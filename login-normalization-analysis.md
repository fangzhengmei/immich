# Immich OAuth 与 LDAP 登录归一化处理分析

## 概述

Immich 当前版本（本代码库）**没有原生 LDAP 支持**，仅实现了 OAuth 2.0 / OpenID Connect (OIDC) 认证协议。LDAP 用户可通过支持 LDAP 的身份提供者（IdP，如 Authelia、Keycloak、Authentik 等）间接接入，由 IdP 完成 LDAP 认证后，通过 OIDC 协议与 Immich 交互。

本文档分析 Immich 对 OAuth/OIDC 登录的归一化处理机制，包括身份映射、账号关联、首次登录和冲突处理四个核心环节。

---

## 一、核心数据模型

用户表 (`user` table) 关键字段设计 `server/src/schema/tables/user.table.ts`：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | UUID | 用户唯一标识（内部生成） |
| `email` | string (unique) | 邮箱，作为用户身份的关键标识 |
| `password` | string | bcrypt 哈希后的密码（密码登录用） |
| `oauthId` | string | OAuth/OIDC 的 `sub` 声明，用于外部身份映射 |
| `name` | string | 用户显示名称 |
| `isAdmin` | boolean | 管理员标记 |
| `storageLabel` | string | 存储标签（可从 OAuth claim 映射） |
| `quotaSizeInBytes` | bigint | 存储配额（可从 OAuth claim 映射） |

**归一化设计**：
- 所有登录方式最终都映射到同一个 `UserTable` 实体
- `oauthId` 为空字符串表示密码登录用户
- `email` 是跨登录方式的统一身份锚点

---

## 二、登录归一化流程总览

```
          +-------------------+
          |   登录请求入口    |
          +---------+---------+
                    |
      +-------------+-------------+
      |                           |
+-----v-----+              +------v------+
| 密码登录   |              |  OAuth登录   |
| /auth/login|              | /oauth/callback|
+-----+-----+              +------+------+
      |                           |
      | 验证邮箱密码                | 验证 OAuth token
      |                           | 获取用户 profile
      +-------------+-------------+
                    |
          +---------v---------+
          |   查找/创建用户    |
          |  userRepository   |
          +---------+---------+
                    |
          +---------v---------+
          |  创建登录会话      |
          |  createLoginResponse |
          +---------+---------+
                    |
          +---------v---------+
          |  返回统一响应      |
          |  LoginResponseDto  |
          +-------------------+
```

**归一化点**：无论哪种登录方式，最终都调用 `createLoginResponse()` 创建会话并返回相同格式的响应。

---

## 三、身份映射 (Identity Mapping)

### 3.1 OAuth 身份映射

代码位置：`server/src/services/auth.service.ts:308-382`

**映射优先级**：

1. **优先通过 OAuth ID 映射**（`profile.sub` → `oauthId` 字段）
   ```typescript
   let user: UserAdmin | undefined = await this.userRepository.getByOAuthId(profile.sub);
   ```

2. **次要通过邮箱映射**（`profile.email` → `email` 字段）
   - 邮箱规范化：`trim().toLowerCase()`
   - 仅当 `oauthId` 未找到用户时触发

3. **Claim 映射（用户创建时）**：
   - `storageLabelClaim` → `storageLabel`（默认 `preferred_username`）
   - `storageQuotaClaim` → `quotaSizeInBytes`（默认 `immich_quota`）
   - `roleClaim` → `isAdmin`（默认 `immich_role`，值为 `admin` 时设为管理员）
   - 姓名映射优先级：`profile.name` > `given_name + family_name` > `preferred_username` > `email`

### 3.2 LDAP 身份映射（间接）

由于 Immich 无原生 LDAP 支持，LDAP 身份映射由上游 IdP 完成：

```
LDAP 目录 ──(LDAP协议)──> IdP (Authelia/Keycloak) ──(OIDC协议)──> Immich
    |                         |                                  |
    |                         |                                  |
    +-- uid/cn --> username --+-- sub/email/claims --> oauthId/email/storageLabel/quota
```

