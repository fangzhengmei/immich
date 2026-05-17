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

### 3.3 LDAP 经 IdP 接入的 Claim 映射与冲突对照表

下表完整描述了 LDAP 属性如何通过 IdP 映射到 Immich 字段，以及各字段冲突时的处理路径和后果：

| LDAP 属性 | IdP Claim | Immich 字段 | 映射时机 | 冲突场景 | 处理路径 | 处理结果 |
|-----------|-----------|-------------|----------|----------|----------|----------|
| `entryUUID` / `uidNumber` | `sub` (必填) | `oauthId` | 每次登录 | sub 未找到用户，邮箱也未匹配 | 自动注册路径（autoRegister=true） | 创建新用户 |
| | | | | sub 未找到用户，邮箱匹配到无 oauthId 的用户 | 自动关联路径 `auth.service.ts:320-330` | 更新该用户的 oauthId，登录成功 |
| | | | | sub 未找到用户，邮箱匹配到有其他 oauthId 的用户 | 冲突拒绝路径 `auth.service.ts:324-327` | 抛出 `OAuth authentication failed` |
| | | | | **IdP 端 sub 发生变化**（如 LDAP 条目重建） | 视为新用户，按首次登录处理 | 若 autoRegister=false 则登录失败；若 autoRegister=true 且邮箱已被占用则邮箱冲突 |
| `mail` / `email` | `email` (必填) | `email` | 每次登录 | OAuth profile 无 email | 直接拒绝 `auth.service.ts:341-343` | 抛出 `OAuth profile does not have an email address` |
| | | | | 自动注册时邮箱已存在 | 用户创建前检查 `base.service.ts:219-223` | 抛出 `Email is not available` |
| | | | | **邮箱已绑定他人 OAuth 账号** | 冲突拒绝路径 `auth.service.ts:324-327` | 抛出 `OAuth authentication failed` |
| `uid` / `sAMAccountName` | `preferred_username` | `storageLabel` | 首次注册 | storageLabel 已被其他用户使用 | 数据库唯一约束 | 抛出 `duplicate key value violates unique constraint` |
| `cn` / `displayName` | `name` / `given_name` + `family_name` | `name` | 首次注册 | （无唯一约束） | 直接使用 | 可能出现重名，不影响登录 |
| `immich_role` (自定义) | `immich_role` | `isAdmin` | 首次注册 | 首个用户 role 不为 admin | 管理员检查 `base.service.ts:225-230` | 抛出 `The first registered account must the administrator.` |
| `immich_quota` (自定义) | `immich_quota` | `quotaSizeInBytes` | 首次注册 | claim 值非数字或负数 | 默认值回退 `auth.service.ts:352-356` | 使用 `defaultStorageQuota` |

---

### 3.4 典型冲突场景的处理路径详解

#### 3.4.1 sub 变化场景

**场景描述**：
- LDAP 目录中用户条目被删除后重建，导致 `entryUUID` 变化
- IdP 配置变更，导致 `sub` 声明的生成规则改变
- 用户在 LDAP 中的 DN 变化，而 IdP 使用 DN 作为 sub

**处理路径**：
```
OAuth 登录回调
    ↓
getByOAuthId(new_sub) → 返回 undefined
    ↓
尝试通过 email 查找用户
    ├─ 找到用户且该用户 oauthId 为空 → 自动关联（更新 oauthId）
    ├─ 找到用户但该用户 oauthId ≠ new_sub → 冲突拒绝（邮箱已绑定他人）
    └─ 未找到用户 → 进入自动注册流程
        ├─ autoRegister=true → 尝试创建新用户
        │   ├─ 邮箱未被占用 → 创建成功
        │   └─ 邮箱已被占用 → 邮箱冲突，创建失败
        └─ autoRegister=false → 登录失败
```

**管理员应对措施**：
1. 若旧账号仍在，可让用户先通过密码登录，使用 `POST /oauth/unlink` 解除旧绑定，再重新 OAuth 登录
2. 或直接在数据库中更新用户的 `oauthId` 字段为新的 sub 值

---

#### 3.4.2 邮箱已绑定他人 OAuth 场景

**场景描述**：
- 用户 A 的邮箱 `user@example.com` 已绑定 OAuth 账号 sub_1
- 用户 B 在 IdP 中被分配了相同邮箱 `user@example.com`，其 sub 为 sub_2
- 用户 B 尝试通过 OAuth 登录 Immich

