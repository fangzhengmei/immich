# Immich API Key Scope 权限链路分析

## 概述

Immich 的 API Key 权限系统采用**四层防护**机制：路由级权限声明 → 认证中间件拦截 → Scope 权限校验 → 资源级访问控制。整个链路从请求进入到最终授权形成完整的安全闭环。

---

## 一、权限链路总览

```
HTTP Request
    ↓
[AuthGuard] NestJS 守卫 (auth.guard.ts:78-109)
    ├─ 读取路由元数据 (@Authenticated 装饰器)
    ├─ 调用 authService.authenticate()
    │   ├─ 1. 认证身份 (validate)
    │   │   └─ 按优先级判定凭证类型 → 解析对应信息
    │   ├─ 2. 管理员路由检查 (adminRoute)
    │   ├─ 3. 共享链接路由检查 (sharedLinkRoute)
    │   └─ 4. API Key Scope 权限检查 (isGranted)
    └─ 注入 AuthDto 到 Request
    ↓
[Controller] 路由处理器
    ↓
[Service] 业务逻辑
    └─ 资源级访问控制 (requireAccess / checkAccess)
        ├─ sharedLink 分支: checkSharedLinkAccess
        └─ 其他分支: checkOtherAccess
    ↓
返回结果 / 抛出异常
```

---

## 二、多凭证并存时认证优先级

### 2.1 凭证判定顺序与短路机制

**核心代码**: `server/src/services/auth.service.ts:244-271`

```typescript
private async validate({ headers, queryParams }: Omit<ValidateRequest, 'metadata'>): Promise<AuthDto> {
  const shareKey = (headers[ImmichHeader.SharedLinkKey] || queryParams[ImmichQuery.SharedLinkKey]) as string;
  const shareSlug = (headers[ImmichHeader.SharedLinkSlug] || queryParams[ImmichQuery.SharedLinkSlug]) as string;
  const session = (headers[ImmichHeader.UserToken] ||
    headers[ImmichHeader.SessionToken] ||
    queryParams[ImmichQuery.SessionKey] ||
    this.getBearerToken(headers) ||
    this.getCookieToken(headers)) as string;
  const apiKey = (headers[ImmichHeader.ApiKey] || queryParams[ImmichQuery.ApiKey]) as string;

  if (shareKey) {
    return this.validateSharedLinkKey(shareKey);    // 优先级1: 共享链接 Key
  }
  if (shareSlug) {
    return this.validateSharedLinkSlug(shareSlug);  // 优先级2: 共享链接 Slug
  }
  if (session) {
    return this.validateSession(session, headers);  // 优先级3: Session
  }
  if (apiKey) {
    return this.validateApiKey(apiKey);             // 优先级4: API Key
  }

  throw new UnauthorizedException('Authentication required');
}
```

### 2.2 优先级与短路关系详解

| 优先级 | 凭证类型 | 提取来源 | 短路条件 | 对 API Key Scope 检查的影响 |
|-------|---------|---------|---------|---------------------------|
| **1** | `shareKey` | header `x-immich-share-key` / query `shareKey` | 只要存在就走此分支 | ❌ API Key 完全被绕过，不会触发 Scope 检查 |
| **2** | `shareSlug` | header `x-immich-share-slug` / query `shareSlug` | 只要存在且 shareKey 不存在就走此分支 | ❌ API Key 完全被绕过，不会触发 Scope 检查 |
| **3** | `session` | header `x-immich-user-token`/`x-immich-session-token`/`authorization: Bearer <token>`/Cookie `immich_access_token`/query `sessionKey` | 只要存在且 shareKey/shareSlug 不存在就走此分支 | ❌ API Key 完全被绕过，不会触发 Scope 检查 |
| **4** | `apiKey` | header `x-api-key` / query `apiKey` | 上述三者都不存在时才走此分支 | ✅ 触发 API Key Scope 检查 |

**关键结论**:
1. **优先级是固定的**：shareKey > shareSlug > session > apiKey，高优先级凭证存在时直接短路返回
2. **API Key Scope 检查仅在第4层触发**：只要前面任何凭证存在，无论 API Key 是否提供，都不会走 API Key 分支
3. **多凭证并存的隐患**：如果请求同时携带 session cookie 和 API Key，实际认证的是 session，API Key 被完全忽略

### 2.3 不同凭证类型的 AuthDto 结构差异