**配置示例**（Authelia + LDAP）：
- LDAP 属性 `uid` → IdP claim `preferred_username` → Immich `storageLabel`
- LDAP 属性 `mail` → IdP claim `email` → Immich `email`
- LDAP 自定义属性 `immichquota` → IdP claim `immich_quota` → Immich `quotaSizeInBytes`

---

## 四、账号关联 (Account Linking)

### 4.1 自动关联（OAuth 登录时）

代码位置：`server/src/services/auth.service.ts:320-330`

**触发条件**：
- 通过 `oauthId` 未找到用户
- OAuth profile 包含 `email`
- 通过 `email` 找到已存在的用户
- 该用户尚未绑定任何 OAuth 账号（`oauthId` 为空）

**关联操作**：
```typescript
user = await this.userRepository.update(emailUser.id, { oauthId: profile.sub });
```

**效果**：原密码登录用户现在可以通过 OAuth 登录，原有数据完全保留。

### 4.2 手动关联

代码位置：`server/src/services/auth.service.ts:407-435`

**端点**：`POST /oauth/link`

**流程**：
1. 用户已通过密码登录（获取 `AuthDto`）
2. 完成 OAuth 授权流程获取 `oauthId`
3. 检查 `oauthId` 是否已被其他用户绑定
4. 关联当前用户与 `oauthId`
5. 更新当前会话的 `oauthSid`

**冲突检测**：
```typescript
const duplicate = await this.userRepository.getByOAuthId(oauthId);
if (duplicate && duplicate.id !== auth.user.id) {
  throw new BadRequestException('This OAuth account has already been linked to another user.');
}
```

### 4.3 取消关联

代码位置：`server/src/services/auth.service.ts:437-444`

**端点**：`POST /oauth/unlink`

**操作**：
- 将用户 `oauthId` 设为空字符串
- 清除当前会话的 `oauthSid`
- 用户恢复为密码登录模式

---

## 五、首次登录 (First-time Login)

### 5.1 OAuth 自动注册

代码位置：`server/src/services/auth.service.ts:332-375`

**前置条件**：
- `oauth.autoRegister` 配置为 `true`（默认 `true`）
- 通过 `oauthId` 和 `email` 均未找到用户
- OAuth profile 包含有效 `email`

**用户创建流程**：
1. 从 OAuth profile 提取各项 claim（姓名、邮箱、角色、存储标签、配额）
2. 调用 `createUser()` 创建用户 `base.service.ts:218-245`
3. 触发 `UserCreate` 事件
4. 同步 OAuth 头像（如果有 `picture` claim 且用户无头像）

**注意**：
- Claim 仅在用户创建时使用，后续不会同步更新
- 第一个注册的用户必须是管理员

### 5.2 首次登录失败分支分析

#### 5.2.1 OAuth 资料缺少 email

**触发条件**：
- OAuth/OIDC 返回的 profile 中不包含 `email` 字段
- 或 `email` 字段为空字符串

**代码位置**：`server/src/services/auth.service.ts:341-343`

```typescript
if (!normalizedEmail) {
  throw new BadRequestException('OAuth profile does not have an email address');
}
```

**处理逻辑**：
1. `normalizedEmail` 来自 `profile.email ? profile.email.trim().toLowerCase() : undefined`
2. 若 `normalizedEmail` 为 `undefined` 或空字符串，直接抛出异常
3. 无论 `autoRegister` 开启与否，均无法完成登录或注册

**返回结果**：
- HTTP 状态码：`400 Bad Request`
- 错误消息：`OAuth profile does not have an email address`
- 用户无法登录，需要在 IdP 端配置 email 字段映射

**注意**：即使 IdP 的 scope 包含 `email`，也可能因为用户未验证邮箱等原因导致 `email` 字段为空。

---

#### 5.2.2 autoRegister 关闭

**触发条件**：
- `oauth.autoRegister` 配置为 `false`
- 通过 `oauthId` 未找到用户
- 通过 `email` 也未找到用户（或 profile 无 email）

**代码位置**：`server/src/services/auth.service.ts:333-339`