**处理路径** `auth.service.ts:320-330`：
```
getByOAuthId(sub_2) → undefined
    ↓
getByEmail(user@example.com) → 找到用户 A（oauthId=sub_1）
    ↓
检查 emailUser.oauthId → 非空且 ≠ sub_2
    ↓
抛出 BadRequestException('OAuth authentication failed')
```

**日志输出**：
```
OAuth login conflict: email already linked to different account
```

**管理员应对措施**：
1. 核实邮箱所有权归属
2. 若邮箱应归用户 B，需先解除用户 A 的 OAuth 绑定（`POST /oauth/unlink` 或数据库操作）
3. 确保 IdP 中邮箱分配的唯一性

---

#### 3.4.3 手动 link 冲突场景

**场景描述**：
- 用户 A 已通过密码登录，尝试将 OAuth 账号 sub_x 关联到自己
- 但 sub_x 已被用户 B 绑定

**处理路径** `auth.service.ts:407-435`：
```
POST /oauth/link（用户 A 已认证）
    ↓
完成 OAuth 授权，获取 oauthId=sub_x
    ↓
getByOAuthId(sub_x) → 找到用户 B
    ↓
检查 duplicate.id !== auth.user.id → true
    ↓
抛出 BadRequestException('This OAuth account has already been linked to another user.')
```

**管理员应对措施**：
1. 核实 OAuth 账号归属
2. 若应归用户 A，需先解除用户 B 的绑定
3. 或让用户 A 使用其他 OAuth 账号

---

### 3.5 LDAP 经 IdP 接入归一化冲突对照表

下表按实际执行顺序，完整列出 LDAP 经 IdP 接入时可能出现的所有归一化冲突场景。异常类型说明：
- **BadRequestException**：NestJS 内置异常，返回 `400 Bad Request`，客户端可看到具体错误消息
- **普通 Error**：被全局异常过滤器捕获，统一返回 `500 Internal Server Error`，客户端仅看到通用错误

| 序号 | Claim 来源 | Immich 落库字段 | 触发条件 | 代码处理路径 | 代码抛错类型 | 服务端日志文案 | 客户端 HTTP 响应 |
|------|-----------|----------------|----------|--------------|--------------|----------------|------------------|
| 1 | `sub` (LDAP entryUUID) | `oauthId` | `getByOAuthId(sub)` 找到用户 | `auth.service.ts:318` 直接使用该用户 | 无异常 | （无） | `200 OK`，登录成功 |
| 2 | `sub` (LDAP entryUUID) | `oauthId` | `getByOAuthId(sub)` 未命中，但 `getByEmail(email)` 找到用户且 `oauthId` 为空 | `auth.service.ts:320-330` 自动关联分支 | 无异常 | （无，静默更新） | `200 OK`，登录成功 |
| 3 | **`sub` 变化** (LDAP 条目重建) | `oauthId` | `getByOAuthId(new_sub)` 未命中，`getByEmail(email)` 找到用户但 `oauthId ≠ new_sub` | `auth.service.ts:324-327` 冲突拒绝分支 | `BadRequestException` | `OAuth login conflict: email already linked to different account` | `400 Bad Request`，`message: "OAuth authentication failed"` |
| 4 | **`sub` 变化** (LDAP 条目重建) | `oauthId` | `getByOAuthId(new_sub)` 未命中，`getByEmail(email)` 未命中，`autoRegister=true` | `auth.service.ts:332-375` 自动注册分支 | 无异常（成功路径） | `Registering new user: {sub}/{email}` | `200 OK`，创建新用户成功 |
| 5 | **`sub` 变化** (LDAP 条目重建) | `oauthId` | `getByOAuthId(new_sub)` 未命中，`getByEmail(email)` 未命中，`autoRegister=false` | `auth.service.ts:333-339` 拒绝分支 | `BadRequestException` | `Unable to register {sub}/{email}. User does not exist and auto registering is disabled.` | `400 Bad Request`，`message: "OAuth authentication failed"` |
| 6 | `email` (LDAP mail) | `email` | OAuth profile 无 `email` 字段或为空 | `auth.service.ts:341-343` 直接拒绝 | `BadRequestException` | （无，直接抛错） | `400 Bad Request`，`message: "OAuth profile does not have an email address"` |
| 7 | `email` (LDAP mail) | `email` | `getByOAuthId(sub)` 未命中，`getByEmail(email)` 找到用户但该用户的 `oauthId` 是**其他用户的 sub** | `auth.service.ts:324-327` 冲突拒绝分支 | `BadRequestException` | `OAuth login conflict: email already linked to different account` | `400 Bad Request`，`message: "OAuth authentication failed"` |
| 8 | `email` (LDAP mail) | `email` | 自动注册时 `createUser()` 发现邮箱已存在 | `base.service.ts:219-223` 邮箱唯一性检查 | `BadRequestException` | `User creation rejected: user already exists` | `400 Bad Request`，`message: "Email is not available"` |
| 9 | `sub` (手动 link 流程) | `oauthId` | `POST /oauth/link` 时 `getByOAuthId(sub)` 找到其他用户 | `auth.service.ts:424-427` 重复绑定检查 | `BadRequestException` | `OAuth link account failed: sub is already linked to another user ({email}).` | `400 Bad Request`，`message: "This OAuth account has already been linked to another user."` |
| 10 | `sub` (手动 link 流程) | `oauthId` | `POST /oauth/link` 时 `getByOAuthId(sub)` 找到当前用户自己 | `auth.service.ts:424-427` 检查通过 | 无异常 | （无） | `200 OK`，关联成功 |
| 11 | `immich_role` (LDAP 自定义属性) | `isAdmin` | 系统无管理员，新用户 `roleClaim` 不为 `admin` | `base.service.ts:225-230` 管理员检查 | `BadRequestException` | （无，直接抛错） | `400 Bad Request`，`message: "The first registered account must the administrator."` |
| 12 | `preferred_username` (LDAP uid) | `storageLabel` | 自动注册时 `storageLabel` 与已有用户重复 | 数据库层唯一约束违反 | 普通 `Error` | `duplicate key value violates unique constraint "user_storageLabel_key"` | `500 Internal Server Error` |
| 13 | - | - | **Token 交换失败**（code 无效/过期、PKCE 不匹配、签名算法错误等） | `oauth.repository.ts:81-130` token 交换流程 | 普通 `Error` | `OAuth login failed: {具体错误原因}` | `500 Internal Server Error` |

