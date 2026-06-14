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

### 3.6 插件三阶段初始化：代码串联详解

插件初始化的入口链如下：

```
cmd/wire_gen.go →  wire 自动装配
  └─ plugin_common.NewPluginCommonService(...)      # 构造函数
       └─ p.initPluginData()                        # 立即执行，同步完成
```

[wire_gen.go#L272](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/cmd/wire_gen.go#L272) 在 Wire 依赖注入容器创建 PluginCommonService 时触发其构造函数 [NewPluginCommonService](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/plugin_common/plugin_common_service.go#L66-L82)，后者在返回前立即调用 `p.initPluginData()`，**阻塞执行**直到基础设施阶段和状态/配置阶段全部完成。

`initPluginData()` [plugin_common_service.go#L143-L230](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/plugin_common/plugin_common_service.go#L143-L230) 在代码层面自然分为三个连续阶段：

**阶段一：基础设施注入（同步，super=true 扩展点准备）** [L144-L151]

```go
_ = plugin.CallKVStorage(func(k plugin.KVStorage) error {
    k.SetOperator(plugin.NewKVOperator(
        ps.data.DB, ps.data.Cache, k.Info().SlugName,
    ))
    return nil
})
```

- 调用 `CallKVStorage`（super=true）遍历所有实现 KVStorage 的插件，为每个注入含 DB/Cache/SlugName 的 `KVOperator`。
- **为什么放在最前面？** 后续两个阶段中插件的 ConfigReceiver/UserConfigReceiver 可能需要访问自己的 KV 数据；若晚于状态恢复，一个被禁用的插件在 UserConfigReceiver（super=false）中会被跳过，但如果它的 KV 还未注入也无所谓。
- 此阶段没有任何 StatusManager 判断——因为 KVStorage 是 super=true。

**阶段二：状态恢复 + 管理员配置恢复（同步，系统可用的临界点）** [L153-L190]

```go
// 2a. 从 config 表还原插件启停状态
pluginStatus, _ := ps.configService.GetStringValueFromDB(
    context.TODO(), constant.PluginStatus)
plugin.StatusManager.UnmarshalJSON([]byte(pluginStatus))

// 2b. 从 plugin_config 表还原每个插件的管理员配置
pluginConfigs, _ := ps.pluginConfigRepo.GetPluginConfigAll(context.Background())
for _, pluginConfig := range pluginConfigs {
    plugin.CallConfig(func(fn plugin.Config) error {
        if fn.Info().SlugName == pluginConfig.PluginSlugName {
            return fn.ConfigReceiver([]byte(pluginConfig.Value))
        }
        return nil
    })
}

// 2c. 若有 Cache 插件，替换全局 Cache 实例
plugin.CallCache(func(cache plugin.Cache) error {
    ps.data.Cache = cache
    return nil
})

// 2d. 为 VectorSearch 插件注册数据同步器
plugin.CallVectorSearch(func(vs plugin.VectorSearch) error {
    vs.RegisterSyncer(...)
    return nil
})
```

- 先还原 StatusManager，**再**还原 ConfigReceiver。这个顺序至关重要：`CallConfig` 是 super=true，不受 StatusManager 影响，所以顺序其实不影响 ConfigReceiver 本身；但紧接着 `CallCache`、`CallVectorSearch` 都是 super=false，此时 StatusManager 已就绪，被禁用的插件能被正确跳过。
- 此阶段完成后，系统对 HTTP 请求已具备基本响应能力（插件启停/配置/缓存均就绪）。

**阶段三：用户配置恢复（异步，后台 goroutine 分页加载）** [L192-L230]

```go
// 3a. 注入"跨层读取用户配置"函数（plugin 包不直接依赖 repo 层）
plugin.RegisterGetPluginUserConfigFunc(func(userID, pluginSlugName string) []byte {
    pluginUserConfig, _, _ := ps.pluginUserConfigRepo.GetPluginUserConfig(
        context.Background(), userID, pluginSlugName)
    return []byte(pluginUserConfig.Value)
})

// 3b. 后台 goroutine 分页加载所有用户配置并回调插件
go func() {
    page, pageSize := 1, 1000
    for {
        userConfigs, _, _ := ps.pluginUserConfigRepo.GetPluginUserConfigPage(
            context.Background(), page, pageSize)
        if len(userConfigs) == 0 { return }
        for _, userConfig := range userConfigs {
            plugin.CallUserConfig(func(fn plugin.UserConfig) error {
                if fn.Info().SlugName == userConfig.PluginSlugName {
                    return fn.UserConfigReceiver(userConfig.UserID,
                        []byte(userConfig.Value))
                }
                return nil
            })
        }
        page++
    }
}()
```

- **为什么异步？** 用户配置数据量可能极大（用户数 × 插件数），同步加载会阻塞 HTTP 服务启动。分页 1000 条/页是典型的批量优化。
- `CallUserConfig` 是 super=false，所以被禁用的插件的 `UserConfigReceiver` 会被自动跳过——这也是为什么它必须放在阶段二之后（StatusManager 已恢复）。
- 3a 先注入读取函数，再启动 3b 的恢复 goroutine：顺序保证了恢复过程中如果某个插件需要读其他用户的配置，`GetPluginUserConfig` 函数已经可用。

**三阶段与 super 标记的耦合关系**：

| 阶段 | 涉及扩展点 | super 值 | 执行模式 | 依赖前提 |
|---|---|---|---|---|
| 阶段一 | KVStorage | `true` | 同步 | 仅依赖 DB/Cache 已连接 |
| 阶段二 | Config / Cache / VectorSearch | Config:`true` / 其余:`false` | 同步 | 阶段一完成 + StatusManager 先于 Cache/VS 恢复 |
| 阶段三 | UserConfig | `false` | 异步 goroutine | 阶段二完成（StatusManager 就绪）+ 读取函数已注入 |

### 3.7 Config 与 UserConfig super 不对称的设计解析

两者在扩展点声明上存在明确不对称：

| 维度 | `Config`（管理员配置） | `UserConfig`（用户配置） |
|---|---|---|
| super 标记 | `MakePlugin[Config](true)` [config.go#L133](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/config.go#L133) | `MakePlugin[UserConfig](false)` [user_config.go#L36](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_config.go#L36) |
| 调用方（Controller） | `controller_admin/plugin_controller.go` 管理端三处调用 [L81/L184/L211](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller_admin/plugin_controller.go#L81) | `controller/user_plugin_controller.go` 用户端三处调用 [L58/L89/L141](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/user_plugin_controller.go#L58) |
| 面向对象 | 管理员（全局） | 已登录用户（个人维度） |
| 数据量 | 小（插件数条） | 大（用户数 × 插件数） |

**不对称的三层设计原因**：

1. **管理端需要"看见"被禁用插件的配置**
   - 管理员在插件管理列表（`GetPluginList`）中需要区分哪些插件有配置、哪些没有，以便决定是否启用。
   - 管理员点击某个被禁用插件时，需要能预先填写/修改它的配置，再点击"启用"——如果 Config 是 super=false，`GetPluginConfig`（`CallConfig`）会直接跳过，管理员无法为被禁用插件预填配置。
   - 这与 `Base` 设为 super=true 的逻辑完全一致（禁用的插件也必须在列表中可见才能被启用）。

2. **用户端必须"看不见"被禁用插件的配置**
   - 插件一旦禁用，用户在个人设置页不应再看到该插件的配置入口（`CallUserConfig` 返回空），也不应再允许写入。
   - UserConfig 面向用户行为，插件不启用就意味着此功能对该用户不可用。

3. **初始化恢复顺序的依赖不同**
   - ConfigReceiver 在阶段二同步恢复，且需要在 `CallCache` 之前完成（Cache 插件自身的 Config 必须先被喂进内存才能正确接管全局 Cache）。Config super=true 保证即便 Cache 插件处于"尚未启用"的中间状态，它的 ConfigReceiver 仍能被调用。
   - UserConfigReceiver 在阶段三异步恢复，此时 StatusManager 已完全就绪，super=false 可以正确过滤掉被禁用的插件。

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

### 4.3 UserCenter 的选择顺序：CallUserCenter 与 GetUserCenter 的真实行为

**代码事实**：UserCenter **没有** coordinated 互斥函数，`statusManager.Enable` [plugin.go#L186-L202](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/plugin.go#L186-L202) 中只处理 Captcha 和 CDN。但 UserCenter 的选择行为需要区分 `CallUserCenter`（泛型 Stack 遍历）和 `GetUserCenter`（便捷封装）两个层面。

**CallUserCenter 的遍历机制**：

[CallUserCenter](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L100-L101) 来自 `MakePlugin[UserCenter](false)`，底层就是 [plugin.go#L155-L167](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/plugin.go#L155-L167) 的标准 `call` 闭包：

```go
call := func(fn Caller[T]) error {
    for _, p := range stack.plugins {
        if !super && !StatusManager.IsEnabled(p.Info().SlugName) {
            continue
        }
        if err := fn(p); err != nil {
            return err          // ← 遇到 error 就 break
        }
    }
    return nil
}
```

**关键细节**：遍历中如果 `fn(p)` 返回 `error != nil`，循环会**立即终止**。这与 CallCaptcha 的"两次遍历+不 break"完全不同。

**三个便捷函数的行为差异**：

| 函数 | 代码位置 | 闭包返回值 | 遍历行为 | 最终取到哪个 |
|---|---|---|---|---|
| `UserCenterEnabled()` | [user_center.go#L104-L110](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L104-L110) | `return nil`（永远不报错） | 遍历所有已启用 UserCenter，每次都执行 `enabled = true` | 仅判断"有没有"，不区分几个 |
| `RankAgentEnabled()` | [user_center.go#L112-L118](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L112-L118) | `return nil`（永远不报错） | 遍历所有已启用 UserCenter，每次覆盖 `enabled` | 取**注册顺序最后一个**已启用的 `RankAgentEnabled` 值 |
| `GetUserCenter()` | [user_center.go#L120-L127](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/user_center.go#L120-L127) | `return nil`（永远不报错） | 遍历所有已启用 UserCenter，每次覆盖 `uc` | 返回**注册顺序最后一个**已启用的 UserCenter 实例 |

**与 CallCaptcha 的选择对比**：

| 维度 | CallCaptcha | CallUserCenter |
|---|---|---|
| 遍历模式 | 两次遍历（第一次选 slugName，第二次精确匹配） | 一次遍历，闭包不 break |
| 多启用时取哪个 | **注册顺序最后一个**已启用（slugName 覆盖赋值） | **注册顺序最后一个**已启用（uc 覆盖赋值） |
| 结果是否相同 | 是 | 是 |
| 互斥保障 | coordinatedCaptchaPlugins（强互斥） | 无 coordinated（弱事实单实例） |

**Controller 层的调用模式**：

- `UserCenterAgent` [L79-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/plugin_user_center_controller.go#L79-L96)：直接用 `CallUserCenter`，闭包不 break → 取**注册顺序最后一个**已启用的 `Description()`
- `UserCenterLoginRedirect` / `UserCenterSignUpRedirect` [L112-L130](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/plugin_user_center_controller.go#L112-L130)：直接用 `CallUserCenter`，闭包不 break → 取**注册顺序最后一个**已启用的 `LoginRedirectURL`
- `UserCenterLoginCallback` / `UserCenterSignUpCallback` [L132-L202](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/controller/plugin_user_center_controller.go#L132-L202)：用 `GetUserCenter()` → 返回**注册顺序最后一个**已启用的实例，然后直接调用该实例的 `LoginCallback()`/`SignUpCallback()`

**结论**：UserCenter 和 Captcha 在多启用异常场景下的选择方向一致——都取**注册顺序最后一个**已启用。区别仅在于 Captcha 用两次遍历 + coordinated 强互斥保障，UserCenter 用一次遍历 + 无互斥仅靠覆盖赋值的弱事实单实例。

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

### 5.2 CallCaptcha 的选择逻辑

CallCaptcha 在 [captcha.go#L42-L57](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L42-L57) 采用**两次遍历**策略，从所有**已启用**的 Captcha 中选出**唯一的**一个来执行回调：

```go
func CallCaptcha(fn func(fn Captcha) error) error {
    slugName := ""
    _ = callCaptcha(func(captcha Captcha) error {
        slugName = captcha.Info().SlugName
        return nil   // 不 break，slugName 每次被覆盖
    })
    if slugName == "" {
        return nil
    }
    return callCaptcha(func(captcha Captcha) error {
        if captcha.Info().SlugName == slugName {
            return fn(captcha)
        }
        return nil
    })
}
```

**逐层代码分析**：

1. 底层 `callCaptcha` 来自 `MakePlugin[Captcha](false)`，Stack 在遍历中会先检查 `StatusManager.IsEnabled(slugName)`，所以闭包只对**已启用**的 Captcha 执行。
2. 第一次遍历：闭包对每个已启用 Captcha 执行 `slugName = captcha.Info().SlugName`，**不 return error 也不 break**，`slugName` 被逐个覆盖，最终值为**注册顺序最后一个**已启用的 slugName。
3. 第二次遍历：只对 slugName 精确匹配的那一个执行 `fn`。

**选择结论（与 5.7 时序图、7.6 总结完全一致）**：
- **正常路径**（coordinatedCaptchaPlugins 保证最多一个启用）：slugName 就是那个唯一的，两次遍历等价于"调用那一个"，顺序问题无实际影响。
- **异常路径**（绕过 coordinated 手动启用多个）：**生效的是注册顺序最后一个已启用的** Captcha，而非第一个。这是因为闭包不 break、slugName 被覆盖赋值的代码行为决定的。
- 整个系统中 `plugin.CaptchaEnabled()`、`GenerateCaptcha`、`VerifyCaptcha`、`CaptchaController.GetCaptchaConfig` 等所有 Captcha 调用点**全部走 CallCaptcha**，因此上述选择结论对全系统一致有效。

### 5.3 Captcha 双闸门验证机制

每个业务操作的验证码校验不是单一判断，而是通过 `ActionRecordVerifyCaptcha` 串联的**两道闸门**，代码在 [captcha_service.go#L95-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/action/captcha_service.go#L95-L107)：

```go
func (cs *CaptchaService) ActionRecordVerifyCaptcha(
    ctx context.Context, actionType, unit, captchaID, captchaCode string,
) bool {
    // 闸门一：频率/行为策略判断
    verificationResult := cs.ValidationStrategy(ctx, unit, actionType)
    if verificationResult {
        return true   // 闸门一放行，直接通过，不进入闸门二
    }
    // 闸门二：验证码本身校验
    pass, err := cs.VerifyCaptcha(ctx, captchaID, captchaCode)
    if err != nil {
        return false
    }
    return pass
}
```

**闸门一：ValidationStrategy（频率/行为策略）** [captcha_strategy.go#L32-L73](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/action/captcha_strategy.go#L32-L73)

```go
func (cs *CaptchaService) ValidationStrategy(ctx, unit, actionType string) bool {
    // 前置短路：未启用任何 Captcha 插件 → 全部放行
    if !plugin.CaptchaEnabled() {
        return true
    }
    info, _ := cs.captchaRepo.GetActionType(ctx, unit, actionType)
    switch actionType {
    case entity.CaptchaActionEmail:    return cs.CaptchaActionEmail(ctx, unit, info)   // 每次都需要
    case entity.CaptchaActionPassword: return cs.CaptchaActionPassword(ctx, unit, info) // 30min≥3次
    // ...共 12 种 action，各有独立阈值
    }
    return false
}
```

- 返回值语义：`true` = 放行（无需验证码），`false` = 需进入闸门二。
- 每种操作类型有独立的阈值规则（详见 5.6 表格），基于"操作频率 + 时间窗口"双重判定。
- 对 password/search/edit_userinfo 三种操作还附带**窗口过期自动清零**：超过时间窗口后将计数器重置为 0。

**闸门二：VerifyCaptcha（验证码本身校验）** [captcha_service.go#L152-L162](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/action/captcha_service.go#L152-L162)

```go
func (cs *CaptchaService) VerifyCaptcha(ctx, key, captcha string) (bool, error) {
    realCaptcha, _ := cs.captchaRepo.GetCaptcha(ctx, key)  // 从 Cache 取正确答案
    _ = plugin.CallCaptcha(func(fn plugin.Captcha) error {
        isCorrect = fn.Verify(realCaptcha, captcha)          // 交给插件 Verify() 比对
        return nil
    })
    _ = cs.captchaRepo.DelCaptcha(ctx, key)  // 无论对错，一次性验证码立即删除
    return isCorrect, nil
}
```

- 从 Cache 按 `captcha_id` 取出 `realCaptcha`（GenerateCaptcha 时写入，含 TTL）。
- 通过 CallCaptcha 委托给当前生效的（注册顺序最后一个已启用的）Captcha 插件执行 `Verify(realCaptcha, userInput)`——后端生成模式直接字符串比对，第三方服务模式调外部 API。
- **删除即失效**：`DelCaptcha` 确保同一个验证码不能被重复使用（防重放）。

**两道闸门的组合语义**：

| 闸门一（策略） | 闸门二（验证码） | 整体结果 | 含义 |
|---|---|---|---|
| `true`（放行） | 不执行 | `true` 通过 | 频率/行为未触发阈值，不需要验证码 |
| `false`（触发） | `true`（正确） | `true` 通过 | 触发了阈值，但验证码正确 |
| `false`（触发） | `false`（错误/过期） | `false` 拒绝 | 触发了阈值且验证码校验失败 |

**设计意图**：闸门一用极低成本的内存/缓存计数挡住绝大多数正常请求，只有高频或可疑请求才进入闸门二（可能涉及第三方 API 调用或额外计算），在保证安全的前提下把性能开销降到最低。

### 5.4 互斥机制（纠正后的完整说明）

Captcha 插件的互斥通过两层保障：

1. **StatusManager 层面**：[coordinatedCaptchaPlugins](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L67-L82) 在 `Enable(name, true)` 时，收集除 `name` 之外的所有 Captcha slugName，然后由 StatusManager 把它们都置为 false。
2. **CallCaptcha 层面**：即便多个同时启用（如绕过 StatusManager），CallCaptcha 的两次遍历逻辑也只会调用"**注册顺序最后一个已启用**"的插件（结论与 5.2、7.6 一致）。

两者结合使得 Captcha 在正常使用下始终只有一个生效。

### 5.5 前端配置获取

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

通过 CallCaptcha 返回当前激活的（注册顺序最后一个已启用的）Captcha 插件的 slug_name 和配置信息，前端据此初始化验证码组件。

### 5.6 CaptchaService — 验证策略引擎

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
2. `ValidationStrategy` — 闸门一：根据操作类型和频率计算是否需要验证码
3. `GenerateCaptcha` — 调用 `plugin.CallCaptcha` 生成验证码，正确答案存入 Cache
4. `VerifyCaptcha` — 闸门二：从 Cache 取出正确答案，调用 `plugin.CallCaptcha` 的 `Verify()` 校验用户输入
5. `ActionRecordVerifyCaptcha` — 双闸门串联：先跑策略，需要则再验验证码
6. `ActionRecordAdd` — 操作计数 +1
7. `ActionRecordDel` — 清除该操作的频率统计（成功后）

**关键短路**：[ValidationStrategy](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/internal/service/action/captcha_strategy.go#L35-L39) 的第一行：

```go
if !plugin.CaptchaEnabled() {
    return true  // 直接通过，不触发验证码
}
```

`plugin.CaptchaEnabled()` 内部也是通过 CallCaptcha 实现的（注册顺序最后一个已启用即视为启用），因此与 5.2 节选择结论一致——系统内所有 Captcha 相关判断**全部走同一条 CallCaptcha 选择路径**。

### 5.7 Captcha 业务接入完整时序

以**用户提问**（CaptchaActionQuestion）为例，端到端时序如下（CallCaptcha 选择结论与 5.2、7.6 完全一致）：

```
前端                     QuestionController           CaptchaService          CaptchaRepo    plugin.CallCaptcha      Cache
  │                              │                           │                      │            (最后一个已启用)    │
  │ 1. GET /user/action/record   │                           │                      │                  │          │
  │    ?action=question          │                           │                      │                  │          │
  │─────────────────────────────▶│                           │                      │                  │          │
  │                              │ ActionRecord(req)         │                      │                  │          │
  │                              │──────────────────────────▶│                      │                  │          │
  │                              │                           │ 1a. ValidationStrategy (闸门一)          │          │
  │                              │                           │   ├─ plugin.CaptchaEnabled()             │          │
  │                              │                           │   │   └─ CallCaptcha: 两次遍历取           │          │
  │                              │                           │   │      注册顺序最后一个已启用 → true     │          │
  │                              │                           │   ├─ captchaRepo.GetActionType(unit=UID) │          │
  │                              │                           │   └─ 5秒内 或 累计10次? → false(需验证)   │          │
  │                              │                           │                                              │
  │                              │                           │ 1b. 若需要验证: GenerateCaptcha()         │          │
  │                              │                           │   ├─ token.GenerateToken() → key          │          │
  │                              │                           │   ├─ CallCaptcha → Create()               │          │
  │                              │                           │   │          (两次遍历取注册顺序最后一个)    │          │
  │                              │                           │   │     ───────────────────────────────────────────▶│
  │                              │                           │   │     ◀──── captcha_img + realCaptcha    │          │
  │                              │                           │   └─ SetCaptcha(key, realCaptcha)         │          │
  │                              │                           │                ──────────────────────────────────────▶│
  │                              │◀──────────────────────────│ verify=true + captcha_id + captcha_img    │          │
  │◀─────────────────────────────│                           │                      │                  │          │
  │                              │                           │                      │                  │          │
  │ 2. 前端 useCaptchaModal      │                           │                      │                  │          │
  │    弹窗，用户输入后提交       │                           │                      │                  │          │
  │    POST /question/add        │                           │                      │                  │          │
  │    (携带 captcha_id+code)    │                           │                      │                  │          │
  │─────────────────────────────▶│                           │                      │                  │          │
  │                              │ 非管理员:                  │                      │                  │          │
  │                              │ ActionRecordVerifyCaptcha │                      │                  │          │
  │                              │ (question, UID, id, code) │                      │                  │          │
  │                              │──────────────────────────▶│                      │                  │          │
  │                              │                           │ 2a. ValidationStrategy (闸门一) ——再次判定 │          │
  │                              │                           │   仍需验证 → false                     │          │
  │                              │                           │                      │                  │          │
  │                              │                           │ 2b. VerifyCaptcha (闸门二)               │          │
  │                              │                           │   ├─ GetCaptcha(key)                       │          │
  │                              │                           │   │           ──────────────────────────────────────▶│
  │                              │                           │   │   ◀──────────────── realCaptcha        │          │
  │                              │                           │   ├─ CallCaptcha → Verify(realCaptcha, code)         │
  │                              │                           │   │          (两次遍历取注册顺序最后一个)    │          │
  │                              │                           │   │     ───────────────────────────────────────────▶│
  │                              │                           │   │     ◀─────── true/false               │          │
  │                              │                           │   └─ DelCaptcha(key)                       │          │
  │                              │                           │                ──────────────────────────────────────▶│
  │◀──── 验证失败：400 + captcha错误 │                        │                      │                  │          │
  │                              │                           │                      │                  │          │
  │                              │ 验证通过则继续：            │                      │                  │          │
  │                              │ ActionRecordAdd           │                      │                  │          │
  │                              │ (question, UID)           │                      │                  │          │
  │                              │──────────────────────────▶│ SetActionType(+1)    │                  │          │
  │                              │                           │─────────────────────▶│                  │          │
  │                              │ questionService           │                      │                  │          │
  │                              │ .AddQuestion()            │                      │                  │          │
  │                              │  ...持久化提问...          │                      │                  │          │
  │◀─────────────────────────────│ 成功：{question_id}       │                      │                  │          │
```

**CallCaptcha 选择结论在本时序中的三处体现（与 5.2、7.6 对齐）**：
- ① `plugin.CaptchaEnabled()`：CallCaptcha 能取到即视为启用
- ② `GenerateCaptcha` → `CallCaptcha → Create()`
- ③ `VerifyCaptcha` → `CallCaptcha → Verify()`
- 三处全部遵循"注册顺序最后一个已启用"的同一选择逻辑。

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

### 5.8 useCaptchaModal 前端状态机

前端验证码交互由 [useCaptchaModal/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/ui/src/hooks/useCaptchaModal/index.tsx) 这个自定义 Hook 管理，内部是一个状态机。

**核心状态变量**：

| 状态变量 | 类型 | 含义 |
|---|---|---|
| `stateShow` | `boolean` | Modal 弹窗是否可见 |
| `captcha` | `{captcha_id, captcha_img, verify}` | 后端下发的验证码信息。`verify=true` 表示当前业务被策略闸门判定为需要验证 |
| `imgCode` | `{value, isInvalid, errorMsg}` | 用户输入状态 |
| `pending` | `ref<boolean>` | 正在请求验证码数据，防重入 |
| `refCallback` | `ref<SubmitCallback>` | 挂起的业务提交回调，验证码通过后执行 |

**状态机变迁图**：

```
                ┌─────────────────────┐
                │      初始态          │
                │ stateShow=false      │
                │ captcha.verify=false │
                │ imgCode.value=''     │
                └──────────┬──────────┘
                           │
          autoInitCaptcha? │  (email类立即预取)
                           │
                           ▼
                ┌─────────────────────┐
                │  预取验证码待触发     │
                │ captcha.verify 待后端 │
                │   ActionRecord 返回   │
                └──────────┬──────────┘
                           │
                   check(submitFunc)
                           │
             ┌─────────────┴─────────────┐
             │                           │
    captcha.verify=false         captcha.verify=true
    (闸门一放行，直接提交)       (闸门一触发，需验证)
             │                           │
             ▼                           ▼
    ┌────────────────┐         ┌────────────────────┐
    │ 直接执行业务    │         │ show() 弹窗显现      │
    │ submitFunc()   │         │ stateShow=true      │
    └────────────────┘         └─────────┬──────────┘
                                         │
                               用户输入验证码
                               handleChange()
                                         │
                                         ▼
                               ┌────────────────────┐
                               │ 点击"验证"按钮       │
                               │ handleSubmit()      │
                               └─────────┬──────────┘
                                         │
                                         ▼
                               ┌────────────────────┐
                               │ 执行 refCallback    │
                               │ (提交业务请求，附带  │
                               │  captcha_id+code)   │
                               └─────────┬──────────┘
                                         │
                        ┌────────────────┴────────────────┐
                        │                                 │
               后端返回非captcha错误               后端返回captcha_code错误
               (业务成功或其他校验失败)            (验证码错误或需要重新触发)
                        │                                 │
                        ▼                                 ▼
               ┌────────────────┐               ┌────────────────────┐
               │ handleCaptcha  │               │ handleCaptchaError │
               │ Error() → close │               │ → fetchCaptchaData │
               │ 关闭弹窗，清空状态 │             │ → 保留弹窗，刷新图片 │
               └────────────────┘               │ → (首次触发不显示错 │
                                                │    误，非首次显示)  │
                                                └────────────────────┘
```

**对外 API 的行为语义**：

- `check(submitFunc)`：业务代码的统一入口。若 `captcha.verify=false` 直接执行 `submitFunc()` 并返回 `true`；若 `captcha.verify=true` 仅弹窗并返回 `false`，真实提交延后到 `handleSubmit()`。
- `resolveCaptchaReq(req)`：把当前持有的 `captcha_id` 和 `captcha_code` 注入业务请求体——仅当 `captcha.verify=true` 时才注入，`verify=false` 不注入字段。
- `handleCaptchaError(fieldErrors)`：统一处理后端返回的错误。若含 `captcha_code` 错误：
  - 首次触发（`imgCode.value` 为空）：只刷新验证码并弹窗，**不显示错误红框**（用户还没输过）。
  - 非首次（`imgCode.value` 非空）：显示错误红框 + 刷新验证码 + 弹窗。
  - 若不含 captcha 错误（业务成功或其他失败）：关闭弹窗并清空所有状态。
- `fetchCaptchaData()`：调用 `checkImgCode` API（后端走 `ActionRecord` → `ValidationStrategy` → `GenerateCaptcha`），拉取新的 `captcha_id`/`captcha_img`/`verify` 三元组。

**与后端双闸门的协同**：
前端的 `captcha.verify` 字段就是后端 `ActionRecord` 中闸门一（ValidationStrategy）判定结果的直接下发；用户提交后后端会再次跑闸门一+闸门二（双保险），前端 `verify` 只是 UX 优化（提前弹窗减少一次失败往返）。

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

### 7.4 运行时配置恢复：三阶段代码串联

系统启动时 `initPluginData` 分为三个严格顺序的阶段（详见 3.6 节）：
- **阶段一（同步）**：KVStorage 注入 Operator，super=true，不依赖状态
- **阶段二（同步）**：StatusManager 恢复 → ConfigReceiver 恢复（super=true）→ Cache/VectorSearch 插件接管（super=false，此时状态已就绪）。此阶段完成后 HTTP 服务可响应请求
- **阶段三（异步 goroutine）**：UserConfig 读取函数注入 → 分页恢复 UserConfigReceiver（super=false，自动跳过被禁用插件）

三阶段与 super 标记精确耦合：阶段一只操作 super=true 的 KVStorage，阶段二先恢复状态再依次操作 super=true 的 Config 和 super=false 的 Cache/VectorSearch，阶段三在状态完全就绪后异步处理 super=false 的 UserConfig。

### 7.5 互斥协调的真实范围

- **明确互斥**（有 coordinated 函数）：Captcha、CDN，在 `StatusManager.Enable` 中自动禁用同类型其他插件
- **事实上单实例**（无 coordinated，但调用方都只取一个）：UserCenter —— 所有便捷函数和 Controller 都取遍历顺序第一个已启用的实例
- **多实例并行**（每个都被调用）：Connector、Parser、Filter、Notification、Reviewer 等

### 7.6 CallCaptcha 的选择逻辑（与 5.2、5.7 结论完全一致）

CallCaptcha 采用**两次遍历**策略（[captcha.go#L42-L57](file:///d:/fz/0601-1/solo-dogfeeding/code/68-answer/plugin/captcha.go#L42-L57)）：
1. 第一次遍历：对每个已启用 Captcha 执行 `slugName = captcha.Info().SlugName`，闭包不 break，slugName 被逐个**覆盖赋值**，最终值为**注册顺序最后一个**已启用的 slugName
2. 第二次遍历：只对 slugName 精确匹配的那一个执行业务回调

**统一结论（在 5.2、5.7、7.6 三处完全对齐）**：
- **正常路径**（coordinatedCaptchaPlugins 保证最多一个启用）：两次遍历等价于"调用那一个"
- **异常路径**（绕过 coordinated 手动启用多个）：生效的是**注册顺序最后一个已启用**的 Captcha，而非第一个
- 系统内所有 Captcha 调用点（`CaptchaEnabled()`、`GetCaptchaConfig`、`GenerateCaptcha`、`VerifyCaptcha`）**全部走同一条 CallCaptcha 路径**，选择结论全局一致

### 7.7 Config 与 UserConfig 的 super 不对称设计

`Config` super=true 而 `UserConfig` super=false 的三层设计原因（详见 3.7 节）：
1. **管理端可见性**：管理员必须能为被禁用插件预填配置后再启用，Config super=true 保证禁用状态下 `CallConfig` 仍可返回配置字段，逻辑与 `Base` super=true 一致
2. **用户端隐藏性**：插件禁用后用户不应再看到或修改其个人配置，UserConfig super=false 保证 `CallUserConfig` 自动跳过
3. **初始化顺序依赖**：ConfigReceiver 在阶段二同步恢复且必须先于 Cache 插件接管；UserConfigReceiver 在阶段三异步恢复，依赖 StatusManager 已就绪才能正确过滤

### 7.8 双闸门验证

Captcha 业务接入采用**双闸门**设计（详见 5.3 节）：第一闸门 `ValidationStrategy` 判断频率是否超阈值，第二闸门 `VerifyCaptcha` 校验验证码正确性。第一闸门通过则短路跳过第二闸门。无 Captcha 插件时第一闸门首行 `!plugin.CaptchaEnabled()` 短路返回 true，整个验证码机制零成本。两次闸门之间职责分离：闸门一用内存/缓存计数挡掉绝大多数正常请求，闸门二仅对高风险请求才可能产生第三方 API 调用等昂贵计算。

### 7.9 useCaptchaModal 前端状态机

前端通过 `check(submitFunc)` 实现**两阶段提交**（详见 5.8 节）：频率未触发时直接执行业务函数；频率触发时弹窗等用户输入后通过 `refCallback` 延迟提交。email 操作因策略为"每次都需要"而自动预加载验证码。`handleCaptchaError` 对首次触发静默弹窗、对输错显示错误信息。`pending` ref 防止并发请求。前端 `captcha.verify` 只是后端闸门一判定结果的 UX 镜像，实际提交时后端会再次跑双闸门校验，双端不互信。

