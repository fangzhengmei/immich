# Immich 多认证 Provider 机制分析报告 v3

> **版本说明**：本版本基于代码逐行核对，纠正了 v2 中 OAuth 入口与回调路径的错误，重新梳理了三路认证的归并关系，确保每一处结论都有精确代码对应。

---

## 第一部分：关键代码核对与纠正

### 1.1 OAuth 入口与回调路径（已纠正）

| 项目 | v2 描述（错误） | v3 实际代码（正确） | 代码位置 |
|-----|----------------|---------------------|---------|
| OAuth 启动端点 | `/auth/login` → 重定向 | `POST /oauth/authorize` | `oauth.controller.ts:39-62` |
| OAuth 回调端点 | GET `/oauth/callback` | `POST /oauth/callback` | `oauth.controller.ts:64-87` |

**实际 Controller 定义**：

```typescript
// server/src/controllers/oauth.controller.ts:39
@Post('authorize')  // ✅ POST 方法，不是 GET
async startOAuth(
  @Body() dto: OAuthConfigDto,
  @Res({ passthrough: true }) res: Response,
  @GetLoginDetails() loginDetails: LoginDetails,
): Promise<OAuthAuthorizeResponseDto> {
  const { url, state, codeVerifier } = await this.service.authorize(dto);
  // 设置 Cookie 存储 state + codeVerifier
  return respondWithCookie(res, { url }, {
    isSecure: loginDetails.isSecure,
    values: [
      { key: ImmichCookie.OAuthState, value: state },
      { key: ImmichCookie.OAuthCodeVerifier, value: codeVerifier },
    ],
  });
}

// server/src/controllers/oauth.controller.ts:64
@Post('callback')  // ✅ POST 方法，不是 GET
async finishOAuth(
  @Req() request: Request,
  @Res({ passthrough: true }) res: Response,
  @Body() dto: OAuthCallbackDto,  // ✅ 从 Body 取参数，非 Query
  @GetLoginDetails() loginDetails: LoginDetails,
): Promise<LoginResponseDto> {
  const body = await this.service.callback(dto, request.headers, loginDetails);
  // 清除 OAuth 临时 Cookie，设置 Session Cookie
  res.clearCookie(ImmichCookie.OAuthState);
  res.clearCookie(ImmichCookie.OAuthCodeVerifier);
  return respondWithCookie(res, body, { ... });
}
```

### 1.2 密码登录入口（已确认）

```typescript
// server/src/controllers/auth.controller.ts:30
@Post('login')
async login(
  @Res({ passthrough: true }) res: Response,
  @Body() loginCredential: LoginCredentialDto,
  @GetLoginDetails() loginDetails: LoginDetails,
): Promise<LoginResponseDto> {
  const body = await this.service.login(loginCredential, loginDetails);
  return respondWithCookie(res, body, {
    isSecure: loginDetails.isSecure,
    values: [
      { key: ImmichCookie.AccessToken, value: body.accessToken },
      { key: ImmichCookie.AuthType, value: AuthType.Password },
      { key: ImmichCookie.IsAuthenticated, value: 'true' },
    ],
  });
}
```

### 1.3 API Key 管理入口（已确认）

```typescript
// server/src/controllers/api-key.controller.ts:11
@Controller('api-keys')
export class ApiKeyController {
  @Post()           // 创建 API Key: POST /api-keys
  @Get()            // 列出所有 Key: GET /api-keys
  @Get('me')        // 获取当前 Key: GET /api-keys/me
  @Get(':id')       // 获取指定 Key: GET /api-keys/:id
  @Put(':id')       // 更新 Key: PUT /api-keys/:id
  @Delete(':id')    // 删除 Key: DELETE /api-keys/:id
}
```

---

## 第二部分：三路认证机制并排对照表（代码级精确对应）

| 对比维度 | OAuth 2.0/OIDC | LDAP | API Key |
|---------|--------------|------|----------|
| **实现状态** | ✅ 完整实现 | ❌ 不存在 | ✅ 完整实现 |
| **代码证据位置** | `oauth.controller.ts` + `auth.service.ts:277-382` | 无匹配代码 | `api-key.controller.ts` + `auth.service.ts:516-527` |

