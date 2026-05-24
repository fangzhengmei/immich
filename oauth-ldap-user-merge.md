# OAuth 与 LDAP 账号匹配与合并策略分析

## 前置说明：LDAP 的实现方式

当前版本的 Immich **没有原生 LDAP 支持**。LDAP 用户认证是通过 OAuth/OIDC 身份提供者（Identity Provider）间接实现的，典型架构如下：

```
LDAP 目录服务 ←→ OAuth 提供者 (Authelia/Authentik/Keycloak) ←→ Immich (OAuth 客户端)
```

LDAP 用户信息由 OAuth 提供者通过 OIDC Claim 传递给 Immich，Immich 只实现 OAuth 协议。文档中提到的 "LDAP 入口" 本质上仍是 OAuth 协议入口，区别仅在于 OAuth 提供者后端对接的是 LDAP 目录。

---

## 核心数据模型

用户表 (`user.table.ts:27-85`) 关键字段：

| 字段 | 类型 | 约束 | 作用 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | 本地用户唯一标识 |
| `email` | string | UNIQUE | 用户邮箱，用于账号匹配 |
| `oauthId` | string | **无 UNIQUE 约束**，NOT NULL DEFAULT '' | 外部身份标识，存储 OAuth `sub` Claim |
| `password` | string | NOT NULL DEFAULT '' | 本地密码哈希，OAuth 用户可为空字符串 |
| `storageLabel` | string | UNIQUE, NULLABLE | 存储目录标签 |
| `isAdmin` | boolean | DEFAULT false | 管理员标识 |

会话表 (`session.table.ts:14-58`) 关键字段：

| 字段 | 类型 | 约束 | 作用 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | 会话唯一标识 |
| `token` | bytea | INDEX | 会话令牌哈希 |
| `userId` | UUID | FOREIGN KEY | 关联用户 |
| `oauthSid` | string | NULLABLE, INDEX | OAuth 会话 ID，用于 backchannel logout |

关键查询方法 (`user.repository.ts`)：
- `getByOAuthId(oauthId)` - 按外部身份 ID 查询
- `getByEmail(email)` - 按邮箱查询（邮箱标准化后匹配）

关键会话操作 (`session.repository.ts:114-133`)：
- `invalidateOAuth({ oauthSid, oauthId })` - 按 OAuth 标识批量失效会话

---

## oauthId 在数据层的约束边界

**重要发现**：`oauthId` 在数据库层面**没有 UNIQUE 约束** (`1744910873969-InitialMigration.ts:111,405`)。

```sql
-- 用户表建表语句，oauthId 无唯一约束
CREATE TABLE "users" (
  "id" uuid NOT NULL DEFAULT uuid_generate_v4(),
  "email" character varying NOT NULL,
  "oauthId" character varying NOT NULL DEFAULT '',  -- 仅 NOT NULL + DEFAULT ''
  ...
);
-- 只有 email 和 storageLabel 有 UNIQUE 约束
ALTER TABLE "users" ADD CONSTRAINT "UQ_97672ac88f789774dd47f7c8be3" UNIQUE ("email");
ALTER TABLE "users" ADD CONSTRAINT "UQ_b309cf34fa58137c416b32cea3a" UNIQUE ("storageLabel");
```

### 约束边界分析：

1. **允许多个用户同时为空**：`oauthId` 默认值是空字符串 `''`，所有未绑定 OAuth 的用户都拥有相同的空字符串值，这在数据库层面是允许的。

2. **无唯一性保证**：理论上可以将同一个 `oauthId` 绑定到多个用户（但业务层有冲突检测防止这种情况）。

3. **空字符串 vs null**：代码中有 TODO 注释 (`user.table.ts:67`) 标记要将 `oauthId` 改为 nullable，但当前实现使用空字符串表示"未绑定"状态。这导致：
   - `unlink()` 操作是将 `oauthId` 设为 `''` 而非 `null`
   - 邮箱匹配时检查 `if (emailUser.oauthId)`，空字符串会被判定为 falsy