> **关于场景 8 的说明**：在 OAuth 回调主路径中，`createUser()` 之前已通过 `getByEmail()` 检查过邮箱不存在，因此"邮箱已存在"主要属于**并发竞争场景**（两个相同邮箱的用户几乎同时发起首次登录），而非常规必经分支。

> **关于场景 12、13 的说明**：普通 `Error` 分支在客户端统一返回 `500 Internal Server Error`，具体错误原因仅在服务端日志中可见，需排查时查看服务端日志定位真实原因。

---

### 3.5.1 排障指引结论

**排障时的判断顺序**：
1. **先看客户端状态码**：
   - `400 Bad Request` → 业务逻辑问题（参数错误、权限不足、冲突拒绝等），错误消息可直接定位原因
   - `500 Internal Server Error` → 服务端/IdP 问题（token 交换失败、数据库异常、网络问题等），需查看服务端日志
2. **再看服务端日志**：
   - 搜索 `OAuth login failed` → 定位 token 交换失败的具体原因（code 过期、PKCE 不匹配、签名算法错误等）
   - 搜索 `email already linked` → 定位账号绑定冲突
   - 搜索 `duplicate key` → 定位数据库唯一约束违反
   - 搜索 `auto registering is disabled` → 定位自动注册未开启问题

**设计意图**：通过"状态码粗分问题类型，具体原因藏于日志，既保证客户端安全（不暴露内部错误详情），又方便运维排障（日志保留完整上下文）。

---

### 3.6 归一化冲突串联说明

LDAP 经 IdP 接入时，OAuth 登录回调的实际执行顺序与冲突分流如下：