| 凭证类型 | `authDto.user` | `authDto.apiKey` | `authDto.session` | `authDto.sharedLink` |
|---------|---------------|------------------|-------------------|---------------------|
| shareKey | ✅ 存在 | ❌ 不存在 | ❌ 不存在 | ✅ 存在 |
| shareSlug | ✅ 存在 | ❌ 不存在 | ❌ 不存在 | ✅ 存在 |
| session | ✅ 存在 | ❌ 不存在 | ✅ 存在 | ❌ 不存在 |
| apiKey | ✅ 存在 | ✅ 存在 (含 permissions) | ❌ 不存在 | ❌ 不存在 |

**重要**: 只有 `authDto.apiKey` 存在时，才会进入 Scope 校验分支。

---

## 三、Scope 解析阶段

### 3.1 API Key 认证流程

**文件**: `server/src/services/auth.service.ts:516-527`

```typescript
private async validateApiKey(key: string): Promise<AuthDto> {
  const hashed = this.cryptoRepository.hashSha256(key);
  const apiKey = await this.apiKeyRepository.getKey(hashed);
  if (apiKey?.user) {
    return {
      user: apiKey.user,
      apiKey,  // 包含 permissions 字段
    };
  }
  throw new UnauthorizedException('Invalid API key');
}
```

**解析逻辑**:
1. 从请求头 `x-api-key` 或 query 参数 `apiKey` 获取原始 key
2. 使用 SHA256 哈希后与数据库中存储的哈希比对
3. 验证通过后返回包含 `apiKey.permissions` 的 `AuthDto` 对象

### 3.2 数据结构定义

**DTO 定义**: `server/src/dtos/api-key.dto.ts`

```typescript
const ApiKeyCreateSchema = z.object({
  name: z.string().optional(),
  permissions: z.array(PermissionSchema).min(1),  // 至少一个权限
});
```

**Permission 枚举**: `server/src/enum.ts:104-203`

采用 `领域.操作` 的命名规范，例如：
- `asset.read` - 读取资产
- `asset.delete` - 删除资产
- `album.create` - 创建相册
- `all` - 全部权限（特殊值）

---

## 四、路由限制阶段

### 4.1 @Authenticated 装饰器

**文件**: `server/src/middleware/auth.guard.ts:22-46`

```typescript
export const Authenticated = (options: AuthenticatedOptions = {}): MethodDecorator => {
  return applyDecorators(
    ApiBearerAuth(),
    ApiCookieAuth(),
    ApiSecurity(MetadataKey.ApiKeySecurity),
    SetMetadata(MetadataKey.AuthRoute, options),  // 存储路由权限元数据
    // ... 其他装饰器
  );
};
```

### 4.2 路由权限声明的三种模式

**模式1: 显式指定权限** (最常用)
```typescript
@Post()
@Authenticated({ permission: Permission.ApiKeyCreate })  // 需要 apiKey.create 权限
createApiKey(@Auth() auth: AuthDto, @Body() dto: ApiKeyCreateDto) {
  return this.service.create(auth, dto);
}
```

**模式2: permission=false (仅跳过 Scope 检查)**
```typescript
@Get('me')
@Authenticated({ permission: false })  // 仅跳过 Scope 校验，adminRoute/sharedLinkRoute 仍检查
getMyApiKey(@Auth() auth: AuthDto) {
  return this.service.getMine(auth);
}
```

**模式3: 未指定 permission (默认 = Permission.All)**
```typescript
@Get('some-endpoint')
@Authenticated({ admin: true })  // 未显式指定 permission，默认要求 Permission.All
someEndpoint(@Auth() auth: AuthDto) {
  // 默认要求 Permission.All
}
```

### 4.3 permission=false 行为修正

> **重要修正**: `permission=false` **只跳过 Scope 校验**，不会跳过 `adminRoute` 与 `sharedLinkRoute` 的拦截。

**核心校验流程**: `server/src/services/auth.service.ts:195-222`