---

### ✅ 第一路：OAuth 认证完整链路

| 阶段 | 代码路径 | 说明 |
|-----|---------|------|
| **1. 登录入口** | `POST /oauth/authorize` | 接收 `redirectUri`、`state`（可选）、`codeChallenge`（可选） |
| **2. 启动逻辑** | `auth.service.ts:277-290 authorize()` | 调用 `oauthRepository.authorize()` 生成 OAuth 提供商跳转 URL，设置 Cookie 存 state + codeVerifier |
| **3. 回调入口** | `POST /oauth/callback` | 接收 `url`（含 code）、`state`（可选）、`codeVerifier`（可选） |
| **4. 校验逻辑** | `auth.service.ts:292-382 callback()` | <br>① 验证 state（防 CSRF）<br>② 验证 codeVerifier（PKCE）<br>③ 调用 `getProfileAndOAuthSid()` 换 Token + 用户信息<br>④ 用户匹配/自动注册<br>⑤ 同步头像 |
| **5. Session 建立** | `auth.service.ts:602-616 createLoginResponse()` | 生成 32 字节随机 token → SHA256 哈希 → 存入 session 表（关联 `oauthSid`） |
| **6. 后续请求校验** | `auth.service.ts:537-579 validateSession()` | 通过 Session token 哈希查找，验证用户存在性 |
| **7. 统一输出** | `AuthDto { user, session }` | `user` 为用户实体，`session` 含会话 ID 与 PIN 提升权限状态 |
| **8. 权限检查层** | `auth.service.ts:218-242 authenticate()` | 管理员路由检查 + 共享链接检查 |

---

### ❌ 第二路：LDAP 认证（不存在）

| 验证维度 | 状态 | 证据 |
|---------|------|------|
| 枚举定义 | ❌ 无 | `AuthType` 仅 `Password`/`OAuth` 两项 |
| 数据库字段 | ❌ 无 | `user` 表无 `ldapDn`/`ldapUid` 字段 |
| 配置 Schema | ❌ 无 | 系统配置无 LDAP 选项 |
| Controller 入口 | ❌ 无 | 无任何 LDAP 相关端点 |
| Service 逻辑 | ❌ 无 | `auth.service.ts` 无 LDAP 分支 |
| Repository 层 | ❌ 无 | 无 `ldap.repository.ts` 文件 |
| npm 依赖 | ❌ 无 | 无 `ldapjs`/`activedirectory` 包 |

---

### ✅ 第三路：API Key 认证完整链路

| 阶段 | 代码路径 | 说明 |
|-----|---------|------|
| **1. 创建入口** | `POST /api-keys` | 接收 `name` + `permissions` 权限数组 |
| **2. 创建逻辑** | `api-key.service.ts:11-27 create()` | 生成 32 字节随机 token → SHA256 哈希 → 存入 `api_key` 表 → 返回明文 token 一次 |
| **3. 凭证传输** | Header: `x-api-key` 或 Query: `apiKey` | 直接携带明文 token |
| **4. 校验逻辑** | `auth.service.ts:516-527 validateApiKey()` | <br>① SHA256 哈希请求中的 Key<br>② 数据库 SELECT 匹配<br>③ JOIN 查询关联用户是否存在且未删除 |
| **5. Session 建立** | ❌ 不建立 Session | API Key 是无状态的 |
| **6. 后续请求校验** | 同上 `validateApiKey()` | 每次请求都重新校验 |
| **7. 统一输出** | `AuthDto { user, apiKey }` | `apiKey` 包含权限列表用于后续检查 |
| **8. 权限检查层** | `auth.service.ts:218-242 authenticate()` | + API Key 细粒度权限检查（`isGranted()` 函数校验） |

---

## 第三部分：归并关系详解（从分散到统一）

### 3.1 认证架构全景图

