# 访问控制系统深度分析（Deep Dive）

## 一、纠正：匿名访问的真实判定路径

### 1.1 之前的错误理解

> ❌ 错误：匿名用户走到声望门槛检查

### 1.2 真实的权限判定决策树

```
用户请求操作
    │
    ├─ 未登录（匿名）
    │   │
    │   ├─ 路由组是 mustUnAuthV1 / unAuthV1？
    │   │   ├─ 是 → ✅ 匿名放行（不进入服务层权限检查）
    │   │   └─ 否 → MustAuth* 中间件拦截 → 401 Unauthorized
    │   │
    │   └─ 业务代码中 CheckOperationPermission(userID="", ...)
    │       └─ len(userID) == 0 → 直接 return false, nil ❌ 无权限
    │          （根本不会走到声望检查！）
    │
    └─ 已登录 → Token 有效
        │
        ├─ MustAuthWithoutAccountAvailable
        │   ├─ Token 是 API Key (sk_ 前缀)？ → AuthAPIKeyScope 分支
        │   └─ 普通 Token → 只校验是否存在 + 非删除状态
        │
        ├─ MustAuthAndAccountAvailable
        │   ├─ Token 是 API Key (sk_ 前缀)？ → AuthAPIKeyScope 分支
        │   └─ 普通 Token → 邮箱已验证 + 未停用 + 未删除
        │
        └─ AdminAuth
            └─ 管理员缓存 + 与普通 Token 状态联动刷新
                ├─ 邮箱未验证 → 403
                ├─ 已停用 → 403
                └─ 已删除 → 401
```

### 1.3 关键代码证据

