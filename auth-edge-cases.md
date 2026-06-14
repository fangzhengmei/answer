# 认证系统 Edge Case 深度剖析

## 一、AdminAuth 下 sk_ 实际拿 403 不是 401

### 1.1 之前的错误结论

> ❌ 之前的理解：`adminauthV1` → `AdminAuth()` → API Key 走不通 → ❌ 永远 401

这个结论虽然最终结果是"不可用"，但 HTTP 状态码是错的。实际拿的是 **403 Forbidden**，不是 401 Unauthorized。

### 1.2 真实执行流

[auth.go L180-L219](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L180-L219)：

```
API Key 请求进入 AdminAuth()
    │
    ├─ token = "sk_abc123..."
    │
    ├─ GetAdminUserCacheInfo(ctx, "sk_abc123...")
    │   │
    │   ├─ Step 1: 读 ⑤ answer:admin:token:sk_abc123...
    │   │   └─ 不存在 → adminCacheInfo = nil
    │   │
    │   ├─ Step 2: if adminCacheInfo == nil → return nil, nil
    │   │
    │   └─ 返回 (nil, nil)
    │
    └─ if err != nil || userInfo == nil  ← userInfo == nil 命中
        │
        └─ handler.HandleResponse(ctx, errors.Forbidden(reason.UnauthorizedError), nil)
                                  ^^^^^^^^^
                                  这里是 403 Forbidden！不是 401！
```

### 1.3 各中间件对 sk_ 的状态码对比

