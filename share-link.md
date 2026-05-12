# Immich 分享链接权限模型分析报告

## 一、权限模型设计

### 1.1 三种分享渠道

Immich 照片分享系统设计了三种分享渠道，共用同一套权限模型：

| 渠道类型 | 描述 | 实现机制 |
|---------|------|---------|
| **同事相册共享** | 将团队成员加入相册，共同查看和管理 | `album_user` 表 + `AlbumUserRole` 角色 |
| **带密码分享链接** | 生成带访问密码的链接，指定人员凭密码访问 | `shared_link` 表 + `password` 字段 |
| **公开分享链接** | 无密码公开链接，可对外发布给陌生人浏览 | `shared_link` 表，`password` 字段为 `null` |

### 1.2 共享链接核心数据模型

`shared_link` 表核心字段设计（位于 `server/src/schema/tables/shared-link.table.ts`）：

```typescript
class SharedLinkTable {
  id!: string;                    // 链接唯一ID
  key!: Buffer;                   // 访问密钥（Base64URL编码）
  type!: SharedLinkType;          // 链接类型：Individual / Album
  userId!: string;                // 创建者用户ID
  albumId!: string | null;        // 关联相册ID（Album类型时）
  description!: string | null;    // 链接描述
  password!: string | null;       // 访问密码（null表示公开链接）
  slug!: string | null;           // 自定义URL别名
  expiresAt!: Timestamp | null;   // 过期时间
  allowUpload!: boolean;          // 是否允许上传
  allowDownload!: Generated<boolean>;  // 是否允许下载
  showExif!: Generated<boolean>;  // 是否显示EXIF元数据
  createdAt!: Generated<Timestamp>;
}
```

### 1.3 权限枚举设计

系统采用细粒度权限控制（位于 `server/src/enum.ts`），与分享链接相关的核心权限包括：

```typescript
enum Permission {
  // 资源操作权限
  AssetRead = 'asset.read',        // 读取资产
  AssetView = 'asset.view',        // 查看资产
  AssetDownload = 'asset.download', // 下载资产
  AssetShare = 'asset.share',      // 分享资产
  AssetUpload = 'asset.upload',    // 上传资产
  
  // 相册操作权限
  AlbumRead = 'album.read',             // 读取相册
  AlbumShare = 'album.share',           // 分享相册
  AlbumDownload = 'album.download',     // 下载相册
  AlbumAssetCreate = 'albumAsset.create', // 添加相册资产
  
  // 分享链接管理权限
  SharedLinkCreate = 'sharedLink.create',
  SharedLinkRead = 'sharedLink.read',
  SharedLinkUpdate = 'sharedLink.update',
  SharedLinkDelete = 'sharedLink.delete',
  
  All = 'all'  // 全部权限
}
```

### 1.4 相册用户角色设计

```typescript
enum AlbumUserRole {
  Owner = 'owner',    // 所有者
  Editor = 'editor',  // 编辑者
  Viewer = 'viewer',  // 查看者
}
```

---

## 二、链接 Token 验证机制

### 2.1 认证流程总览

认证守卫（`AuthGuard`）在请求进入时触发，通过 `AuthService.authenticate()` 方法完成验证：

```
请求进入 → AuthGuard.canActivate() 
         → AuthService.authenticate()
         → 解析请求头/查询参数中的认证信息
         → 按优先级验证：shareKey → shareSlug → session → apiKey
         → 构建 AuthDto 对象
```

### 2.2 共享链接密钥验证

#### 验证优先级：
1. **Share Key**：通过 `key` 查询参数或 `x-immich-share-key` 请求头
2. **Share Slug**：通过 `slug` 查询参数或 `x-immich-share-slug` 请求头（自定义别名）

#### 核心验证代码（`auth.service.ts:validateSharedLinkKey`）：

```typescript
async validate({ headers, queryParams }): Promise<AuthDto> {
  const shareKey = headers[ImmichHeader.SharedLinkKey] || 
                   queryParams[ImmichQuery.SharedLinkKey];
  const shareSlug = headers[ImmichHeader.SharedLinkSlug] || 
                    queryParams[ImmichQuery.SharedLinkSlug];
  
  if (shareKey) {
    return this.validateSharedLinkKey(shareKey);
  }
  if (shareSlug) {
    return this.validateSharedLinkSlug(shareSlug);
  }
  // ...其他认证方式
}
```

### 2.3 密码保护链接的登录流程

对于设置了密码的分享链接，需要通过登录接口获取访问 token：

**登录端点**：`POST /shared-links/login`

#### 核心实现（`shared-link.service.ts:login`）：