```typescript
if (!user) {
  if (!autoRegister) {
    this.logger.warn(
      `Unable to register ${profile.sub}/${normalizedEmail || '(no email)'}. User does not exist and auto registering is disabled. To enable set OAuth Auto Register to true in admin settings.`,
    );
    throw new BadRequestException('OAuth authentication failed');
  }
}
```

**处理逻辑**：
1. 经过 `oauthId` 查找和 `email` 自动关联后，`user` 仍为 `undefined`
2. 检查 `autoRegister` 配置，若为 `false` 则拒绝
3. 记录警告日志，包含 `profile.sub` 和 `email`（用于排查）

**返回结果**：
- HTTP 状态码：`400 Bad Request`
- 错误消息：`OAuth authentication failed`
- 服务端日志包含详细原因：`User does not exist and auto registering is disabled`

**解决方案**：
- 管理员预先在 Immich 中创建用户（匹配邮箱）
- 或开启 `autoRegister` 允许自动注册
- 或用户通过密码登录后手动关联 OAuth 账号

---

#### 5.2.3 首个用户非 admin

**触发条件**：
- Immich 系统中尚无任何用户（`userRepository.getAdmin()` 返回 `null`）
- 首次注册的用户 `isAdmin` 为 `false`

**代码位置**：`server/src/services/base.service.ts:225-230`

```typescript
if (!dto.isAdmin) {
  const localAdmin = await this.userRepository.getAdmin();
  if (!localAdmin) {
    throw new BadRequestException('The first registered account must the administrator.');
  }
}
```

**处理逻辑**：
1. `createUser()` 被调用时检查 `dto.isAdmin`
2. 若非管理员，查询系统中是否已存在管理员
3. 若无管理员，则拒绝创建

**常见场景**：
- 通过 OAuth 自动注册首个用户，但 `roleClaim` 未返回 `admin`
- 普通用户通过 API 注册（绕过管理员）

**返回结果**：
- HTTP 状态码：`400 Bad Request`
- 错误消息：`The first registered account must the administrator.`

**解决方案**：
- 在 IdP 中为首个用户配置 `roleClaim`（如 `immich_role=admin`）
- 或先通过 `/auth/admin-signup` 手动创建管理员
- 或临时修改 OAuth 配置，让首个用户的 `roleClaim` 返回 `admin`

---

#### 5.2.4 其他首次登录失败场景

| 失败场景 | 触发条件 | 错误消息 | 处理方式 |
|----------|----------|----------|----------|
| OAuth 未启用 | `oauth.enabled = false` | `OAuth is not enabled` | 在管理后台启用 OAuth |
| State 参数缺失 | Cookie 中无 OAuth state | `OAuth state is missing` | 检查 OAuth 发起流程 |
| PKCE verifier 缺失 | Cookie 中无 code_verifier | `OAuth code verifier is missing` | 检查 OAuth 发起流程 |
| Token 验证失败 | OAuth token 无效或过期 | `OAuth login failed` | 重新发起 OAuth 登录 |
| 邮箱已被占用 | 创建用户时邮箱已存在 | `Email is not available` | 使用现有账号登录或联系管理员 |

### 5.3 禁用自动注册

当 `autoRegister` 为 `false` 时：
- 未注册的 OAuth 用户登录失败
- 错误信息：`User does not exist and auto registering is disabled`
- 管理员需预先创建用户（通过 API 或管理界面），用户登录时通过邮箱自动关联

### 5.4 密码登录首次注册

仅管理员可通过 `/auth/admin-signup` 注册首个管理员账户，普通用户需由管理员创建。

---

## 六、冲突处理 (Conflict Handling)

### 6.1 OAuth 账号冲突

**场景 1**：邮箱已被其他 OAuth 账号绑定
```typescript
// auth.service.ts:324-327
if (emailUser.oauthId) {
  this.logger.debug('OAuth login conflict: email already linked to different account');
  throw new BadRequestException('OAuth authentication failed');
}
```
- 处理：直接拒绝登录
- 原因：防止账号劫持，确保邮箱与 OAuth 账号一一对应