4. **已删除用户不清理**：软删除（`deletedAt` 非空）的用户其 `oauthId` 仍保留在数据库中，`getByOAuthId()` 方法可能返回已删除用户。

5. **业务层唯一性保障**：唯一性完全由应用层代码保证：
   - 自动合并时检查目标账号是否已有 `oauthId`
   - 手动绑定时检查 `oauthId` 是否已被其他用户占用
   - 但这些检查存在 TOCTOU（时间差）竞态风险

---

## 登录入口的安全防护：state + codeVerifier + 移动端重定向

### state 与 codeVerifier 完整生命周期

**生成阶段** (`auth.service.ts:277-290` → `oauth.repository.ts:44-74`)：

```typescript
// authorize 端点
async authorize(dto: OAuthConfigDto) {
  return await this.oauthRepository.authorize(
    oauth,
    this.resolveRedirectUri(oauth, dto.redirectUri),
    dto.state,      // 移动端可传入自定义 state
    dto.codeChallenge, // 移动端可传入自定义 codeChallenge
  );
}

// oauthRepository.authorize 内部
state ??= randomState();           // 未传入则随机生成
codeVerifier = randomPKCECodeVerifier();  // 未传入 codeChallenge 则生成
codeChallenge = await calculatePKCECodeChallenge(codeVerifier);
```

**存储阶段** (`oauth.controller.ts:45-62`)：

```typescript
// 写入浏览器 Cookie，SameSite 策略保护
return respondWithCookie(res, { url }, {
  isSecure: loginDetails.isSecure,
  values: [
    { key: ImmichCookie.OAuthState, value: state },
    { key: ImmichCookie.OAuthCodeVerifier, value: codeVerifier },
  ],
});
```

**验证阶段** (`auth.service.ts:298-306`)：

```typescript
// callback 端点
const expectedState = dto.state ?? this.getCookieOauthState(headers);
if (!expectedState?.length) {
  throw new BadRequestException('OAuth state is missing');
}

const codeVerifier = dto.codeVerifier ?? this.getCookieCodeVerifier(headers);
if (!codeVerifier?.length) {
  throw new BadRequestException('OAuth code verifier is missing');
}
```

**双来源设计**：`state` 和 `codeVerifier` 优先从请求体 DTO 取，其次从 Cookie 取。这种设计同时支持：
- **Web 端**：使用 Cookie 存储，防止 CSRF
- **移动端**：客户端自行管理 state 和 codeVerifier，通过请求体传递

### 移动端重定向替换机制

**重定向 URI 解析** (`auth.service.ts:623-631`)：

```typescript
private resolveRedirectUri(
  { mobileRedirectUri, mobileOverrideEnabled }: { mobileRedirectUri: string; mobileOverrideEnabled: boolean },
  url: string,
) {
  if (mobileOverrideEnabled && mobileRedirectUri) {
    return url.replace(/app\.immich:\/+oauth-callback/, mobileRedirectUri);
  }
  return url;
}
```

**替换规则**：
- 当 `mobileOverrideEnabled=true` 且配置了 `mobileRedirectUri` 时
- 使用正则 `/app\.immich:\/+oauth-callback/` 匹配默认的移动端 Deep Link
- 替换为管理员配置的自定义重定向 URI

**独立移动端重定向端点** (`oauth.controller.ts:24-37`)：

```typescript
@Get('mobile-redirect')
@Redirect()
redirectOAuthToMobile(@Req() request: Request) {
  return {
    url: this.service.getMobileRedirect(request.url),
    statusCode: HttpStatus.TEMPORARY_REDIRECT,
  };
}

// auth.service.ts:273-275
getMobileRedirect(url: string) {
  return `${MOBILE_REDIRECT}?${url.split('?')[1] || ''}`;
}
```

该端点用于 OAuth 提供者不能正确处理 Deep Link 的场景，通过服务端 307 重定向将 OAuth 授权码传递给移动应用。

---

## 入口一：OAuth 回调自动登录 (`auth.service.ts:292-382`)

### 核心方法：`callback()`

