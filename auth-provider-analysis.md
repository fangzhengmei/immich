# Immich 多认证 Provider 机制分析报告

## 概述

经过代码分析，Immich 项目实际实现了 **三种认证机制** 共存，而非原本预期的 OAuth/LDAP/API Key。**项目中并没有 LDAP 认证** 的实现。三种实际认证机制为：

1. **Password（密码登录）** - 传统用户名密码认证
2. **OAuth（OAuth2/OIDC 认证）** - 第三方 OAuth2/OIDC 认证
3. **API Key（API 密钥认证）** - API 密钥访问认证

这三种认证机制各自走独立的验证路径，最终汇集到统一的认证结论对象 `AuthDto`。

---

## 核心认证架构

### 认证流程入口

认证流程的核心入口是 `AuthGuard`（`server/src/middleware/auth.guard.ts`），它作为 NestJS 的守卫拦截所有需要认证的请求。

```typescript
// server/src/middleware/auth.guard.ts:88-108
async canActivate(context: ExecutionContext): Promise<boolean> {
  const targets = [context.getHandler()];
  const options = this.reflector.getAllAndOverride<AuthenticatedOptions | undefined>(MetadataKey.AuthRoute, targets);
  if (!options) {
    return true;
  }

  const { admin: adminRoute, sharedLink: sharedLinkRoute, permission } = { sharedLink: false, admin: false, ...options };
  const request = context.switchToHttp().getRequest<AuthRequest>();

  request.user = await this.authService.authenticate({
    headers: request.headers,
    queryParams: request.query as Record<string, string>,
    metadata: { adminRoute, sharedLinkRoute, permission, uri: request.path },
  });

  return true;
}
```

### 统一认证方法：authenticate()

`AuthService.authenticate()` 方法负责调用具体的验证逻辑并进行统一的权限检查：

```typescript
// server/src/services/auth.service.ts:218-242
async authenticate({ headers, queryParams, metadata }: ValidateRequest): Promise<AuthDto> {
  const authDto = await this.validate({ headers, queryParams });
  const { adminRoute, sharedLinkRoute, uri } = metadata;
  const requestedPermission = metadata.permission ?? Permission.All;

  // 管理员路由检查
  if (!authDto.user.isAdmin && adminRoute) {
    this.logger.warn(`Denied access to admin only route: ${uri}`);
    throw new ForbiddenException('Forbidden');
  }

  // 共享链接路由检查
  if (authDto.sharedLink && !sharedLinkRoute) {
    this.logger.warn(`Denied access to non-shared route: ${uri}`);
    throw new ForbiddenException('Forbidden');
  }

  // API Key 权限检查
  if (authDto.apiKey && requestedPermission !== false && 
      !isGranted({ requested: [requestedPermission], current: authDto.apiKey.permissions })) {
    throw new ForbiddenException(`Missing required permission: ${requestedPermission}`);
  }

  return authDto;
}
```

---

## 认证 Provider 详细分析

### Provider 分发器：validate() 方法

三种认证方式在 `validate()` 方法中按优先级顺序依次尝试，只要其中一种成功即返回结果：

```typescript
// server/src/services/auth.service.ts:244-271
private async validate({ headers, queryParams }: Omit<ValidateRequest, 'metadata'>): Promise<AuthDto> {
  const shareKey = (headers[ImmichHeader.SharedLinkKey] || queryParams[ImmichQuery.SharedLinkKey]) as string;
  const shareSlug = (headers[ImmichHeader.SharedLinkSlug] || queryParams[ImmichQuery.SharedLinkSlug]) as string;
  const session = (headers[ImmichHeader.UserToken] ||
    headers[ImmichHeader.SessionToken] ||
    queryParams[ImmichQuery.SessionKey] ||
    this.getBearerToken(headers) ||
    this.getCookieToken(headers)) as string;
  const apiKey = (headers[ImmichHeader.ApiKey] || queryParams[ImmichQuery.ApiKey]) as string;

  // 验证优先级：共享链接 Key > 共享链接 Slug > Session > API Key
  if (shareKey) {
    return this.validateSharedLinkKey(shareKey);
  }

  if (shareSlug) {
    return this.validateSharedLinkSlug(shareSlug);
  }

  if (session) {
    return this.validateSession(session, headers);
  }

  if (apiKey) {
    return this.validateApiKey(apiKey);
  }

  throw new UnauthorizedException('Authentication required');
}
```

