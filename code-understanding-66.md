# 访问控制与权限判定系统代码理解

## 一、系统架构概述

Apache Answer 采用**多层级权限控制模型**，从外到内依次为：

```
┌─────────────────────────────────────────────────────────────┐
│                     HTTP 请求入口                             │
├─────────────────────────────────────────────────────────────┤
│  1. 中间件层（Middleware）                                    │
│     ├─ Token 提取与解析                                       │
│     ├─ 登录态认证（Auth）                                     │
│     ├─ API Key 认证                                           │
│     └─ 站点私有模式检查                                       │
├─────────────────────────────────────────────────────────────┤
│  2. 路由层（Router）                                          │
│     ├─ 完全公开路由（无需登录）                                │
│     ├─ 可选登录路由（有登录态则注入）                          │
│     ├─ 必须登录路由（活跃用户）                                │
│     ├─ 必须登录路由（任意状态用户）                            │
│     └─ 管理员路由                                             │
├─────────────────────────────────────────────────────────────┤
│  3. 服务层（Service）                                         │
│     ├─ 角色权限（Role-based Power）                           │
│     ├─ 声望权限（Rank-based Permission）                      │
│     └─ 资源所有者权限（Object Owner）                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、核心概念区分

### 2.1 登录态（Authentication）

**定义**：验证用户身份的过程，确认"你是谁"。

**实现位置**：[auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go)

#### 核心方法：

| 中间件方法 | 作用 | 适用场景 |
|-----------|------|---------|
| `Auth()` | 尝试解析 Token，有则注入用户信息，无则继续 | 公开页面，支持登录用户个性化 |
| `MustAuthWithoutAccountAvailable()` | 必须登录，但不检查用户状态（激活/停用） | 登出、邮箱验证等操作 |
| `MustAuthAndAccountAvailable()` | 必须登录且账户状态正常（邮箱已验证、未被停用） | 业务操作接口 |
| `AdminAuth()` | 必须是管理员身份 | 后台管理接口 |
| `EjectUserBySiteInfo()` | 站点私有模式下强制登录 | 全站访问控制 |

#### Token 提取机制 [L321-L327](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L321-L327)：

```go
func ExtractToken(ctx *gin.Context) (token string) {
    token = ctx.GetHeader("Authorization")
    if len(token) == 0 {
        token = ctx.Query("Authorization")
    }
    return strings.TrimPrefix(token, "Bearer ")
}
```

#### 用户缓存信息结构 [auth_user_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/entity/auth_user_entity.go)：

```go
type UserCacheInfo struct {
    UserID      string `json:"user_id"`      // 用户ID
    UserStatus  int    `json:"user_status"`  // 用户状态：1=正常, 9=停用, 10=删除
    EmailStatus int    `json:"email_status"` // 邮箱状态：1=已验证, 2=待验证
    RoleID      int    `json:"role_id"`      // 角色ID
    ExternalID  string `json:"external_id"`  // 外部系统ID
    VisitToken  string `json:"visit_token"`  // 访问令牌
}
```

#### 账户状态检查流程：

1. **邮箱状态检查** [L158-L163](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L158-L163)
   - `EmailStatusAvailable` (1)：邮箱已验证，正常访问
   - `EmailStatusToBeVerified` (2)：邮箱待验证，返回 403 Forbidden

2. **用户状态检查** [L164-L174](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L164-L174)
   - `UserStatusAvailable` (1)：正常用户
   - `UserStatusSuspended` (9)：已停用，返回 403 Forbidden
   - `UserStatusDeleted` (10)：已删除，返回 401 Unauthorized

---

### 2.2 角色（Role）

**定义**：用户的身份分组，不同角色拥有不同的默认权限集合。

**实现位置**：[role_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/role/role_service.go)

#### 系统内置角色：

| 角色ID | 角色名称 | 标识常量 | 说明 |
|-------|---------|---------|------|
| 1 | 普通用户 | `RoleUserID` | 默认角色，新用户自动分配 |
| 2 | 管理员 | `RoleAdminID` | 系统管理员，拥有最高权限 |
| 3 | 版主 | `RoleModeratorID` | 内容审核人员 |

#### 角色分配机制 [user_role_rel_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/role/user_role_rel_service.go)：

```go
func (us *UserRoleRelService) GetUserRole(ctx context.Context, userID string) (roleID int, err error) {
    rolePowerRel, exist, err := us.userRoleRelRepo.GetUserRoleRel(ctx, userID)
    if err != nil {
        return 0, err
    }
    if !exist {
        return 1, nil  // 默认角色为普通用户
    }
    return rolePowerRel.RoleID, nil
}
```

**关键点**：用户未分配角色时，默认使用角色ID=1（普通用户）。

#### 角色权限关联 [role_power_rel_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/role/role_power_rel_service.go)：

```go
func (rs *RolePowerRelService) GetUserPowerList(ctx context.Context, userID string) (powers []string, err error) {
    roleID, err := rs.userRoleRelService.GetUserRole(ctx, userID)
    if err != nil {
        return nil, err
    }
    return rs.rolePowerRelRepo.GetRolePowerTypeList(ctx, roleID)
}
```

---

### 2.3 资源级权限（Resource Permission）

**定义**：用户对特定资源（问题、回答、评论等）执行特定操作的权限。

**实现位置**：[rank_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/rank/rank_service.go)

#### 权限检查三层模型：

```
权限检查请求
      │
      ▼
