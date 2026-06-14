# Session 生命周期与边界深度分析

## 一、API Key 在 MustAuth 链路的致命边界：永远 401

### 1.1 问题根源

之前的分析忽略了一个关键的控制流问题：当 API Key 通过 `AuthAPIKeyScope` 校验后，代码**不会**停止，而是会 fallthrough 到 `GetUserCacheInfo`，用 `sk_xxx` 作为 token 查询缓存，**永远查不到**，最终返回 401。

**真实执行流** [auth.go L111-L136](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L111-L136)：

```
API Key 请求进入 MustAuthWithoutAccountAvailable()
    │
    ├─ token = "sk_abc123..."
    │
    ├─ AuthAPIKeyScope(ctx, "sk_abc123...")
    │   ├─ 检测到 sk_ 前缀
    │   ├─ 调用 AuthService.AuthAPIKey()
    │   ├─ Key 有效且权限匹配 → pass=true
    │   └─ return false  // API Key 校验通过，继续执行
    │
    └─ ⚠️  Fallthrough！调用 GetUserCacheInfo(ctx, "sk_abc123...")
        │
        ├─ 查询缓存键 answer:user:token:sk_abc123...
        ├─ 不存在（因为用户登录时生成的是 UUIDv7 token，不是 sk_ 前缀）
        └─ return nil, nil
            │
            └─ if err != nil || userInfo == nil → 401 Unauthorized ❌
```

### 1.2 边界结论矩阵

| 路由组 | 中间件链 | API Key 实际结果 | 说明 |
|-------|---------|----------------|------|
| `mustUnAuthV1` | 无认证 | ✅ 放行 | 不走任何认证中间件 |
| `unAuthV1` | `Auth()` + `EjectUserBySiteInfo()` | ⚠️ 身份注入失败，但请求继续 | `Auth()` 中 `GetUserCacheInfo(sk_xxx)` 返回 nil，不会注入用户信息，但不会 Abort |
| `authWithoutStatusV1` | `MustAuthWithoutAccountAvailable()` | ❌ 永远 401 | Key 通过后 fallthrough 到缓存查询失败 |
| `authV1` | `MustAuthAndAccountAvailable()` | ❌ 永远 401 | 同上 |
| `adminauthV1` | `AdminAuth()` | ❌ 永远 401 | 不支持 API Key 旁路，直接查缓存失败 |
| `mcpAPIGroup` | `AuthMcpEnable()` + `AuthAPIKey()` | ✅ 正常通过 | 专门的 API Key 中间件，**不调用** `GetUserCacheInfo`，通过后直接 `ctx.Next()` |

### 1.3 AuthAPIKeyScope 的设计意图（反模式）

从代码看，`AuthAPIKeyScope` 只处理了**校验失败**的场景（返回 true，Abort 请求），但**完全没有处理校验通过后应该做什么**。它没有：
- 注入用户信息到 Context
- 跳过后续的 `GetUserCacheInfo` 查询
- 或者在通过时也返回 true 提前结束

这意味着 `MustAuth*` 系列中的 API Key 支持是**完全不可用的**，只有 MCP 路由组使用的独立 `AuthAPIKey()` 中间件才能正常工作。

---

## 二、Token 清理的四个触发点

### 2.1 触发点全景

```
Token 清理触发点
    │
    ├─ 🔴 触发点 1：用户主动登出
    │   ├─ Controller: POST /user/logout [user_controller.go L244-L249]
    │   ├─ 清理：① answer:user:token + ⑤ answer:admin:token
    │   └─ 范围：仅当前 AccessToken
    │
    ├─ 🔴 触发点 2：管理员修改用户角色
    │   ├─ Controller: PUT /admin/user/role [user_backyard_controller.go L80]
    │   ├─ Service: UpdateUserRole [user_backyard.go L220-L233]
    │   ├─ 清理：RemoveUserAllTokens → 遍历 ③ user-token:mapping 删除所有 ①
    │   └─ 范围：该用户所有设备的所有 Token
    │
    ├─ 🔴 触发点 3：用户修改密码（其他设备登出）
    │   ├─ Controller: PUT /user/password [user_controller.go L?]
    │   ├─ Service: UserService.UpdatePassword [user_service.go L310]
    │   ├─ 清理：RemoveTokensExceptCurrentUser → 删除除当前外的所有 Token
    │   └─ 范围：该用户其他设备的 Token（保留当前会话）
    │
    └─ 🔴 触发点 4：管理员/CLI 修改用户状态（停用/删除）
        ├─ 场景 A：管理员 PUT /admin/user/status → UpdateUserStatus [user_backyard.go L128]
        ├─ 场景 B：CLI reset-password → reset_password.go L151
        ├─ 场景 C：用户删除自己账号 → UserService.DeleteUser [user_service.go L267]
        ├─ 清理：RemoveUserAllTokens → 删除该用户所有 Token
        └─ 范围：该用户所有设备的所有 Token
```