这是外部用户首次或后续登录的主要入口，完整的账号匹配与合并流程如下：

```typescript
async callback(dto: OAuthCallbackDto, headers: IncomingHttpHeaders, loginDetails: LoginDetails)
```

### 匹配优先级与合并策略

**步骤 0：获取 OAuth Profile 与 oauthSid** (`auth.service.ts:308-314`)

```typescript
const url = this.resolveRedirectUri(oauth, dto.url);
const { profile, sid: oauthSid } = await this.oauthRepository.getProfileAndOAuthSid(
  oauth, url, expectedState, codeVerifier,
);
```

- `profile.sub` - 外部用户唯一标识（后续匹配核心）
- `profile.email` - 用于邮箱匹配
- `sid` - OAuth 会话 ID，存储到 session 表用于后续 backchannel logout

**步骤 1：按 OAuth ID 精确匹配** (`auth.service.ts:318`)

```typescript
let user: UserAdmin | undefined = await this.userRepository.getByOAuthId(profile.sub);
```

- 使用 OAuth `sub` Claim 作为唯一键查询
- 这是最高优先级的匹配方式
- 匹配成功直接进入登录流程，跳过后续匹配

**步骤 2：按邮箱匹配并自动关联** (`auth.service.ts:321-330`)

```typescript
if (!user && normalizedEmail) {
  const emailUser = await this.userRepository.getByEmail(normalizedEmail);
  if (emailUser) {
    if (emailUser.oauthId) {  // 空字符串 '' 会被判定为 falsy
      this.logger.debug('OAuth login conflict: email already linked to different account');
      throw new BadRequestException('OAuth authentication failed');
    }
    user = await this.userRepository.update(emailUser.id, { oauthId: profile.sub });
  }
}
```

**邮箱标准化处理** (`auth.service.ts:315`)：
```typescript
const normalizedEmail = profile.email ? profile.email.trim().toLowerCase() : undefined;
```

**合并逻辑分支（修正"必然新建重复账号"的误判）**：

| 场景 | 行为 | 说明 |
|------|------|------|
| 邮箱匹配 + 用户 `oauthId=''` (空字符串) | **合并账号** | 将新 `oauthId` 写入该用户，两个身份合二为一 |
| 邮箱匹配 + 用户已有 `oauthId` (非空) | **拒绝登录** | 抛出冲突异常，防止账号劫持 |
| 邮箱无匹配 | 进入自动注册 | 可能创建新账号 |

> **sub 变更后的实际行为**：如果 OAuth 提供者的 `sub` 发生变化（例如重新安装 OAuth 提供者、重建用户），系统行为分三种情况：
> 1. **已执行 `unlink` 或 `unlinkAll`**：邮箱匹配成功且 `oauthId=''` → 自动合并，绑定新 sub
> 2. **未解绑但邮箱匹配**：`oauthId` 仍为旧值（非空）→ 拒绝登录（冲突）
> 3. **邮箱也变化**：无匹配 → 若 `autoRegister=true` 则创建重复账号

**步骤 3：自动注册新用户** (`auth.service.ts:332-375`)

```typescript
if (!user) {
  if (!autoRegister) {
    this.logger.warn(
      `Unable to register ${profile.sub}/${normalizedEmail || '(no email)'}. User does not exist and auto registering is disabled. To enable set OAuth Auto Register to true in admin settings.`,
    );
    throw new BadRequestException('OAuth authentication failed');
  }
  // 创建新用户...
}
```

**注册条件分支：**

| `autoRegister` 配置 | 行为 |
|---------------------|------|
| `true` (默认) | 使用 OAuth Profile 信息创建新本地用户，自动绑定 `oauthId` |
| `false` | 拒绝登录，要求管理员先手动创建账号 |

**新用户属性映射** (`auth.service.ts:347-374`)：
- `name`: 优先 `name` Claim，其次 `given_name + family_name`，再次 `preferred_username`，最后用邮箱
- `email`: 标准化后的邮箱
- `oauthId`: OAuth `sub` Claim
- `quotaSizeInBytes`: 从 `storageQuotaClaim` 读取或使用 `defaultStorageQuota`
- `storageLabel`: 从 `storageLabelClaim` 读取
- `isAdmin`: 从 `roleClaim` 读取（值为 `admin` 时为管理员）