┌─────────────────────┐
│  1. 角色权限检查     │  用户角色是否拥有该操作权限（Power）
└─────────────────────┘
      │ 通过
      ▼
┌─────────────────────┐
│  2. 资源所有者检查   │  用户是否为资源创建者
└─────────────────────┘
      │ 通过
      ▼
┌─────────────────────┐
│  3. 声望门槛检查     │  用户声望是否满足操作要求（Rank）
└─────────────────────┘
      │
      ▼
  权限判定结果
```

#### 核心检查方法 [L88-L121](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/rank/rank_service.go#L88-L121)：

```go
func (rs *RankService) CheckOperationPermission(ctx context.Context, userID string, action string, objectID string) (can bool, err error) {
    // 1. 获取用户基本信息
    userInfo, exist, err := rs.userCommon.GetUserBasicInfoByID(ctx, userID)
    
    // 2. 检查角色权限（Power）
    powerMapping := rs.getUserPowerMapping(ctx, userID)
    if powerMapping[action] {
        return true, nil
    }

    // 3. 检查资源所有权
    if len(objectID) > 0 {
        objectInfo, err := rs.objectInfoService.GetInfo(ctx, objectID)
        if objectInfo != nil && objectInfo.ObjectCreatorUserID == userID {
            return true, nil
        }
    }

    // 4. 检查声望门槛（Rank）
    can, _ = rs.checkUserRank(ctx, userInfo.ID, userInfo.Rank, PermissionPrefix+action)
    return can, nil
}
```

#### 权限名称定义 [permission_name.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/permission/permission_name.go)：

| 分类 | 权限常量 | 说明 |
|-----|---------|------|
| 问题 | `QuestionAdd` | 提问 |
| | `QuestionEdit` | 编辑问题 |
| | `QuestionDelete` | 删除问题 |
| | `QuestionClose` | 关闭问题 |
| | `QuestionVoteUp/Down` | 问题投票 |
| 回答 | `AnswerAdd` | 回答 |
| | `AnswerEdit` | 编辑回答 |
| | `AnswerDelete` | 删除回答 |
| | `AnswerAccept` | 采纳回答 |
| 评论 | `CommentAdd` | 添加评论 |
| | `CommentEdit` | 编辑评论 |
| | `CommentDelete` | 删除评论 |
| 标签 | `TagAdd` | 创建标签 |
| | `TagEdit` | 编辑标签 |
| | `TagDelete` | 删除标签 |
| 审核 | `QuestionAudit` | 审核问题 |
| | `AnswerAudit` | 审核回答 |
| | `TagAudit` | 审核标签 |
| 管理 | `AdminAccess` | 访问管理后台 |

#### 声望（Rank）权限配置 [privilege.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/constant/privilege.go)：

声望权限是可配置的，每个操作对应一个声望门槛值，存储在系统配置中：

```go
const (
    RankQuestionAddKey               = "rank.question.add"
    RankQuestionEditKey              = "rank.question.edit"
    RankQuestionDeleteKey            = "rank.question.delete"
    // ... 约30种声望权限
)
```

**声望检查逻辑** [L246-L260](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/rank/rank_service.go#L246-L260)：

```go
func (rs *RankService) checkUserRank(ctx context.Context, userID string, userRank int, action string) (can bool, rank int) {
    requireRank, err := rs.configService.GetIntValue(ctx, action)
    if userRank < requireRank || requireRank < 0 {
        return false, requireRank
    }
    return true, requireRank
}
```

---

### 2.4 匿名访问（Anonymous Access）

**定义**：未登录用户可以访问的接口和页面。

**实现位置**：[http.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/server/http.go)

#### 路由分组策略：

| 路由组 | 中间件 | 说明 | 示例接口 |
|-------|--------|------|---------|
| `mustUnAuthV1` | 无 | 完全公开，绝对不能要求登录 | 登录、注册、邮箱验证 |
| `unAuthV1` | `Auth()` + `EjectUserBySiteInfo()` | 支持匿名，但私有模式下需登录 | 查看问题、搜索、用户主页 |

#### 路由注册示例 [L81-L87](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/server/http.go#L81-L87)：

```go
// 完全公开路由（无需登录）
mustUnAuthV1 := r.Group(uiConf.APIBaseURL + "/answer/api/v1")
answerRouter.RegisterMustUnAuthAnswerAPIRouter(authUserMiddleware, mustUnAuthV1)