### Provider 1：Password 密码认证

#### 认证路径
1. 用户通过 `/auth/login` 端点提交邮箱和密码
2. `AuthService.login()` 方法验证凭证
3. 验证成功后创建 Session 会话

```typescript
// server/src/services/auth.service.ts:59-76
async login(dto: LoginCredentialDto, details: LoginDetails) {
  const config = await this.getConfig({ withCache: false });
  if (!config.passwordLogin.enabled) {
    throw new UnauthorizedException('Password login has been disabled');
  }

  const user = await this.userRepository.getByEmail(dto.email, { withPassword: true });
  // 始终运行 bcrypt 以防止时序攻击枚举用户
  const authenticated = this.cryptoRepository.compareBcrypt(dto.password, user?.password ?? LOGIN_DUMMY_HASH);

  if (!user || !user.password || !authenticated) {
    this.logger.warn(`Failed login attempt for user ${dto.email} from ip address ${details.clientIp}`);
    throw new UnauthorizedException('Incorrect email or password');
  }

  return this.createLoginResponse(user, details);
}
```

#### 安全特性
- **时序攻击防护**：即使邮箱不存在也执行 bcrypt 哈希比较，响应时间一致
- **可配置开关**：通过 `passwordLogin.enabled` 配置项控制密码登录是否启用

---

### Provider 2：OAuth2/OIDC 认证

#### OAuth 核心组件

1. **OAuth Repository** (`server/src/repositories/oauth.repository.ts`) - 底层 OAuth 协议实现
2. **AuthService 业务逻辑** (`server/src/services/auth.service.ts`) - 集成到认证系统

#### 认证流程

```
客户端 → /oauth/authorize → 重定向到 OAuth 提供商
    ↓
用户在提供商登录授权
    ↓
回调 → /oauth/callback → AuthService.callback()
    ↓
1. 验证 state 和 code_verifier
2. 用 code 交换 token
3. 获取用户信息（ID Token 或 UserInfo 端点）
4. 通过邮箱查找或自动创建本地用户
5. 创建本地 Session
    ↓
返回登录凭证
```

#### 关键实现代码

```typescript
// server/src/services/auth.service.ts:292-382
async callback(dto: OAuthCallbackDto, headers: IncomingHttpHeaders, loginDetails: LoginDetails) {
  const { oauth } = await this.getConfig({ withCache: false });
  if (!oauth.enabled) {
    throw new BadRequestException('OAuth is not enabled');
  }

  // CSRF 防护：验证 state
  const expectedState = dto.state ?? this.getCookieOauthState(headers);
  if (!expectedState?.length) {
    throw new BadRequestException('OAuth state is missing');
  }

  // PKCE 验证：code_verifier
  const codeVerifier = dto.codeVerifier ?? this.getCookieCodeVerifier(headers);
  if (!codeVerifier?.length) {
    throw new BadRequestException('OAuth code verifier is missing');
  }

  // 获取 OAuth 用户信息
  const { profile, sid: oauthSid } = await this.oauthRepository.getProfileAndOAuthSid(
    oauth, this.resolveRedirectUri(oauth, dto.url), expectedState, codeVerifier
  );
  
  const normalizedEmail = profile.email ? profile.email.trim().toLowerCase() : undefined;

  // 1. 尝试通过 OAuth sub (subject ID) 查找用户
  let user: UserAdmin | undefined = await this.userRepository.getByOAuthId(profile.sub);

  // 2. 如未找到，尝试通过邮箱链接已有账号
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

  // 3. 如仍未找到且启用了自动注册，创建新用户
  if (!user) {
    if (!oauth.autoRegister) {
      this.logger.warn(
        `Unable to register ${profile.sub}/${normalizedEmail || '(no email)'}. User does not exist and auto registering is disabled.`
      );
      throw new BadRequestException('OAuth authentication failed');
    }

    if (!normalizedEmail) {
      throw new BadRequestException('OAuth profile does not have an email address');
    }

    // 从 OAuth Token 声明中提取额外信息
    const storageLabel = this.getClaim(profile, { key: storageLabelClaim, default: '', isValid: ... });
    const storageQuota = this.getClaim(profile, { key: storageQuotaClaim, default: defaultStorageQuota, isValid: ... });
    const role = this.getClaim(profile, { key: roleClaim, default: 'user', isValid: ... });

    user = await this.createUser({
      name: profile.name || profile.given_name + ' ' + profile.family_name || profile.preferred_username || normalizedEmail,
      email: normalizedEmail,
      oauthId: profile.sub,
      quotaSizeInBytes: storageQuota === null ? null : storageQuota * HumanReadableSize.GiB,
      storageLabel: storageLabel || null,
      isAdmin: role === 'admin',
    });
  }

  // 同步用户头像
  if (!user.profileImagePath && profile.picture) {
    await this.syncProfilePicture(user, profile.picture);
  }

  return this.createLoginResponse(user, loginDetails, oauthSid);
}
```