| 中间件 | Token 来源 | 失败原因 | 实际 HTTP 状态码 | 代码位置 |
|-------|-----------|---------|----------------|---------|
| `MustAuthWithoutAccountAvailable()` | `Authorization: sk_xxx` | fallthrough 到 `GetUserCacheInfo` 查不到 → 401 | **401 Unauthorized** | [auth.go L124-L127](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L124-L127) |
| `MustAuthAndAccountAvailable()` | `Authorization: sk_xxx` | 同上 | **401 Unauthorized** | [auth.go L153-L156](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L153-L156) |
| `AdminAuth()` | `Authorization: sk_xxx` | `GetAdminUserCacheInfo` 返回 nil → 403 | **403 Forbidden** | [auth.go L189-L192](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L189-L192) |
| `AuthAPIKey()` (MCP) | `Authorization: sk_xxx`（有效） | 通过 | **200 OK** | [api_key_auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/api_key_auth.go) |
| `AuthAPIKey()` (MCP) | `Authorization: sk_xxx`（无效） | Key 不存在/不匹配 | **401 Unauthorized** | [api_key_auth.go L38-L41](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/api_key_auth.go#L38-L41) |
| `AuthAPIKeyScope` 内部拦截 | MustAuth* 中 | Key 越权/错误 | **403 Forbidden** | [auth.go L248-L255](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L248-L255) |

### 1.4 为什么 AdminAuth 用 403 而不是 401

代码注释没有说明，但从语义上看：
- 401 = "你没有身份"（未认证）
- 403 = "你有身份，但权限不足"

AdminAuth 假定调用者应该是管理员，拿不到管理员缓存就视为"权限不足"。但实际上 API Key 的场景下调用者**根本没有用户身份**，这是一个语义不匹配的设计。

---

## 二、UpdateUserStatus 的冗余链路：SetUserStatus 先写再被全清

### 2.1 完整调用链路

管理员通过后台 API 修改用户状态时，实际执行流如下：

```
Controller: PUT /admin/user/status
    │
    ▼
Service: UserAdminService.UpdateUserStatus [user_backyard.go L128-L179]
    │
    ├─ 1. 校验权限（不能改自己、用户不能是已删除）
    │
    ├─ 2. 组装 userStatus / mailStatus
    │
    ├─ 3. 调用 userRepo.UpdateUserStatus(...)
    │       │
    │       ▼
    │     Repo: user_backyard_repo.UpdateUserStatus [user_backyard_repo.go L54-L85]
    │       │
    │       ├─ 3a. 更新数据库（xorm Update）
    │       │
    │       └─ 3b. ⭐ SetUserStatus(ctx, userID, userCacheInfo)
    │              │   写入 ④ answer:user:status:<userID>
    │              │   TTL = 7 天
    │              │
    │              └─ 期望：下次用户访问时刷新状态
    │
    ├─ 4. 可选：删除用户所有内容（RemoveAllContent）
    │
    ├─ 5. 可选：删除用户所有配置（IsDeleted 时）
    │
    └─ 6. 可选：UserActive（恢复正常且声望为 0 时）
```

### 2.2 问题：UpdateUserStatus 没有调用 RemoveUserAllTokens

**之前的分析有误**。实际看 [user_backyard.go L128-L179](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go#L128-L179)：

```go
func (us *UserAdminService) UpdateUserStatus(ctx context.Context, req *schema.UpdateUserStatusReq) (err error) {
    // ... 省略校验和数据库更新 ...
    
    // 注意：这里完全没有 RemoveUserAllTokens 调用！
    // 只有 UpdateUserRole 和 UpdateUserPassword 才会调用 RemoveUserAllTokens。
    // UpdateUserStatus 不踢用户下线！
}
```

**真正调用 RemoveUserAllTokens 的是这三个场景**：

| 方法 | 调用 RemoveUserAllTokens | 代码位置 |
|-----|------------------------|---------|
| `UpdateUserRole` | ✅ 是 | [user_backyard.go L231](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go#L231) |
| `UpdateUserPassword` | ✅ 是 | [user_backyard.go L393](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go#L393) |
| `UpdateUserStatus` | ❌ 否 | [user_backyard.go L128-L179](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go#L128-L179) |

### 2.3 正确的链路描述

所以实际链路不是"先写再被清"，而是：

```
管理员修改用户状态（如：停用）
    │
    ├─ 1. user_backyard_repo.UpdateUserStatus
    │   ├─ 数据库 UPDATE status=suspended
    │   └─ ④ answer:user:status:<userID> = {Status: 9, EmailStatus: 1}
    │       （写入状态缓存，不踢用户下线）
    │
    └─ 用户下次任意请求访问
        │
        ├─ GetUserCacheInfo(accessToken)
        │   ├─ ① → 旧状态 {Status: 1}
        │   ├─ ④ → 新状态 {Status: 9}
        │   ├─ 合并：Status = 9
        │   └─ 回写刷新 ①
        │
        └─ MustAuthAndAccountAvailable()
            └─ userInfo.UserStatus == Suspended (9)
                └─ 403 Forbidden (UserSuspended)
```

**这才是设计意图**：修改用户状态不立即踢下线，而是让用户下次请求时通过状态缓存合并感知到变化，返回 403。

### 2.4 但确实存在的冗余

虽然没有"先写再被全清"的问题，但 `UpdateUserPassword` 和 `UpdateUserRole` 中存在真正的冗余链路：

**UpdateUserRole 冗余链路**：

```
UpdateUserRole(userID, RoleID=2)
    │
    ├─ 1. SaveUserRole(userID, 2) → 数据库更新
    │
    └─ 2. RemoveUserAllTokens(userID)
        │
        ├─ 删除所有 ① answer:user:token:*
        ├─ 删除 ③ answer:user-token:mapping:<userID>
        └─ 删除 ④ answer:user:status:<userID>
```

但这里 **④ answer:user:status 从未在 UpdateUserRole 之前被写入**，所以删除 ④ 是无意义的空操作。

只有在 `UpdateUserStatus` + `UpdateUserRole` 连续操作（先改状态再改角色）时，RemoveUserAllTokens 才会清理掉 SetUserStatus 写入的 ④ 号缓存。这种极端场景下存在"写了马上删"的冗余。

---

## 三、Logout 三清实际有三清 + 之前的分析错误修正

### 3.1 之前的错误理解

> ❌ 之前的理解：Logout 只清理了 ① 和 ⑤，漏掉了 ② VisitToken

实际上让我重新看 [user_controller.go L240-L251](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/controller/user_controller.go#L240-L251)：

```go
func (uc *UserController) UserLogout(ctx *gin.Context) {
    accessToken := middleware.ExtractToken(ctx)
    if len(accessToken) == 0 {
        handler.HandleResponse(ctx, nil, nil)
        return
    }
    // 清 1: ① answer:user:token:<AccessToken>
    _ = uc.authService.RemoveUserCacheInfo(ctx, accessToken)
    // 清 2: ⑤ answer:admin:token:<AccessToken>
    _ = uc.authService.RemoveAdminUserCacheInfo(ctx, accessToken)
    // 清 3: ② answer:user:visit:<VisitToken>
    visitToken, _ := ctx.Cookie(constant.UserVisitCookiesCacheKey)
    _ = uc.authService.RemoveUserVisitCacheInfo(ctx, visitToken)
    
    handler.HandleResponse(ctx, nil, nil)
}
```

**实际是三清，没有遗漏！** 三条清理：

| 序号 | 清理内容 | 缓存 Key 模式 | 代码 |
|-----|---------|-------------|------|
| 1 | AccessToken → 用户缓存 | `answer:user:token:<AccessToken>` | `RemoveUserCacheInfo` |
| 2 | 管理员独立缓存 | `answer:admin:token:<AccessToken>` | `RemoveAdminUserCacheInfo` |
| 3 | VisitToken → AccessToken 反向映射 | `answer:user:visit:<VisitToken>` | `RemoveUserVisitCacheInfo` |

### 3.2 真正遗漏的清理

虽然 Logout 的三清是完整的，但批量登出（`RemoveUserAllTokens` / `RemoveTokensExceptCurrent`）中确实**遗漏了 ② VisitToken**：

[auth.go (repo) L211-L239](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/auth/auth.go#L211-L239)：

```go
func (ar *authRepo) RemoveUserTokens(ctx context.Context, userID string, remainToken string) {
    // 1. 读 ③ 获取所有 AccessToken
    key := constant.UserTokenMappingCacheKey + userID
    resp, _, _ := ar.data.Cache.GetString(ctx, key)
    mapping := make(map[string]bool, 0)
    json.Unmarshal([]byte(resp), &mapping)

    // 2. 遍历删除每个 AccessToken 的 ① 号缓存
    for token := range mapping {
        if token == remainToken {
            continue
        }
        ar.RemoveUserCacheInfo(ctx, token)  // 删 ①
        // ⚠️  遗漏：没有删除对应 AccessToken 的 VisitToken（② 号缓存）
        // ⚠️  遗漏：没有删除对应 AccessToken 的 ⑤ 号管理员缓存
    }

    // 3. 删除 ④ 号状态缓存
    ar.RemoveUserStatus(ctx, userID)

    // 4. 删除 ③ 号映射缓存
    ar.data.Cache.Del(ctx, key)
}
```

**批量登出的遗漏项**：

| 缓存 | Logout 单独登出 | RemoveUserTokens 批量登出 | 说明 |
|-----|---------------|------------------------|------|
| ① `user:token:*` | ✅ 清理当前 | ✅ 清理所有（除保留） | AccessToken → UserCacheInfo |
| ② `user:visit:*` | ✅ 清理当前 | ❌ **未清理** | VisitToken → AccessToken |
| ③ `user-token:mapping` | ❌ 不清理 | ✅ 清理 | UserID → [AccessToken 列表] |
| ④ `user:status:*` | ❌ 不清理 | ✅ 清理 | UserID → 最新状态 |
| ⑤ `admin:token:*` | ✅ 清理当前 | ❌ **未清理** | AccessToken → AdminCacheInfo |

**后果**：
1. 用户 A 在设备 1 登录，获得 `AccessToken=A1`，`VisitToken=V1`
2. 用户 A 在设备 2 修改密码 → 触发 `RemoveTokensExceptCurrent`（保留 A2，删除 A1）
3. 设备 1 的 `A1` 被删除了，但 `V1` 还在
4. 私有模式下，设备 1 访问图片时携带 Cookie `visit=V1`
5. `VisitAuth` 中间件查到 `V1 → A1`（② 号缓存仍在）→ 放行
6. 但设备 1 调 API 时用 `A1` 已被删 → 401

**不一致状态**：静态资源能访问（VisitToken 残留），但 API 不通（AccessToken 已删）。

---

## 四、unAuthV1 私有模式下 sk_ 实际 401

### 4.1 unAuthV1 中间件链

路由配置 [http.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/server/http.go)：

```go
unAuthV1 := r.Group(uiConf.APIBaseURL + "/answer/api/v1")
unAuthV1.Use(authUserMiddleware.Auth(), authUserMiddleware.EjectUserBySiteInfo())
answerRouter.RegisterUnAuthAnswerAPIRouter(unAuthV1)
```

执行顺序：`Auth()` → `EjectUserBySiteInfo()`

### 4.2 私有模式下携带 sk_ 的真实执行流

```
请求：GET /answer/api/v1/siteinfo
Header: Authorization: Bearer sk_abc123...
配置：LoginRequired = true（私有模式）
    │
    ▼
Step 1: Auth() 中间件 [auth.go L60-L77]
    │
    ├─ token = "sk_abc123..."
    │
    ├─ GetUserCacheInfo(ctx, "sk_abc123...")
    │   └─ 查不到 → userInfo = nil
    │
    ├─ userInfo == nil → 不 ctx.Set
    │
    └─ ctx.Next() → 继续执行
    │
    ▼
Step 2: EjectUserBySiteInfo() 中间件 [auth.go L80-L107]
    │
    ├─ LoginRequired = true
    │
    ├─ GetUserInfoFromContext(ctx) → nil（Auth() 没注入）
    │
    └─ userInfo == nil → 401 Unauthorized
         handler.HandleResponse(ctx, errors.Unauthorized(reason.UnauthorizedError), nil)
         ctx.Abort()
```

### 4.3 结论矩阵

| 模式 | Token | unAuthV1 结果 | 说明 |
|-----|-------|--------------|------|
| 私有模式 | 无 Token | ❌ 401 | 正常：私有模式需登录 |
| 私有模式 | 有效 AccessToken | ✅ 200 | Auth() 注入用户，Eject 放行 |
| 私有模式 | 无效 AccessToken | ❌ 401 | Auth() 查不到，Eject 拦截 |
| 私有模式 | 有效 sk_ API Key | ❌ **401** | Auth() 不识别 sk_，不注入用户，Eject 拦截 |
| 公开模式 | 有效 sk_ API Key | ✅ 200 | 但 Auth() 未注入用户，业务层拿不到 userID |

**核心问题**：`Auth()` 中间件完全没有 API Key 支持。它只知道调 `GetUserCacheInfo`，对 `sk_` 前缀毫无感知。所以在 unAuthV1 路由组中，API Key 要么被当作"无效用户 Token"（私有模式 401），要么被当作"匿名用户"（公开模式 userID 为空）。

---

## 五、GetAdminUserCacheInfo 清管理员缓存的 Fail-Safe

### 5.1 完整实现

[auth.go (service) L147-L181](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go#L147-L181)：

```go
func (as *AuthService) GetAdminUserCacheInfo(ctx context.Context, accessToken string) (userInfo *entity.UserCacheInfo, err error) {
    // Step 1: 读管理员独立缓存 ⑤
    adminCacheInfo, err := as.authRepo.GetAdminUserCacheInfo(ctx, accessToken)
    if adminCacheInfo == nil {
        return nil, nil  // 管理员缓存不存在 → 不是管理员
    }

    // Step 2: ⭐ Fail-Safe：联动普通用户 Token 的生命周期和状态
    refreshedUserCacheInfo, err := as.GetUserCacheInfo(ctx, accessToken)
    if err != nil {
        return nil, err
    }

    // Step 3: 如果普通用户 Token 已失效（被删/过期/不存在）
    if refreshedUserCacheInfo == nil {
        // ⭐ Fail-Safe 触发：立即清理管理员缓存！
        if err = as.authRepo.RemoveAdminUserCacheInfo(ctx, accessToken); err != nil {
            return nil, err
        }
        return nil, nil  // 假装管理员缓存也不存在
    }

    // Step 4: 用普通用户的最新状态覆盖管理员缓存
    adminCacheInfo.UserStatus = refreshedUserCacheInfo.UserStatus
    adminCacheInfo.EmailStatus = refreshedUserCacheInfo.EmailStatus
    if refreshedUserCacheInfo.RoleID > 0 {
        adminCacheInfo.RoleID = refreshedUserCacheInfo.RoleID
    }
    if len(refreshedUserCacheInfo.ExternalID) > 0 {
        adminCacheInfo.ExternalID = refreshedUserCacheInfo.ExternalID
    }

    // Step 5: 回写刷新管理员缓存 ⑤
    if err = as.authRepo.SetAdminUserCacheInfo(ctx, accessToken, adminCacheInfo); err != nil {
        return nil, err
    }
    return adminCacheInfo, nil
}
```

### 5.2 Fail-Safe 的设计意图

管理员缓存（⑤）和普通用户缓存（①）是**独立的 Redis Key**。如果只清理了 ① 而没清理 ⑤，管理员可能"偷偷"保持登录状态。

**场景**：
1. 管理员登录，同时生成 ① 和 ⑤ 两个缓存
2. 管理员去"其他设备登出" → `RemoveTokensExceptCurrent` 只清理了 ①（如之前分析，批量登出没清理 ⑤）
3. 如果没有 Fail-Safe：设备 2 的 ⑤ 还在 → 直接 `GetAdminUserCacheInfo` 读到 → 继续当管理员
4. 有 Fail-Safe：`GetUserCacheInfo` 发现 ① 已失效 → 主动 `RemoveAdminUserCacheInfo` 清理 ⑤

### 5.3 管理员缓存的三重清理保证

| 清理场景 | 触发方 | 清理 ⑤ 的机制 |
|---------|-------|--------------|
| 主动登出 | 用户 | `Logout` → `RemoveAdminUserCacheInfo` |
| 管理员被改角色 | `UpdateUserRole` | `RemoveUserAllTokens` → **未清理 ⑤**（漏洞！） |
| 管理员被改状态 | `UpdateUserStatus` | 不踢下线，下次访问 Fail-Safe 感知状态变化 |
| 管理员被改密码 | `UpdateUserPassword` | `RemoveUserAllTokens` → **未清理 ⑤**（漏洞！） |
| 普通用户 Token 过期 | Redis TTL | 下次访问 Fail-Safe 主动清理 ⑤ |
| 状态异常（邮箱未验证/停用/删除） | `AdminAuth` 中间件 | `RemoveAdminUserCacheInfo` |

### 5.4 AdminAuth 中间件中的 Fail-Safe 清理

除了 `GetAdminUserCacheInfo` 内部的 Fail-Safe，`AdminAuth` 中间件在检测到状态异常时也会主动清理：

[auth.go L194-L214](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L194-L214)：

```go
if userInfo != nil {
    if userInfo.EmailStatus == entity.EmailStatusToBeVerified {
        // ⭐ Fail-Safe：邮箱未验证 → 清除管理员缓存
        _ = am.authService.RemoveAdminUserCacheInfo(ctx, token)
        handler.HandleResponse(ctx, errors.Forbidden(reason.EmailNeedToBeVerified), ...)
        return
    }
    if userInfo.UserStatus == entity.UserStatusSuspended {
        // ⭐ Fail-Safe：已停用 → 清除管理员缓存
        _ = am.authService.RemoveAdminUserCacheInfo(ctx, token)
        handler.HandleResponse(ctx, errors.Forbidden(reason.UserSuspended), ...)
        return
    }
    if userInfo.UserStatus == entity.UserStatusDeleted {
        // ⭐ Fail-Safe：已删除 → 清除管理员缓存
        _ = am.authService.RemoveAdminUserCacheInfo(ctx, token)
        handler.HandleResponse(ctx, errors.Unauthorized(reason.UnauthorizedError), nil)
        return
    }
}
```

**但注意**：这三处清理都是"清理当前 Token 的管理员缓存"，不是清理用户的全部管理员缓存。如果用户有多个设备，每个设备的管理员缓存需要各自触发一次状态检查才能被清理。

---

## 六、Edge Case 总览矩阵

| # | Edge Case | 之前的理解 | 真实情况 | 代码证据 |
|---|-----------|-----------|---------|---------|
| 1 | AdminAuth 下 sk_ 的状态码 | 401 Unauthorized | **403 Forbidden** | [auth.go L189-L192](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L189-L192) |
| 2 | UpdateUserStatus 调用链路 | 先 SetUserStatus 再 RemoveUserAllTokens（冗余） | **只 SetUserStatus，不 RemoveUserAllTokens**（不踢下线，状态缓存合并） | [user_backyard.go L128-L179](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go#L128-L179) |
| 3 | SetUserStatus 在 user_backyard_repo 中的位置 | 没注意到 | 实际在 `user_backyard_repo.UpdateUserStatus` 中写入 ④ 号缓存 | [user_backyard_repo.go L73-L83](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/user/user_backyard_repo.go#L73-L83) |
| 4 | Logout 清理项 | 二清，漏了 VisitToken | **三清**，AccessToken + AdminToken + VisitToken 都清理了 | [user_controller.go L246-L249](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/controller/user_controller.go#L246-L249) |
| 5 | RemoveUserAllTokens 清理项 | 以为清理了所有 | **漏了 ② VisitToken 和 ⑤ AdminToken** | [auth.go (repo) L211-L239](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/auth/auth.go#L211-L239) |
| 6 | unAuthV1 私有模式下 sk_ 的结果 | 未分析 | **401 Unauthorized**（Auth() 不识别 sk_，Eject 拦截） | [auth.go L60-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L60-L107) |
| 7 | GetAdminUserCacheInfo Fail-Safe | 未深入 | 普通用户 Token 失效时主动清理管理员缓存，AdminAuth 状态异常时也清理 | [auth.go (service) L156-L166](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go#L156-L166) |

---

## 七、关键文件索引

| Edge Case | 文件 | 关键行 |
|----------|------|--------|
| AdminAuth 下 sk_ 拿 403 | [auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go) | L189-L192 |
| UpdateUserStatus SetUserStatus 写入 | [user_backyard_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/user/user_backyard_repo.go) | L73-L83 |
| UpdateUserStatus 不调用 RemoveUserAllTokens | [user_backyard.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go) | L128-L179 |
| UpdateUserRole / UpdateUserPassword 调用 RemoveUserAllTokens | [user_backyard.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/user_admin/user_backyard.go) | L231, L393 |
| Logout 三清 | [user_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/controller/user_controller.go) | L240-L251 |
| RemoveUserTokens 漏清理 VisitToken 和 AdminToken | [auth.go (repo)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/auth/auth.go) | L211-L239 |
| unAuthV1 Auth() 不识别 sk_ | [auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go) | L60-L77 |
| unAuthV1 EjectUserBySiteInfo 私有模式拦截 | [auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go) | L80-L107 |
| GetAdminUserCacheInfo Fail-Safe | [auth.go (service)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go) | L147-L181 |
| AdminAuth 状态异常时清理缓存 | [auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go) | L194-L214 |