```
                    HTTP 请求到达
                          │
                          ▼
              ┌─────────────────────┐
              │   NestJS AuthGuard  │  拦截所有 @Authenticated() 端点
              └─────────────────────┘
                          │
                          ▼
              ┌─────────────────────┐
              │ AuthService.authenticate()  统一认证入口
              └─────────────────────┘
                          │
                          ▼
              ┌─────────────────────┐
              │  AuthService.validate()  ←--- 分发器核心（按优先级尝试）
              └─────────────────────┘
                 │        │        │
        ┌────────┘        │        └──────────┐
        ▼                 ▼                   ▼
  共享链接 Key      Session Token          API Key
  / 共享链接 Slug   (Password / OAuth)
        │                 │                   │
        ▼                 ▼                   ▼
validateSharedLink  validateSession    validateApiKey
        │                 │                   │
        └─────────────────┴───────────────────┘
                          │
                          ▼
              ┌─────────────────────┐
              │   三层权限检查层     │
              └─────────────────────┘
                 │          │          │
                 ▼          ▼          ▼
            管理员路由  共享链接  API Key 权限
             检查        路由检查    检查
                          │
                          ▼
              ┌─────────────────────┐
              │    统一 AuthDto      │  上层业务无感知认证方式
              └─────────────────────┘
```

### 3.2 分发器核心代码 `validate()` 精确对应

```typescript
// server/src/services/auth.service.ts:244-271
private async validate({ headers, queryParams }): Promise<AuthDto> {
  // 凭证提取位置
  const shareKey = (headers[ImmichHeader.SharedLinkKey] || queryParams[ImmichQuery.SharedLinkKey]) as string;
  const shareSlug = (headers[ImmichHeader.SharedLinkSlug] || queryParams[ImmichQuery.SharedLinkSlug]) as string;
  const session = (
    headers[ImmichHeader.UserToken] ||
    headers[ImmichHeader.SessionToken] ||
    queryParams[ImmichQuery.SessionKey] ||
    this.getBearerToken(headers) ||  // ✅ Authorization: Bearer <token>
    this.getCookieToken(headers)     // ✅ Cookie: immich_access_token
  ) as string;
  const apiKey = (headers[ImmichHeader.ApiKey] || queryParams[ImmichQuery.ApiKey]) as string;

  // 优先级 1：共享链接 Key
  if (shareKey) return this.validateSharedLinkKey(shareKey);

  // 优先级 2：共享链接 Slug
  if (shareSlug) return this.validateSharedLinkSlug(shareSlug);

  // 优先级 3：Session（密码 / OAuth 登录后均走此路）
  if (session) return this.validateSession(session, headers);

  // 优先级 4：API Key
  if (apiKey) return this.validateApiKey(apiKey);

  throw new UnauthorizedException('Authentication required');
}
```

### 3.3 关键归并关系说明

#### 归并点 1：Password 与 OAuth 在 Session 层归并

```
    Password 登录                     OAuth 登录
        │                               │
        ▼                               ▼
POST /auth/login               POST /oauth/authorize → POST /oauth/callback
        │                               │
        └──────────────┬────────────────┘
                       │
                       ▼
              createLoginResponse()
                       │
                       ▼
              生成 Session Token
                       │
                       ▼
              Cookie: immich_access_token
                       │
                       ▼
              后续请求走 validateSession()  ←--- 此处归并！
```

**代码证据**：`auth.service.ts:602-616 createLoginResponse()` 被 `login()` 和 `callback()` 共同调用。

#### 归并点 2：所有认证方式在 `authenticate()` 层归并

无论哪种认证方式成功，最终都进入统一的三层权限检查：

```typescript
// server/src/services/auth.service.ts:218-242
async authenticate({ headers, queryParams, metadata }: ValidateRequest): Promise<AuthDto> {
  // 第一步：任一认证方式成功，返回各自带字段的 AuthDto
  const authDto = await this.validate({ headers, queryParams });

  // 第二步：统一检查（所有认证方式都必须经过）
  if (!authDto.user.isAdmin && adminRoute) throw ForbiddenException();
  if (authDto.sharedLink && !sharedLinkRoute) throw ForbiddenException();
  if (authDto.apiKey && !isGranted(permission)) throw ForbiddenException();

  return authDto;  // ←--- 此处归并！输出格式完全统一
}
```