```typescript
async authenticate({ headers, queryParams, metadata }: ValidateRequest): Promise<AuthDto> {
  const authDto = await this.validate({ headers, queryParams });  // 第一步: 身份认证
  const { adminRoute, sharedLinkRoute, uri } = metadata;
  const requestedPermission = metadata.permission ?? Permission.All;

  // 检查1: 管理员路由限制 - 无论 permission 是否为 false 都会执行
  if (!authDto.user.isAdmin && adminRoute) {
    this.logger.warn(`Denied access to admin only route: ${uri}`);
    throw new ForbiddenException('Forbidden');
  }

  // 检查2: 共享链接路由限制 - 无论 permission 是否为 false 都会执行
  if (authDto.sharedLink && !sharedLinkRoute) {
    this.logger.warn(`Denied access to non-shared route: ${uri}`);
    throw new ForbiddenException('Forbidden');
  }

  // 检查3: API Key Scope 权限校验 - 仅当 permission !== false 时执行
  if (
    authDto.apiKey &&
    requestedPermission !== false &&        // permission=false 时短路跳过此检查
    !isGranted({ requested: [requestedPermission], current: authDto.apiKey.permissions })
  ) {
    throw new ForbiddenException(`Missing required permission: ${requestedPermission}`);
  }

  return authDto;
}
```

**permission=false 校验流程图**:
```
身份认证通过
    ↓
adminRoute 检查? → 不通过 → 403 Forbidden
    ↓ 通过
sharedLinkRoute 检查? → 不通过 → 403 Forbidden
    ↓ 通过
permission === false? → 是 → 跳过 Scope 检查，放行
    ↓ 否
Scope 校验 → 不通过 → 403 Forbidden
    ↓ 通过
放行
```

### 4.4 permission=false vs permission=all 行为差异

**差异对比表**:

| 维度 | `permission=false` | `permission=Permission.All` | 未指定 (默认 `= Permission.All`) |
|-----|-------------------|----------------------------|---------------------------------|
| `requestedPermission` 值 | `false` | `Permission.All` (`'all'`) | `Permission.All` (`'all'`) |
| 是否检查 adminRoute | ✅ 检查 | ✅ 检查 | ✅ 检查 |
| 是否检查 sharedLinkRoute | ✅ 检查 | ✅ 检查 | ✅ 检查 |
| 是否触发 Scope 校验 | ❌ 不触发 | ✅ 触发 | ✅ 触发 |
| 短路条件 | `requestedPermission !== false` → `false`，跳过 `isGranted` | 执行 `isGranted` 检查 | 执行 `isGranted` 检查 |
| `isGranted` 判定逻辑 | - | 检查 `current` 是否包含 `'all'` | 检查 `current` 是否包含 `'all'` |
| 放行条件 (API Key) | 身份认证通过 + adminRoute 通过 + sharedLinkRoute 通过 | 身份认证通过 + adminRoute 通过 + sharedLinkRoute 通过 + API Key 含 `all` | 身份认证通过 + adminRoute 通过 + sharedLinkRoute 通过 + API Key 含 `all` |
| 拒绝返回 (Scope) | - | `403 Forbidden: Missing required permission: all` | `403 Forbidden: Missing required permission: all` |
| 拒绝返回 (adminRoute) | `403 Forbidden: Forbidden` | `403 Forbidden: Forbidden` | `403 Forbidden: Forbidden` |
| 拒绝返回 (sharedLinkRoute) | `403 Forbidden: Forbidden` | `403 Forbidden: Forbidden` | `403 Forbidden: Forbidden` |
| 适用场景 | 获取当前 API Key 自身信息等无需权限的操作 | 管理员操作、高风险操作 | 未显式声明权限的路由 |

**代码佐证**: `server/src/services/auth.service.ts:198`
```typescript
const requestedPermission = metadata.permission ?? Permission.All;
```
当 `metadata.permission` 为 `undefined` 时，默认赋值为 `Permission.All`。

### 4.5 AuthGuard 守卫拦截

**文件**: `server/src/middleware/auth.guard.ts:88-109`

```typescript
async canActivate(context: ExecutionContext): Promise<boolean> {
  const targets = [context.getHandler()];
  const options = this.reflector.getAllAndOverride<AuthenticatedOptions | undefined>(
    MetadataKey.AuthRoute, 
    targets
  );
  if (!options) {
    return true;  // 无权限要求, 直接放行
  }

  const request = context.switchToHttp().getRequest<AuthRequest>();
  request.user = await this.authService.authenticate({
    headers: request.headers,
    queryParams: request.query as Record<string, string>,
    metadata: { adminRoute, sharedLinkRoute, permission, uri: request.path },
  });

  return true;
}
```

---

## 五、权限校验阶段

### 5.1 authenticate 核心校验

**文件**: `server/src/services/auth.service.ts:195-222`

