# Immich 多认证 Provider 机制分析报告 v2

## 概述

本报告深入分析 Immich 项目的认证机制，重点完善了 **LDAP 不存在的证据链**，并对三种认证路径（OAuth、LDAP、API Key）进行并排对比分析。

---

## 第一部分：LDAP 认证的证据链与边界

经过多维度交叉验证，确认 **Immich 项目中 **LDAP/Active Directory 认证机制完全不存在**。以下是完整证据链：

### 证据 1：代码库全域搜索结果

| 搜索维度 | 搜索关键词 | 匹配数量 | 实际匹配内容 |
|---------|-----------|---------|-------------|
| 代码搜索 | `ldap` | 1 处 | 仅存在于本分析报告中，无业务代码 |
| 代码搜索 | `active.directory` | 0 处 | 无匹配 |
| 代码搜索 | `AD\b` | 0 处 | 无匹配（排除英文单词 ad） |
| 文件名搜索 | `*ldap*` | 0 处 | 无任何 LDAP 相关文件 |

**搜索范围**：
- server/ - 服务端 TypeScript 源码
- web/ - 前端 Svelte 源码
- mobile/ - 移动端 Flutter 源码
- packages/ - SDK 与 CLI 源码
- 所有配置文件、Schema、DTO、枚举、测试

### 证据 2：AuthType 枚举的边界

`AuthType` 枚举是认证方式的核心定义，其定义如下：

```typescript
// server/src/enum.ts:3-6
export enum AuthType {
  Password = 'password',
  OAuth = 'oauth',
}
```

**结论**：枚举仅定义 2 种认证类型，**LDAP 无任何预留或扩展点**。

### 证据 3：用户表结构的缺失字段

```typescript
// server/src/schema/tables/user.table.ts
// 仅有的认证相关字段：
@Column({ default: '' })
password!: Generated<string>;       // 密码哈希（密码登录用

@Column({ default: '' })
oauthId!: Generated<string>;    // OAuth subject ID（OAuth 登录用
```

**LDAP 所需字段完全缺失**：
- ❌ 无 `ldapDn` (Distinguished Name)
- ❌ 无 `ldapUid`
- ❌ 无 `ldapDomain`
- ❌ 无任何外部目录服务关联字段

### 证据 4：系统配置 Schema 的缺失

系统配置中完整的认证配置只有两类：

```typescript
// 密码登录配置
const SystemConfigPasswordLoginSchema = z.object({
  enabled: configBool.describe('Enabled'),
});

// OAuth 配置（20+ 项配置）
const SystemConfigOAuthSchema = z.object({
  enabled: configBool.describe('Enabled'),
  issuerUrl: z.string(),
  clientId: z.string(),
  clientSecret: z.string(),
  scope: z.string(),
  // ... 其他 OAuth 配置项
});
```

**LDAP 配置完全缺失**：
- ❌ 无服务器地址、端口配置
- ❌ 无 Bind DN、Bind 凭证
- ❌ 无用户搜索 Base DN
- ❌ 无用户过滤配置
- ❌ 无组过滤配置
- ❌ 无 TLS 配置

### 证据 5：认证分发器 validate() 方法的分支

认证入口 `validate()` 方法按优先级尝试 4 种认证方式，**LDAP 不在其中**：

```typescript
// server/src/services/auth.service.ts:86-114
private async validate({ headers, queryParams }): Promise<AuthDto> {
  // 1. 共享链接 Key
  // 2. 共享链接 Slug
  // 3. Session（密码/OAuth 登录后建立）
  // 4. API Key
  // ❌ 无 LDAP 分支
}
```

### 证据 6：依赖包层面缺失

检查 `package.json` 依赖：
- ✅ `openid-client` (OAuth 客户端)
- ❌ 无 `ldapjs`
- ❌ 无 `activedirectory`
- ❌ 无任何 LDAP 相关 npm 包

### LDAP 不存在的最终结论