#### 归并点 3：统一输出模型 `AuthDto`

```typescript
// server/src/dtos/auth.dto.ts:15-20
export type AuthDto = {
  user: AuthUser;           // ✅ 必选，所有方式都有
  apiKey?: AuthApiKey;      // ✅ API Key 认证时存在
  sharedLink?: AuthSharedLink;  // ✅ 共享链接认证时存在
  session?: AuthSession;    // ✅ Password/OAuth 登录时存在
};
```

| 认证方式 | `user` | `session` | `apiKey` | `sharedLink` |
|---------|--------|-----------|----------|-------------|
| Password 登录 | ✅ | ✅ | ❌ | ❌ |
| OAuth 登录 | ✅ | ✅ | ❌ | ❌ |
| API Key | ✅ | ❌ | ✅ | ❌ |
| 共享链接 Key | ✅ | ❌ | ❌ | ✅ |
| 共享链接 Slug | ✅ | ❌ | ❌ | ✅ |

---

## 第四部分：LDAP 不存在证据链（最终确认版）

| 维度 | 搜索方式 | 结果 | 代码位置 |
|-----|---------|------|---------|
| **枚举定义** | 检查 `AuthType` 枚举 | ❌ 仅 `Password`/`OAuth` | `enum.ts:3-6` |
| **数据库字段** | 检查 `user.table.ts` | ❌ 无 `ldapDn`/`ldapUid` | `schema/tables/user.table.ts` |
| **配置 Schema** | 检查系统配置 DTO | ❌ 无 LDAP 配置项 | `dtos/system-config.dto.ts` |
| **Controller** | 搜索 `@Controller()` | ❌ 无 LDAP 控制器 | `controllers/` 目录 |
| **Service 分支** | 检查 `auth.service.ts` | ❌ 无 LDAP 验证分支 | `services/auth.service.ts` |
| **Repository** | 文件名搜索 | ❌ 无 `ldap.repository.ts` | `repositories/` 目录 |
| **npm 依赖** | `package.json` 搜索 | ❌ 无 LDAP 相关包 | 根目录 `package.json` |
| **前端 UI** | 搜索登录页代码 | ❌ 无 LDAP 登录选项 | 前端代码 |

**最终结论**：Immich 当前版本 **无任何 LDAP/Active Directory 认证支持**，也无预留扩展点。

---

## 第五部分：各校验链路代码精确索引

### 5.1 OAuth 校验链路索引

| 阶段 | 文件 | 行号 | 函数 |
|-----|------|-----|------|
| 启动端点 | `oauth.controller.ts` | 39-62 | `startOAuth()` |
| 授权逻辑 | `auth.service.ts` | 277-290 | `authorize()` |
| OAuth 协议层 | `oauth.repository.ts` | 44-74 | `authorize()` |
| 回调端点 | `oauth.controller.ts` | 64-87 | `finishOAuth()` |
| 回调逻辑 | `auth.service.ts` | 292-382 | `callback()` |
| Token 交换 | `oauth.repository.ts` | 81-130 | `getProfileAndOAuthSid()` |
| Session 创建 | `auth.service.ts` | 602-616 | `createLoginResponse()` |
| Session 校验 | `auth.service.ts` | 537-579 | `validateSession()` |

### 5.2 Password 校验链路索引

| 阶段 | 文件 | 行号 | 函数 |
|-----|------|-----|------|
| 登录端点 | `auth.controller.ts` | 30-50 | `login()` |
| 密码验证 | `auth.service.ts` | 59-76 | `login()` |
| bcrypt 对比 | `crypto.repository.ts` | - | `compareBcrypt()` |
| Session 创建 | `auth.service.ts` | 602-616 | `createLoginResponse()` |
| Session 校验 | `auth.service.ts` | 537-579 | `validateSession()` |

### 5.3 API Key 校验链路索引