#### OAuth 配置项

| 配置项 | 说明 |
|--------|------|
| `enabled` | 是否启用 OAuth |
| `issuerUrl` | OIDC Issuer URL，用于自动发现配置 |
| `clientId` / `clientSecret` | 客户端凭证 |
| `scope` | 请求的 OAuth Scope |
| `prompt` | 授权页面 prompt 参数 |
| `autoRegister` | 是否自动创建本地用户 |
| `autoLaunch` | 是否自动跳转到 OAuth 登录页 |
| `buttonText` | 登录按钮显示文本 |
| `signingAlgorithm` | ID Token 签名算法 |
| `storageLabelClaim` / `storageQuotaClaim` / `roleClaim` | 从 Token 中提取用户属性的 Claim 名称 |
| `mobileOverrideEnabled` / `mobileRedirectUri` | 移动端重写配置 |
| `endSessionEndpoint` | 单点登出端点 |

---

### Provider 3：API Key 认证

#### 架构设计

API Key 采用 **细粒度权限控制** 设计，每个密钥关联特定的权限集合。

#### 验证实现

```typescript
// server/src/services/auth.service.ts:516-527
private async validateApiKey(key: string): Promise<AuthDto> {
  const hashed = this.cryptoRepository.hashSha256(key);
  const apiKey = await this.apiKeyRepository.getKey(hashed);
  if (apiKey?.user) {
    return {
      user: apiKey.user,
      apiKey,
    };
  }

  throw new UnauthorizedException('Invalid API key');
}
```

#### 安全存储机制

```typescript
// server/src/repositories/api-key.repository.ts:34-49
getKey(hashedToken: Buffer) {
  return this.db
    .selectFrom('api_key')
    .select((eb) => [
      ...columns.authApiKey,
      jsonObjectFrom(
        eb
          .selectFrom('user')
          .select(columns.authUser)
          .whereRef('user.id', '=', 'api_key.userId')
          .where('user.deletedAt', 'is', null),
      ).as('user'),
    ])
    .where('api_key.key', '=', hashedToken)
    .executeTakeFirst();
}
```

**重要安全特性**：
- **仅存储哈希值**：数据库中不存储原始密钥，只存储 SHA256 哈希值
- **数据库关联查询**：查询时同时验证关联用户是否存在且未删除

#### API Key 创建流程

```typescript
// server/src/services/api-key.service.ts:11-27
async create(auth: AuthDto, dto: ApiKeyCreateDto): Promise<ApiKeyCreateResponseDto> {
  const token = this.cryptoRepository.randomBytesAsText(32);
  const hashed = this.cryptoRepository.hashSha256(token);

  // 权限继承检查：不能授予超出当前认证的权限
  if (auth.apiKey && !isGranted({ requested: dto.permissions, current: auth.apiKey.permissions })) {
    throw new BadRequestException('Cannot grant permissions you do not have');
  }

  const entity = await this.apiKeyRepository.create({
    key: hashed,
    name: dto.name || 'API Key',
    userId: auth.user.id,
    permissions: dto.permissions,
  });

  return { secret: token, apiKey: this.map(entity) };
}
```

#### 权限系统