```typescript
async authenticate({ headers, queryParams, metadata }: ValidateRequest): Promise<AuthDto> {
  const authDto = await this.validate({ headers, queryParams });  // 第一步: 身份认证
  const { adminRoute, sharedLinkRoute, uri } = metadata;
  const requestedPermission = metadata.permission ?? Permission.All;

  // 检查1: 管理员路由限制
  if (!authDto.user.isAdmin && adminRoute) {
    this.logger.warn(`Denied access to admin only route: ${uri}`);
    throw new ForbiddenException('Forbidden');
  }

  // 检查2: 共享链接路由限制
  if (authDto.sharedLink && !sharedLinkRoute) {
    this.logger.warn(`Denied access to non-shared route: ${uri}`);
    throw new ForbiddenException('Forbidden');
  }

  // 检查3: API Key Scope 权限校验 (核心)
  if (
    authDto.apiKey &&
    requestedPermission !== false &&
    !isGranted({ requested: [requestedPermission], current: authDto.apiKey.permissions })
  ) {
    throw new ForbiddenException(`Missing required permission: ${requestedPermission}`);
  }

  return authDto;
}
```

### 5.2 isGranted 权限判定算法

**文件**: `server/src/utils/access.ts:8-19`

```typescript
export type GrantedRequest = {
  requested: Permission[];  // 路由要求的权限
  current: Permission[];    // API Key 实际拥有的权限
};

export const isGranted = ({ requested, current }: GrantedRequest) => {
  if (current.includes(Permission.All)) {
    return true;  // 拥有 all 权限直接放行
  }
  return setIsSuperset(new Set(current), new Set(requested));  // 集合超集判断
};
```

**判定逻辑**:
1. 若 API Key 包含 `Permission.All`，直接返回 `true`
2. 否则检查 `current` 权限集合是否是 `requested` 集合的超集
3. 即：API Key 必须拥有路由要求的**全部**权限

---

## 六、资源访问判定阶段

路由级权限检查通过后，业务逻辑层还会进行**资源级**的访问控制，确保用户只能访问自己有权限的具体资源。

### 6.1 requireAccess 强制检查

**文件**: `server/src/utils/access.ts:37-42`

```typescript
export const requireAccess = async (access: AccessRepository, request: AccessRequest) => {
  const allowedIds = await checkAccess(access, request);
  if (!setIsEqual(new Set(request.ids), allowedIds)) {
    throw new BadRequestException(`Not found or no ${request.permission} access`);
  }
};
```

### 6.2 checkAccess 资源访问检查

**文件**: `server/src/utils/access.ts:44-56`

```typescript
export const checkAccess = async (
  access: AccessRepository,
  { ids, auth, permission }: AccessRequest,
): Promise<Set<string>> => {
  const idSet = Array.isArray(ids) ? new Set(ids) : ids;
  if (idSet.size === 0) {
    return new Set<string>();
  }

  return auth.sharedLink
    ? checkSharedLinkAccess(access, { sharedLink: auth.sharedLink, permission, ids: idSet })
    : checkOtherAccess(access, { auth, permission, ids: idSet });
};
```

### 6.3 Shared Link 分支决策表

当 `auth.sharedLink` 存在时，进入共享链接访问判定分支。共享链接有两个关键开关：
- `allowDownload`: 是否允许下载
- `allowUpload`: 是否允许上传

**核心代码**: `server/src/utils/access.ts:58-98`

```typescript
const checkSharedLinkAccess = async (
  access: AccessRepository,
  request: SharedLinkAccessRequest,
): Promise<Set<string>> => {
  const { sharedLink, permission, ids } = request;
  const sharedLinkId = sharedLink.id;

  switch (permission) {
    case Permission.AssetRead: {
      return await access.asset.checkSharedLinkAccess(sharedLinkId, ids);
    }
    case Permission.AssetDownload: {
      return sharedLink.allowDownload ? await access.asset.checkSharedLinkAccess(sharedLinkId, ids) : new Set();
    }
    case Permission.AssetUpload: {
      return sharedLink.allowUpload ? ids : new Set();
    }
    // ... 其他权限
  }
};
```

**Shared Link 资源访问决策表**:

| 请求权限 | `allowDownload` | `allowUpload` | 判定逻辑 | 结果 | HTTP 状态 |
|---------|----------------|--------------|---------|------|----------|
| **asset.read** | 任意 | 任意 | 直接调用 `checkSharedLinkAccess` 检查资源是否在共享链接中 | ✅ 资源在共享链接中 → 放行<br>❌ 资源不在 → 返回空 Set | `400 Bad Request` |
| **asset.view** | 任意 | 任意 | 直接调用 `checkSharedLinkAccess` 检查资源是否在共享链接中 | ✅ 资源在共享链接中 → 放行<br>❌ 资源不在 → 返回空 Set | `400 Bad Request` |
| **asset.download** | `true` | 任意 | 调用 `checkSharedLinkAccess` 检查资源是否在共享链接中 | ✅ 资源在共享链接中 → 放行<br>❌ 资源不在 → 返回空 Set | `400 Bad Request` |
| **asset.download** | `false` | 任意 | 直接返回空 Set | ❌ 全部拒绝 | `400 Bad Request` |
| **asset.upload** | 任意 | `true` | 直接返回 `ids` (不检查共享链接资源) | ✅ 全部放行 | - |
| **asset.upload** | 任意 | `false` | 直接返回空 Set | ❌ 全部拒绝 | `400 Bad Request` |
| **album.read** | 任意 | 任意 | 调用 `checkSharedLinkAccess` 检查相册是否在共享链接中 | ✅ 相册在共享链接中 → 放行<br>❌ 相册不在 → 返回空 Set | `400 Bad Request` |
| **album.download** | `true` | 任意 | 调用 `checkSharedLinkAccess` 检查相册是否在共享链接中 | ✅ 相册在共享链接中 → 放行<br>❌ 相册不在 → 返回空 Set | `400 Bad Request` |
| **album.download** | `false` | 任意 | 直接返回空 Set | ❌ 全部拒绝 | `400 Bad Request` |
| **albumAsset.create** | 任意 | `true` | 调用 `checkSharedLinkAccess` 检查相册是否在共享链接中 | ✅ 相册在共享链接中 → 放行<br>❌ 相册不在 → 返回空 Set | `400 Bad Request` |
| **albumAsset.create** | 任意 | `false` | 直接返回空 Set | ❌ 全部拒绝 | `400 Bad Request` |
| **其他权限** | 任意 | 任意 | 直接返回空 Set | ❌ 全部拒绝 | `400 Bad Request` |

**决策表说明**:
1. `asset.read` 和 `asset.view` 不受 `allowDownload`/`allowUpload` 影响，只要资源在共享链接中即可访问
2. `asset.download` 和 `album.download` 依赖 `allowDownload: true` 开关
3. `asset.upload` 不检查资源是否在共享链接中，只看 `allowUpload` 开关（因为是上传新资源）
4. `albumAsset.create` 需要同时满足 `allowUpload: true` 且相册在共享链接中
5. 未在 switch 中列出的权限（如 `asset.delete`、`asset.update` 等）共享链接一概拒绝

### 6.4 非 Shared Link 资源访问判定示例 (AssetRead)

**文件**: `server/src/utils/access.ts:116-121`

```typescript
case Permission.AssetRead: {
  const isOwner = await access.asset.checkOwnerAccess(auth.user.id, ids, auth.session?.hasElevatedPermission);
  const isAlbum = await access.asset.checkAlbumAccess(auth.user.id, setDifference(ids, isOwner));
  const isPartner = await access.asset.checkPartnerAccess(auth.user.id, setDifference(ids, isOwner, isAlbum));
  return setUnion(isOwner, isAlbum, isPartner);  // 合并所有有权限的资源ID
}
```

**资产可读判定逻辑**:
1. **所有者访问**: 用户自己上传的资产
2. **相册共享**: 资产所在相册被共享给用户
3. **伙伴共享**: 用户的伙伴共享的资产
4. 最终返回三者的并集

---

## 七、凭证类型 × 路由元数据 × 最终拒绝状态矩阵

### 7.1 矩阵说明

本矩阵覆盖三种凭证类型（session、apiKey、sharedLink）在不同路由元数据配置下的拒绝返回情况。

**符号说明**:
- ✅ 放行
- ❌ 401 Unauthorized (身份认证失败)
- 🚫 403 Forbidden (权限不足)
- ⚠️ 400 Bad Request (资源级访问失败)
- N/A 不适用

### 7.2 完整拒绝状态矩阵