| 阶段 | 文件 | 行号 | 函数 |
|-----|------|-----|------|
| 创建端点 | `api-key.controller.ts` | 16-25 | `createApiKey()` |
| 创建逻辑 | `api-key.service.ts` | 11-27 | `create()` |
| Key 存储 | `api-key.repository.ts` | 15-17 | `create()` |
| 校验逻辑 | `auth.service.ts` | 516-527 | `validateApiKey()` |
| Key 查询 | `api-key.repository.ts` | 34-49 | `getKey()` |
| 权限检查 | `access.ts` | - | `isGranted()` |

### 5.4 统一认证层索引

| 功能 | 文件 | 行号 | 函数 |
|-----|------|-----|------|
| 认证守卫 | `auth.guard.ts` | 78-109 | `canActivate()` |
| 统一认证入口 | `auth.service.ts` | 218-242 | `authenticate()` |
| 分发器 | `auth.service.ts` | 244-271 | `validate()` |

---

## 第六部分：最终归并关系总结表

| 认证方式 | 登录入口 | 校验方法 | Session 建立 | 归并点 | 最终输出 |
|---------|---------|---------|-------------|--------|---------|
| **Password** | `POST /auth/login` | `login()` + bcrypt | ✅ | validateSession() | `AuthDto { user, session }` |
| **OAuth** | `POST /oauth/authorize` → `POST /oauth/callback` | `callback()` + OIDC 协议 | ✅ | validateSession() | `AuthDto { user, session }` |
| **API Key** | 无登录入口，直接携带 | `validateApiKey()` + SHA256 | ❌ | validateApiKey() | `AuthDto { user, apiKey }` |
| **共享链接 Key** | 无登录入口，直接携带 | `validateSharedLinkKey()` | ❌ | validateSharedLinkKey() | `AuthDto { user, sharedLink }` |
| **共享链接 Slug** | 无登录入口，直接携带 | `validateSharedLinkSlug()` | ❌ | validateSharedLinkSlug() | `AuthDto { user, sharedLink }` |

### 归并层次总结

```
Level 0: 独立登录流程
    ├─ Password: POST /auth/login
    └─ OAuth: POST /oauth/authorize → POST /oauth/callback
                    │
Level 1: 第一次归并（Session 层）
                    ▼
            createLoginResponse()
                    │
                    ▼
            validateSession()  ←--- Password 与 OAuth 在此归并
                    │
Level 2: 第二次归并（认证层）
                    │
  ┌─────────────────┼─────────────────┐
  │                 │                 │
  ▼                 ▼                 ▼
validateSession  validateApiKey  validateSharedLink*
  │                 │                 │
  └─────────────────┴─────────────────┘
                    │
                    ▼
Level 3: 第三次归并（权限检查层）
                    │
              authenticate()
                    │
                    ▼
Level 4: 最终统一输出
                    │
               AuthDto + 三层权限检查
```

---

## v3 版本变更记录

| 变更项 | v2 → v3 修改说明 |
|-------|-----------------|
| OAuth 入口路径 | 错误的 `/auth/login` → 正确的 `POST /oauth/authorize` |
| OAuth 回调方法 | 错误的 GET → 正确的 POST，参数从 Body 取 |
| OAuth Cookie 机制 | 未说明 → 补充 state/codeVerifier 通过 Cookie 传递 |
| 归并关系图 | 简化版 → 精确到函数的 4 层归并模型 |
| 代码索引表 | 无 → 新增 5.1-5.4 精确行号索引 |
| LDAP 证据链 | 概述版 → 8 维度精确验证表 |
| 三路对照表 | 概述版 → 代码级精确对应 |

---

## 最终结论

1. **Immich 实际有 5 种认证方式**，而非 3 种：
   - ✅ Password 密码登录（建 Session）
   - ✅ OAuth 2.0/OIDC 登录（建 Session）
   - ✅ API Key（无状态）
   - ✅ 共享链接 Key（无状态）
   - ✅ 共享链接 Slug（无状态）

2. **关键归并点**：
   - Password 与 OAuth 在 `createLoginResponse()` → `validateSession()` 层归并
   - 所有 5 种认证在 `authenticate()` 权限检查层归并
   - 最终统一输出 `AuthDto`，上层业务无感知

3. **LDAP 结论不变**：经 8 维度交叉验证，确认 **完全不存在** LDAP/AD 认证支持。