```typescript
async login(auth: AuthDto, dto: SharedLinkLoginDto) {
  const sharedLink = await this.findOrFail(auth.user.id, auth.sharedLink.id);
  
  // 验证密码
  if (sharedLink.password !== dto.password) {
    throw new UnauthorizedException('Invalid password');
  }
  
  // 生成认证 token
  const token = this.asToken({ id: sharedLink.id, password: sharedLink.password });
  
  return {
    sharedLink: mapSharedLink(sharedLink, { stripAssetMetadata: !sharedLink.showExif }),
    token,
  };
}

// Token 生成算法：SHA256 哈希链接ID和密码的组合
private asToken(sharedLink: { id: string; password: string }) {
  return this.cryptoRepository
    .hashSha256(`${sharedLink.id}-${sharedLink.password}`)
    .toString('base64');
}
```

#### Cookie 存储机制：

验证成功后，token 存储在 `immich_shared_link_token` Cookie 中，后续请求自动携带：

```typescript
// Controller 层处理（shared-link.controller.ts）
return respondWithCookie(res, sharedLink, {
  isSecure: loginDetails.isSecure,
  values: [{ key: ImmichCookie.SharedLinkToken, value: merge(req.cookies, token) }],
});
```

### 2.4 权限路由保护

装饰器 `@Authenticated()` 标记需要认证的路由，并指定权限要求：

```typescript
@Authenticated({ sharedLink: true })  // 允许共享链接访问此路由
@Authenticated({ permission: Permission.SharedLinkRead })  // 需要特定权限
```

---

## 三、访问范围裁剪机制

### 3.1 访问控制核心架构

访问范围裁剪由 `access.ts` 工具函数统一实现，采用**权限矩阵 + 资源访问边界**双层控制：

```
请求权限
    ↓
checkAccess()
    ├─ 是共享链接访问？ → checkSharedLinkAccess()
    └─ 是普通用户访问？ → checkOtherAccess()
              ↓
         过滤允许访问的资源ID集合
              ↓
         返回裁剪后的结果
```

### 3.2 共享链接访问控制矩阵

共享链接的访问控制严格根据链接配置裁剪访问范围（`access.ts:checkSharedLinkAccess`）：

```typescript
const checkSharedLinkAccess = async (request: SharedLinkAccessRequest) => {
  const { sharedLink, permission, ids } = request;
  
  switch (permission) {
    // 资产读取/查看：始终允许（只要在共享范围内）
    case Permission.AssetRead:
    case Permission.AssetView:
      return await access.asset.checkSharedLinkAccess(sharedLinkId, ids);
    
    // 资产下载：受 allowDownload 配置控制
    case Permission.AssetDownload:
      return sharedLink.allowDownload 
        ? await access.asset.checkSharedLinkAccess(sharedLinkId, ids) 
        : new Set();
    
    // 资产上传：受 allowUpload 配置控制
    case Permission.AssetUpload:
      return sharedLink.allowUpload ? ids : new Set();
    
    // 相册读取：始终允许（只要在共享范围内）
    case Permission.AlbumRead:
      return await access.album.checkSharedLinkAccess(sharedLinkId, ids);
    
    // 相册下载：受 allowDownload 配置控制
    case Permission.AlbumDownload:
      return sharedLink.allowDownload 
        ? await access.album.checkSharedLinkAccess(sharedLinkId, ids) 
        : new Set();
    
    // 相册资产添加：受 allowUpload 配置控制
    case Permission.AlbumAssetCreate:
      return sharedLink.allowUpload 
        ? await access.album.checkSharedLinkAccess(sharedLinkId, ids) 
        : new Set();
    
    // 其他权限：全部拒绝
    default:
      return new Set<string>();
  }
};
```

### 3.3 共享资产边界验证

数据库层验证资产是否在共享链接范围内（`access.repository.ts`）：

```sql
-- 验证单个资产是否属于共享链接
SELECT asset.id
FROM asset
INNER JOIN shared_link_asset ON asset.id = shared_link_asset.assetId
WHERE shared_link_asset.sharedLinkId = $1
  AND asset.id = ANY($2)
  AND asset.deletedAt IS NULL
```

### 3.4 元数据脱敏控制

返回共享链接资产时，根据 `showExif` 配置裁剪 EXIF 元数据：

```typescript
// shared-link.service.ts:mapSharedLink
return mapSharedLink(sharedLink, { 
  stripAssetMetadata: !sharedLink.showExif 
});
```

### 3.5 上传权限验证

上传接口验证共享链接是否允许上传：