| 凭证类型 | 路由元数据配置 | 条件 | 身份认证 | adminRoute 检查 | sharedLinkRoute 检查 | Scope 检查 | 资源级检查 | 最终结果 | HTTP 状态码 |
|---------|---------------|------|---------|----------------|---------------------|-----------|-----------|---------|------------|
| **session** | `permission=false` | 非管理员访问 admin=true | ✅ 通过 | ❌ 不通过 | N/A (无 sharedLink) | N/A (permission=false) | 不涉及 | 🚫 拒绝 | 403 |
| **session** | `permission=false` | 非管理员访问 admin=false | ✅ 通过 | ✅ 通过 | N/A | N/A | 正常执行 | ✅ 放行 / ⚠️ 可能拒绝 | 200 / 400 |
| **session** | `permission=all` | 非管理员访问 admin=true | ✅ 通过 | ❌ 不通过 | N/A | N/A | 不涉及 | 🚫 拒绝 | 403 |
| **session** | `permission=all` | 非管理员访问 admin=false | ✅ 通过 | ✅ 通过 | N/A | N/A (session 无 Scope) | 正常执行 | ✅ 放行 / ⚠️ 可能拒绝 | 200 / 400 |
| **session** | `permission=asset.read` | 非管理员访问 admin=false | ✅ 通过 | ✅ 通过 | N/A | N/A (session 无 Scope) | 正常执行 | ✅ 放行 / ⚠️ 可能拒绝 | 200 / 400 |
| **session** | `sharedLink=true` | session 访问共享路由 | ✅ 通过 | ✅ 通过 | N/A (无 sharedLink) | N/A | 正常执行 | ✅ 放行 / ⚠️ 可能拒绝 | 200 / 400 |
| **session** | `sharedLink=false` | session 访问非共享路由 | ✅ 通过 | ✅ 通过 | N/A | N/A | 正常执行 | ✅ 放行 / ⚠️ 可能拒绝 | 200 / 400 |
| **apiKey** | `permission=false` | 非管理员访问 admin=true | ✅ 通过 | ❌ 不通过 | N/A | N/A | 不涉及 | 🚫 拒绝 | 403 |
| **apiKey** | `permission=false` | 非管理员访问 admin=false | ✅ 通过 | ✅ 通过 | N/A | N/A (permission=false) | 正常执行 | ✅ 放行 / ⚠️ 可能拒绝 | 200 / 400 |
| **apiKey** | `permission=all` | API Key 不含 all 权限 | ✅ 通过 | ✅ 通过 | N/A | ❌ 不通过 | 不涉及 | 🚫 拒绝 | 403 |
| **apiKey** | `permission=all` | API Key 含 all 权限 | ✅ 通过 | ✅ 通过 | N/A | ✅ 通过 | 正常执行 | ✅ 放行 / ⚠️ 可能拒绝 | 200 / 400 |
| **apiKey** | `permission=asset.delete` | API Key 不含 asset.delete | ✅ 通过 | ✅ 通过 | N/A | ❌ 不通过 | 不涉及 | 🚫 拒绝 | 403 |
| **apiKey** | `permission=asset.delete` | API Key 含 asset.delete | ✅ 通过 | ✅ 通过 | N/A | ✅ 通过 | 正常执行 | ✅ 放行 / ⚠️ 可能拒绝 | 200 / 400 |
| **apiKey** | `sharedLink=true` | API Key 访问共享路由 | ✅ 通过 | ✅ 通过 | N/A (无 sharedLink) | ✅ 通过 | 正常执行 | ✅ 放行 / ⚠️ 可能拒绝 | 200 / 400 |
| **apiKey** | `sharedLink=false` | API Key 访问非共享路由 | ✅ 通过 | ✅ 通过 | N/A | ✅ 通过 | 正常执行 | ✅ 放行 / ⚠️ 可能拒绝 | 200 / 400 |
| **sharedLink** | `permission=false` | sharedLinkRoute=false | ✅ 通过 | ✅ 通过 | ❌ 不通过 | N/A | 不涉及 | 🚫 拒绝 | 403 |
| **sharedLink** | `permission=false` | sharedLinkRoute=true | ✅ 通过 | ✅ 通过 | ✅ 通过 | N/A (无 apiKey) | 正常执行 | ✅ 放行 / ⚠️ 可能拒绝 | 200 / 400 |
| **sharedLink** | `permission=asset.read` | sharedLinkRoute=false | ✅ 通过 | ✅ 通过 | ❌ 不通过 | N/A | 不涉及 | 🚫 拒绝 | 403 |
| **sharedLink** | `permission=asset.read` | sharedLinkRoute=true, 资源在链接中 | ✅ 通过 | ✅ 通过 | ✅ 通过 | N/A | ✅ 通过 | ✅ 放行 | 200 |
| **sharedLink** | `permission=asset.read` | sharedLinkRoute=true, 资源不在链接中 | ✅ 通过 | ✅ 通过 | ✅ 通过 | N/A | ❌ 不通过 | ⚠️ 拒绝 | 400 |
| **sharedLink** | `permission=asset.download` | allowDownload=false | ✅ 通过 | ✅ 通过 | ✅ 通过 | N/A | ❌ 不通过 | ⚠️ 拒绝 | 400 |
| **sharedLink** | `permission=asset.download` | allowDownload=true, 资源在链接中 | ✅ 通过 | ✅ 通过 | ✅ 通过 | N/A | ✅ 通过 | ✅ 放行 | 200 |
| **sharedLink** | `permission=asset.upload` | allowUpload=false | ✅ 通过 | ✅ 通过 | ✅ 通过 | N/A | ❌ 不通过 | ⚠️ 拒绝 | 400 |
| **sharedLink** | `permission=asset.upload` | allowUpload=true | ✅ 通过 | ✅ 通过 | ✅ 通过 | N/A | ✅ 通过 | ✅ 放行 | 200 |
| **sharedLink** | `permission=asset.delete` | sharedLinkRoute=true | ✅ 通过 | ✅ 通过 | ✅ 通过 | N/A | ❌ 不通过 | ⚠️ 拒绝 | 400 |
| **任意** | 任意 | 未提供任何凭证 | ❌ 不通过 | N/A | N/A | N/A | N/A | ❌ 拒绝 | 401 |
| **apiKey** | 任意 | API Key 哈希不匹配 | ❌ 不通过 | N/A | N/A | N/A | N/A | ❌ 拒绝 | 401 |
| **sharedLink** | 任意 | shareKey 无效 | ❌ 不通过 | N/A | N/A | N/A | N/A | ❌ 拒绝 | 401 |
| **session** | 任意 | session token 无效 | ❌ 不通过 | N/A | N/A | N/A | N/A | ❌ 拒绝 | 401 |

