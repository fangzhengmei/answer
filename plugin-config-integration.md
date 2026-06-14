# 插件配置与用户中心集成深度理解

本文档围绕 Apache Answer 项目的扩展点机制、运行时配置、插件发现、配置保存、用户中心集成以及 Captcha 插件接入进行系统性理解。

---

## 一、扩展点体系（Extension Points）

### 1.1 核心设计：接口即扩展点

Answer 采用 **Go 接口 + 泛型 Stack** 的方式定义扩展点。每个扩展点在 [plugin](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin) 包下都有独立文件，定义一个继承自 `Base` 的接口。

`Base` 接口是所有插件的根基，定义在 [base.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/base.go#L33-L36)：

```go
type Base interface {
    Info() Info
}
```

所有扩展点均嵌入 `Base`，形成如下层次：

| 扩展点接口 | 文件 | 职责 |
|---|---|---|
| `Config` | [config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/config.go#L118-L129) | 管理员级插件配置（字段定义 + 接收） |
| `UserConfig` | [user_config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_config.go#L22-L32) | 用户级插件配置（字段定义 + 接收） |
| `Connector` | [connector.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/connector.go#L22-L45) | 第三方 OAuth 登录连接器 |
| `UserCenter` | [user_center.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L22-L44) | 外部用户中心（登录/注册/用户同步） |
| `Captcha` | [captcha.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L22-L34) | 验证码生成与校验 |
| `Parser` | [parser.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/parser.go#L23-L25) | 文本解析 |
| `Filter` | plugin/filter.go | 内容过滤 |
| `Storage` | plugin/storage.go | 文件存储 |
| `Cache` | plugin/cache.go | 缓存策略 |
| `Search` | plugin/search.go | 搜索引擎 |
| `Agent` | plugin/agent.go | AI 代理 |
| `Notification` | plugin/notification.go | 通知推送 |
| `Reviewer` | plugin/reviewer.go | 内容审核 |
| `Embed` | plugin/embed.go | 嵌入内容 |
| `Render` | plugin/render.go | 渲染引擎 |
| `CDN` | [cdn.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/cdn.go#L39-L42) | CDN 静态资源 |
| `Importer` | plugin/importer.go | 数据导入 |
| `KVStorage` | [kv_storage.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/kv_storage.go#L318-L321) | 插件键值存储 |
| `Sidebar` | plugin/sidebar.go | 侧边栏扩展 |
| `VectorSearch` | plugin/vector_search.go | 向量搜索 |

### 1.2 泛型 Stack 与 MakePlugin 工厂

核心机制在 [plugin.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/plugin.go#L139-L179) 的 `MakePlugin` 函数：

```go
func MakePlugin[T Base](super bool) (CallFn[T], RegisterFn[T])
```

- **`super` 参数**：标记该扩展点是否为"超级"类型。`super=true` 表示插件不可被禁用（如 Base、Config、KVStorage），`super=false` 表示可被 StatusManager 控制启停。
- **返回值**：一对函数 —— `CallFn`（遍历调用已注册插件）和 `RegisterFn`（注册新插件到 Stack）。

调用 `CallFn` 时，如果 `super=false`，会先检查 `StatusManager.IsEnabled(slugName)`，被禁用的插件将被跳过。

### 1.3 Register：统一注册入口

[plugin.go#L55-L137](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/plugin.go#L55-L137) 中的 `Register(p Base)` 函数是所有插件注册的唯一入口。它通过 Go 类型断言 `p.(Config)` 等检查插件实现了哪些扩展点接口，然后分别调用对应的 `registerXxx` 函数完成注册。

这意味着：**一个插件结构体只需实现多个扩展点接口，`Register` 就会自动将它注册到所有匹配的扩展点 Stack 中。**

---

## 二、插件发现机制

### 2.1 编译期发现

Answer 采用 **编译期插件发现**，而非运行时动态加载。插件通过 Go `init()` 函数在程序启动时调用 `plugin.Register()`，将自身注册到对应扩展点的 Stack 中。

由于 Go 的 `init()` 机制要求包被 `import` 才会执行，所以插件的发现本质上是 **包导入链** 决定的。

### 2.2 运行时状态管理

虽然插件在编译期就已注册，但运行时通过 [StatusManager](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/plugin.go#L50-L52) 控制每个插件的启用/禁用状态：

```go
var StatusManager = statusManager{
    status: make(map[string]bool),
}
```

- `StatusManager.Enable(name, enabled)`：启用或禁用插件
- `StatusManager.IsEnabled(name)`：查询插件是否启用
- 状态持久化到数据库 `config` 表，key 为 `plugin.status`，值为 JSON 格式的 `map[string]bool`

**互斥协调机制**：某些扩展点同一时刻只能有一个插件激活。`StatusManager.Enable` 内部会调用 `coordinatedCaptchaPlugins` 和 `coordinatedCDNPlugins`，当启用一个 Captcha/CDN 插件时，自动禁用同类型的其他插件。详见 [captcha.go#L67-L82](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L67-L82) 和 [cdn.go#L50-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/cdn.go#L50-L65)。

### 2.3 管理端插件列表发现

管理员通过 [PluginController.GetPluginList](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller_admin/plugin_controller.go#L74-L110) 查看所有已注册插件：

1. 遍历 `CallBase` 获取所有插件基本信息（名称、版本、SlugName）
2. 遍历 `CallConfig` 标记哪些插件拥有配置表单
3. 读取 `StatusManager` 判断启用状态
4. 支持按 status（active/inactive）和 have_config 过滤

---

## 三、配置保存与运行时配置

### 3.1 配置的分层体系

Answer 的插件配置分为三层：

| 层级 | 接口 | 存储表 | 作用域 |
|---|---|---|---|
| 管理员配置 | `Config` | `plugin_config` | 全局，管理员设置 |
| 用户配置 | `UserConfig` | `plugin_user_config` | 用户级别，每个用户独立 |
| 插件内部 KV | `KVStorage` | `plugin_kv_storage` | 插件自定义数据 |

### 3.2 管理员配置（Config 接口）

**接口定义** 在 [config.go#L118-L129](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/config.go#L118-L129)：

```go
type Config interface {
    Base
    ConfigFields() []ConfigField        // 声明配置表单字段
    ConfigReceiver(config []byte) error  // 接收并处理配置数据
}
```

**ConfigField 类型系统** 支持：input、textarea、checkbox、radio、select、upload、timezone、switch、button、legend、tag_selector 等，以及 input 子类型（text、password、email、number、color 等）。

**配置保存流程**：

1. 管理员通过 `PUT /answer/admin/api/plugin/config` 提交配置
2. [PluginController.UpdatePluginConfig](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller_admin/plugin_controller.go#L204-L224) 先序列化 `configFields` 为 JSON
3. 调用 `plugin.CallConfig` 找到匹配的插件，执行 `ConfigReceiver` 让插件在内存中更新配置
4. 调用 `PluginCommonService.UpdatePluginConfig` 将配置持久化到 `plugin_config` 表

**实体层** 在 [plugin_config_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/entity/plugin_config_entity.go#L23-L27)：

```go
type PluginConfig struct {
    ID             int    `xorm:"not null pk autoincr INT(11) id"`
    PluginSlugName string `xorm:"unique VARCHAR(128) plugin_slug_name"`
    Value          string `xorm:"TEXT value"`
}
```

以 `plugin_slug_name` 为唯一键，`value` 存储 JSON 序列化的配置字段映射。

### 3.3 用户配置（UserConfig 接口）

**接口定义** 在 [user_config.go#L22-L32](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_config.go#L22-L32)：

```go
type UserConfig interface {
    Base
    UserConfigFields() []ConfigField
    UserConfigReceiver(userID string, config []byte) error
}
```

与 `Config` 的区别在于 `UserConfigReceiver` 多了 `userID` 参数，表明配置是用户维度的。

**存储实体** 在 [plugin_user_config_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/entity/plugin_user_config_entity.go#L23-L28)：

```go
type PluginUserConfig struct {
    ID             int    `xorm:"not null pk autoincr INT(11) id"`
    UserID         string `xorm:"not null default 0 BIGINT(20) UNIQUE(uk_up) user_id"`
    PluginSlugName string `xorm:"VARCHAR(128) UNIQUE(uk_up) plugin_slug_name"`
    Value          string `xorm:"TEXT value"`
}
```

`(user_id, plugin_slug_name)` 联合唯一，每个用户对每个插件只有一条配置记录。

**UserConfig 的跨层读取** 通过 [RegisterGetPluginUserConfigFunc](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_config.go#L50-L51) 实现。plugin 包不直接依赖 repo 层，而是在 `initPluginData` 时注入一个从数据库读取配置的函数。插件内部通过 `plugin.GetPluginUserConfig(userID, slugName)` 即可获取自己的用户配置。

### 3.4 插件 KV 存储（KVStorage）

[KVStorage](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/kv_storage.go#L318-L321) 提供结构化的键值存储能力：

```go
type KVStorage interface {
    Info() Info
    SetOperator(operator *KVOperator)
}
```

**KVOperator** 是核心操作类，支持：
- `Get(ctx, KVParams)` — 按 group+key 查询
- `Set(ctx, KVParams)` — 写入或更新
- `Del(ctx, KVParams)` — 按 group/key 删除
- `GetByGroup(ctx, KVParams)` — 按 group 分页查询
- `Tx(ctx, fn)` — 事务操作

**缓存策略**：默认 30 分钟 TTL，带 ±10% 随机抖动防止缓存雪崩。可通过 `WithCacheTTL` 选项自定义，设为负数禁用缓存。

**存储实体** 在 [plugin_kv_storage_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/entity/plugin_kv_storage_entity.go#L22-L28)：

```go
type PluginKVStorage struct {
    ID             int    `xorm:"not null pk autoincr INT(11) id"`
    PluginSlugName string `xorm:"not null VARCHAR(128) UNIQUE(uk_psg) plugin_slug_name"`
    Group          string `xorm:"not null VARCHAR(128) UNIQUE(uk_psg) 'group'"`
    Key            string `xorm:"not null VARCHAR(128) UNIQUE(uk_psg) 'key'"`
    Value          string `xorm:"not null TEXT value"`
}
```

`(plugin_slug_name, group, key)` 三字段联合唯一。

### 3.5 插件状态持久化

插件启停状态通过 `config` 表持久化，key 为 `plugin.status`（定义在 [plugin_config_key.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/base/constant/plugin_config_key.go#L23-L24)），值为 JSON 序列化的 `map[string]bool`。

### 3.6 初始化流程

[PluginCommonService.initPluginData](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/plugin_common/plugin_common_service.go) 在服务创建时执行，完成以下初始化：

1. **KVStorage 初始化**：为所有实现了 `KVStorage` 的插件注入 `KVOperator`（含 DB 和 Cache 实例）
2. **插件状态恢复**：从 `config` 表读取 `plugin.status`，反序列化到 `StatusManager`
3. **管理员配置恢复**：从 `plugin_config` 表读取所有插件配置，调用对应插件的 `ConfigReceiver` 在内存中恢复
4. **Cache 插件注入**：如果存在实现了 `Cache` 的插件，替换全局 Cache 实例
5. **VectorSearch 同步器注册**：为向量搜索插件注册数据同步器
6. **UserConfig 读取函数注入**：通过 `RegisterGetPluginUserConfigFunc` 注入数据库读取函数
7. **用户配置恢复**：后台 goroutine 分页加载 `plugin_user_config`，调用各插件的 `UserConfigReceiver` 恢复用户级配置

---

## 四、用户中心集成

### 4.1 UserCenter 接口

[user_center.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L22-L44) 定义了完整的用户中心扩展点：

```go
type UserCenter interface {
    Base
    Description() UserCenterDesc
    ControlCenterItems() []ControlCenter
    LoginCallback(ctx *GinContext) (userInfo *UserCenterBasicUserInfo, err error)
    SignUpCallback(ctx *GinContext) (userInfo *UserCenterBasicUserInfo, err error)
    UserInfo(externalID string) (userInfo *UserCenterBasicUserInfo, err error)
    UserStatus(externalID string) (userStatus UserStatus)
    UserList(externalIDs []string) (userInfo []*UserCenterBasicUserInfo, err error)
    UserSettings(externalID string) (userSettings *SettingInfo, err error)
    PersonalBranding(externalID string) (branding []*PersonalBranding)
    AfterLogin(externalID, accessToken string)
}
```

### 4.2 UserCenterDesc 能力声明

[UserCenterDesc](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L46-L58) 是用户中心的能力声明结构体：

| 字段 | 含义 |
|---|---|
| `Name` / `DisplayName` / `Icon` / `Url` | 用户中心基本信息 |
| `LoginRedirectURL` / `SignUpRedirectURL` | 登录/注册跳转地址 |
| `RankAgentEnabled` | 是否接管用户等级管理 |
| `UserStatusAgentEnabled` | 是否接管用户状态管理 |
| `UserRoleAgentEnabled` | 是否接管用户角色管理 |
| `MustAuthEmailEnabled` | 是否强制邮箱授权 |
| `EnabledOriginalUserSystem` | 是否保留原系统用户体系 |

这些能力声明直接影响管理后台的功能开关，见 [UserCenterAdminFunctionAgent](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/user_external_login/user_center_login_service.go#L241-L262)：

- 如果 `UserStatusAgentEnabled=true`，管理后台禁用修改用户状态
- 如果 `UserRoleAgentEnabled=true`，管理后台禁用修改用户角色
- 如果 `EnabledOriginalUserSystem=false`，管理后台禁用创建用户和修改密码

### 4.3 登录/注册回调流程

路由定义在 [plugin_api_router.go#L63-L68](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/router/plugin_api_router.go#L63-L68)：

```
GET /user-center/login/redirect   → UserCenterLoginRedirect
GET /user-center/sign-up/redirect → UserCenterSignUpRedirect
GET /user-center/login/callback   → UserCenterLoginCallback
GET /user-center/sign-up/callback → UserCenterSignUpCallback
```

**登录回调完整流程**（以 Login 为例）：

1. 前端访问 `/user-center/login/redirect`，Controller 从 `UserCenter.Description()` 获取 `LoginRedirectURL`，302 跳转到外部用户中心
2. 用户在外部用户中心完成认证后，回调到 `/user-center/login/callback`
3. [UserCenterController.UserCenterLoginCallback](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/plugin_user_center_controller.go#L132-L167) 调用 `userCenter.LoginCallback(ctx)` 获取 `UserCenterBasicUserInfo`
4. 调用 `UserCenterLoginService.ExternalLogin` 处理用户匹配/注册：
   - 如果 `externalID` 已存在 → 直接登录（生成 accessToken）
   - 如果 `externalID` 不存在 → 自动注册新用户 → 登录
   - 如果 `MustAuthEmailEnabled` 且未提供 email → 返回错误
5. 调用 `userCenter.AfterLogin(externalID, accessToken)` 通知插件登录完成
6. 302 跳转到 `/users/auth-landing?access_token=xxx`

### 4.4 Connector 与 UserCenter 的关系

两者都提供外部登录能力，但定位不同：

| 维度 | Connector | UserCenter |
|---|---|---|
| 定位 | 第三方 OAuth 登录（如 GitHub、Google） | 外部用户中心（完全接管用户体系） |
| 登录方式 | OAuth 重定向流程 | 自定义跳转/回调 |
| 用户数据 | 只提供基础信息 | 提供完整用户生命周期管理 |
| 数量限制 | 可同时启用多个 | 只能启用一个（互斥） |
| 用户状态管理 | 无 | 可接管用户状态/角色/等级 |
| 原系统保留 | 保留 | 可选择禁用原系统 |

### 4.5 用户设置与个人品牌

- **UserSettings**：通过 `GET /user-center/user/settings` 返回用户中心的重定向 URL（如个人资料设置、账号设置的跳转链接），使得用户在 Answer 中的设置页面可以跳转到外部用户中心
- **PersonalBranding**：通过 `GET /user-center/personal/branding` 返回用户在外部用户中心的品牌信息（如社交媒体链接），展示在用户个人页面

### 4.6 辅助查询函数

[user_center.go#L104-L127](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L104-L127) 提供三个便捷函数：

- `UserCenterEnabled()` — 判断是否有用户中心插件启用
- `RankAgentEnabled()` — 判断用户中心是否启用了等级代理
- `GetUserCenter()` — 获取当前启用的 UserCenter 实例

---

## 五、Captcha 插件接入

### 5.1 Captcha 接口

[captcha.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L22-L34) 定义了三个方法：

```go
type Captcha interface {
    Base
    GetConfig() (configJsonStr string)          // 返回前端需要的配置（如第三方服务 token）
    Create() (captcha, code string)             // 可选：后端生成验证码
    Verify(captchaCode, userInput string) bool  // 校验用户输入
}
```

**两种验证码模式**：

1. **后端生成模式**：实现 `Create()` 返回验证码图片 base64 和正确答案，`Verify()` 比对答案
2. **第三方服务模式**：不实现 `Create()`（返回空），`GetConfig()` 返回第三方服务配置，`Verify()` 调用第三方 API 校验

### 5.2 互斥机制

Captcha 插件是互斥的 —— 同一时刻只能有一个 Captcha 插件激活。[CallCaptcha](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L42-L57) 的实现确保只调用第一个注册的 Captcha 插件。

[coordinatedCaptchaPlugins](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L67-L82) 在启用一个 Captcha 插件时，自动禁用其他所有 Captcha 插件。

### 5.3 前端配置获取

路由 `GET /captcha/config` 由 [CaptchaController.GetCaptchaConfig](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/plugin_captcha_controller.go#L45-L53) 处理：

```go
func (uc *CaptchaController) GetCaptchaConfig(ctx *gin.Context) {
    resp := &GetCaptchaConfigResp{}
    _ = plugin.CallCaptcha(func(fn plugin.Captcha) error {
        resp.SlugName = fn.Info().SlugName
        _ = json.Unmarshal([]byte(fn.GetConfig()), &resp.Config)
        return nil
    })
    handler.HandleResponse(ctx, nil, resp)
}
```

返回当前激活的 Captcha 插件的 slug_name 和配置信息，前端据此初始化验证码组件。

### 5.4 CaptchaService — 验证策略引擎

[action/captcha_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/action/captcha_service.go) 和 [captcha_strategy.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/action/captcha_strategy.go) 实现了完整的验证码策略：

**Action 类型** 定义在 [captcha_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/entity/captcha_entity.go#L22-L35)：

| Action | 含义 | 触发策略 |
|---|---|---|
| `email` | 发送邮件 | 每次都需要验证码 |
| `password` | 修改密码 | 30 分钟内 3 次后需要 |
| `edit_userinfo` | 修改用户信息 | 30 分钟内 3 次后需要 |
| `question` | 提问 | 5 秒内或累计 10 次后需要 |
| `answer` | 回答 | 5 秒内或累计 10 次后需要 |
| `comment` | 评论 | 1 秒内或累计 30 次后需要 |
| `edit` | 编辑 | 累计 10 次后需要 |
| `invitation_answer` | 邀请回答 | 累计 30 次后需要 |
| `search` | 搜索 | 60 秒内 20 次后需要 |
| `report` | 举报 | 1 秒内或累计 30 次后需要 |
| `delete` | 删除 | 5 秒内或累计 5 次后需要 |
| `vote` | 投票 | 累计 40 次后需要 |

**核心流程**：

1. `ActionRecord` — 记录用户操作，判断是否需要验证码
2. `ValidationStrategy` — 根据操作类型和频率判断是否触发验证码
3. `GenerateCaptcha` — 调用 `plugin.CallCaptcha` 生成验证码
4. `VerifyCaptcha` — 调用 `plugin.CallCaptcha` 校验验证码
5. `ActionRecordVerifyCaptcha` — 组合验证：先判断策略，再验证验证码

**关键点**：如果 `plugin.CaptchaEnabled()` 返回 false（即没有启用任何 Captcha 插件），`ValidationStrategy` 直接返回 true（通过），相当于整个验证码机制被跳过。

### 5.5 前端 Captcha 集成

[useCaptchaModal](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/ui/src/hooks/useCaptchaModal/index.tsx) 是前端验证码的 React Hook，负责：

- 调用 `checkImgCode` API 获取验证码配置和图片
- 渲染验证码弹窗（图片 + 输入框 + 刷新按钮）
- 通过 `resolveCaptchaReq` 将 `captcha_id` 和 `captcha_code` 注入业务请求
- 通过 `handleCaptchaError` 处理后端返回的验证码错误，自动弹窗重试

### 5.6 Captcha 与业务流程的集成点

Captcha 不是独立的，而是嵌入在各个业务操作中：

1. 用户提交操作（如提问、回答）
2. 后端先调用 `CaptchaService.ActionRecord` 检查是否需要验证码
3. 如果需要，返回验证码要求（`verify: true` + `captcha_id` + `captcha_img`）
4. 前端弹出验证码弹窗
5. 用户输入验证码后，前端将 `captcha_id` 和 `captcha_code` 随请求发送
6. 后端调用 `ActionRecordVerifyCaptcha` 验证
7. 验证通过后继续执行业务逻辑
8. 操作完成后调用 `ActionRecordAdd` 增加操作计数

---

## 六、整体架构关系图

```
┌─────────────────────────────────────────────────────────────┐
│                     Admin API (/admin/api/)                  │
│  GetPluginList | UpdatePluginStatus | GetPluginConfig        │
│  UpdatePluginConfig                                          │
│                    ↕                                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │          PluginCommonService                         │    │
│  │  - initPluginData()  ← 系统启动时初始化              │    │
│  │  - UpdatePluginStatus()  ← 状态持久化到 config 表    │    │
│  │  - UpdatePluginConfig()  ← 配置持久化到 plugin_config │    │
│  │  - UpdatePluginUserConfig() ← 用户配置到             │    │
│  │                              plugin_user_config      │    │
│  └─────────────────────────────────────────────────────┘    │
│                    ↕                                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │             plugin 包 (扩展点层)                      │    │
│  │                                                     │    │
│  │  Register() → 类型断言 → 分别注册到各 Stack          │    │
│  │                                                     │    │
│  │  StatusManager ──→ 控制插件启停                      │    │
│  │  MakePlugin[T]() ──→ CallFn + RegisterFn            │    │
│  │                                                     │    │
│  │  Config    ─→ 管理员配置字段 + 接收器                │    │
│  │  UserConfig ─→ 用户配置字段 + 接收器                 │    │
│  │  UserCenter ─→ 外部用户中心集成                      │    │
│  │  Captcha   ─→ 验证码生成/校验                        │    │
│  │  Connector ─→ 第三方 OAuth 登录                      │    │
│  │  KVStorage ─→ 插件数据存储                          │    │
│  │  ...其他扩展点                                       │    │
│  └─────────────────────────────────────────────────────┘    │
│                    ↕                                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           数据持久层                                  │    │
│  │                                                     │    │
│  │  config 表          ← plugin.status 等全局配置       │    │
│  │  plugin_config 表   ← 管理员级插件配置               │    │
│  │  plugin_user_config ← 用户级插件配置                 │    │
│  │  plugin_kv_storage  ← 插件自定义 KV 数据             │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   Plugin API (/api/v1/)                      │
│                                                             │
│  /captcha/config           → CaptchaController              │
│  /connector/login/:name    → ConnectorController            │
│  /connector/redirect/:name → ConnectorController            │
│  /connector/info           → ConnectorController            │
│  /user-center/agent        → UserCenterController           │
│  /user-center/login/*      → UserCenterController           │
│  /user-center/sign-up/*    → UserCenterController           │
│  /user-center/user/settings → UserCenterController          │
│  /user/plugin/configs      → UserPluginController           │
│  /user/plugin/config       → UserPluginController           │
└─────────────────────────────────────────────────────────────┘
```

---

## 七、关键设计总结

### 7.1 扩展点设计模式

- **接口组合**：通过 Go 接口嵌入实现扩展点的灵活组合，一个插件可实现多个扩展点
- **泛型 Stack**：`MakePlugin[T]` 统一了注册和调用的模式，避免大量重复代码
- **super 标记**：区分可禁用和不可禁用的扩展点，保证核心功能不受影响

### 7.2 配置分层设计

- **管理员配置（Config）**：全局生效，影响插件行为
- **用户配置（UserConfig）**：用户维度，支持个性化
- **KV 存储（KVStorage）**：插件自定义数据，支持事务和缓存

三层配置各司其职，通过不同的表结构和 API 端点管理，互不干扰。

### 7.3 运行时配置恢复

系统启动时，`initPluginData` 按序恢复：状态 → 管理员配置 → KV → 用户配置，确保插件在接收配置前 StatusManager 已经就绪，保证 `super=false` 的插件能正确被跳过。

### 7.4 互斥协调

Captcha 和 CDN 扩展点具有互斥语义。通过 `coordinatedXxxPlugins` 函数在 `StatusManager.Enable` 时自动处理冲突，确保同一类型只有一个插件激活。

### 7.5 用户中心的代理模式

UserCenter 插件采用代理模式：它不替换原有用户体系，而是通过能力声明（`UserCenterDesc`）告诉 Answer 哪些功能由外部用户中心接管。Answer 据此在管理后台禁用相应功能，避免操作冲突。