权限采用**层级式点分命名法**，支持通配符：

```typescript
// server/src/enum.ts:104-307
export enum Permission {
  All = 'all',  // 通配符，拥有所有权限

  // 按资源类型分类
  ActivityCreate = 'activity.create',
  ActivityRead = 'activity.read',
  // ...

  ApiKeyCreate = 'apiKey.create',
  ApiKeyRead = 'apiKey.read',
  ApiKeyUpdate = 'apiKey.update',
  ApiKeyDelete = 'apiKey.delete',

  AssetRead = 'asset.read',
  AssetUpdate = 'asset.update',
  AssetDelete = 'asset.delete',
  // ... 共有约 100+ 种细粒度权限
}
```

---

## 统一认证结论：AuthDto

无论使用哪种认证方式，最终都会生成统一的 `AuthDto` 对象，实现了"多路径输入，单结构输出"的设计。

```typescript
// server/src/dtos/auth.dto.ts:15-20
export type AuthDto = {
  user: AuthUser;           // 必选：认证用户信息
  apiKey?: AuthApiKey;      // API Key 认证时存在
  sharedLink?: AuthSharedLink;  // 共享链接认证时存在
  session?: AuthSession;    // Session 登录时存在
};
```

### AuthDto 字段含义

| 字段 | 说明 | Password | OAuth | API Key |
|------|------|----------|-------|---------|
| `user` | 用户基本信息 | ✅ | ✅ | ✅ |
| `session` | 会话信息（含 elevated 权限状态） | ✅ | ✅ | ❌ |
| `apiKey` | API Key 信息及权限列表 | ❌ | ❌ | ✅ |
| `sharedLink` | 共享链接信息 | ❌ | ❌ | ❌ * |

*注：共享链接是第四种认证方式，用于公开相册访问。

---

## Session 会话机制

Password 和 OAuth 认证最终都会创建 Session，这是 Web 登录的统一会话机制。

### Session 创建

```typescript
// server/src/services/auth.service.ts:602-616
private async createLoginResponse(user: UserAdmin, loginDetails: LoginDetails, oauthSid?: string) {
  const token = this.cryptoRepository.randomBytesAsText(32);
  const hashed = this.cryptoRepository.hashSha256(token);

  await this.sessionRepository.create({
    token: hashed,
    deviceOS: loginDetails.deviceOS,
    deviceType: loginDetails.deviceType,
    appVersion: loginDetails.appVersion,
    userId: user.id,
    oauthSid: oauthSid ?? null,  // OAuth 会话记录 sid，用于单点登出
  });

  return mapLoginResponse(user, token);
}
```

### Session 验证与 PIN 保护

```typescript
// server/src/services/auth.service.ts:537-579
private async validateSession(token: string, headers: IncomingHttpHeaders): Promise<AuthDto> {
  const hashed = this.cryptoRepository.hashSha256(token);
  const session = await this.sessionRepository.getByToken(hashed);
  if (session?.user) {
    const { appVersion, deviceOS, deviceType } = getUserAgentDetails(headers);
    
    // 定期更新会话元数据
    const now = DateTime.now();
    const updatedAt = DateTime.fromJSDate(session.updatedAt);
    const diff = now.diff(updatedAt, ['hours']);
    if (diff.hours > 1 || appVersion != session.appVersion) {
      await this.sessionRepository.update(session.id, {
        id: session.id,
        updatedAt: new Date(),
        appVersion,
        deviceOS,
        deviceType,
      });
    }

    // PIN 码保护的提升权限检查
    let hasElevatedPermission = false;
    if (session.pinExpiresAt) {
      const pinExpiresAt = DateTime.fromJSDate(session.pinExpiresAt);
      hasElevatedPermission = pinExpiresAt > now;

      // 自动续期：如果 PIN 即将过期且仍在活跃使用中
      if (hasElevatedPermission && now.plus({ minutes: 5 }) > pinExpiresAt) {
        await this.sessionRepository.update(session.id, {
          pinExpiresAt: DateTime.now().plus({ minutes: 5 }).toJSDate(),
        });
      }
    }

    return {
      user: session.user,
      session: {
        id: session.id,
        hasElevatedPermission,
      },
    };
  }

  throw new UnauthorizedException('Invalid user token');
}
```