### 7.3 矩阵关键结论

1. **Session 无 Scope 检查**: session 认证的请求不会触发 Scope 校验，因为 `authDto.apiKey` 不存在
2. **Shared Link 无 Scope 检查**: sharedLink 认证的请求也不会触发 Scope 校验
3. **adminRoute 优先级最高**: 无论 permission 如何配置，adminRoute 检查始终先执行
4. **sharedLinkRoute 是 Shared Link 的第一道关卡**: Shared Link 访问非共享路由直接 403
5. **资源级检查是最后一道防线**: 即使所有路由级检查通过，资源级检查仍可能返回 400
6. **401 只发生在身份认证阶段**: 凭证无效或缺失时才返回 401

---

## 八、拒绝返回阶段

### 8.1 异常类型与场景

| 异常类型 | HTTP 状态码 | 触发场景 |
|---------|------------|----------|
| `UnauthorizedException` | 401 | API Key 无效、未提供认证信息、session 无效、shareKey 无效 |
| `ForbiddenException` | 403 | 权限不足（Scope 不匹配、非管理员访问管理员路由、共享链接访问非共享路由） |
| `BadRequestException` | 400 | 资源不存在或无访问权限（资源级检查失败、allowDownload=false、allowUpload=false） |

### 8.2 拒绝返回示例

**场景1: API Key 无效**
```
Status: 401 Unauthorized
{
  "message": "Invalid API key",
  "error": "Unauthorized",
  "statusCode": 401
}
```

**场景2: Scope 权限不足 (asset.delete)**
```
Status: 403 Forbidden
{
  "message": "Missing required permission: asset.delete",
  "error": "Forbidden",
  "statusCode": 403
}
```

**场景3: 默认 permission=all 被拒绝**
```
Status: 403 Forbidden
{
  "message": "Missing required permission: all",
  "error": "Forbidden",
  "statusCode": 403
}
```

**场景4: permission=false 但非管理员访问 admin 路由**
```
Status: 403 Forbidden
{
  "message": "Forbidden",
  "error": "Forbidden",
  "statusCode": 403
}
```

**场景5: Shared Link 访问非共享路由**
```
Status: 403 Forbidden
{
  "message": "Forbidden",
  "error": "Forbidden",
  "statusCode": 403
}
```