// 可选登录路由（有登录态则注入用户信息）
unAuthV1 := r.Group(uiConf.APIBaseURL + "/answer/api/v1")
unAuthV1.Use(authUserMiddleware.Auth(), authUserMiddleware.EjectUserBySiteInfo())
answerRouter.RegisterUnAuthAnswerAPIRouter(unAuthV1)
```

#### 完全公开接口列表 [answer_api_router.go L143-L166](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/router/answer_api_router.go#L143-L166)：

```go
func (a *AnswerAPIRouter) RegisterMustUnAuthAnswerAPIRouter(...) {
    // i18n
    r.GET("/language/config", ...)
    r.GET("/language/options", ...)
    // siteinfo
    r.GET("/siteinfo", ...)
    r.GET("/siteinfo/legal", ...)
    // user
    r.GET("/user/info", ...)
    r.POST("/user/login/email", ...)
    r.POST("/user/register/email", ...)
    // ...
}
```

#### 站点私有模式检查 [L80-L108](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L80-L108)：

```go
func (am *AuthUserMiddleware) EjectUserBySiteInfo() gin.HandlerFunc {
    return func(ctx *gin.Context) {
        siteInfo, _ := am.siteInfoCommonService.GetSiteSecurity(ctx)
        if siteInfo != nil && siteInfo.LoginRequired {
            userInfo := GetUserInfoFromContext(ctx)
            if userInfo == nil {
                handler.HandleResponse(ctx, errors.Unauthorized(reason.UnauthorizedError), nil)
                ctx.Abort()
                return
            }
        }
        ctx.Next()
    }
}
```

---

### 2.5 密钥访问（API Key Access）

**定义**：使用 API Key 进行身份认证的访问方式，主要用于服务间调用和自动化集成。

**实现位置**：[api_key_auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/api_key_auth.go)

#### API Key 实体结构 [api_key_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/entity/api_key_entity.go)：

```go
type APIKey struct {
    ID          int       `xorm:"id"`
    AccessKey   string    `xorm:"access_key"`   // sk_ 开头的密钥
    Scope       string    `xorm:"scope"`        // read-only / full-access
    UserID      string    `xorm:"user_id"`      // 所属用户
    Description string    `xorm:"description"`  // 描述
    LastUsedAt  time.Time `xorm:"last_used_at"` // 最后使用时间
}
```

#### API Key 识别规则 [auth.go L238-L255](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go#L238-L255)：

```go
func (am *AuthUserMiddleware) AuthAPIKeyScope(ctx *gin.Context, accessToken string) bool {
    // API Key 以 sk_ 前缀标识
    if !strings.HasPrefix(accessToken, "sk_") {
        return false
    }
    // 根据请求方法和 Key 权限范围校验
    pass, err := am.authService.AuthAPIKey(ctx, ctx.Request.Method == "GET", accessToken)
    // ...
}
```

#### API Key 认证逻辑 [auth.go L191-L206](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go#L191-L206)：

```go
func (as *AuthService) AuthAPIKey(ctx context.Context, read bool, apiKey string) (pass bool, err error) {
    apiKeyInfo, exist, err := as.apiKeyRepo.GetAPIKey(ctx, apiKey)
    if !exist {
        return false, nil
    }
    // 只读权限不能执行写操作
    if !read && apiKeyInfo.Scope == "read-only" {
        log.Warnf("API key %s does not have write permissions", apiKeyInfo.AccessKey)
        return false, nil
    }
    return true, nil
}
```

#### API Key 专属中间件 [api_key_auth.go L29-L51](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/api_key_auth.go#L29-L51)：

```go
func (am *AuthUserMiddleware) AuthAPIKey() gin.HandlerFunc {
    return func(ctx *gin.Context) {
        token := ExtractToken(ctx)
        if len(token) == 0 {
            handler.HandleResponse(ctx, errors.Unauthorized(...), nil)
            ctx.Abort()
            return
        }
        pass, err := am.authService.AuthAPIKey(ctx, ctx.Request.Method == "GET", token)
        // ... 校验结果处理
    }
}
```

#### MCP 路由的 API Key 认证 [http.go L118-L120](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/server/http.go#L118-L120)：

```go
mcpAPIGroup := r.Group(uiConf.APIBaseURL + "/answer/api/v1")
mcpAPIGroup.Use(authUserMiddleware.AuthMcpEnable(), authUserMiddleware.AuthAPIKey())
answerRouter.RegisterMCPRouter(mcpAPIGroup)
```

---

## 三、完整权限判定流程

### 3.1 请求处理流水线

```
HTTP 请求
    │
    ▼