**步骤 4：创建会话并存储 oauthSid** (`auth.service.ts:381` → `602-616`)

```typescript
return this.createLoginResponse(user, loginDetails, oauthSid);

private async createLoginResponse(user: UserAdmin, loginDetails: LoginDetails, oauthSid?: string) {
  await this.sessionRepository.create({
    token: hashed,
    userId: user.id,
    oauthSid: oauthSid ?? null,  // 存储 OAuth 会话 ID
    // ... 其他字段
  });
}
```

- `oauthSid` 从 ID Token 的 `sid` Claim 提取
- 存储到 session 表，用于后续 backchannel logout 精确匹配会话

**步骤 5：头像同步** (`auth.service.ts:377-379`)

```typescript
if (!user.profileImagePath && profile.picture) {
  await this.syncProfilePicture(user, profile.picture);
}
```

- 仅在用户没有头像时同步
- 下载 `picture` Claim 指向的 URL 并生成本地图标

---

## 入口二：已登录用户手动链接 OAuth (`auth.service.ts:407-435`)

### 核心方法：`link()`

用于已通过密码登录的用户，手动绑定 OAuth 账号。

```typescript
async link(auth: AuthDto, dto: OAuthCallbackDto, headers: IncomingHttpHeaders): Promise<UserAdminResponseDto>
```

### 合并策略

**步骤 1：验证 state 和 codeVerifier** (`auth.service.ts:408-416`)
- 与 callback 相同的验证逻辑
- 防止 CSRF 和授权码注入

**步骤 2：获取 OAuth Profile** (`auth.service.ts:418-422`)
```typescript
const { profile: { sub: oauthId }, sid } = 
  await this.oauthRepository.getProfileAndOAuthSid(oauth, dto.url, expectedState, codeVerifier);
```

**步骤 3：冲突检测** (`auth.service.ts:423-427`)
```typescript
const duplicate = await this.userRepository.getByOAuthId(oauthId);
if (duplicate && duplicate.id !== auth.user.id) {
  this.logger.warn(`OAuth link account failed: sub is already linked to another user (${duplicate.email}).`);
  throw new BadRequestException('This OAuth account has already been linked to another user.');
}
```

**步骤 4：执行绑定** (`auth.service.ts:429-434`)
```typescript
if (auth.session) {
  await this.sessionRepository.update(auth.session.id, { oauthSid: sid });
}
const user = await this.userRepository.update(auth.user.id, { oauthId });
```

**关键区别**：与自动合并不同，手动链接时**不进行邮箱匹配**，直接绑定到当前登录用户。但会检查该 OAuth ID 是否已被其他用户占用。

---

## 会话失效路径：oauthSid 与 Backchannel Logout

### 完整失效链路

**接收注销请求** (`oauth.controller.ts:117-128`)：
```typescript
@Post('backchannel-logout')
async logoutOAuth(@Body() dto: OAuthBackchannelLogoutDto): Promise<void> {
  return this.service.backchannelLogout(dto);
}
```

**验证 Logout Token** (`auth.service.ts:90-112` → `oauth.repository.ts:145-195`)：
```typescript
async backchannelLogout(dto: OAuthBackchannelLogoutDto): Promise<void> {
  const claims = await this.oauthRepository.validateLogoutToken(oauth, dto.logout_token);
  
  // RFC 8963 验证：必须包含 sub 或 sid
  if (!claims.sub && !claims.sid) {
    throw new BadRequestException('Invalid logout token: it must contain either a sub or a sid claim');
  }
  // ...
}
```

Token 验证遵循 OIDC Back-Channel Logout 规范 (RFC 8963)：
- 验证 JWT 签名、issuer、audience、algorithm
- 验证 `events` Claim 包含 `http://schemas.openid.net/event/backchannel-logout`
- 验证不包含 `nonce`
- Token 有效期 2 分钟，时钟容差 5 秒