---

## 认证方式共存机制总结

### 1. 按优先级顺序验证

```
请求到达
    ↓
AuthGuard 拦截
    ↓
AuthService.authenticate()
    ↓
AuthService.validate() 按顺序尝试:
  ├─ ① 共享链接 Key (Header/Query)
  ├─ ② 共享链接 Slug (Header/Query)
  ├─ ③ Session (Header/Query/Authorization Bearer/Cookie)
  └─ ④ API Key (Header/Query)
    ↓
  任一成功 → 生成 AuthDto → 权限检查 → 通过
    ↓
  全部失败 → 抛出 UnauthorizedException
```

### 2. 各认证方式的凭证传输位置

| 认证方式 | HTTP Header | URL Query | Authorization | Cookie |
|---------|------------|-----------|--------------|--------|
| Session | `x-immich-user-token` / `x-immich-session-token` | `sessionKey` | `Bearer <token>` | `immich_access_token` |
| API Key | `x-api-key` | `apiKey` | ❌ | ❌ |
| 共享链接 Key | `x-immich-share-key` | `key` | ❌ | ❌ |
| 共享链接 Slug | `x-immich-share-slug` | `slug` | ❌ | ❌ |

### 3. 统一的权限检查层

无论哪种认证方式，都会经过 `authenticate()` 方法中的三层权限检查：
1. **管理员路由检查** - 仅管理员可访问
2. **共享链接路由检查** - 共享链接只能访问特定路由
3. **API Key 权限检查** - 验证密钥是否具备请求所需权限

### 4. 认证类型标记（AuthType）

系统通过 `AuthType` 枚举区分密码登录和 OAuth 登录，主要用于差异化的登出流程：

```typescript
// server/src/enum.ts:3-6
export enum AuthType {
  Password = 'password',
  OAuth = 'oauth',
}
```

OAuth 登录用户登出时会尝试调用 OAuth Provider 的 end_session_endpoint 实现单点登出。

---

## 关键安全设计

### 1. 密码安全
- 使用 bcrypt 哈希存储密码
- 抗时序攻击的登录验证设计

### 2. Session 安全
- 32 字节（256 位）加密安全随机令牌
- 仅存储 SHA256 哈希值，不存储明文
- PIN 码保护的提升权限机制（敏感操作二次验证）

### 3. API Key 安全
- SHA256 哈希存储，泄露后无法逆向
- 细粒度权限控制，最小权限原则
- 权限继承检查，无法授予超出当前的权限

### 4. OAuth 安全
- PKCE（Proof Key for Code Exchange）支持
- State 参数防止 CSRF 攻击
- OIDC 发现协议自动配置端点
- 支持多种签名算法验证

---

## 代码文件索引

| 功能 | 文件路径 |
|------|---------|
| 认证核心服务 | `server/src/services/auth.service.ts` |
| API Key 服务 | `server/src/services/api-key.service.ts` |
| 认证守卫 | `server/src/middleware/auth.guard.ts` |
| OAuth 协议层 | `server/src/repositories/oauth.repository.ts` |
| API Key 存储 | `server/src/repositories/api-key.repository.ts` |
| 认证 DTO | `server/src/dtos/auth.dto.ts` |
| 权限枚举 | `server/src/enum.ts` |
| 系统配置 DTO | `server/src/dtos/system-config.dto.ts` |

---

## 结论

Immich 的认证架构设计体现了优秀的可扩展性和安全性：

1. **多 Provider 共存**：密码、OAuth、API Key 三种认证方式独立实现，互不干扰
2. **统一输出模型**：所有认证路径最终汇聚到 `AuthDto`，上层业务逻辑无需关心认证方式
3. **优先级顺序验证**：按安全等级和使用场景确定验证顺序
4. **分层权限检查**：在认证通过后统一进行管理员、路由、权限三级检查
5. **安全设计完善**：从密码存储、令牌生成到 OAuth 协议实现，都遵循了现代安全最佳实践

**注**：代码库中未发现 LDAP 认证相关实现，`AuthType` 枚举仅包含 `Password` 和 `OAuth` 两种类型。