**RankService.CheckOperationPermission 入口** [rank_service.go L88-L92](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/rank/rank_service.go#L88-L92)：

```go
func (rs *RankService) CheckOperationPermission(ctx context.Context, userID string, action string, objectID string) (can bool, err error) {
    // 匿名用户 userID 为空，直接返回 false，不会走声望检查
    if len(userID) == 0 {
        return false, nil
    }
    // ... 后续逻辑只有已登录用户才会走到
}
```

**同理批量检查** [rank_service.go L124-L130](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/rank/rank_service.go#L124-L130)：

```go
func (rs *RankService) CheckOperationPermissionsForRanks(ctx context.Context, userID string, actions []string) (
    can []bool, requireRanks []int, err error) {
    can = make([]bool, len(actions))
    requireRanks = make([]int, len(actions))
    // 匿名用户直接返回全 false + 全 0 requireRanks
    if len(userID) == 0 {
        return can, requireRanks, nil
    }
    // ...
}
```

---

## 二、AuthAPIKeyScope 返回语义深度解析

### 2.1 函数签名与返回值

```go
func (am *AuthUserMiddleware) AuthAPIKeyScope(ctx *gin.Context, accessToken string) (apiHaveNoScope bool)
```

### 2.2 返回值语义真值表

| 返回值 | 含义 | 后续行为 | 调用方位置 |
|-------|------|---------|-----------|
| `false` | **不是** API Key（无 `sk_` 前缀），或者 API Key 校验通过 → 让流程继续按普通 Token 走 | `ctx.Next()` 走后续普通 Token 校验逻辑 | MustAuth* 系列 |
| `true` | **是** API Key，但校验失败（不存在/越权/报错）→ 请求已在内部被 Abort | 直接 return，不再走普通 Token 校验 | MustAuth* 系列 |

### 2.3 调用位置与上下文

**MustAuthWithoutAccountAvailable 中** [auth.go L111-L137](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L111-L137)：

```go
func (am *AuthUserMiddleware) MustAuthWithoutAccountAvailable() gin.HandlerFunc {
    return func(ctx *gin.Context) {
        token := ExtractToken(ctx)
        if len(token) == 0 {
            handler.HandleResponse(ctx, errors.Unauthorized(reason.UnauthorizedError), nil)
            ctx.Abort()
            return
        }
        // 关键点：如果是 API Key 且通过校验，内部不会 Abort，返回 false，继续走
        // 如果是 API Key 但校验失败，内部已 Abort，返回 true，这里直接 return
        if am.AuthAPIKeyScope(ctx, token) {
            return  // 已在 AuthAPIKeyScope 内部处理了 403 响应
        }
        // 只有非 API Key 的普通 Token 才走到这里
        userInfo, err := am.authService.GetUserCacheInfo(ctx, token)
        // ... 普通 Token 校验
    }
}
```

### 2.4 AuthAPIKeyScope 内部执行流

```
accessToken 传入
    │
    ├─ !strings.HasPrefix(accessToken, "sk_")
    │   └─ return false  → 不是 API Key，让调用方继续走普通 Token 校验
    │
    └─ 是 sk_ 开头（API Key）
        │
        ├─ AuthService.AuthAPIKey(...) 调用
        │   │
        │   ├─ err != nil → 返回 403 Forbidden，ctx.Abort()，return true
        │   ├─ pass == false → 返回 403 Forbidden，ctx.Abort()，return true
        │   └─ pass == true → return false（API Key 校验通过，不 Abort，走 ctx.Next()）
        │
        └─ 返回 true（已拦截）或 false（放行）
```

---

## 三、API Key 在 MustAuth 的边界

### 3.1 边界概览

| 中间件 | API Key 校验位置 | 普通 Token 校验 | API Key 通过后注入用户信息？ |
|--------|----------------|----------------|---------------------------|
| `Auth()` | ❌ 不处理 API Key | 有 Token 则注入 | ❌ |
| `MustAuthWithoutAccountAvailable()` | ✅ AuthAPIKeyScope | ✅ GetUserCacheInfo（但不检查邮箱验证/停用） | ❌ API Key 通过后 **不会** 注入 UserCacheInfo |
| `MustAuthAndAccountAvailable()` | ✅ AuthAPIKeyScope | ✅ GetUserCacheInfo（全量检查） | ❌ 同上 |
| `AdminAuth()` | ❌ 不支持 API Key | ✅ GetAdminUserCacheInfo | ❌ API Key 无法访问 Admin 路由 |
| `AuthAPIKey()`（MCP 专用） | ✅ 必须是 API Key | ❌ 不支持普通 Token | ❌ 只校验 Key 有效性 |

### 3.2 关键设计：API Key 通过后不注入 UserCacheInfo

**影响 1**：后续业务代码调用 `GetLoginUserIDFromContext(ctx)` 会返回空字符串 `""`

```go
func GetLoginUserIDFromContext(ctx *gin.Context) (userID string) {
    userInfo := GetUserInfoFromContext(ctx)  // API Key 场景下为 nil
    if userInfo == nil {
        return ""  // ⚠️ 业务层需注意 API Key 访问拿不到 userID
    }
    return userInfo.UserID
}
```

**影响 2**：API Key 走的是"旁路"，通过后直接 `ctx.Next()`，不执行普通 Token 后面的 `ctx.Set(ctxUUIDKey, userInfo)` 逻辑。

### 3.3 AuthAPIKeyScope 与 AuthAPIKey() 的区别

| 维度 | AuthAPIKeyScope | AuthAPIKey() |
|------|----------------|-------------|
| 调用位置 | MustAuth* 系列内部，作为普通 Token 的旁路替代 | MCP 路由组独立中间件 |
| Token 类型判断 | 先判断 `sk_` 前缀，非 API Key 直接放行 | 不判断前缀，任何 Token 缺失直接 401 |
| 校验失败行为 | 返回 403 Forbidden | 返回 401 Unauthorized |
| 注入用户信息 | 否 | 否 |

---

## 四、双令牌机制（AccessToken + VisitToken）

### 4.1 两种令牌的定位

| 令牌 | 生成方式 | 存储位置 | 用途 | 有效期 |
|------|---------|---------|------|--------|
| **AccessToken** | `token.GenerateToken()` | HTTP Header `Authorization: Bearer <token>` | API 接口调用认证 | 7 天 |
| **VisitToken** | `token.GenerateToken()` | Cookie (`visit`) | 静态文件/图片访问（私有模式下） | 7 天 |

### 4.2 令牌生成与写入缓存 [auth.go L93-L102](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go#L93-L102)

```go
func (as *AuthService) SetUserCacheInfo(ctx context.Context, userInfo *entity.UserCacheInfo) (
    accessToken string, visitToken string, err error) {
    accessToken = token.GenerateToken()  // 随机字符串
    visitToken = token.GenerateToken()   // 另一个随机字符串
    err = as.authRepo.SetUserCacheInfo(ctx, accessToken, visitToken, userInfo)
    return accessToken, visitToken, err
}
```

### 4.3 双缓存写入 [auth.go (repo) L63-L86](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/auth/auth.go#L63-L86)

```go
func (ar *authRepo) SetUserCacheInfo(ctx context.Context,
    accessToken, visitToken string, userInfo *entity.UserCacheInfo) (err error) {

    userInfo.VisitToken = visitToken  // VisitToken 存入 UserCacheInfo

    // 缓存 1：AccessToken → UserCacheInfo
    // Key: answer:user:token:<accessToken>
    // Value: JSON(UserCacheInfo)
    // TTL: 7 days
    ar.data.Cache.SetString(ctx, constant.UserTokenCacheKey+accessToken,
        string(userInfoCache), constant.UserTokenCacheTime)

    // 缓存 2：VisitToken → AccessToken（反向映射）
    // Key: answer:user:visit:<visitToken>
    // Value: <accessToken>
    // TTL: 7 days
    ar.data.Cache.SetString(ctx, constant.UserVisitTokenCacheKey+visitToken,
        accessToken, constant.UserTokenCacheTime)

    // 缓存 3：UserID → [AccessToken1, AccessToken2, ...]（用于批量登出）
    // Key: answer:user-token:mapping:<userID>
    // Value: JSON(map[accessToken]bool)
    ar.AddUserTokenMapping(ctx, userInfo.UserID, accessToken)
}
```

### 4.4 VisitToken 使用场景（静态资源保护）

**VisitAuth 中间件** [visit_img_auth.go L31-L66](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/visit_img_auth.go#L31-L66)：

```go
func (am *AuthUserMiddleware) VisitAuth() gin.HandlerFunc {
    return func(ctx *gin.Context) {
        // 品牌图片公开访问，跳过
        if strings.HasPrefix(ctx.Request.URL.Path, "/uploads/branding/") {
            ctx.Next()
            return
        }

        // 非私有模式下，图片公开访问
        siteSecurity, _ := am.siteInfoCommonService.GetSiteSecurity(ctx)
        if !siteSecurity.LoginRequired {
            ctx.Next()
            return
        }

        // 私有模式下，必须带 visit Cookie
        visitToken, _ := ctx.Cookie(constant.UserVisitCookiesCacheKey) // Cookie 名: "visit"
        if len(visitToken) == 0 {
            ctx.Redirect(http.StatusFound, "/403")
            return
        }

        // VisitToken → AccessToken 反向查找
        if !am.authService.CheckUserVisitToken(ctx, visitToken) {
            ctx.Redirect(http.StatusFound, "/403")
            return
        }
        ctx.Next()
    }
}
```

**VisitToken 校验链路**：

```
Cookie: visit=<visitToken>
    │
    ▼
Cache: answer:user:visit:<visitToken> → 找到 AccessToken？
    ├─ 否 → 403
    └─ 是 → 放行
```

### 4.5 路由中的双令牌保护 [http.go L76-L78](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/server/http.go#L76-L78)

```go
static := r.Group(uiConf.APIBaseURL)
// 静态资源路由使用 VisitAuth（基于 Cookie 的 VisitToken）
static.Use(avatarMiddleware.AvatarThumb(), authUserMiddleware.VisitAuth())
staticRouter.RegisterStaticRouter(static)
```

---

## 五、Token 双缓存与状态动态刷新机制

### 5.1 缓存层次结构

```
┌─────────────────────────────────────────────────────────────┐
│                     缓存层（Cache Layer）                      │
├─────────────────────────────────────────────────────────────┤
│  ① answer:user:token:<AccessToken>                           │
│    └─ Value: UserCacheInfo {UserID, Status, Email, RoleID}  │
│    └─ TTL: 7 days                                            │
│                                                              │
│  ② answer:user:visit:<VisitToken>                            │
│    └─ Value: <AccessToken>（反向映射）                        │
│    └─ TTL: 7 days                                            │
│                                                              │
│  ③ answer:user-token:mapping:<UserID>                        │
│    └─ Value: {AccessToken1: true, AccessToken2: true, ...}  │
│    └─ TTL: 7 days                                            │
│                                                              │
│  ④ answer:user:status:<UserID>                               │
│    └─ Value: UserCacheInfo（**最新状态**）                     │
│    └─ TTL: 7 days                                            │
│    └─ 👉 管理员修改用户状态/角色时写入                         │
│                                                              │
│  ⑤ answer:admin:token:<AccessToken>                          │
│    └─ Value: UserCacheInfo（管理员独立缓存）                   │
│    └─ TTL: 7 days                                            │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 状态动态刷新：用户登录时的状态合并

**GetUserCacheInfo 核心逻辑** [auth.go (service) L63-L91](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go#L63-L91)：

```go
func (as *AuthService) GetUserCacheInfo(ctx context.Context, accessToken string) (userInfo *entity.UserCacheInfo, err error) {
    // Step 1: 从 ① 号缓存读取 Token 对应的用户信息
    userCacheInfo, err := as.authRepo.GetUserCacheInfo(ctx, accessToken)
    if userCacheInfo == nil {
        return nil, nil
    }

    // Step 2: 检查 ④ 号缓存是否有待生效的状态变更
    cacheInfo, _ := as.authRepo.GetUserStatus(ctx, userCacheInfo.UserID)
    if cacheInfo != nil {
        // 用最新状态覆盖 Token 缓存中的旧状态
        userCacheInfo.UserStatus = cacheInfo.UserStatus
        userCacheInfo.EmailStatus = cacheInfo.EmailStatus
        userCacheInfo.RoleID = cacheInfo.RoleID

        // 回写刷新 ① 号缓存（延长生命周期 + 更新状态）
        err := as.authRepo.SetUserCacheInfo(ctx, accessToken, userCacheInfo.VisitToken, userCacheInfo)
    }

    // Step 3: 如有 UserCenter 插件，再从外部用户中心同步最新状态
    uc, ok := plugin.GetUserCenter()
    if ok && len(userCacheInfo.ExternalID) > 0 {
        if userStatus := uc.UserStatus(userCacheInfo.ExternalID); userStatus != plugin.UserStatusAvailable {
            userCacheInfo.UserStatus = int(userStatus)  // 外部状态覆盖本地
        }
    }
    return userCacheInfo, nil
}
```

### 5.3 状态变更的写入点

当管理员修改用户状态/角色时：

```
管理员操作（如停用用户、修改角色）
    │
    ▼
调用 AuthService.SetUserStatus(ctx, userInfo)
    │
    ▼
写入 ④ answer:user:status:<UserID> 缓存
    │
    ▼
下次用户用任意 AccessToken 访问时
    → GetUserCacheInfo 读取到 ④ 号缓存有内容
    → 合并新状态到 UserCacheInfo
    → 回写刷新 ① 号缓存
```

### 5.4 管理员 Token 的特殊同步

**GetAdminUserCacheInfo** [auth.go (service) L147-L181](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go#L147-L181)：

```go
func (as *AuthService) GetAdminUserCacheInfo(ctx context.Context, accessToken string) (userInfo *entity.UserCacheInfo, err error) {
    // Step 1: 读取管理员独立缓存 ⑤
    adminCacheInfo, err := as.authRepo.GetAdminUserCacheInfo(ctx, accessToken)
    if adminCacheInfo == nil {
        return nil, nil
    }

    // Step 2: 调用 GetUserCacheInfo 获取普通用户侧的最新状态
    //         （内部已包含 ④ 号缓存 + UserCenter 同步）
    refreshedUserCacheInfo, err := as.GetUserCacheInfo(ctx, accessToken)
    if refreshedUserCacheInfo == nil {
        // 普通 Token 已失效 → 管理员 Token 也一并失效
        as.authRepo.RemoveAdminUserCacheInfo(ctx, accessToken)
        return nil, nil
    }

    // Step 3: 用刷新后的普通用户状态合并到管理员缓存
    adminCacheInfo.UserStatus = refreshedUserCacheInfo.UserStatus
    adminCacheInfo.EmailStatus = refreshedUserCacheInfo.EmailStatus
    adminCacheInfo.RoleID = refreshedUserCacheInfo.RoleID

    // Step 4: 回写刷新管理员缓存 ⑤
    as.authRepo.SetAdminUserCacheInfo(ctx, accessToken, adminCacheInfo)
    return adminCacheInfo, nil
}
```

**设计意图**：管理员会话与普通用户会话生命周期绑定，用户被停用/删除后管理员权限也同步失效。

### 5.5 批量登出：UserTokenMapping 的作用

**RemoveUserTokens** [auth.go (repo) L210-L239](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/auth/auth.go#L210-L239)：

```go
func (ar *authRepo) RemoveUserTokens(ctx context.Context, userID string, remainToken string) {
    // 从 ③ 号缓存读取该用户的所有 AccessToken
    key := constant.UserTokenMappingCacheKey + userID
    resp, _, _ := ar.data.Cache.GetString(ctx, key)
    mapping := make(map[string]bool, 0)
    json.Unmarshal([]byte(resp), &mapping)

    // 遍历删除每个 Token 的 ① 号缓存（可选保留当前 Token）
    for token := range mapping {
        if token == remainToken {
            continue  // "其他设备登出" 场景保留当前 Token
        }
        ar.RemoveUserCacheInfo(ctx, token)
    }

    // 清理 ④ 号状态缓存 + ③ 号映射缓存
    ar.RemoveUserStatus(ctx, userID)
    ar.data.Cache.Del(ctx, key)
}
```

对应 Service 层 API：
- `RemoveUserAllTokens(ctx, userID)` → 全部登出（包括当前）
- `RemoveTokensExceptCurrentUser(ctx, userID, accessToken)` → 其他设备登出

---

## 六、UserCenter 插件挂钩机制

### 6.1 UserCenter 插件接口

[user_center.go L22-L44](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/plugin/user_center.go#L22-L44)：

```go
type UserCenter interface {
    Description() UserCenterDesc              // 功能开关描述
    LoginCallback(ctx *GinContext)             // 登录回调
    SignUpCallback(ctx *GinContext)            // 注册回调
    UserInfo(externalID string)                // 获取用户信息
    UserStatus(externalID string) UserStatus   // 👉 获取用户最新状态（每次认证时调用）
    UserList(externalIDs []string)
    UserSettings(externalID string)
    PersonalBranding(externalID string)
    AfterLogin(externalID, accessToken string)
}
```

### 6.2 功能开关矩阵

[user_center.go L46-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/plugin/user_center.go#L46-L58)：

```go
type UserCenterDesc struct {
    RankAgentEnabled          bool  // 外部声望代理
    UserStatusAgentEnabled    bool  // 外部用户状态代理（认证时同步）
    UserRoleAgentEnabled      bool  // 外部角色代理
    MustAuthEmailEnabled      bool  // 强制邮箱验证
    EnabledOriginalUserSystem bool  // 是否保留原始用户系统（密码、邮箱登录等）
}
```

### 6.3 认证流程中的 UserCenter 挂钩点

**挂钩点 1：用户状态实时同步** [auth.go (service) L83-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go#L83-L89)：

```go
uc, ok := plugin.GetUserCenter()
if ok && len(userCacheInfo.ExternalID) > 0 {
    // 每次读取缓存时，都从外部用户中心拉取最新状态
    if userStatus := uc.UserStatus(userCacheInfo.ExternalID); userStatus != plugin.UserStatusAvailable {
        userCacheInfo.UserStatus = int(userStatus)
        // 外部状态优先于本地状态
    }
}
```

**挂钩点 2：禁用原始用户系统 API** [user_center_plugin_auth.go L30-L42](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/user_center_plugin_auth.go#L30-L42)：

```go
func BanAPIForUserCenter(ctx *gin.Context) {
    uc, ok := plugin.GetUserCenter()
    if !ok {
        return  // 未启用 UserCenter 插件，不拦截
    }
    // 启用了 UserCenter 但禁用了原始用户系统
    if !uc.Description().EnabledOriginalUserSystem {
        handler.HandleResponse(ctx, errors.Forbidden(reason.ForbiddenError), nil)
        ctx.Abort()
        return
    }
    ctx.Next()
}
```

被 BanAPIForUserCenter 保护的接口（原始用户系统功能）：
- `POST /user/login/email` 邮箱密码登录
- `POST /user/register/email` 邮箱注册
- `POST /user/email/verification` 邮箱验证
- `PUT /user/email` 修改邮箱
- `POST /user/password/reset` 重置密码
- `POST /user/password/replacement` 重置密码确认
- `PUT /user/password` 修改密码
- `POST /user/email/change/code` 发送邮箱变更验证码
- `POST /user/email/verification/send` 重发邮箱验证邮件

---

## 七、RateLimit 防重放机制

### 7.1 DuplicateRequestRejection 算法

[rate_limit.go L46-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/rate_limit.go#L46-L65)：

```go
func (rm *RateLimitMiddleware) DuplicateRequestRejection(ctx *gin.Context, req any) (reject bool, key string) {
    userID := GetLoginUserIDFromContext(ctx)
    fullPath := ctx.FullPath()
    reqJson, _ := json.Marshal(req)
    // MD5(userID + fullPath + reqBody) 作为幂等键
    key = encryption.MD5(fmt.Sprintf("%s:%s:%s", userID, fullPath, string(reqJson)))

    // 检查缓存中是否已存在该键
    reject, err = rm.limitRepo.CheckAndRecord(ctx, key)
    if reject {
        handler.HandleResponse(ctx, errors.BadRequest(reason.DuplicateRequestError), nil)
        return true, key
    }
    return false, key
}
```

### 7.2 底层缓存实现

[limit.go L46-L60](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/limit/limit.go#L46-L60)：

```go
func (lr *LimitRepo) CheckAndRecord(ctx context.Context, key string) (limit bool, err error) {
    // Key: answer:rate-limit:<MD5Hash>
    _, exist, err := lr.data.Cache.GetString(ctx, constant.RateLimitCacheKeyPrefix+key)
    if exist {
        return true, nil  // 键已存在 → 重复请求
    }
    // 写入缓存，5 分钟后自动过期
    err = lr.data.Cache.SetString(ctx, constant.RateLimitCacheKeyPrefix+key,
        fmt.Sprintf("%d", time.Now().Unix()), constant.RateLimitCacheTime) // TTL=5分钟
    return false, nil
}
```

### 7.3 Controller 中的标准用法模式

以 [question_controller.go L390-L398](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/controller/question_controller.go#L390-L398) 为例：

```go
func (qc *QuestionController) AddQuestion(ctx *gin.Context) {
    req := &schema.QuestionAdd{}
    if handler.BindAndCheck(ctx, req) {
        return
    }

    // Step 1: 防重检查（在业务逻辑执行前）
    reject, rejectKey := qc.rateLimitMiddleware.DuplicateRequestRejection(ctx, req)
    if reject {
        return  // 已返回 400 DuplicateRequestError
    }

    // Step 2: 执行业务逻辑
    resp, err := qc.questionService.AddQuestion(ctx, req)

    // Step 3: 失败时清除防重记录（允许重试）
    if err != nil {
        qc.rateLimitMiddleware.DuplicateRequestClear(ctx, rejectKey)
    }

    handler.HandleResponse(ctx, err, resp)
}
```

### 7.4 已接入防重的接口

| Controller | 接口 | 说明 |
|-----------|------|------|
| QuestionController | AddQuestion | 发布问题 |
| AnswerController | AddAnswer | 发布回答 |
| CommentController | AddComment | 发布评论 |
| ImporterService | (注释中) | 导入功能（预留） |

---

## 八、CheckPrivateMode 与私有模式双保险

### 8.1 CheckPrivateMode 中间件

[auth.go L221-L236](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L221-L236)：

```go
func (am *AuthUserMiddleware) CheckPrivateMode() gin.HandlerFunc {
    return func(ctx *gin.Context) {
        resp, err := am.siteInfoCommonService.GetSiteSecurity(ctx)
        if err != nil {
            ShowIndexPage(ctx)  // 读不到配置就渲染前端 SPA
            ctx.Abort()
            return
        }
        if resp.LoginRequired {
            // 私有模式 → 前端接管，由前端路由守卫判断跳转
            ShowIndexPage(ctx)
            ctx.Abort()
            return
        }
        ctx.Next()
    }
}
```

### 8.2 私有模式的两层保护机制

```
浏览器访问 SPA 页面（非 API 请求）
    │
    ▼
Layer 1: CheckPrivateMode 中间件（UI 路由层）
    ├─ 私有模式 → 返回 index.html（前端 SPA），前端再根据登录态跳转登录页
    └─ 公开模式 → 放行
    │
    ▼
浏览器/客户端访问 API 接口
    │
    ▼
Layer 2: EjectUserBySiteInfo 中间件（API 路由层）
    ├─ 私有模式 + 未登录 → 401 Unauthorized（JSON 响应）
    └─ 公开模式 → 放行
    │
    ▼
访问静态资源（图片/上传文件）
    │
    ▼
Layer 3: VisitAuth 中间件
    ├─ 私有模式 + 无 visit Cookie → 302 重定向到 /403
    └─ 公开模式 → 放行
```

### 8.3 EjectUserBySiteInfo 与 CheckPrivateMode 的区别

| 维度 | EjectUserBySiteInfo | CheckPrivateMode |
|------|--------------------|-----------------|
| 应用路由组 | `unAuthV1`（API 接口） | UI 模板路由（页面访问） |
| 未登录私有模式行为 | 返回 401 JSON（API 友好） | 返回 index.html（SPA 接管） |
| 检查账户状态 | ✅ 检查邮箱验证状态 | ❌ 只检查 LoginRequired 开关 |
| 对已登录用户 | 检查账户状态，异常返回 403 | 一律放行，不检查账户状态 |

---

## 九、完整认证时序图（含所有深度细节）

```
HTTP Request
    │
    ├─ 1. ExtractToken(ctx)
    │   ├─ Header: Authorization → Bearer <token>
    │   └─ Query: ?Authorization=<token>
    │
    ├─ 2. 路由分组匹配（决定使用哪套中间件链）
    │   │
    │   ├─ mustUnAuthV1 → 无认证中间件 → 直接放行
    │   │
    │   ├─ unAuthV1
    │   │   ├─ Auth() → 有 Token 则尝试注入 UserCacheInfo（失败不报错）
    │   │   └─ EjectUserBySiteInfo() → 私有模式 + 未登录 → 401
    │   │
    │   ├─ authWithoutStatusV1
    │   │   └─ MustAuthWithoutAccountAvailable()
    │   │       ├─ 无 Token → 401
    │   │       ├─ sk_ 前缀 → AuthAPIKeyScope()
    │   │       │   ├─ Key 无效/越权 → 403 Abort → return true
    │   │       │   └─ Key 有效 → return false → 不会注入 UserInfo
    │   │       └─ 普通 Token → GetUserCacheInfo()
    │   │           ├─ ① answer:user:token:<token> 缓存读取
    │   │           ├─ ④ answer:user:status:<uid> 状态刷新 → 回写 ①
    │   │           ├─ UserCenter.UserStatus() 外部同步
    │   │           └─ 状态检查：仅禁止删除状态 (status=10 → 401)
    │   │
    │   ├─ authV1
    │   │   └─ MustAuthAndAccountAvailable()
    │   │       ├─ 无 Token → 401
    │   │       ├─ sk_ 前缀 → AuthAPIKeyScope()（同上）
    │   │       └─ 普通 Token → GetUserCacheInfo() + 全量状态检查
    │   │           ├─ 邮箱未验证 (EmailStatus=2) → 403 inactive
    │   │           ├─ 用户已停用 (Status=9) → 403 user_suspended
    │   │           ├─ 用户已删除 (Status=10) → 401
    │   │           └─ 全部正常 → ctx.Set(ctxUUIDKey, userInfo) 注入
    │   │
    │   ├─ adminauthV1
    │   │   └─ AdminAuth()
    │   │       └─ GetAdminUserCacheInfo()
    │   │           ├─ ⑤ answer:admin:token:<token> 读取
    │   │           └─ 联动调用 GetUserCacheInfo() 同步最新状态
    │   │               └─ 状态异常时 RemoveAdminUserCacheInfo() + 401/403
    │   │
    │   └─ mcpAPIGroup
    │       ├─ AuthMcpEnable() → MCP 功能未开启 → 403
    │       └─ AuthAPIKey() → 必须是有效 API Key
    │           └─ 无 Token/Key 无效 → 401
    │
    ├─ 3. 业务层权限检查（仅已登录场景）
    │   └─ RankService.CheckOperationPermission()
    │       ├─ userID 为空 → return false（匿名直接拒绝，不走声望）
    │       ├─ 角色 Power 检查 → 有权限 → return true
    │       ├─ 资源 Owner 检查 → 创建者本人 → return true
    │       └─ 声望 Rank 检查 → userRank >= requireRank → return true
    │
    ├─ 4. RateLimit 防重（写操作前）
    │   └─ DuplicateRequestRejection(MD5(userID+path+body))
    │       ├─ 缓存已存在 → 400 DuplicateRequestError
    │       └─ 业务失败 → DuplicateRequestClear(key) 允许重试
    │
    └─ 5. 响应返回
```

---

## 十、关键文件索引

| 功能模块 | 文件 | 关键行 |
|---------|------|--------|
| AuthAPIKeyScope 返回语义 | [auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go) | L238-L255 |
| MustAuth* 中 API Key 旁路 | [auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go) | L111-L178 |
| MCP 专用 API Key 中间件 | [api_key_auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/api_key_auth.go) | L29-L51 |
| 双令牌 VisitAuth | [visit_img_auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/visit_img_auth.go) | L31-L66 |
| Token 双缓存写入 | [auth.go (repo)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/auth/auth.go) | L63-L86 |
| 状态动态刷新 | [auth.go (service)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go) | L63-L91 |
| 管理员缓存联动刷新 | [auth.go (service)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go) | L147-L181 |
| 批量登出实现 | [auth.go (repo)](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/auth/auth.go) | L210-L239 |
| UserCenter 接口定义 | [user_center.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/plugin/user_center.go) | L22-L127 |
| UserCenter 禁用原始 API | [user_center_plugin_auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/user_center_plugin_auth.go) | L30-L42 |
| 匿名用户快速短路 | [rank_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/rank/rank_service.go) | L88-L92 |
| 防重放算法 | [rate_limit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/rate_limit.go) | L46-L73 |
| 防重放缓存层 | [limit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/repo/limit/limit.go) | L46-L65 |
| CheckPrivateMode | [auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go) | L221-L236 |
| EjectUserBySiteInfo | [auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go) | L80-L108 |
| 路由分组总览 | [http.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/server/http.go) | L74-L121 |
| 缓存 Key 常量 | [cache_key.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/constant/cache_key.go) | L24-L54 |