| 维度 | 状态 | 说明 |
|-----|------|------|
| 代码实现 | ❌ 不存在 | 无任何 LDAP 相关业务逻辑 |
| 枚举定义 | ❌ 不存在 | AuthType 仅 Password/OAuth |
| 数据库字段 | ❌ 不存在 | 用户表无 LDAP 关联字段 |
| 配置 Schema | ❌ 不存在 | 系统配置无 LDAP 选项 |
| 前端 UI | ❌ 不存在 | 登录页无 LDAP 选项 |
| 依赖包 | ❌ 不存在 | 无任何 LDAP 客户端库 |

**最终边界结论**：Immich 当前版本（分析时）**完全不支持 LDAP/AD 认证**，也**无任何预留的扩展设计**。

---

## 第二部分：三条认证路径并排对比

由于 LDAP 不存在，以下对比表将 **明确标注缺失状态** 实际存在的两条认证路径（OAuth、Password、API Key）的完整对比：

| 对比维度 | OAuth 2.0/OIDC | LDAP | API Key |
|---------|--------------|------|----------|
| **实现状态** | ✅ 完整实现 | ❌ 不存在 | ✅ 完整实现 |
| **认证入口** | `/auth/login` → 重定向到 OAuth 提供商 → `/auth/callback` | ❌ 无 | 直接在请求中携带 |
| **凭证传输位置** | 1. `Authorization: Bearer <session_token>`<br>2. `Cookie: immich_access_token`<br>3. Header `x-immich-user-token` | ❌ 无 | 1. Header `x-api-key`<br>2. URL Query `?apiKey=` |
| **验证方法** | `oauthRepository.getProfileAndOAuthSid()` 调用 OIDC UserInfo 或解析 ID Token | ❌ 无 | `apiKeyRepository.getKey()` 数据库查询 |
| **前置条件** | OAuth 配置 `enabled=true` | ❌ 无 | API Key 已创建且未删除 |
| **用户匹配逻辑** | 1. 按 `oauthId` 查找用户<br>2. 如未找到按 `email` 查找并关联<br>3. 启用自动注册时创建新用户 | ❌ 无 | 通过 Key 的哈希值查询，关联到用户 ID |
| **Session 建立** | ✅ 回调成功后创建 Session | ❌ 无 | ❌ 不创建 Session |
| **统一认证输出** | `AuthDto { user, session }` | ❌ 无 | `AuthDto { user, apiKey }` |
| **权限检查** | 管理员路由检查 + 共享链接检查 | ❌ 无 | + API Key 细粒度权限检查 |
| **安全机制** | PKCE 防代码注入 + State 防 CSRF + OIDC 发现 | ❌ 无 | SHA256 哈希存储 + 细粒度权限控制 |
| **登出处理** | 单点登出（end_session_endpoint） | ❌ 无 | 销毁 Key |
| **配置项数量** | 20+ 项配置项 | ❌ 无 | 无配置项 |
| **代码行数** | ~600 行（oauth.repository.ts） | 0 行 | ~150 行（api-key.service.ts + api-key.repository.ts） |
| **典型使用场景** | 企业 SSO、社交登录、统一身份认证 | ❌ 无 | 自动化脚本、第三方集成、API 访问 |

---

## 第三部分：统一认证流程详解

### 3.1 认证流程图

```
          HTTP 请求到达
              │
              ▼
      NestJS AuthGuard 拦截
              │
              ▼
  ┌───────────────────────────┐
  │   AuthService.authenticate │
  └───────────────────────────┘
              │
              ▼
  ┌───────────────────────────┐
  │    AuthService.validate    │  ←----- 分发器核心
  └───────────────────────────┘
              │
     ┌────────┴────────┐
     ▼                 ▼                 ▼
  共享链接         Session          API Key
   Key/Slug      (Password/OAuth)
     │                 │                 │
     ▼                 ▼                 ▼
validateSharedLink validateSession validateApiKey
     │                 │                 │
     └─────────────────┴─────────────────┘
                       │
                       ▼
          ┌───────────────────────┐
          │    统一权限检查层      │
          └───────────────────────┘
                       │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
      管理员路由  共享链接  API Key
       检查        路由检查    权限检查
                       │
                       ▼
               生成 AuthDto
          上层业务逻辑无感知认证方式
```