**失效会话** (`session.repository.ts:114-133`)：
```typescript
async invalidateOAuth({ oauthSid, oauthId }: { oauthSid?: string; oauthId?: string }): Promise<string[]> {
  let query = this.db.deleteFrom('session').returning('session.id');

  if (oauthSid && oauthId) {
    // 精确匹配：指定会话 + 指定用户
    query = query
      .using('user')
      .whereRef('user.id', '=', 'session.userId')
      .where('session.oauthSid', '=', oauthSid)
      .where('user.oauthId', '=', oauthId);
  } else if (!oauthSid && oauthId) {
    // 批量失效：该用户的所有 OAuth 会话
    query = query.using('user').whereRef('user.id', '=', 'session.userId').where('user.oauthId', '=', oauthId);
  } else if (oauthSid && !oauthId) {
    // 精确匹配：指定会话 ID（可能跨用户）
    query = query.where('session.oauthSid', '=', oauthSid);
  }
  // ...
}
```

**多维度匹配策略**：

| 传入参数 | 匹配范围 | 适用场景 |
|---------|---------|---------|
| `oauthSid` + `oauthId` | 指定用户的指定会话 | 单点登出（最精确） |
| 仅 `oauthId` | 指定用户的所有会话 | 用户全局登出 |
| 仅 `oauthSid` | 所有用户中匹配该 sid 的会话 | 会话级登出 |

**发布事件** (`auth.service.ts:119-121`)：
```typescript
for (const sessionId of deletedSessionIds) {
  await this.eventRepository.emit('SessionDelete', { sessionId });
}
```

---

## 解除关联与管理操作

### 个人解除绑定 (`auth.service.ts:437-444`)

```typescript
async unlink(auth: AuthDto): Promise<UserAdminResponseDto> {
  if (auth.session) {
    await this.sessionRepository.update(auth.session.id, { oauthSid: null });
  }
  const user = await this.userRepository.update(auth.user.id, { oauthId: '' });
  return mapUserAdmin(user);
}
```

- 将 `oauthId` 设为空字符串 `''`（不是 `null`，代码中有 TODO 标记要改为 null）
- 清除当前会话的 `oauthSid`
- 解绑后该邮箱账号可以被新的 OAuth sub 匹配合并

### 管理员批量解绑 (`auth-admin.service.ts:7-10`)

```typescript
async unlinkAll(_auth: AuthDto) {
  await this.userRepository.updateAll({ oauthId: '' });
}
```

- 重置所有用户的 `oauthId` 为空字符串
- 用于 OAuth 配置变更、sub 迁移或提供者更换场景
- **注意**：此操作不区分已删除用户，所有用户都会被重置
- 重置后用户首次登录时会通过邮箱匹配自动合并

---

## 两条入口对比总结

| 对比项 | 入口一：OAuth 回调自动登录 | 入口二：手动链接 OAuth |
|--------|--------------------------|----------------------|
| 触发时机 | OAuth 登录回调 | 已登录用户在设置页操作 |
| 前置验证 | state + codeVerifier | state + codeVerifier |
| 匹配键优先级 | 1. `oauthId` → 2. `email` | 直接绑定当前用户 |
| 邮箱自动合并 | 支持（`oauthId` 为空时） | 不检查邮箱，直接绑定 |
| 冲突处理 | 邮箱已绑定其他 OAuth ID 时拒绝 | `oauthId` 已被其他用户使用时拒绝 |
| 自动注册 | 支持（`autoRegister=true`） | 不涉及 |
| 会话存储 | 自动存储 `oauthSid` | 更新当前会话的 `oauthSid` |
| 适用场景 | 普通用户登录、首次登录合并已有账号 | 用户主动关联外部身份 |
| 代码位置 | `auth.service.ts:292-382` | `auth.service.ts:407-435` |

---

## 安全设计要点

1. **邮箱标准化**：所有邮箱比较前都经过 `trim().toLowerCase()` 处理，避免大小写和空格导致的匹配失败或重复账号