```typescript
export const requireUploadAccess = (auth: AuthDto | null): AuthDto => {
  if (!auth || (auth.sharedLink && !auth.sharedLink.allowUpload)) {
    throw new UnauthorizedException();
  }
  return auth;
};
```

---

## 四、三种分享渠道的权限统一模型

### 4.1 渠道权限对比

| 权限维度 | 同事相册共享 | 带密码分享链接 | 公开分享链接 |
|---------|-------------|---------------|-------------|
| **身份认证** | 用户登录认证 | 链接密钥 + 密码验证 | 仅链接密钥 |
| **访问控制** | AlbumUserRole 角色 | 链接配置 | 链接配置 |
| **可访问资源** | 相册内全部资产 | 链接关联的资产/相册 | 链接关联的资产/相册 |
| **上传权限** | 有（Editor角色） | 由 `allowUpload` 控制 | 由 `allowUpload` 控制 |
| **下载权限** | 有 | 由 `allowDownload` 控制 | 由 `allowDownload` 控制 |
| **EXIF可见性** | 可见 | 由 `showExif` 控制 | 由 `showExif` 控制 |
| **有效期** | 永久（直到被移除） | 由 `expiresAt` 控制 | 由 `expiresAt` 控制 |

### 4.2 统一访问控制入口

三种渠道通过 `checkAccess()` 函数实现统一入口：

```typescript
export const checkAccess = async (
  access: AccessRepository,
  { ids, auth, permission }: AccessRequest,
): Promise<Set<string>> => {
  // 共享链接访问路径
  if (auth.sharedLink) {
    return checkSharedLinkAccess(access, { 
      sharedLink: auth.sharedLink, 
      permission, 
      ids: idSet 
    });
  }
  
  // 普通用户/同事共享路径
  return checkOtherAccess(access, { auth, permission, ids: idSet });
};
```

### 4.3 普通用户访问控制（同事共享）

对于同事加入相册的场景，通过多维度权限叠加计算可访问范围：

```typescript
case Permission.AssetRead: {
  const isOwner = await access.asset.checkOwnerAccess(userId, ids);        // 自己的资产
  const isAlbum = await access.asset.checkAlbumAccess(userId, diff1);      // 加入的相册资产
  const isPartner = await access.asset.checkPartnerAccess(userId, diff2);  // 伴侣共享资产
  return setUnion(isOwner, isAlbum, isPartner);  // 三者取并集
}
```

---

## 五、安全边界设计

### 5.1 密钥安全性

- `key` 字段使用 50 字节随机 Buffer，经 Base64URL 编码，熵充足
- 密码保护链接需同时持有 **链接密钥** + **访问密码** 双重认证
- 密码验证采用直接字符串比对（密码存储在数据库中，非哈希）

### 5.2 路由隔离

```typescript
// 共享链接只能访问标记为 sharedLink: true 的路由
if (authDto.sharedLink && !sharedLinkRoute) {
  throw new ForbiddenException('Forbidden');
}
```

### 5.3 过期时间检查

在 `getByKey` / `getBySlug` 查询时自动检查 `expiresAt` 字段，过期链接拒绝访问。

### 5.4 API Key 权限检查

```typescript
if (authDto.apiKey && requestedPermission !== false) {
  if (!isGranted({ requested: [requestedPermission], current: authDto.apiKey.permissions })) {
    throw new ForbiddenException(`Missing required permission: ${requestedPermission}`);
  }
}
```

---

## 六、架构总结

### 核心设计思想

1. **统一模型**：三种分享渠道共用 `checkAccess` 权限框架，避免重复逻辑
2. **范围裁剪**：返回**允许访问的资源ID集合**，而非布尔值，支持细粒度过滤
3. **配置驱动**：共享链接行为完全由字段配置驱动（`allowUpload`/`allowDownload`/`showExif`）
4. **最小权限**：共享链接默认仅开放读取类权限，其他权限需显式开启
5. **分层验证**：认证层 → 权限层 → 资源边界层，三层防护确保安全

### 文件位置索引

| 模块 | 文件路径 |
|------|---------|
| 权限模型 | `server/src/enum.ts` |
| 共享链接服务 | `server/src/services/shared-link.service.ts` |
| 访问控制工具 | `server/src/utils/access.ts` |
| 认证服务 | `server/src/services/auth.service.ts` |
| 认证守卫 | `server/src/middleware/auth.guard.ts` |
| 共享链接数据表 | `server/src/schema/tables/shared-link.table.ts` |
| 共享链接控制器 | `server/src/controllers/shared-link.controller.ts` |
| 共享链接仓库 | `server/src/repositories/shared-link.repository.ts` |
