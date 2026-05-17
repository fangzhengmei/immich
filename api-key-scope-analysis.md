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
    │   │   └─ validateApiKey() → 解析 API Key 及 Scope
    │   ├─ 2. 管理员路由检查
    │   ├─ 3. 共享链接路由检查
    │   └─ 4. API Key Scope 权限检查 (isGranted)
    └─ 注入 AuthDto 到 Request
    ↓
[Controller] 路由处理器
    ↓
[Service] 业务逻辑
    └─ 资源级访问控制 (requireAccess / checkAccess)
        └─ 检查具体资源所有权/共享关系
    ↓
返回结果 / 抛出异常
```

---

## 二、Scope 解析阶段

### 2.1 API Key 认证流程

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

### 2.2 数据结构定义

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

## 三、路由限制阶段

### 3.1 @Authenticated 装饰器

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

### 3.2 路由权限声明示例

**文件**: `server/src/controllers/api-key.controller.ts`

```typescript
@Post()
@Authenticated({ permission: Permission.ApiKeyCreate })  // 需要 apiKey.create 权限
createApiKey(@Auth() auth: AuthDto, @Body() dto: ApiKeyCreateDto) {
  return this.service.create(auth, dto);
}

@Get('me')
@Authenticated({ permission: false })  // 特殊值: 不需要权限检查
getMyApiKey(@Auth() auth: AuthDto) {
  return this.service.getMine(auth);
}
```

**文件**: `server/src/controllers/asset.controller.ts`

```typescript
@Get(':id')
@Authenticated({ permission: Permission.AssetRead, sharedLink: true })
getAssetInfo(@Auth() auth: AuthDto, @Param() { id }: UUIDParamDto) {
  return this.service.get(auth, id);
}

@Delete()
@Authenticated({ permission: Permission.AssetDelete })
deleteAssets(@Auth() auth: AuthDto, @Body() dto: AssetBulkDeleteDto) {
  return this.service.deleteAll(auth, dto);
}
```

### 3.3 AuthGuard 守卫拦截

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

## 四、权限校验阶段

### 4.1 authenticate 核心校验

**文件**: `server/src/services/auth.service.ts:218-242`

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

### 4.2 isGranted 权限判定算法

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

## 五、资源访问判定阶段

路由级权限检查通过后，业务逻辑层还会进行**资源级**的访问控制，确保用户只能访问自己有权限的具体资源。

### 5.1 requireAccess 强制检查

**文件**: `server/src/utils/access.ts:37-42`

```typescript
export const requireAccess = async (access: AccessRepository, request: AccessRequest) => {
  const allowedIds = await checkAccess(access, request);
  if (!setIsEqual(new Set(request.ids), allowedIds)) {
    throw new BadRequestException(`Not found or no ${request.permission} access`);
  }
};
```

### 5.2 checkAccess 资源访问检查

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

### 5.3 资源访问判定示例 (AssetRead)

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

## 六、拒绝返回阶段

### 6.1 异常类型与场景

| 异常类型 | HTTP 状态码 | 触发场景 |
|---------|------------|----------|
| `UnauthorizedException` | 401 | API Key 无效、未提供认证信息 |
| `ForbiddenException` | 403 | 权限不足（Scope 不匹配、非管理员访问管理员路由） |
| `BadRequestException` | 400 | 资源不存在或无访问权限（资源级检查失败） |

### 6.2 拒绝返回示例

**场景1: API Key 无效**
```
Status: 401 Unauthorized
{
  "message": "Invalid API key",
  "error": "Unauthorized",
  "statusCode": 401
}
```

**场景2: Scope 权限不足**
```
Status: 403 Forbidden
{
  "message": "Missing required permission: asset.delete",
  "error": "Forbidden",
  "statusCode": 403
}
```

**场景3: 资源级访问被拒**
```
Status: 400 Bad Request
{
  "message": "Not found or no asset.read access",
  "error": "Bad Request",
  "statusCode": 400
}
```

---

## 七、API Key 自限制机制

### 7.1 创建时的权限限制

**文件**: `server/src/services/api-key.service.ts:15-17`

```typescript
if (auth.apiKey && !isGranted({ requested: dto.permissions, current: auth.apiKey.permissions })) {
  throw new BadRequestException('Cannot grant permissions you do not have');
}
```

**安全特性**: 使用 API Key 创建新的 API Key 时，新 Key 的权限不能超过当前 Key 的权限，防止权限逃逸。

### 7.2 更新时的权限限制

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

## 八、关键设计总结

### 8.1 安全设计亮点

1. **分层防御**: 路由级 Scope 检查 + 资源级所有权检查，形成纵深防御
2. **最小权限原则**: 默认不授予权限，需显式声明
3. **权限不可提升**: API Key 创建子 Key 时无法超越自身权限
4. **哈希存储**: API Key 采用 SHA256 哈希存储，泄露后无法还原
5. **细粒度控制**: 支持 50+ 种细分权限，可精确控制 API 访问范围

### 8.2 核心数据流向

```
请求 → [AuthGuard] 读取路由权限元数据
     → [authService.validateApiKey] 解析 Key 及 Scope
     → [authService.authenticate] Scope 与路由要求比对
     → [Controller] 执行业务逻辑
     → [checkAccess] 资源级访问控制
     → 返回结果 / 抛出异常
```

### 8.3 关键组件关系

| 组件 | 职责 | 核心文件 |
|-----|-----|---------|
| `@Authenticated` | 声明路由权限要求 | `auth.guard.ts:22-46` |
| `AuthGuard` | 拦截请求，触发认证 | `auth.guard.ts:78-109` |
| `AuthService.validateApiKey` | 解析 API Key 及 Scope | `auth.service.ts:516-527` |
| `AuthService.authenticate` | 执行四层权限检查 | `auth.service.ts:218-242` |
| `isGranted` | Scope 权限判定算法 | `access.ts:13-19` |
| `checkAccess` | 资源级访问控制 | `access.ts:44-335` |