2. **冲突防止**：
   - 自动合并时检查目标账号是否已有 `oauthId`（非空字符串）
   - 手动绑定时检查 `oauthId` 是否已被其他用户占用
   - 冲突时直接抛出通用错误信息，不暴露具体原因（防止用户枚举）

3. **安全日志** (`auth.service.ts:325-326`)：
   ```typescript
   this.logger.debug('OAuth login conflict: email already linked to different account');
   throw new BadRequestException('OAuth authentication failed');
   ```
   - 详细错误仅记录日志，用户只看到通用错误信息

4. **密码登录防护**：OAuth 用户的 `password` 字段默认为空字符串，密码登录时会验证失败

5. **Backchannel Logout 规范遵循**：严格实现 RFC 8963，支持按 sid 和/或 sub 精确失效会话

6. **移动端安全设计**：
   - 支持客户端传入自定义 state 和 codeChallenge（PKCE）
   - 支持重定向 URI 替换，适配不同移动平台的 Deep Link 限制
   - 独立的 mobile-redirect 端点处理特殊重定向场景

---

## 测试覆盖验证

核心逻辑测试用例位于 `auth.service.spec.ts`：

| 测试场景 | 位置 |
|----------|------|
| 邮箱标准化后匹配 | `auth.service.spec.ts:725-743` |
| 邮箱已绑定其他 OAuth ID 时拒绝 | `auth.service.spec.ts:745-763` |
| 自动注册新用户 | `auth.service.spec.ts:765-781` |
| 自动注册但无 Email Claim 时拒绝 | `auth.service.spec.ts:783-801` |
| 手动链接冲突检测 | `auth.service.spec.ts` (link 相关测试) |
| 批量解绑所有用户 | `auth-admin.service.spec.ts:24-65` |
| Backchannel Logout Token 验证 | `auth.service.spec.ts:231-248` |
| 移动端重定向替换 | `auth.service.spec.ts:808` |
| state/codeVerifier 缺失时拒绝 | `auth.service.spec.ts` (callback 前置检查) |

---

## 常见问题与边界情况

**Q: LDAP 用户如何同步到 Immich？**
A: LDAP 用户属性（如配额 `immich_quota`、角色 `immich_role`）需由 OAuth 提供者映射为 OIDC Claim，Immich 在用户首次创建时读取这些 Claim。**注意**：Claim 仅在用户创建时使用，后续不同步。

**Q: 修改 OAuth 提供者的 `sub` 会怎样？（修正版）**
A: 行为分三种情况，不是必然创建重复账号：
1. **已解绑（`oauthId=''`）**：邮箱匹配成功 → 自动合并，绑定新 sub，不会重复
2. **未解绑但邮箱匹配**：原账号仍有旧 `oauthId` → 拒绝登录（冲突），需管理员执行 `unlinkAll`
3. **邮箱也变化**：无匹配 → 若 `autoRegister=true` 才会创建重复账号

**Q: 能否同时绑定多个 OAuth 账号？**
A: 不能。`oauthId` 字段是单一字符串，每个用户只能绑定一个外部身份。虽然数据库没有 UNIQUE 约束，但业务层会检查冲突。

**Q: 邮箱变更后会自动重新匹配吗？**
A: 不会。匹配仅在登录回调时执行一次，且仅在 `oauthId` 未匹配时才尝试邮箱匹配。如果用户已绑定 `oauthId`，邮箱变更不会触发重新匹配。

**Q: Backchannel Logout 能精确失效单个会话吗？**
A: 可以。如果 Logout Token 同时包含 `sid` 和 `sub`，会精确匹配并失效指定用户的指定会话，不影响其他登录会话。

**Q: 为什么 `oauthId` 没有数据库 UNIQUE 约束？**
A: 历史设计选择。这允许多个用户同时处于未绑定状态（`oauthId=''`），但也带来了竞态风险。应用层通过前置检查尽量避免冲突，但理论上存在 TOCTOU 漏洞。代码中已有 TODO 注释计划将 `oauthId` 改为 nullable，未来可能添加条件唯一索引。