### 2.2 各触发点的清理细节

**触发点 1：主动登出** [user_controller.go L244-L249](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/controller/user_controller.go#L244-L249)：

```go
func (uc *UserController) Logout(ctx *gin.Context) {
    accessToken := middleware.ExtractToken(ctx)
    // 只清理当前 AccessToken 对应的 ① 和 ⑤ 号缓存
    _ = uc.authService.RemoveUserCacheInfo(ctx, accessToken)
    _ = uc.authService.RemoveAdminUserCacheInfo(ctx, accessToken)
    handler.HandleResponse(ctx, nil, nil)
}
```

**触发点 2：管理员修改角色** [user_backyard.go L220-L233](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go#L220-L233)：

```go
func (us *UserAdminService) UpdateUserRole(ctx context.Context, req *schema.UpdateUserRoleReq) (err error) {
    err = us.userRoleRelService.SaveUserRole(ctx, req.UserID, req.RoleID)
    // 修改角色后强制该用户所有设备重新登录
    // 确保新角色权限立即生效
    us.authService.RemoveUserAllTokens(ctx, req.UserID)
    return
}
```

**触发点 3：修改密码（保留当前会话）** [user_service.go L310](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go#L310)：

```go
func (us *UserService) UpdatePassword(ctx context.Context, req *schema.UserUpdatePasswordReq) (err error) {
    // ... 验证旧密码、更新新密码 ...
    // 其他设备登出，保留当前 Token
    us.authService.RemoveTokensExceptCurrentUser(ctx, userInfo.ID, req.AccessToken)
    return nil
}
```

**触发点 4：状态变更（停用/删除）**：

- 管理员更新用户状态 → 调用 `RemoveUserAllTokens` [user_backyard.go L231](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go#L231)
- 管理员删除用户 → 调用 `RemoveUserAllTokens` [user_backyard.go L393](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go#L393)
- 用户自删账号 → 调用 `RemoveUserAllTokens` [user_service.go L267](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go#L267)
- CLI 重置密码 → 调用 `RemoveUserAllTokens` [reset_password.go L151](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/cli/reset_password.go#L151)

### 2.3 RemoveUserAllTokens 实现细节

[auth.go (repo) L210-L239](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/auth/auth.go#L210-L239)：

```go
func (ar *authRepo) RemoveUserTokens(ctx context.Context, userID string, remainToken string) {
    // 1. 从 ③ 号缓存读取该用户的所有 AccessToken 映射
    key := constant.UserTokenMappingCacheKey + userID
    resp, _, _ := ar.data.Cache.GetString(ctx, key)
    mapping := make(map[string]bool, 0)
    json.Unmarshal([]byte(resp), &mapping)

    // 2. 遍历删除每个 Token 的 ① 号缓存
    for token := range mapping {
        if token == remainToken {  // 可选：保留当前 Token
            continue
        }
        ar.RemoveUserCacheInfo(ctx, token)
    }

    // 3. 清理 ④ 号状态变更缓存
    ar.RemoveUserStatus(ctx, userID)

    // 4. 清理 ③ 号映射缓存本身
    ar.data.Cache.Del(ctx, key)
}
```

**注意**：此实现**不清理** ② 号 VisitToken 缓存，也不清理 ⑤ 号管理员缓存。

---

## 三、SetUserStatus 写状态缓存的实际场景

### 3.1 SetUserStatus 的作用

`SetUserStatus` 写入 ④ 号缓存 `answer:user:status:<UserID>`，作为"状态变更通知"。下次用户任意 Token 访问时，`GetUserCacheInfo` 会读到这个缓存，合并新状态并回写刷新 ① 号缓存。

### 3.2 实际写入场景（只有 3 处！）

| 场景 | 代码位置 | 说明 |
|-----|---------|------|
| **1. 用户登录** | [user_service.go L631](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go#L631) | 登录成功后写入当前状态，确保后续请求刷新 |
| **2. 用户验证邮箱** | [user_service.go L766](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go#L766) | 邮箱验证通过后，EmailStatus 从 2→1，需要通知所有 Token |
| **3. 后台手动封禁** | [user_backyard_repo.go L80](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/user/user_backyard_repo.go#L80) | 管理员后台操作时写入状态缓存 |

**关键反常识**：管理员通过 `UpdateUserStatus` API 修改用户状态时，**不调用 SetUserStatus**，而是直接调用 `RemoveUserAllTokens` 让用户强制重新登录。

证据：[user_backyard.go L128-L179](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go#L128-L179) 中完全没有 `SetUserStatus` 调用，而是：
- 修改用户角色 → `RemoveUserAllTokens`（全部踢下线）
- 修改用户状态 → 不调用 SetUserStatus，直接 `RemoveUserAllTokens`（全部踢下线）

### 3.3 场景 1：用户登录时写入

[user_service.go L620-L633](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go#L620-L633)：

```go
accessToken, userCacheInfo, err := us.userCommonService.CacheLoginUserInfo(
    ctx, userInfo.ID, userInfo.MailStatus, userInfo.Status, "")
// ...
// 注释说明：User verified email will update user email status.
// So user status cache should be updated.
if err = us.authService.SetUserStatus(ctx, userCacheInfo); err != nil {
    return nil, err
}
```

**设计意图**：如果用户在其他设备登录时邮箱被验证通过，本次登录写入的状态会作为"最新状态"，通知其他设备下次请求时刷新。

### 3.4 场景 2：邮箱验证后写入

[user_service.go L747-L768](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go#L747-L768)：

```go
// 更新数据库中的 EmailStatus = Available
err = us.userRepo.UpdateEmailStatus(ctx, data.UserID, entity.EmailStatusAvailable)
// ...
userCacheInfo := &entity.UserCacheInfo{
    UserID:      userInfo.ID,
    EmailStatus: entity.EmailStatusAvailable,  // 新状态
    UserStatus:  userInfo.Status,
    RoleID:      roleID,
}
// 生成新 Token（旧 Token 仍有效，但下次访问会被刷新）
resp.AccessToken, resp.VisitToken, err = us.authService.SetUserCacheInfo(ctx, userCacheInfo)
// 写入 ④ 号缓存，通知该用户的其他 Token 下次访问时刷新状态
if err = us.authService.SetUserStatus(ctx, userCacheInfo); err != nil {
    return nil, err
}
```

### 3.5 SetUserStatus 缓存生命周期

```
SetUserStatus(UserID, userInfo)
    │
    ▼
写入 ④ answer:user:status:<UserID> = JSON(userInfo)
    │  TTL = UserStatusChangedCacheTime（7 天）
    │
    ▼
下次用户任意 AccessToken 访问
    │
    ├─ GetUserCacheInfo(AccessToken) → 读到 ① 号旧缓存
    ├─ GetUserStatus(UserID) → 读到 ④ 号新状态
    ├─ 合并新状态到 UserCacheInfo
    ├─ SetUserCacheInfo(AccessToken, ...) → 回写刷新 ① 号缓存
    └─ （④ 号缓存仍保留，直到 TTL 过期或被 RemoveUserStatus 清除）
```

**问题**：④ 号缓存一旦写入，除非调用 `RemoveUserStatus`（只有在批量登出时才会调用），否则会一直存在 7 天。每次用户访问都会触发一次"状态合并 + 缓存回写"，造成不必要的写放大。

---

## 四、UUIDv7 GenerateToken 语义

### 4.1 实现代码

[token.go L24-L28](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/pkg/token/token.go#L24-L28)：

```go
import "github.com/google/uuid"

// GenerateToken generate token
func GenerateToken() string {
    uid, _ := uuid.NewV7()
    return uid.String()
}
```

### 4.2 UUIDv7 格式详解

UUIDv7 格式（RFC 9562）：

```
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    unix_ts_ms (48 bits)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  ver  |    rand_a (12 bits)    | var |      rand_b (30 bits) |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         rand_b (32 bits)                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**示例**：`018b8a3f-7b2c-7a1b-9c3d-2e4f5a6b7c8d`
- `018b8a3f`：前 8 位十六进制 = 时间戳高 32 位
- `7b2c`：接下来 4 位 = 时间戳低 16 位（总计 48 位毫秒时间戳）
- `7`：版本号 = 7（UUIDv7）
- `a1b`：随机数 A
- `9`：变体 = 1001（RFC 4122）
- `c3d2e4f5a6b7c8d`：随机数 B

### 4.3 UUIDv7 作为 Token 的设计考量

| 特性 | 说明 | 优势 |
|-----|------|------|
| **可排序** | 前 48 位是毫秒时间戳 | 便于按时间排序、审计、追踪 |
| **随机性** | 74 位随机数 | 足够的熵防止猜测 |
| **全局唯一** | 时间戳 + 随机数组合 | 分布式环境下无需中心化生成器 |
| **无状态** | 自包含时间戳 | 无需额外查表即可判断 Token 生成时间 |

### 4.4 使用位置

| 用途 | 生成位置 |
|-----|---------|
| AccessToken | `SetUserCacheInfo()` 登录时生成 |
| VisitToken | `SetUserCacheInfo()` 登录时生成 |
| 邮箱验证码 | `UserChangeEmailSendCode()` 发送验证码时生成 |
| 其他一次性 Token | 各处调用 `token.GenerateToken()` |

### 4.5 Token 格式总结

| Token 类型 | 前缀 | 格式 | 示例 |
|-----------|------|------|------|
| AccessToken | 无 | UUIDv7 | `018b8a3f-7b2c-7a1b-9c3d-2e4f5a6b7c8d` |
| VisitToken | 无 | UUIDv7 | `018b8a3f-1234-5678-9abc-def012345678` |
| API Key | `sk_` | `sk_` + 随机字符串 | `sk_abc123def456ghi789` |
| 邮箱验证码 | 无 | UUIDv7 | `018b8a3f-...` |

---

## 五、前端 RouteGuard 与私有模式后端的协同

### 5.1 三层防护架构

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: 后端 CheckPrivateMode 中间件（UI 路由）             │
│  作用：非 API 路径访问时，私有模式返回 index.html 给前端        │
│  位置：[auth.go L221-L236]                                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: 前端 shouldLoginRequired Guard（路由级）            │
│  作用：前端路由切换前检查 login_required 标志，未登录跳登录页  │
│  位置：[guard.ts L265-L281]                                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: 后端 EjectUserBySiteInfo 中间件（API 路由）         │
│  作用：API 请求时，私有模式 + 未登录返回 401 JSON             │
│  位置：[auth.go L80-L108]                                     │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Layer 1：CheckPrivateMode 后端中间件

[auth.go L221-L236](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L221-L236)：

```go
func (am *AuthUserMiddleware) CheckPrivateMode() gin.HandlerFunc {
    return func(ctx *gin.Context) {
        resp, err := am.siteInfoCommonService.GetSiteSecurity(ctx)
        if resp.LoginRequired {
            // 私有模式 → 返回 SPA index.html，让前端接管路由和登录
            ShowIndexPage(ctx)  // 渲染 HTML，不是 JSON
            ctx.Abort()
            return
        }
        ctx.Next()
    }
}
```

**关键点**：
- 返回的是 HTML 页面，不是 401 JSON
- 前端 SPA 加载后，自己判断是否需要跳转登录页
- 这样做的目的是让私有模式下的 URL 能正常被前端路由解析（用户复制链接后能正常打开）

### 5.3 Layer 2：shouldLoginRequired 前端 Guard

[guard.ts L265-L281](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/ui/src/utils/guard.ts#L265-L281)：

```typescript
export const shouldLoginRequired = () => {
  const gr: TGuardResult = { ok: true };
  const { login_required } = siteSecurityStore.getState();
  if (!login_required) {
    return gr;  // 非私有模式，直接放行
  }
  const us = deriveLoginState();
  if (us.isLogged) {
    return gr;  // 已登录，放行
  }
  if (isIgnoredPath(IGNORE_PATH_LIST)) {
    return gr;  // 白名单路径（登录页、注册页等）放行
  }
  gr.ok = false;
  gr.redirect = RouteAlias.login;  // 未登录 → 跳转登录页
  return gr;
};
```

**白名单路径** [guard.ts L102-L113](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/ui/src/utils/guard.ts#L102-L113)：

```typescript
export const IGNORE_PATH_LIST = [
  RouteAlias.login,           // /users/login
  RouteAlias.signUp,          // /users/signup
  RouteAlias.accountRecovery, // /users/account-recovery
  RouteAlias.changeEmail,     // /users/change-email
  RouteAlias.passwordReset,   // /users/password-reset
  RouteAlias.accountActivation,   // /users/account-activation
  RouteAlias.confirmNewEmail,     // /users/confirm-new-email
  RouteAlias.confirmEmail,        // /users/confirm-email
  RouteAlias.authLanding,         // /users/auth-landing
  '/user-center/',                // 用户中心插件路径
];
```

### 5.4 Layer 3：EjectUserBySiteInfo 后端中间件

[auth.go L80-L108](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L80-L108)：

```go
func (am *AuthUserMiddleware) EjectUserBySiteInfo() gin.HandlerFunc {
    return func(ctx *gin.Context) {
        siteInfo, _ := am.siteInfoCommonService.GetSiteSecurity(ctx)
        if siteInfo != nil && siteInfo.LoginRequired {
            userInfo := GetUserInfoFromContext(ctx)
            if userInfo == nil {
                // API 场景返回 JSON 401
                handler.HandleResponse(ctx, errors.Unauthorized(reason.UnauthorizedError), nil)
                ctx.Abort()
                return
            }
        }
        ctx.Next()
    }
}
```

### 5.5 初始化时序与状态同步

前端初始化流程 [guard.ts L417-L442](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/ui/src/utils/guard.ts#L417-L442)：

```
App 启动 → setupApp()
    │
    ├─ Promise.allSettled([
    │   ├─ initAppSettingsStore() → GET /siteinfo → siteSecurityStore.login_required
    │   └─ pullLoggedUser(true)   → GET /user/info → loggedUserInfoStore
    │  ])
    │
    ├─ pullUcAgent() → UserCenter 同步
    └─ appInitialized = true
          │
          ▼
        路由挂载 → 根路由 guard: shouldLoginRequired()
              └─ 读取 siteSecurityStore.login_required
```

**关键顺序保证**：`guard.setupApp()` 作为根路由的 `loader` 先执行，完成状态初始化后才会调用 `shouldLoginRequired()`，避免竞态。

### 5.6 完整协同流程示例（私有模式下访问首页）

```
浏览器请求 GET /
    │
    ├─ 后端 CheckPrivateMode 中间件
    │   ├─ 检测到 LoginRequired=true
    │   └─ 返回 index.html（SPA）
    │
    ├─ 前端加载 JS → setupApp()
    │   ├─ GET /siteinfo → login_required=true → 存入 siteSecurityStore
    │   └─ GET /user/info → 401（未登录）→ loggedUserInfoStore 清空
    │
    ├─ 根路由 guard: shouldLoginRequired()
    │   ├─ login_required=true
    │   ├─ isLogged=false
    │   ├─ 路径不在白名单
    │   └─ return { ok:false, redirect: "/users/login" }
    │
    └─ 前端自动跳转到 /users/login
          │
          ├─ 路径在白名单 → guard 放行
          └─ 用户输入账号密码登录
```

---

## 六、GetUserCacheInfo 回写一号缓存的并发竞态

### 6.1 问题场景

当管理员修改了用户状态/角色，或用户验证了邮箱后，④ 号缓存被写入。此时如果该用户同时有多个并发请求（浏览器通常会并行加载多个 API），就会触发竞态。

### 6.2 竞态演示

```
并发请求 1                    并发请求 2                    并发请求 3
    │                            │                            │
    ├─ GetUserCacheInfo(T1)      ├─ GetUserCacheInfo(T1)      ├─ GetUserCacheInfo(T1)
    │  ① → 旧状态                 │  ① → 旧状态                 │  ① → 旧状态
    │                            │                            │
    ├─ GetUserStatus(UID)        ├─ GetUserStatus(UID)        ├─ GetUserStatus(UID)
    │  ④ → 新状态 ✓               │  ④ → 新状态 ✓               │  ④ → 新状态 ✓
    │                            │                            │
    ├─ 合并状态                   ├─ 合并状态                   ├─ 合并状态
    │                            │                            │
    ├─ SetUserCacheInfo(T1, VT1, merged) ────┐                │
    │  ① = 新状态 + VT1                      │                │
    │                                        │                │
    │                            ├─ SetUserCacheInfo(T1, VT2, merged)
    │                            │  ① = 新状态 + VT2（覆盖！）
    │                            │                            │
    │                            │               ┌─────────────┘
    │                            │               │
    │                            │               ├─ SetUserCacheInfo(T1, VT3, merged)
    │                            │               │  ① = 新状态 + VT3（再覆盖！）
    │                            │               │
    │                            │               │
    ▼                            ▼               ▼
  结果：
  1. ① 号缓存被多次写入，最后一次的 VisitToken "获胜"
  2. ② 号缓存（VisitToken → AccessToken）只有 VT3 能查到
  3. VT1 和 VT2 对应的 ② 号缓存可能已存在，但不会被清理
  4. ③ 号映射中可能残留无效的 Token 记录
```

### 6.3 竞态代码位置

[auth.go (service) L71-L81](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go#L71-L81)：

```go
func (as *AuthService) GetUserCacheInfo(ctx context.Context, accessToken string) (userInfo *entity.UserCacheInfo, err error) {
    userCacheInfo, err := as.authRepo.GetUserCacheInfo(ctx, accessToken)
    // ...
    cacheInfo, _ := as.authRepo.GetUserStatus(ctx, userCacheInfo.UserID)
    if cacheInfo != nil {
        userCacheInfo.UserStatus = cacheInfo.UserStatus
        userCacheInfo.EmailStatus = cacheInfo.EmailStatus
        userCacheInfo.RoleID = cacheInfo.RoleID
        // ⚠️ 并发竞态点：多个请求同时到达这里
        err := as.authRepo.SetUserCacheInfo(ctx, accessToken, userCacheInfo.VisitToken, userCacheInfo)
        // 每次调用都会重新写入 ① 和 ② 号缓存，VisitToken 都是同一个？
        // 不！userCacheInfo.VisitToken 是从 ① 号缓存读出来的，所以是一样的？
        // 等一下，让我再看 SetUserCacheInfo 的实现...
    }
    // ...
}
```

**重新审视**：`userCacheInfo.VisitToken` 是从 ① 号缓存读取的，所以并发请求读取到的 VisitToken 是相同的。但 `SetUserCacheInfo` 会：
1. 重新写入 ① 号缓存（AccessToken → UserCacheInfo）
2. 重新写入 ② 号缓存（VisitToken → AccessToken）

**实际竞态结果**：

由于 `userCacheInfo.VisitToken` 在各并发请求中是相同的（从同一个 ① 号缓存读出），所以竞态的后果是：
- **无害的写放大**：多次重复写入相同的 Key-Value（Redis SET 是幂等的）
- **无状态不一致**：因为写入的值是相同的（状态都是从 ④ 号缓存合并的，VisitToken 也相同）

但有一个**隐藏问题**：`SetUserCacheInfo` 会重置 TTL 为 7 天。多次并发写入会导致 TTL 被不断刷新延长，理论上如果用户持续有并发请求，Token 可能永不过期（但实际 7 天的窗口很大，影响不大）。

### 6.4 真正的竞态风险：状态合并过程

竞态真正可能出现在状态合并的逻辑中：

```
Request A                          Request B
    │                                │
    ├─ ① → {Status:1, Role:1, VT:X}  ├─ ① → {Status:1, Role:1, VT:X}
    │                                │
    ├─ ④ → {Status:9, Role:2}        ├─ ④ → {Status:9, Role:2}
    │                                │
    ├─ 合并 → {Status:9, Role:2, VT:X}
    │                                ├─ 合并 → {Status:9, Role:2, VT:X}
    │                                │
    ├─ SetUserCacheInfo(A, X, merged)
    │  ① = {Status:9, Role:2, VT:X}
    │                                │
    ├─ UserCenter.UserStatus()       ├─ SetUserCacheInfo(A, X, merged)
    │  返回 1（可用）                 │  ① = {Status:9, Role:2, VT:X}（相同值）
    │                                │
    ├─ 覆盖 Status = 1
    │
    └─ Return {Status:1, Role:2}  ⚠️

结果：Request A 返回了被 UserCenter 覆盖后的 Status=1
      Request B 返回了 Status=9
      同一用户的两个并发请求拿到了不同的状态
```

**但这不是缓存竞态**，而是 UserCenter 同步的设计特性：每次请求都可能从外部获取最新状态。

### 6.5 另一个竞态：④ 号缓存的读取与删除

当管理员调用 `RemoveUserAllTokens` 时，会删除 ④ 号缓存：

```
Request A                          Admin Action
    │                                │
    ├─ 读 ① → 旧状态                 │
    │                                ├─ RemoveUserAllTokens
    ├─ 读 ④ → 新状态 ✓               │  ├─ 遍历删除所有 ①
    │                                │  ├─ 删 ③
    ├─ 合并状态                       │  └─ 删 ④ ✨
    │                                │
    ├─ SetUserCacheInfo(...)         │
    │  重写 ①（刚被删了又写回来）     │
    │                                │
    └─ ④ 已被删除，下次不会再触发刷新
```

**结果**：用户的 Token 在批量登出后又被"复活"了。但这是在管理员操作的瞬间窗口发生的，用户需要重新登录才能拿到新 Token（旧 Token 的状态可能已被更新为停用/删除）。

---

## 七、完整 Session 生命周期全景图

```
用户登录
    │
    ├─ UUIDv7 生成 AccessToken + VisitToken
    ├─ 写入 ① answer:user:token:<AccessToken> = UserCacheInfo
    ├─ 写入 ② answer:user:visit:<VisitToken> = AccessToken
    ├─ 写入 ③ answer:user-token:mapping:<UserID> = {AccessToken: true}
    └─ 写入 ④ answer:user:status:<UserID> = UserCacheInfo（可选）
    │
正常访问（每次请求）
    │
    ├─ ExtractToken → AccessToken
    ├─ GetUserCacheInfo(AccessToken)
    │   ├─ 读 ① → UserCacheInfo
    │   ├─ 读 ④ → 有状态变更？
    │   │   ├─ 是 → 合并 + SetUserCacheInfo 回写刷新 ①
    │   │   └─ 否 → 继续
    │   └─ UserCenter.UserStatus() → 外部状态同步
    ├─ 状态检查（邮箱验证/停用/删除）
    ├─ ctx.Set(userInfo)
    └─ 业务处理
    │
状态变更触发点
    │
    ├─ 邮箱验证 → SetUserStatus(④) → 下次访问刷新
    ├─ 修改角色 → RemoveUserAllTokens → 全量登出
    ├─ 修改状态 → RemoveUserAllTokens → 全量登出
    ├─ 修改密码 → RemoveTokensExceptCurrent → 其他设备登出
    └─ 主动登出 → RemoveUserCacheInfo(当前Token)
    │
Token 过期
    │
    └─ Redis TTL 7 天自动过期（或被主动删除）
```

---

## 八、关键文件索引

| 功能 | 文件 | 关键行 |
|-----|------|--------|
| MustAuth* 中 API Key 边界 | [auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go) | L111-L178 |
| MCP 专用 API Key 中间件 | [api_key_auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/api_key_auth.go) | L29-L51 |
| 主动登出清理 | [user_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/controller/user_controller.go) | L244-L249 |
| 修改角色全量登出 | [user_backyard.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go) | L220-L233 |
| 修改密码其他设备登出 | [user_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go) | L310 |
| 状态变更全量登出 | [user_backyard.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go) | L231, L393 |
| Token 清理实现 | [auth.go (repo)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/auth/auth.go) | L210-L239 |
| SetUserStatus 写入场景1 | [user_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go) | L631 |
| SetUserStatus 写入场景2 | [user_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/content/user_service.go) | L766 |
| SetUserStatus 写入场景3 | [user_backyard_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/user/user_backyard_repo.go) | L80 |
| UUIDv7 Token 生成 | [token.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/pkg/token/token.go) | L24-L28 |
| CheckPrivateMode 中间件 | [auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go) | L221-L236 |
| shouldLoginRequired Guard | [guard.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/ui/src/utils/guard.ts) | L265-L281 |
| 前端白名单路径 | [guard.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/ui/src/utils/guard.ts) | L102-L113 |
| 前端初始化 setupApp | [guard.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/ui/src/utils/guard.ts) | L417-L442 |
| 状态回写与竞态点 | [auth.go (service)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go) | L71-L81 |
