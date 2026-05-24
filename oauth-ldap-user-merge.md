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

| 字段 | 类型 | 作用 |
|------|------|------|
| `id` | UUID | 本地用户唯一标识 |
| `email` | string (unique) | 用户邮箱，用于账号匹配 |
| `oauthId` | string | 外部身份标识，存储 OAuth `sub` Claim |
| `password` | string | 本地密码哈希，OAuth 用户可为空 |
| `storageLabel` | string | 存储目录标签 |
| `isAdmin` | boolean | 管理员标识 |

关键查询方法 (`user.repository.ts`)：
- `getByOAuthId(oauthId)` - 按外部身份 ID 查询
- `getByEmail(email)` - 按邮箱查询（邮箱标准化后匹配）

---

## 入口一：OAuth 回调自动登录 (`auth.service.ts:292-382`)

### 核心方法：`callback()`

这是外部用户首次或后续登录的主要入口，完整的账号匹配与合并流程如下：

```typescript
async callback(dto: OAuthCallbackDto, headers: IncomingHttpHeaders, loginDetails: LoginDetails)
```

### 匹配优先级与合并策略

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
    if (emailUser.oauthId) {
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

**合并逻辑分支：**

| 场景 | 行为 |
|------|------|
| 邮箱匹配成功且用户无 `oauthId` | **合并账号**：将 `oauthId` 写入该用户记录，两个身份合二为一 |
| 邮箱匹配成功但已有不同 `oauthId` | **拒绝登录**：抛出冲突异常，防止账号劫持 |
| 邮箱无匹配 | 进入下一步 |

> **关键安全设计**：只有当本地账号尚未绑定任何 OAuth ID 时才允许自动合并。如果邮箱已被其他 OAuth 账号占用，直接拒绝，避免账号越权。

**步骤 3：自动注册新用户** (`auth.service.ts:332-375`)

```typescript
if (!user) {
  if (!autoRegister) {
    this.logger.warn(`Unable to register ${profile.sub}/${normalizedEmail || '(no email)'}. User does not exist and auto registering is disabled.`);
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

**步骤 4：头像同步** (`auth.service.ts:377-379`)

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

**步骤 1：获取 OAuth Profile** (`auth.service.ts:418-422`)
```typescript
const { profile: { sub: oauthId }, sid } = 
  await this.oauthRepository.getProfileAndOAuthSid(oauth, dto.url, expectedState, codeVerifier);
```

**步骤 2：冲突检测** (`auth.service.ts:423-427`)
```typescript
const duplicate = await this.userRepository.getByOAuthId(oauthId);
if (duplicate && duplicate.id !== auth.user.id) {
  this.logger.warn(`OAuth link account failed: sub is already linked to another user (${duplicate.email}).`);
  throw new BadRequestException('This OAuth account has already been linked to another user.');
}
```

**步骤 3：执行绑定** (`auth.service.ts:429-434`)
```typescript
if (auth.session) {
  await this.sessionRepository.update(auth.session.id, { oauthSid: sid });
}
const user = await this.userRepository.update(auth.user.id, { oauthId });
```

**关键区别**：与自动合并不同，手动链接时**不进行邮箱匹配**，直接绑定到当前登录用户。但会检查该 OAuth ID 是否已被其他用户占用。

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

- 将 `oauthId` 设为空字符串（注意：不是 `null`，代码中有 TODO 注释标记要改为 null）
- 清除当前会话的 `oauthSid`

### 管理员批量解绑 (`auth-admin.service.ts:7-10`)

```typescript
async unlinkAll(_auth: AuthDto) {
  await this.userRepository.updateAll({ oauthId: '' });
}
```

- 重置所有用户的 `oauthId`，用于 OAuth 配置变更或迁移场景
- **注意**：此操作不区分已删除用户，所有用户都会被重置

---

## 两条入口对比总结

| 对比项 | 入口一：OAuth 回调自动登录 | 入口二：手动链接 OAuth |
|--------|--------------------------|----------------------|
| 触发时机 | OAuth 登录回调 | 已登录用户在设置页操作 |
| 匹配键优先级 | 1. `oauthId` → 2. `email` | 直接绑定当前用户 |
| 邮箱自动合并 | 支持（无冲突时） | 不检查邮箱，直接绑定 |
| 冲突处理 | 邮箱已绑定其他 OAuth ID 时拒绝 | `oauthId` 已被其他用户使用时拒绝 |
| 自动注册 | 支持（`autoRegister=true`） | 不涉及 |
| 适用场景 | 普通用户登录、首次登录合并已有账号 | 用户主动关联外部身份 |
| 代码位置 | `auth.service.ts:292-382` | `auth.service.ts:407-435` |

---

## 安全设计要点

1. **邮箱标准化**：所有邮箱比较前都经过 `trim().toLowerCase()` 处理，避免大小写和空格导致的匹配失败或重复账号

2. **冲突防止**：
   - 自动合并时检查目标账号是否已有 `oauthId`
   - 手动绑定时检查 `oauthId` 是否已被其他用户占用
   - 冲突时直接抛出通用错误信息，不暴露具体原因（防止用户枚举）

3. **安全日志** (`auth.service.ts:325-326`)：
   ```typescript
   this.logger.debug('OAuth login conflict: email already linked to different account');
   throw new BadRequestException('OAuth authentication failed');
   ```
   - 详细错误仅记录日志，用户只看到通用错误信息

4. **密码登录防护**：OAuth 用户的 `password` 字段默认为空字符串，密码登录时会验证失败

---

## 测试覆盖验证

核心逻辑测试用例位于 `auth.service.spec.ts`：

| 测试场景 | 位置 |
|----------|------|
| 邮箱标准化后匹配 | `auth.service.spec.ts:725-743` |
| 邮箱已绑定其他 OAuth ID 时拒绝 | `auth.service.spec.ts:745-763` |
| 自动注册新用户 | `auth.service.spec.ts:765-781` |
| 手动链接冲突检测 | `auth.service.spec.ts` (link 相关测试) |
| 批量解绑所有用户 | `auth-admin.service.spec.ts:24-65` |
| 自动注册但无 Email Claim 时拒绝 | `auth.service.spec.ts:783-801` |

---

## 常见问题与边界情况

**Q: LDAP 用户如何同步到 Immich？**
A: LDAP 用户属性（如配额 `immich_quota`、角色 `immich_role`）需由 OAuth 提供者映射为 OIDC Claim，Immich 在用户首次创建时读取这些 Claim。**注意**：Claim 仅在用户创建时使用，后续不同步。

**Q: 修改 OAuth 提供者的 `sub` 会怎样？**
A: 会导致用户无法匹配，系统会视为新用户。如果 `autoRegister=true` 会创建重复账号；如果 `autoRegister=false` 会拒绝登录。管理员需执行 `unlinkAll` 后让用户重新登录合并。

**Q: 能否同时绑定多个 OAuth 账号？**
A: 不能。`oauthId` 字段是单一字符串，每个用户只能绑定一个外部身份。

**Q: 邮箱变更后会自动重新匹配吗？**
A: 不会。匹配仅在登录回调时执行一次，且仅在 `oauthId` 未匹配时才尝试邮箱匹配。如果用户已绑定 `oauthId`，邮箱变更不会触发重新匹配。