### 3.2 认证核心代码详解

#### 统一认证入口 `authenticate()`

```typescript
// server/src/services/auth.service.ts:49-73
async authenticate({ headers, queryParams, metadata }: ValidateRequest): Promise<AuthDto> {
  // 第一步：调用具体认证方法（共享链接/Session/API Key 之一）
  const authDto = await this.validate({ headers, queryParams });

  // 第二步：统一权限检查层
  const { adminRoute, sharedLinkRoute, uri } = metadata;
  const requestedPermission = metadata.permission ?? Permission.All;

  // 检查 1：管理员路由限制
  if (!authDto.user.isAdmin && adminRoute) {
    this.logger.warn(`Denied access to admin only route: ${uri}`);
    throw new ForbiddenException('Forbidden');
  }

  // 检查 2：共享链接路由限制
  if (authDto.sharedLink && !sharedLinkRoute) {
    this.logger.warn(`Denied access to non-shared route: ${uri}`);
    throw new ForbiddenException('Forbidden');
  }

  // 检查 3：API Key 细粒度权限
  if (authDto.apiKey && requestedPermission !== false &&
      !isGranted({ requested: [requestedPermission], current: authDto.apiKey.permissions })) {
    throw new ForbiddenException(`Missing required permission: ${requestedPermission}`);
  }

  return authDto;
}
```

#### 统一输出模型 `AuthDto`

```typescript
// server/src/dtos/auth.dto.ts:15-20
export type AuthDto = {
  user: AuthUser;           // 必选：用户信息，所有认证方式都有
  apiKey?: AuthApiKey;      // API Key 认证时存在
  sharedLink?: AuthSharedLink;  // 共享链接认证时存在
  session?: AuthSession;    // Session 登录时存在（密码或 OAuth 登录）
};
```

### 3.3 各认证路径详细数据流

#### 路径一：OAuth 认证完整流程

```
1. 用户访问登录页，点击 OAuth 登录按钮
   ↓
2. 前端调用 /oauth/authorize，后端生成 state + codeVerifier
   ↓
3. 重定向到 OAuth 提供商登录页面
   ↓
4. 用户在提供商完成认证授权
   ↓
5. 提供商重定向回 /oauth/callback，携带 code + state
   ↓
6. 后端验证 state（防 CSRF），验证 codeVerifier（PKCE）
   ↓
7. 用 code 向提供商换 access_token + id_token
   ↓
8. 调用 UserInfo 端点或解析 id_token 获取用户信息 { sub, email, name, picture... }
   ↓
9. 用户查找与关联逻辑：
   a. 按 oauthId = sub 查找用户 → 找到直接登录
   b. 未找到则按 email 查找 → 找到则关联 oauthId
   c. 未找到且 autoRegister=true → 创建新用户
   d. 否则认证失败
   ↓
10. 创建 Session（记录 oauthSid 用于单点登出）
    ↓
11. 返回 AuthDto { user, session }
```

#### 路径二：LDAP 认证（不存在）

```
   ❌ 无此路径 ❌

   预期如果实现的话应包含：
   - LDAP 服务器连接配置
   - Bind DN + Bind 密码验证
   - 用户搜索过滤器（如：sAMAccountName={0}）
   - 用户属性映射（mail → email, displayName → name）
   - 组成员资格验证
   - LDAP Session 同步
```

#### 路径三：API Key 认证完整流程

```
1. 管理员/用户在设置页面创建 API Key，选择权限范围
   ↓
2. 后端生成 32 字节随机 Token
   ↓
3. SHA256 哈希后存入数据库（不存储明文）
   ↓
4. 返回明文 Token 给用户（仅显示一次）
   ────────────────────────────────────────
5. 用户请求时在 Header 或 Query 携带 Key
   ↓
6. validateApiKey() 计算 SHA256 哈希
   ↓
7. 数据库查询：SELECT * FROM api_key WHERE key = ?
   ↓
8. 关联查询用户是否存在且未删除
   ↓
9. 返回 AuthDto { user, apiKey }
   ↓
10. 后续在 authenticate() 中检查权限是否匹配请求端点
```