**场景 2**：OAuth ID 已被其他用户绑定（手动关联时）
```typescript
// auth.service.ts:424-427
const duplicate = await this.userRepository.getByOAuthId(oauthId);
if (duplicate && duplicate.id !== auth.user.id) {
  throw new BadRequestException('This OAuth account has already been linked to another user.');
}
```
- 处理：拒绝关联
- 原因：一个 OAuth 账号只能绑定一个 Immich 用户

### 6.2 邮箱唯一性冲突

**场景**：创建用户时邮箱已存在
```typescript
// base.service.ts:219-223
const exists = await this.userRepository.getByEmail(dto.email);
if (exists) {
  throw new BadRequestException('Email is not available');
}
```
- 处理：拒绝创建
- 原因：`email` 字段有唯一索引约束

### 6.3 存储标签冲突

**场景**：`storageLabel` 已被其他用户使用
- 数据库层面：`storageLabel` 字段有唯一索引
- 应用层面：创建用户时会自动清理（`sanitize` 并移除 `.`），但不主动检查冲突
- 处理：数据库抛出唯一约束违反异常

### 6.4 管理员权限冲突

**场景**：非管理员用户尝试创建（首个用户除外）
```typescript
// base.service.ts:225-230
if (!dto.isAdmin) {
  const localAdmin = await this.userRepository.getAdmin();
  if (!localAdmin) {
    throw new BadRequestException('The first registered account must the administrator.');
  }
}
```
- 处理：拒绝创建
- 原因：确保系统至少有一个管理员

---

## 七、安全设计考量

### 7.1 密码登录安全
- 使用 bcrypt 哈希（`SALT_ROUNDS` 轮）
- 固定时间响应，防止时序攻击枚举用户
```typescript
// auth.service.ts:65-73
const user = await this.userRepository.getByEmail(dto.email, { withPassword: true });
const authenticated = this.cryptoRepository.compareBcrypt(dto.password, user?.password ?? LOGIN_DUMMY_HASH);
```
- 无论邮箱是否存在，都执行一次 bcrypt 比较

### 7.2 OAuth 安全
- 支持 PKCE（Proof Key for Code Exchange）
- State 参数验证，防止 CSRF
- 支持 Backchannel Logout（RFC 8963）
- Token 签名算法可配置（RS256、HS256 等）

### 7.3 会话管理
- 会话 token 使用 SHA-256 哈希存储
- 支持 PIN 码二次验证
- 支持设备指纹跟踪（deviceType、deviceOS、appVersion）

---

## 八、代码关键位置汇总

| 功能模块 | 文件位置 | 关键行号 |
|----------|----------|----------|
| 登录主流程 | `server/src/services/auth.service.ts` | 59-76 (密码登录), 292-382 (OAuth回调) |
| 用户创建 | `server/src/services/base.service.ts` | 218-245 |
| OAuth 授权 | `server/src/services/auth.service.ts` | 277-290 |
| 账号关联/取消 | `server/src/services/auth.service.ts` | 407-444 |
| 会话创建 | `server/src/services/auth.service.ts` | 602-616 |
| OAuth 配置文档 | `docs/docs/administration/oauth.md` | 全文 |
| 用户表结构 | `server/src/schema/tables/user.table.ts` | 全文 |
| 用户仓库 | `server/src/repositories/user.repository.ts` | 144-152 (getByOAuthId) |

---

## 九、总结

Immich 的登录归一化设计遵循以下原则：

1. **统一数据模型**：所有登录方式共享 `UserTable`，通过 `oauthId` 字段区分
2. **邮箱作为锚点**：`email` 是跨登录方式的身份关联桥梁
3. **明确的优先级**：OAuth ID > 邮箱 > 自动注册
4. **安全优先**：防止账号劫持，确保一一对应关系
5. **灵活配置**：支持自动注册、手动关联、Claim 映射等多种策略

对于 LDAP 场景，建议通过支持 LDAP 的 IdP（如 Authelia、Keycloak）进行协议转换，利用 Immich 成熟的 OIDC 实现间接支持 LDAP 认证。