1. Token 提取（Header/Query）
    │
    ▼
2. 路由匹配 → 确定路由组
    ├─ 公开路由 → 跳过认证
    ├─ 可选登录 → Auth() 尝试注入用户
    ├─ 必须登录 → MustAuth* 系列校验
    ├─ 管理员路由 → AdminAuth() 校验
    └─ MCP 路由 → AuthAPIKey() 校验
    │
    ▼
3. 站点私有模式检查（EjectUserBySiteInfo）
    └─ 私有模式 + 匿名用户 → 401 Unauthorized
    │
    ▼
4. 业务层权限检查（Controller → Service）
    ├─ 角色权限检查（Power）
    ├─ 资源所有者检查
    └─ 声望门槛检查（Rank）
    │
    ▼
5. 操作执行 / 拒绝
```

### 3.2 权限判定决策树

```
用户请求操作
    │
    ├─ 未登录 → 匿名访问检查
    │   ├─ 接口支持匿名 → 检查声望门槛（通常为-1，禁止）
    │   └─ 接口需要登录 → 401 Unauthorized
    │
    └─ 已登录 → 账户状态检查
        ├─ 邮箱未验证 → 403 Forbidden（需激活）
        ├─ 用户已停用 → 403 Forbidden（已停用）
        ├─ 用户已删除 → 401 Unauthorized
        └─ 状态正常 → 角色权限检查
            ├─ 角色拥有该 Power → ✅ 通过
            └─ 角色无该 Power → 资源所有者检查
                ├─ 是资源创建者 → ✅ 通过
                └─ 非资源创建者 → 声望门槛检查
                    ├─ 声望 ≥ 要求 → ✅ 通过
                    └─ 声望 < 要求 → ❌ 权限不足
```

---

## 四、关键代码文件索引

| 模块 | 文件路径 | 核心功能 |
|-----|---------|---------|
| 认证中间件 | [auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/auth.go) | 登录态校验、用户信息注入 |
| API Key 中间件 | [api_key_auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/api_key_auth.go) | API Key 认证 |
| MCP 中间件 | [mcp_auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/middleware/mcp_auth.go) | MCP 功能开关检查 |
| 认证服务 | [auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/auth/auth.go) | Token 管理、API Key 校验 |
| 角色服务 | [role_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/role/role_service.go) | 角色管理 |
| 用户角色关联 | [user_role_rel_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/role/user_role_rel_service.go) | 用户-角色绑定 |
| 角色权限关联 | [role_power_rel_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/role/role_power_rel_service.go) | 角色-权限绑定 |
| 权限检查 | [rank_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/rank/rank_service.go) | 三层权限检查逻辑 |
| 权限常量 | [permission_name.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/service/permission/permission_name.go) | 权限名称定义 |
| 声望常量 | [privilege.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/constant/privilege.go) | 声望权限 Key 定义 |
| 路由配置 | [http.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/base/server/http.go) | 路由分组与中间件绑定 |
| API 路由 | [answer_api_router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/router/answer_api_router.go) | 接口注册 |
| 权限控制器 | [permission_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/controller/permission_controller.go) | 权限查询接口 |
| 用户实体 | [user_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/entity/user_entity.go) | 用户状态常量 |
| API Key 实体 | [api_key_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/entity/api_key_entity.go) | API Key 结构 |
| 用户缓存实体 | [auth_user_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/internal/entity/auth_user_entity.go) | 登录用户缓存信息 |

---

## 五、设计特点与最佳实践

### 5.1 分层权限设计优势

1. **灵活性**：角色权限、资源所有权、声望门槛三层检查，适应不同场景
2. **可配置性**：声望门槛通过系统配置动态调整，无需修改代码
3. **扩展性**：支持插件扩展用户中心、权限代理等
4. **性能优化**：用户信息缓存，避免重复查询

### 5.2 安全设计要点

1. **最小权限原则**：默认角色（普通用户）权限最小，高级权限需显式分配
2. **状态联动**：用户状态（邮箱验证、停用、删除）直接影响权限
3. **API Key 隔离**：支持只读权限，降低密钥泄露风险
4. **私有模式**：支持全站私有，保护敏感内容

### 5.3 前端路由守卫

前端配合使用 [RouteGuard.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/66-answer/ui/src/router/RouteGuard.tsx) 进行页面级权限检查，通过 `onEnter` 钩子在路由切换前验证权限，实现前后端双重保护。