**场景6: 资源级访问被拒**
```
Status: 400 Bad Request
{
  "message": "Not found or no asset.read access",
  "error": "Bad Request",
  "statusCode": 400
}
```

**场景7: Shared Link allowDownload=false 时下载被拒**
```
Status: 400 Bad Request
{
  "message": "Not found or no asset.download access",
  "error": "Bad Request",
  "statusCode": 400
}
```

---

## 九、API Key 自限制机制

### 9.1 创建时的权限限制

**文件**: `server/src/services/api-key.service.ts:15-17`

```typescript
if (auth.apiKey && !isGranted({ requested: dto.permissions, current: auth.apiKey.permissions })) {
  throw new BadRequestException('Cannot grant permissions you do not have');
}
```

**安全特性**: 使用 API Key 创建新的 API Key 时，新 Key 的权限不能超过当前 Key 的权限，防止权限逃逸。

### 9.2 更新时的权限限制

**文件**: `server/src/services/api-key.service.ts:35-41`

```typescript
if (
  auth.apiKey &&
  dto.permissions &&
  !isGranted({ requested: dto.permissions, current: auth.apiKey.permissions })
) {
  throw new BadRequestException('Cannot grant permissions you do not have');
}
```

---

## 十、关键设计总结

### 10.1 安全设计亮点

1. **分层防御**: 路由级 Scope 检查 + 资源级所有权检查，形成纵深防御
2. **最小权限原则**: 默认不授予权限，需显式声明
3. **权限不可提升**: API Key 创建子 Key 时无法超越自身权限
4. **哈希存储**: API Key 采用 SHA256 哈希存储，泄露后无法还原
5. **细粒度控制**: 支持 50+ 种细分权限，可精确控制 API 访问范围
6. **灵活的权限声明**: 支持 `permission=false` 跳过 Scope 检查，适用于特殊场景
7. **凭证优先级固定**: 避免凭证优先级不确定导致的安全漏洞

### 10.2 核心数据流向

```
请求 → [AuthGuard] 读取路由权限元数据
     → [authService.validate] 按优先级判定凭证类型
     → [authService.authenticate] adminRoute → sharedLinkRoute → Scope 检查
     → [Controller] 执行业务逻辑
     → [checkAccess] 资源级访问控制 (sharedLink 分支 / 其他分支)
     → 返回结果 / 抛出异常
```

### 10.3 关键组件关系

| 组件 | 职责 | 核心文件 |
|-----|-----|---------|
| `@Authenticated` | 声明路由权限要求 | `auth.guard.ts:22-46` |
| `AuthGuard` | 拦截请求，触发认证 | `auth.guard.ts:78-109` |
| `AuthService.validate` | 按优先级判定凭证类型 | `auth.service.ts:244-271` |
| `AuthService.validateApiKey` | 解析 API Key 及 Scope | `auth.service.ts:516-527` |
| `AuthService.authenticate` | 执行四层权限检查 | `auth.service.ts:195-222` |
| `isGranted` | Scope 权限判定算法 | `access.ts:13-19` |
| `checkSharedLinkAccess` | Shared Link 资源访问控制 | `access.ts:58-98` |
| `checkOtherAccess` | 非 Shared Link 资源访问控制 | `access.ts:100-335` |
| `checkAccess` | 资源级访问控制入口 | `access.ts:44-56` |

### 10.4 权限检查失败速查表

| 阶段 | 检查点 | 失败条件 | 异常类型 | HTTP 状态码 |
|-----|-------|---------|---------|------------|
| 身份认证 | 凭证有效性 | 未提供凭证或凭证无效 | `UnauthorizedException` | 401 |
| 路由检查 | 管理员路由 | 非管理员访问 `admin: true` 路由 | `ForbiddenException` | 403 |
| 路由检查 | 共享链接路由 | Shared Link 访问非 `sharedLink: true` 路由 | `ForbiddenException` | 403 |
| Scope 检查 | permission=all | API Key 不含 `all` 权限 | `ForbiddenException` | 403 |
| Scope 检查 | permission=xxx | API Key 不含 `xxx` 权限 | `ForbiddenException` | 403 |
| 资源检查 | Shared Link download | `allowDownload=false` | `BadRequestException` | 400 |
| 资源检查 | Shared Link upload | `allowUpload=false` | `BadRequestException` | 400 |
| 资源检查 | 资源所有权 | 资源不属于用户且未被共享 | `BadRequestException` | 400 |