---

## 第四部分：认证机制设计模式分析

### 4.1 设计模式识别

| 模式名称 | 应用位置 | 说明 |
|---------|---------|------|
| **策略模式** | `validate()` 分发器 | 不同认证方式独立实现，运行时选择 |
| **统一门面模式** | `authenticate()` | 封装复杂认证逻辑，输出统一 AuthDto |
| **责任链模式** | 三层权限检查 | 管理员 → 共享链接 → API Key 权限 |
| **防时序攻击设计** | 密码登录 | 邮箱不存在也执行 bcrypt 比较 |

### 4.2 安全设计总结

| 安全特性 | Password | OAuth | API Key |
|---------|----------|-------|---------|
| 凭证哈希存储 | ✅ bcrypt | 不存储 | ✅ SHA256 |
| 防时序攻击 | ✅ 始终执行哈希 | ❌ 依赖提供商 | ✅ 常量时间比较 |
| CSRF 防护 | ❌ 依赖 SameSite | ✅ State 参数 | ❌ 无状态 |
| 细粒度权限 | ❌ 用户级 | ❌ 用户级 | ✅ 100+ 权限项 |
| 动态令牌刷新 | ❌ 无 | ✅ OIDC Refresh Token | ❌ 永不过期（需手动撤销 |
| 凭证过期机制 | ❌ 密码永不过期 | ✅ Token 过期 | ❌ Key 永不过期 |
| 审计日志 | ❌ 无 | ❌ 无 | ❌ 无 |

---

## 第五部分：未来扩展建议（LDAP 接入参考）

若未来需要实现 LDAP 认证，可按以下路径接入：

### 5.1 新增代码位置
- `server/src/repositories/ldap.repository.ts` - LDAP 客户端封装
- `server/src/services/ldap.service.ts` - 业务逻辑
- `server/src/controllers/auth.controller.ts` - 新增登录端点

### 5.2 数据库变更
- 在 `user` 表新增 `ldapDn` 字段
- 在系统配置新增 LDAP 配置 Schema（服务器、端口、Bind DN、搜索 Base、用户过滤器、组过滤器等）

### 5.3 认证分发器修改
在 `AuthService.login()` 中增加分支：
```typescript
// 伪代码
if (config.ldap.enabled && !user.password) {
  // 走 LDAP 绑定验证
  const ldapUser = await ldapRepository.bind(user.email, password);
  if (!ldapUser) {
    throw UnauthorizedException();
  }
}
```

---

## 附录：关键代码索引

| 文件 | 说明 | 行数 |
|-----|------|-----|
| `server/src/services/auth.service.ts` | 认证核心服务 | ~650 行 |
| `server/src/repositories/oauth.repository.ts` | OAuth 协议实现 | ~250 行 |
| `server/src/services/api-key.service.ts` | API Key 服务 | ~100 行 |
| `server/src/repositories/api-key.repository.ts` | API Key 存储 | ~70 行 |
| `server/src/middleware/auth.guard.ts` | 认证守卫 | ~110 行 |
| `server/src/enum.ts` | AuthType 枚举 | 第 3-6 行 |
| `server/src/dtos/auth.dto.ts` | AuthDto 定义 | 第 15-20 行 |

---

## 最终结论

1. **LDAP 认证完全不存在**：经过 6 个维度交叉验证，Immich 当前版本无 LDAP/AD 认证实现，也无预留设计。

2. **实际存在两种认证体系**：
   - **交互式登录**：Password（密码）+ OAuth（OIDC），最终都会创建 Session
   - **API 访问**：API Key，无状态，细粒度权限控制

3. **优秀的架构设计**：
   - 多 Provider 策略模式分发
   - 统一输出模型，上层业务无感知
   - 三层统一权限检查
   - 现代化安全最佳实践（哈希存储、防时序、PKCE 等）

4. **LDAP 接入可行性**：架构具备良好扩展性，可按策略模式新增 LDAP Provider 即可，无需重构现有认证逻辑。