```
                  ┌─────────────────────────────────┐
                  │     POST /oauth/callback        │
                  │  携带 OAuth code + state        │
                  └────────────────┬────────────────┘
                                   │
                          验证 state、code_verifier
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │ 失败                    │ 成功                    │ 失败
         ▼                         ▼                         ▼
  400: state missing      换取 token，获取 profile   400: code_verifier missing
                                   │
                        profile.sub 必须存在 ────失败───► 抛出 OAuth login failed
                                   │
                        提取 email 并规范化
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │ email 为空              │ email 有效              │
         ▼                         ▼                         │
  【场景 6】                        │                         │
  400: 无 email 地址               │                         │
                                   ▼                         │
               ┌── getByOAuthId(profile.sub) ──┐             │
               │                                │             │
               ▼ 找到用户                       ▼ 未找到      │
       【场景 1】 登录成功            getByEmail(normalizedEmail) │
                                               │             │
                                    ┌──────────┼──────────┐  │
                                    │ 找到用户 │ 未找到   │  │
                                    ▼          ▼          │  │
                          检查 user.oauthId     autoRegister? │
                          ┌───────┴───────┐    ┌───┴───┐    │
                          │  为空         │    │ true  │    │
                          ▼               ▼    ▼       ▼    │
                   【场景 2】        【场景 3】 【场景 4】【场景 5】│
                   自动关联成功    400: 冲突  创建用户  400: 禁用│
                                                          │  │
                                                          └──┘
```

**执行顺序详解**：

1. **第一步：参数校验**（`auth.service.ts:298-306`）
   - 校验 OAuth state 是否存在（从 cookie 或 dto 读取）
   - 校验 PKCE code_verifier 是否存在（从 cookie 或 dto 读取）
   - 任一缺失抛出 `BadRequestException` → `400 Bad Request`

2. **第二步：Token 交换与 Profile 获取**（`auth.service.ts:308-314`）
   - 调用 `oauthRepository.getProfileAndOAuthSid()` 执行 Authorization Code Flow
   - 内部使用 `authorizationCodeGrant()` 换取 access token 和 ID token
   - 从 ID token claims 或 UserInfo 端点获取用户 profile
   - `profile.sub` 必须存在，否则抛出异常
   - **【场景 13】Token 交换失败**：code 无效/过期、PKCE 不匹配、签名算法错误等
     - `oauth.repository.ts:128` 抛出普通 `Error`
     - 被全局异常过滤器捕获 → `500 Internal Server Error`
     - 服务端日志：`OAuth login failed: {具体原因}`

3. **第三步：邮箱规范化**（`auth.service.ts:315`）
   - `normalizedEmail = profile.email?.trim().toLowerCase()`
   - 后续自动关联和自动注册都依赖此规范化后的值

4. **第四步：按优先级查找用户**
   - **优先查 oauthId**：`getByOAuthId(profile.sub)`
     - 命中 → 【场景 1】直接使用该用户，登录成功
   - **其次查 email**：oauthId 未命中时，`getByEmail(normalizedEmail)`
     - 找到用户且 `oauthId` 为空 → 【场景 2】自动关联（更新 oauthId 后登录）
     - 找到用户但 `oauthId` 非空且 ≠ 当前 sub → 【场景 3/7】冲突拒绝
     - 未找到用户 → 进入自动注册判断

5. **第五步：自动注册判断**
   - `autoRegister=false` → 【场景 5】`BadRequestException`，登录失败
   - `autoRegister=true` 且 `normalizedEmail` 为空 → 【场景 6】`BadRequestException`
   - `autoRegister=true` 且 email 有效 → 调用 `createUser()` 尝试创建
     - 邮箱唯一检查不通过 → 【场景 8】`BadRequestException`（*并发竞争场景*）
     - 系统无管理员且新用户非 admin → 【场景 11】`BadRequestException`
     - storageLabel 数据库唯一约束违反 → 【场景 12】普通 `Error` → `500`
     - 全部检查通过 → 【场景 4】创建用户成功

6. **手动 Link 流程独立路径**（`auth.service.ts:407-435`）
   - 用户已通过密码登录（携带有效 session）
   - 完成 OAuth 授权流程获取 oauthId
   - `getByOAuthId(oauthId)` 检查是否已绑定
     - 绑定到其他用户 → 【场景 9】`BadRequestException`
     - 绑定到自己或未绑定 → 【场景 10】关联成功

**设计意图**：通过"oauthId 优先、email 兜底、自动注册为最后手段"的三级查找策略，在保证安全性（防止账号劫持）的前提下，最大程度实现登录方式的平滑迁移。

**异常处理策略总结**：
- **可预期的业务错误**（如用户不存在、权限不足、参数缺失）→ 使用 `BadRequestException` 返回 `400`
- **不可预期的外部错误**（如 IdP 连接失败、token 验证失败、数据库约束违反）→ 抛出普通 `Error` 返回 `500`
- 这种区分有助于客户端判断是用户操作问题还是服务端/IdP 配置问题

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
