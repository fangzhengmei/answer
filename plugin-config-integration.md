# 插件配置与用户中心集成深度理解

本文档围绕 Apache Answer 项目的扩展点机制、运行时配置、插件发现、配置保存、用户中心集成以及 Captcha 插件接入进行系统性理解。所有结论均以代码实现为依据。

---

## 一、扩展点体系（Extension Points）

### 1.1 核心设计：接口即扩展点

Answer 采用 **Go 接口 + 泛型 Stack** 的方式定义扩展点。每个扩展点在 [plugin](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin) 包下都有独立文件，定义一个继承自 `Base` 的接口。

`Base` 接口是所有插件的根基，定义在 [base.go#L33-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/base.go#L33-L36)：

```go
type Base interface {
    Info() Info
}
```

所有扩展点均嵌入 `Base`，完整清单如下（22 个扩展点）：

| 扩展点接口 | 文件 | 职责 |
|---|---|---|
| `Config` | [config.go#L118-L129](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/config.go#L118-L129) | 管理员级插件配置（字段定义 + 接收） |
| `UserConfig` | [user_config.go#L22-L32](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_config.go#L22-L32) | 用户级插件配置（字段定义 + 接收） |
| `Connector` | [connector.go#L22-L45](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/connector.go#L22-L45) | 第三方 OAuth 登录连接器 |
| `UserCenter` | [user_center.go#L22-L44](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L22-L44) | 外部用户中心（登录/注册/用户同步） |
| `Captcha` | [captcha.go#L22-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L22-L34) | 验证码生成与校验 |
| `Parser` | [parser.go#L23-L25](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/parser.go#L23-L25) | 文本解析 |
| `Filter` | plugin/filter.go | 内容过滤 |
| `Storage` | plugin/storage.go | 文件存储 |
| `Cache` | plugin/cache.go | 缓存策略 |
| `Search` | plugin/search.go | 搜索引擎 |
| `Agent` | plugin/agent.go | AI 代理 |
| `Notification` | plugin/notification.go | 通知推送 |
| `Reviewer` | plugin/reviewer.go | 内容审核 |
| `Embed` | plugin/embed.go | 嵌入内容 |
| `Render` | plugin/render.go | 渲染引擎 |
| `CDN` | [cdn.go#L39-L42](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/cdn.go#L39-L42) | CDN 静态资源 |
| `Importer` | plugin/importer.go | 数据导入 |
| `KVStorage` | [kv_storage.go#L318-L321](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/kv_storage.go#L318-L321) | 插件键值存储 |
| `Sidebar` | plugin/sidebar.go | 侧边栏扩展 |
| `VectorSearch` | plugin/vector_search.go | 向量搜索 |
| `Base` | [base.go#L33-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/base.go#L33-L36) | 所有插件的根接口 |

### 1.2 泛型 Stack 与 MakePlugin 工厂

核心机制在 [plugin.go#L152-L179](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/plugin.go#L152-L179) 的 `MakePlugin` 函数：

```go
func MakePlugin[T Base](super bool) (CallFn[T], RegisterFn[T])
```

- **`super` 参数**：标记该扩展点是否为"超级"类型，决定调用时是否检查 `StatusManager.IsEnabled`。
- **返回值**：一对函数 —— `CallFn`（遍历调用已注册插件）和 `RegisterFn`（注册新插件到 Stack）。

调用 `CallFn` 时的关键分支逻辑：

```go
if !super && !StatusManager.IsEnabled(p.Info().SlugName) {
    continue  // 被禁用的插件跳过
}
```

### 1.3 super 扩展点真实清单与设计目的

通过 grep 全部 22 个 `MakePlugin[...]` 调用，得到确切的分类：

**super=true（4 个，共 4 个）**：调用时**不检查**启用状态，始终生效。

| 扩展点 | MakePlugin 调用位置 | 设计目的 |
|---|---|---|
| `Base` | [base.go#L40-L41](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/base.go#L40-L41) | 插件清单发现的根入口。即使插件被禁用，管理员仍需要在管理端看到它的存在才能启用它 |
| `Config` | [config.go#L133-L134](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/config.go#L133-L134) | 管理端获取插件配置字段、保存配置时需要调用 `ConfigFields()` 和 `ConfigReceiver()`。这些操作不依赖插件是否启用 |
| `Agent` | [agent.go#L34-L35](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/agent.go#L34-L35) | AI 代理作为核心能力，不允许通过状态管理被禁用 |
| `KVStorage` | [kv_storage.go#L324-L325](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/kv_storage.go#L324-L325) | 即使插件被禁用，也可能需要读取或清理它之前写入的 KV 数据；且 `SetOperator` 注入发生在系统启动的 `initPluginData` 阶段，早于状态恢复 |

**super=false（18 个）**：调用时会先检查 `StatusManager.IsEnabled(slugName)`，被禁用则跳过。

| 扩展点 | MakePlugin 调用位置 | 设计目的 |
|---|---|---|
| `UserConfig` | [user_config.go#L36-L37](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_config.go#L36-L37) | 用户配置接口，插件禁用后用户不应再看到或修改其配置 |
| `Connector` | [connector.go#L66-L67](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/connector.go#L66-L67) | OAuth 登录连接器，禁用后不再出现在登录页 |
| `UserCenter` | [user_center.go#L100-L101](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L100-L101) | 外部用户中心，可随时启用/停用 |
| `Captcha` | [captcha.go#L38-L39](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L38-L39) | 验证码插件，可随时启用/停用 |
| `CDN` | [cdn.go#L46-L47](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/cdn.go#L46-L47) | CDN 代理，可随时启用/停用 |
| `Parser` | [parser.go#L29-L30](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/parser.go#L29-L30) | 文本解析器，可按需启用 |
| `Filter` | plugin/filter.go | 内容过滤器，可按需启用 |
| `Storage` | plugin/storage.go | 文件存储后端，可替换 |
| `Cache` | plugin/cache.go | 缓存后端，可替换 |
| `Search` | plugin/search.go | 搜索后端，可替换 |
| `Notification` | plugin/notification.go | 通知渠道，可按需启用 |
| `Reviewer` | plugin/reviewer.go | 内容审核插件，可按需启用 |
| `Embed` | plugin/embed.go | 嵌入解析，可按需启用 |
| `Render` | plugin/render.go | 渲染引擎，可按需启用 |
| `Importer` | plugin/importer.go | 数据导入功能，可按需启用 |
| `Sidebar` | plugin/sidebar.go | 侧边栏扩展，可按需启用 |
| `VectorSearch` | plugin/vector_search.go | 向量搜索，可按需启用 |

### 1.4 Register：统一注册入口

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

**互斥协调机制（真实情况）**：[statusManager.Enable](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/plugin.go#L186-L202) 内部只显式处理两种协调：

```go
// 只有这两个协调
for _, slugName := range coordinatedCaptchaPlugins(name) {
    m.status[slugName] = false
}
for _, slugName := range coordinatedCDNPlugins(name) {
    m.status[slugName] = false
}
```

即：**只有 Captcha 和 CDN 两类扩展点在启用时会自动禁用同类型的其他插件**。详见 [captcha.go#L67-L82](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L67-L82) 和 [cdn.go#L50-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/cdn.go#L50-L65)。

> **重要纠正**：UserCenter **没有** coordinated 函数调用，不在此机制覆盖范围内（详见第四章 4.3 节）。

### 2.3 管理端插件列表发现

管理员通过 [PluginController.GetPluginList](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller_admin/plugin_controller.go#L74-L110) 查看所有已注册插件：

1. 遍历 `CallBase` 获取所有插件基本信息（名称、版本、SlugName）——`CallBase` 是 super=true，不区分启用状态
2. 遍历 `CallConfig` 标记哪些插件拥有配置表单——`CallConfig` 也是 super=true
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

> 注意：`Config` 扩展点是 super=true，因此 **即便插件被禁用**，仍可以通过管理端访问其配置字段和保存配置。

**ConfigField 类型系统** 支持：input、textarea、checkbox、radio、select、upload、timezone、switch、button、legend、tag_selector 等，以及 input 子类型（text、password、email、number、color 等）。

**配置保存流程**：

1. 管理员通过 `PUT /answer/admin/api/plugin/config` 提交配置
2. [PluginController.UpdatePluginConfig](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller_admin/plugin_controller.go#L204-L224) 先序列化 `configFields` 为 JSON
3. 调用 `plugin.CallConfig` 找到匹配的插件，执行 `ConfigReceiver` 让插件在内存中更新配置
4. 调用 `PluginCommonService.UpdatePluginConfig` 将配置持久化到 `plugin_config` 表

**实体层** 在 [plugin_config_entity.go#L23-L27](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/entity/plugin_config_entity.go#L23-L27)：

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

与 `Config` 的区别：① `super=false`（插件禁用后用户看不到配置）；② `UserConfigReceiver` 多了 `userID` 参数，表明配置是用户维度的。

**存储实体** 在 [plugin_user_config_entity.go#L23-L28](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/entity/plugin_user_config_entity.go#L23-L28)：

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

[KVStorage](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/kv_storage.go#L318-L321) 提供结构化的键值存储能力（super=true）：

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

**存储实体** 在 [plugin_kv_storage_entity.go#L22-L28](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/entity/plugin_kv_storage_entity.go#L22-L28)：

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

插件启停状态通过 `config` 表持久化，key 为 `plugin.status`（定义在 [plugin_config_key.go#L23-L24](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/base/constant/plugin_config_key.go#L23-L24)），值为 JSON 序列化的 `map[string]bool`。

### 3.6 初始化流程

[PluginCommonService.initPluginData](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/plugin_common/plugin_common_service.go) 在服务创建时执行，完成以下初始化：

1. **KVStorage 初始化**：为所有实现了 `KVStorage` 的插件注入 `KVOperator`（含 DB 和 Cache 实例）
2. **插件状态恢复**：从 `config` 表读取 `plugin.status`，反序列化到 `StatusManager`
3. **管理员配置恢复**：从 `plugin_config` 表读取所有插件配置，调用对应插件的 `ConfigReceiver` 在内存中恢复
4. **Cache 插件注入**：如果存在实现了 `Cache` 的插件，替换全局 Cache 实例
5. **VectorSearch 同步器注册**：为向量搜索插件注册数据同步器
6. **UserConfig 读取函数注入**：通过 `RegisterGetPluginUserConfigFunc` 注入数据库读取函数
7. **用户配置恢复**：后台 goroutine 分页加载 `plugin_user_config`，调用各插件的 `UserConfigReceiver` 恢复用户级配置

> 注意：KVStorage（步骤1）发生在状态恢复（步骤2）**之前**，因为 KVStorage 是 super=true，不依赖状态。

---

## 四、用户中心集成

### 4.1 UserCenter 接口

[user_center.go#L22-L44](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L22-L44) 定义了完整的用户中心扩展点：

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

### 4.3 UserCenter 的"互斥"真相（重要纠正）

**之前文档中的错误**：将 UserCenter 描述为"只能启用一个（互斥）"。

**代码事实**：

1. `statusManager.Enable` [plugin.go#L186-L202](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/plugin.go#L186-L202) 中只调用了 `coordinatedCaptchaPlugins` 和 `coordinatedCDNPlugins`——**没有** `coordinatedUserCenterPlugins`。
2. UserCenter 的 MakePlugin 声明为 `MakePlugin[UserCenter](false)` [user_center.go#L101](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L101)，super=false。
3. 理论上，管理员可以通过直接修改 `config` 表中的 `plugin.status` JSON，同时启用多个 UserCenter 插件。

**为什么实际上仍然只有一个生效？**

因为所有调用 UserCenter 的便捷函数和 Controller 都只取**遍历顺序的第一个**已启用插件：

- [UserCenterEnabled()](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L104-L110)：找到第一个就设置 `enabled=true` 返回
- [RankAgentEnabled()](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L112-L118)：只取第一个插件的 `RankAgentEnabled`
- [GetUserCenter()](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L120-L127)：只返回遍历到的第一个
- [UserCenterAgent](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/plugin_user_center_controller.go#L59-L99)：`CallUserCenter` 遍历中只取第一个的 `Description()`
- 所有 LoginCallback/SignUpCallback 路由：通过 `GetUserCenter()` 获取实例后调用

**结论**：
- **机制层面**：UserCenter **不是**强制互斥。不存在 coordinated 函数自动禁用同类型其他插件。
- **使用层面**：所有调用方都只取遍历顺序的第一个已启用的 UserCenter，其余被静默忽略。多个同时启用时非第一个插件不生效也不报错。

### 4.4 用户中心登录回调完整时序

以登录流程为例，完整时序如下：

```
浏览器                        Router         UserCenterController        UserCenterLoginService          UserCenter插件       DB/Cache
  │                             │                      │                              │                          │                │
  │ 1. 点击"XX用户中心登录"       │                      │                              │                          │                │
  │─────────────────────────────▶│                      │                              │                          │                │
  │                             │  GET /user-center/    │                              │                          │                │
  │                             │  login/redirect       │                              │                          │                │
  │                             │─────────────────────▶│                              │                          │                │
  │                             │                      │ CallUserCenter 获取          │                          │                │
  │                             │                      │ Description().LoginRedirectURL │                         │                │
  │                             │                      │                              │                          │                │
  │                             │  302 → 外部用户中心    │                              │                          │                │
  │◀────────────────────────────│                      │                              │                          │                │
  │                             │                      │                              │                          │                │
  │ 2. 在外部用户中心完成认证      │                      │                              │                          │                │
  │ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─▶                │
  │                             │                      │                              │                          │                │
  │ 3. 带回调参数跳转回来         │                      │                              │                          │                │
  │─────────────────────────────▶│                      │                              │                          │                │
  │                             │  GET /user-center/    │                              │                          │                │
  │                             │  login/callback       │                              │                          │                │
  │                             │─────────────────────▶│                              │                          │                │
  │                             │                      │ GetUserCenter()              │                          │                │
  │                             │                      │ 获取第一个已启用 UC          │                          │                │
  │                             │                      │                              │                          │                │
  │                             │                      │ 4. LoginCallback(ctx)        │                          │                │
  │                             │                      │─────────────────────────────────────────────────────▶│                │
  │                             │                      │                              │  解析OAuth token          │                │
  │                             │                      │                              │  调用外部UC接口查用户     │                │
  │                             │                      │                              │                          │                │
  │                             │                      │◀─────────────────────────────│ UserCenterBasicUserInfo │                │
  │                             │                      │                              │                          │                │
  │                             │                      │ 5. ExternalLogin(userCenter, userInfo)               │                │
  │                             │                      │─────────────────────────────▶│                          │                │
  │                             │                      │                              │ ┌─ GetByExternalID()    │                │
  │                             │                      │                              │ │  查询是否已绑定         │                │
  │                             │                      │                              │ ├─ 已存在 → 直接登录     │                │
  │                             │                      │                              │ │   CacheLoginUserInfo() │                │
  │                             │                      │                              │ └─ 不存在 → registerNewUser()             │
  │                             │                      │                              │    ├─ 创建User记录      │                │
  │                             │                      │                              │    └─ 创建ExternalLogin绑定                │
  │                             │                      │                              │                          │                │
  │                             │                      │◀─────────────────────────────│ accessToken              │                │
  │                             │                      │                              │                          │                │
  │                             │                      │ 6. AfterLogin(externalID, token)                      │                │
  │                             │                      │─────────────────────────────────────────────────────▶│                │
  │                             │                      │                              │                          │                │
  │                             │   302 → /users/auth-landing?access_token=xxx                                        │
  │◀────────────────────────────│                      │                              │                          │                │
```

路由定义在 [plugin_api_router.go#L63-L68](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/router/plugin_api_router.go#L63-L68)。

注册流程几乎完全相同，唯一区别是 `LoginCallback` 换成 `SignUpCallback`，但实际实现中两个回调的业务逻辑往往一致（都是通过 OAuth code 换 token 再查用户）。

### 4.5 Connector 与 UserCenter 的关系（纠正）

两者都提供外部登录能力，但定位不同：

| 维度 | Connector | UserCenter |
|---|---|---|
| 定位 | 第三方 OAuth 登录（如 GitHub、Google） | 外部用户中心（完全接管用户体系） |
| 登录方式 | OAuth 重定向流程 | 自定义跳转/回调 |
| 用户数据 | 只提供基础信息 | 提供完整用户生命周期管理 |
| 数量限制 | 可同时启用多个（CallConnector 遍历所有） | **事实上单实例**（调用方都取第一个） |
| 状态互斥 | 无 | 无显式互斥机制（见 4.3） |
| 用户状态/角色/等级管理 | 无 | 可通过能力声明接管 |
| 原系统保留 | 始终保留 | 可通过 `EnabledOriginalUserSystem` 选择禁用原系统 |

### 4.6 用户设置与个人品牌

- **UserSettings**：通过 `GET /user-center/user/settings` 返回用户中心的重定向 URL（如个人资料设置、账号设置的跳转链接），使得用户在 Answer 中的设置页面可以跳转到外部用户中心
- **PersonalBranding**：通过 `GET /user-center/personal/branding` 返回用户在外部用户中心的品牌信息（如社交媒体链接），展示在用户个人页面

### 4.7 辅助查询函数

[user_center.go#L104-L127](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L104-L127) 提供三个便捷函数：

- `UserCenterEnabled()` — 判断是否有用户中心插件启用（取第一个）
- `RankAgentEnabled()` — 判断用户中心是否启用了等级代理（取第一个）
- `GetUserCenter()` — 获取当前启用的 UserCenter 实例（取第一个）

> 这三个函数共同印证了：即便多个 UserCenter 同时启用，也只有遍历顺序的第一个会被"看到"。

---

## 五、Captcha 插件接入

### 5.1 Captcha 接口

[captcha.go#L22-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L22-L34) 定义了三个方法：

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

### 5.2 CallCaptcha 的选择逻辑（重要纠正）

**之前文档中的错误**："CallCaptcha 的实现确保只调用第一个注册的 Captcha 插件"。

**代码事实**：CallCaptcha 的实现是**两次遍历 + 选第一个已启用的**，逻辑比单纯"第一个注册"更复杂。

[captcha.go#L42-L57](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L42-L57)：

```go
func CallCaptcha(fn func(fn Captcha) error) error {
    // 第一次遍历：找到第一个已启用的 Captcha 的 slugName
    slugName := ""
    _ = callCaptcha(func(captcha Captcha) error {
        slugName = captcha.Info().SlugName
        return nil   // 不 break，继续遍历，但 slugName 最终会是最后一个
    })
    if slugName == "" {
        return nil  // 没有已启用的 Captcha，直接返回
    }
    // 第二次遍历：只调用 slugName 匹配的那个
    return callCaptcha(func(captcha Captcha) error {
        if captcha.Info().SlugName == slugName {
            return fn(captcha)
        }
        return nil
    })
}
```

**逐层分析**：

1. `callCaptcha` 本身来自 `MakePlugin[Captcha](false)`，因此传入的闭包只对**已启用**的 Captcha 执行（`StatusManager.IsEnabled` 过滤）。
2. 第一次遍历时，遍历顺序由 Stack 中注册顺序决定，`slugName` 在每次执行时被覆盖，最终值等于**遍历顺序最后一个**已启用的 Captcha 的 slugName。
3. 第二次遍历时，只有 slugName 精确匹配的那个 Captcha 才会实际执行 `fn`。

**结论**：
- 在 coordinatedCaptchaPlugins 的保证下，同一时刻最多只有一个 Captcha 启用，此时 slugName 就是那个唯一的，两次遍历等价于"调用那一个"。
- 如果 coordinated 被绕过（如手动改 DB 启用多个），**生效的是注册顺序最后一个**，而非第一个。这与之前描述的"调用第一个注册的"恰好相反。

### 5.3 互斥机制（纠正后的完整说明）

Captcha 插件的互斥通过两层保障：

1. **StatusManager 层面**：[coordinatedCaptchaPlugins](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L67-L82) 在 `Enable(name, true)` 时，收集除 `name` 之外的所有 Captcha slugName，然后由 StatusManager 把它们都置为 false。
2. **CallCaptcha 层面**：即便多个同时启用（如绕过 StatusManager），CallCaptcha 的两次遍历逻辑也只会调用"注册顺序最后一个已启用"的插件。

两者结合使得 Captcha 在正常使用下始终只有一个生效。

### 5.4 前端配置获取

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

### 5.5 CaptchaService — 验证策略引擎

[action/captcha_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/action/captcha_service.go) 和 [captcha_strategy.go](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/action/captcha_strategy.go) 实现了完整的验证码策略。

**Action 类型** 定义在 [captcha_entity.go#L22-L35](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/entity/captcha_entity.go#L22-L35)：

| Action | 含义 | 触发策略 | 统计维度 |
|---|---|---|---|
| `email` | 发送邮件 | **每次都需要**验证码 | IP |
| `password` | 修改密码/登录失败 | 30 分钟内 ≥ 3 次 | IP |
| `edit_userinfo` | 修改用户信息 | 30 分钟内 ≥ 3 次 | UserID |
| `question` | 提问 | 5 秒内 / 累计 ≥ 10 次 | UserID |
| `answer` | 回答 | 5 秒内 / 累计 ≥ 10 次 | UserID |
| `comment` | 评论 | 1 秒内 / 累计 ≥ 30 次 | UserID |
| `edit` | 编辑 | 累计 ≥ 10 次 | UserID |
| `invitation_answer` | 邀请回答 | 累计 ≥ 30 次 | UserID |
| `search` | 搜索 | 60 秒内 ≥ 20 次 | UserID（登录）/ IP（匿名） |
| `report` | 举报 | 1 秒内 / 累计 ≥ 30 次 | UserID |
| `delete` | 删除 | 5 秒内 / 累计 ≥ 5 次 | UserID |
| `vote` | 投票 | 累计 ≥ 40 次 | UserID |

**核心方法**：

1. `ActionRecord` — 预检查是否需要验证码，需要的话生成并返回验证码图片+id
2. `ValidationStrategy` — 根据操作类型和频率计算是否需要验证码
3. `GenerateCaptcha` — 调用 `plugin.CallCaptcha` 生成验证码，正确答案存入 Cache
4. `VerifyCaptcha` — 从 Cache 取出正确答案，调用 `plugin.CallCaptcha` 的 `Verify()` 校验用户输入
5. `ActionRecordVerifyCaptcha` — 组合验证：先判断策略是否需要，需要则再验验证码
6. `ActionRecordAdd` — 操作计数 +1
7. `ActionRecordDel` — 清除该操作的频率统计（成功后）

**关键短路**：[ValidationStrategy](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/action/captcha_strategy.go#L35-L39) 的第一行：

```go
if !plugin.CaptchaEnabled() {
    return true  // 直接通过，不触发验证码
}
```

即：如果没有启用任何 Captcha 插件，整个频率校验 + 验证码机制被完全跳过。

### 5.6 Captcha 业务接入完整时序

以**用户提问**（CaptchaActionQuestion）为例，端到端时序如下：

```
前端                     UserController/QuestionController        CaptchaService          CaptchaRepo        plugin.Captcha     Cache
  │                              │                                     │                      │                   │                │
  │ 1. 先调用 GET /user/action/   │                                     │                      │                   │                │
  │    record?action=question    │                                     │                      │                   │                │
  │─────────────────────────────▶│                                     │                      │                   │                │
  │                              │ ActionRecord(req)                    │                      │                   │                │
  │                              │────────────────────────────────────▶│                      │                   │                │
  │                              │                                     │ 1a. ValidationStrategy(unit=UserID, question)        │                │
  │                              │                                     │   ├─ plugin.CaptchaEnabled()                       │                │
  │                              │                                     │   │   └─ callCaptcha 遍历第一个已启用 → true/false    │                │
  │                              │                                     │   ├─ captchaRepo.GetActionType()                    │                │
  │                              │                                     │   └─ 5秒内 或 累计10次? → 需要验证:false             │                │
  │                              │                                     │                      │                   │                │
  │                              │                                     │ 若需要: GenerateCaptcha()                          │                │
  │                              │                                     │──────────────────────────────────────────────────▶│ Create()         │
  │                              │                                     │                      │                   │ (captcha_img,   │
  │                              │                                     │                      │                   │  code)           │
  │                              │                                     │  SetCaptcha(key, code)                                             │
  │                              │                                     │───────────────────────────────────────────────────────────────────────▶│
  │                              │◀────────────────────────────────────│ verify=true + captcha_id + captcha_img             │                │
  │◀─────────────────────────────│                                     │                      │                   │                │
  │                              │                                     │                      │                   │                │
  │ 2. 前端弹出验证码弹窗，用户输入   │                                     │                      │                   │                │
  │    后提交 POST /question/add  │                                     │                      │                   │                │
  │    (携带 captcha_id + captcha_code)│                                 │                      │                   │                │
  │─────────────────────────────▶│                                     │                      │                   │                │
  │                              │ 非管理员: ActionRecordVerifyCaptcha(question, UserID, id, code)                      │                │
  │                              │────────────────────────────────────▶│                      │                   │                │
  │                              │                                     │ 2a. ValidationStrategy() —— 再次判断是否仍需要验证   │                │
  │                              │                                     │                      │                   │                │
  │                              │                                     │ 若需要验证:                                          │                │
  │                              │                                     │ VerifyCaptcha(key, code)                            │                │
  │                              │                                     │   ├─ GetCaptcha(key)                                  │                │
  │                              │                                     │   │                     ───────────────────────────────────────────────▶│
  │                              │                                     │   │   ◀────────────── realCaptcha                     │                │
  │                              │                                     │   ├─ plugin.CallCaptcha → Verify(realCaptcha, code)  │                │
  │                              │                                     │   │                     ────────────────────────────────────────────▶│
  │                              │                                     │   └─ DelCaptcha(key)                                  │                │
  │                              │                                     │                         ─────────────────────────────────────────────▶│
  │◀──── 验证失败：400 + captcha错误 │                                     │                      │                   │                │
  │                              │                                     │                      │                   │                │
  │                              │ 验证通过则继续：                       │                      │                   │                │
  │                              │ ActionRecordAdd(question, UserID)    │                      │                   │                │
  │                              │────────────────────────────────────▶│ SetActionType(+1)    │                   │                │
  │                              │                                     │─────────────────────▶│                   │                │
  │                              │ questionService.AddQuestion()        │                      │                   │                │
  │                              │  ...持久化提问...                     │                      │                   │                │
  │◀─────────────────────────────│ 成功：{question_id}                  │                      │                   │                │
```

**主要接入点**（代码位置）：

| 业务操作 | 验证位置 | 计数位置 | 清除位置 |
|---|---|---|---|
| 登录（密码） | [user_controller.go#L141](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/user_controller.go#L141) | [user_controller.go#L154](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/user_controller.go#L154) 失败+1 | [user_controller.go#L163](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/user_controller.go#L163) 成功清除 |
| 发送邮件 | [user_controller.go#L190](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/user_controller.go#L190) | — | [user_controller.go#L336](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/user_controller.go#L336) |
| 修改密码 | [user_controller.go#L397](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/user_controller.go#L397) | [user_controller.go#L407](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/user_controller.go#L407) | [user_controller.go#L434](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/user_controller.go#L434) |
| 提问 | [question_controller.go#L419](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/question_controller.go#L419) | [question_controller.go#L484](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/question_controller.go#L484) | — |
| 回答 | [answer_controller.go#L225](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/answer_controller.go#L225) | [answer_controller.go#L273](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/answer_controller.go#L273) | — |
| 评论 | [comment_controller.go#L105](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/comment_controller.go#L105) | [comment_controller.go#L129](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/comment_controller.go#L129) | — |
| 投票 | [vote_controller.go#L90](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/vote_controller.go#L90) | [vote_controller.go#L102](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/vote_controller.go#L102) | — |

> 注意：所有接入点都先判断 `!isAdmin`——管理员**不受验证码约束**。此外，`question`/`answer`/`comment` 等操作对于拥有"免链接限制"权限的用户也跳过验证码检查（由 `canList` 数组控制）。

### 5.7 前端 Captcha 集成

[useCaptchaModal](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/ui/src/hooks/useCaptchaModal/index.tsx) 是前端验证码的 React Hook，负责：

- 调用 `checkImgCode` API 获取验证码配置和图片
- 渲染验证码弹窗（图片 + 输入框 + 刷新按钮）
- 通过 `resolveCaptchaReq` 将 `captcha_id` 和 `captcha_code` 注入业务请求
- 通过 `handleCaptchaError` 处理后端返回的验证码错误，自动弹窗重试

**交互流程**：初次请求时不弹窗（`check()` 先尝试无验证码提交）；如果后端返回 captcha_code 错误，再弹窗提示用户输入并携带重试。

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
│  │    ├─ coordinatedCaptchaPlugins (启用时互斥)         │    │
│  │    └─ coordinatedCDNPlugins (启用时互斥)             │    │
│  │                                                     │    │
│  │  super=true(4): Base / Config / Agent / KVStorage   │    │
│  │  super=false(18): 其余所有，含 UserCenter/Captcha等  │    │
│  │                                                     │    │
│  │  CallCaptcha(fn) = 两次遍历取"最后一个已启用"        │    │
│  │  CallUserCenter使用方 = 全部取"第一个已启用"         │    │
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
- **泛型 Stack**：`MakePlugin[T]` 统一了注册和调用的模式，22 个扩展点复用同一套实现
- **super 标记**：严格区分 4 个始终生效的核心扩展点（Base/Config/Agent/KVStorage）与 18 个可被禁用的业务扩展点

### 7.2 super=true 的精确语义与设计目的

不是"插件不可被禁用"，而是"该扩展点的 CallFn 在调用时不检查 StatusManager"。这意味着：
- 一个实现了 `Config` 的插件即便被禁用（StatusManager 中为 false），管理端依然可以获取其配置字段、保存配置到 DB
- 但同一插件如果也实现了 `Captcha`（super=false），`CallCaptcha` 时它会被跳过
- 即 super 的粒度是**扩展点维度**，不是插件整体维度

### 7.3 配置分层设计

- **管理员配置（Config，super=true）**：全局生效，配置读写不受插件启用状态影响
- **用户配置（UserConfig，super=false）**：用户维度，仅插件启用时可见
- **KV 存储（KVStorage，super=true）**：插件自定义数据，始终可操作以便清理/迁移

### 7.4 运行时配置恢复

系统启动时，`initPluginData` 按序恢复：KVOperator 注入 → 状态恢复 → 管理员配置恢复 → Cache 替换 → VectorSearch 同步器 → UserConfig 读取函数注入 → 用户配置恢复。这个顺序保证 KV 在状态之前初始化，而 `super=false` 的 UserConfig 恢复时可以根据 StatusManager 正确跳过被禁用的插件。

### 7.5 互斥协调的真实范围

- **明确互斥**（有 coordinated 函数）：Captcha、CDN，在 `StatusManager.Enable` 中自动禁用同类型其他插件
- **事实上单实例**（无 coordinated，但调用方都只取一个）：UserCenter —— 所有便捷函数和 Controller 都取遍历顺序第一个已启用的实例
- **多实例并行**（每个都被调用）：Connector、Parser、Filter、Notification、Reviewer 等

### 7.6 CallCaptcha 的选择逻辑

在 coordinated 正常生效的前提下（一个 Captcha 启用），CallCaptcha 的"两次遍历"等价于"调用那个唯一的"。但当 coordinated 被绕过导致多个 Captcha 启用时，生效的是**注册顺序最后一个已启用**的 Captcha，这一细节由 slugName 在第一次遍历中被**覆盖赋值**决定。

### 7.7 UserCenter 的代理模式

UserCenter 插件采用代理模式：它不替换原有用户体系，而是通过能力声明（`UserCenterDesc`）告诉 Answer 哪些功能由外部用户中心接管。Answer 据此在管理后台禁用相应功能（状态/角色/密码/创建用户），避免操作冲突。同时通过 `EnabledOriginalUserSystem` 控制原系统用户体系的去留。

### 7.8 Captcha 业务集成的"三明治"结构

每个业务接入点都遵循统一的**三明治结构**：
1. **上层**：Controller 先判断角色（管理员/免链接限制），非管理员才调用 `ActionRecordVerifyCaptcha`
2. **中层**：CaptchaService 先跑频率策略（ValidationStrategy），需要则再验验证码
3. **下层**：plugin.Captcha 接口屏蔽具体实现（后端生成 or 第三方服务）
4. **收尾**：验证通过则 `ActionRecordAdd` 计数 +1，成功完成则 `ActionRecordDel` 清除
