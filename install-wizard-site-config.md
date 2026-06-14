# 安装向导与站点配置：初始化、配置校验、服务启动与 Schema Form 渲染深度解析

本文档从 **初始化边界**、**配置校验**、**服务启动与恢复边界**、**重复执行保护**、**Schema Form 渲染与校验失败处理** 五个维度，对 Apache Answer 的安装向导与站点配置代码进行系统性分析。

---

## 一、初始化流程：首次安装的完整生命周期

### 1.1 入口：CLI 命令触发

安装向导的启动入口在 [command.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/command.go#L115-L142) 的 `initCmd`：

```
answer init -C ./data/
```

执行流程：

1. **第一步 - 初始化基础环境**：调用 `cli.InstallAllInitialEnvironment(dataDirPath)`
   - 格式化所有路径（配置文件目录、上传目录、i18n 目录、缓存目录）
   - 创建上传目录
   - 安装 i18n 语言包（非覆盖模式）

2. **第二步 - 判断是否已安装**：
   - 调用 `cli.CheckConfigFile()` 检查 `config.yaml` 是否存在
   - 若配置文件存在：调用 `conf.ReadConfig()` 读取配置，再调用 `cli.CheckDBTableExist()` 检查数据库中 `version` 表是否存在
   - 若两者都存在：直接返回，提示 "connect to database successfully and table already exists, do nothing."
   - 否则进入安装向导流程

3. **第三步 - 启动安装 HTTP 服务器**：调用 `install.Run(path.GetConfigFilePath())`

### 1.2 安装服务器启动

在 [install_main.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_main.go#L41-L62) 的 `Run()` 函数中：

```go
func Run(configPath string) {
    confPath = configPath
    // 1. 初始化翻译器（安装过程也需要国际化错误信息）
    _, err := translator.NewTranslator(...)
    
    // 2. 尝试环境变量自动安装
    if installByEnv, err := TryToInstallByEnv(); installByEnv && err != nil {
        fmt.Printf("[auto-install] try to init by env fail: %v\n", err)
    }
    
    // 3. 启动安装专用 HTTP 服务器
    installServer := NewInstallHTTPServer()
    installServer.Run(":" + port)
}
```

**关键设计点**：
- 安装服务器是 **独立的 Gin Engine**，与业务服务完全隔离，只注册安装相关路由
- 支持通过环境变量 `INSTALL_PORT` 自定义端口（默认 80）
- 支持 **无交互自动安装**：检测 `AUTO_INSTALL` 环境变量，若设置则模拟 HTTP 请求自动走完整个安装流程

### 1.3 自动安装模式（Env-Driven）

在 [install_from_env.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_from_env.go#L55-L150) 中实现了 **进程内模拟 HTTP** 的自动安装：

```go
func initByEnv(env *Env) error {
    gin.SetMode(gin.TestMode)
    dbCheck(env)           // POST /installation/db/check
    initConfigAndDb(env)   // POST /installation/init
    initBaseInfo(env)      // POST /installation/base-info
    return nil
}

func requestAPI(req any, method, url string, handlerFunc gin.HandlerFunc) error {
    w := httptest.NewRecorder()
    c, _ := gin.CreateTestContext(w)
    body, _ := json.Marshal(req)
    c.Request, _ = http.NewRequest(method, url, bytes.NewBuffer(body))
    handlerFunc(c)  // 直接调用 handler，不经过网络
    if w.Code != http.StatusOK {
        return errors.New(gjson.Get(w.Body.String(), "msg").String())
    }
    return nil
}
```

**设计特点**：
- 使用 `httptest.NewRecorder()` + `gin.CreateTestContext()` 实现进程内调用，不产生真实网络请求
- 三个步骤与交互式安装完全复用同一套 handler 代码，保证两种安装模式的一致性
- 适用于 Docker/K8s 等容器化部署场景

### 1.4 安装向导的 5 个前端步骤

在 [Install/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/ui/src/pages/Install/index.tsx#L51-L454) 中，安装流程被拆分为 8 个状态：

| Step | 组件 | 内容 | 后端接口 |
|------|------|------|----------|
| 1 | FirstStep | 选择语言 | /installation/language/options |
| 2 | SecondStep | 数据库配置 | /installation/db/check |
| 3 | ThirdStep | 配置文件创建结果 | /installation/init |
| 4 | FourthStep | 站点信息 + 管理员账户 | /installation/base-info |
| 5 | Fifth | 安装完成，跳转引导 | - |
| 6 | - | 配置文件存在但 DB 未连通警告 | /installation/config-file/check |
| 7 | - | 数据库连接失败 | - |
| 8 | - | 已安装完成（DB表存在） | - |

**前端初始化时的自动检测**（`configYmlCheck` 函数）：
1. 页面加载立即调用 `/installation/config-file/check`
2. 若 `config_file_exist=true` 且 `db_connection_success=true` → 跳 step=6（警告已安装）
3. 若 `config_file_exist=true` 但 `db_connection_success=false` → 跳 step=7（DB 失败）

---

## 二、配置校验：后端 + 前端的双层校验体系

### 2.1 后端请求校验：三层结构

#### 第一层：Struct Tag 注解校验

在 [install_req.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_req.go#L43-L133) 中定义的请求结构体带有丰富的 `validate` tag：

```go
type CheckDatabaseReq struct {
    DbType  string `validate:"required,oneof=postgres sqlite3 mysql" json:"db_type"`
    DbHost  string `json:"db_host"`
    DbName  string `json:"db_name"`
    // ...
}

type InitBaseInfoReq struct {
    Language      string `validate:"required,gt=0,lte=30" json:"lang"`
    SiteName      string `validate:"required,sanitizer,gt=0,lte=30" json:"site_name"`
    SiteURL       string `validate:"required,gt=0,lte=512,url" json:"site_url"`
    ContactEmail  string `validate:"required,email,gt=0,lte=500" json:"contact_email"`
    AdminName     string `validate:"required,gte=2,lte=30" json:"name"`
    AdminPassword string `validate:"required,gte=8,lte=32" json:"password"`
    AdminEmail    string `validate:"required,email,gt=0,lte=500" json:"email"`
    ExternalContentDisplay string `validate:"required,oneof=always_display ask_before_display" json:"external_content_display"`
}
```

**校验器自定义扩展**（在 [validator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/validator/validator.go#L121-L180)）：
- `notblank`：不仅非空，还会自动 trim 首尾空格
- `sanitizer`：使用 `bluemonday.UGCPolicy()` 对内容进行 HTML 安全过滤（XSS 防护）
- Tag 名国际化：通过 `RegisterTagNameFunc` 把 json tag 的字段名翻译成用户语言

#### 第二层：Checker 接口的自定义校验

结构体可以实现 `validator.Checker` 接口来追加业务规则校验。例如 `InitBaseInfoReq`：

```go
func (r *InitBaseInfoReq) Check() (errFields []*validator.FormErrorField, err error) {
    if checker.IsInvalidUsername(r.AdminName) {
        errField := &validator.FormErrorField{
            ErrorField: "name",
            ErrorMsg:   reason.UsernameInvalid,
        }
        errFields = append(errFields, errField)
        return errFields, errors.BadRequest(reason.UsernameInvalid)
    }
    return
}
```

#### 第三层：handler 的统一触发入口

在 [handler.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/handler/handler.go#L64-L78) 的 `BindAndCheck` 函数中，三层校验被串联执行：

```go
func BindAndCheck(ctx *gin.Context, data any) bool {
    lang := GetLangByCtx(ctx)
    
    // Step 1: Gin 的 ShouldBind（JSON 反序列化）
    if err := ctx.ShouldBind(data); err != nil {
        HandleResponse(ctx, myErrors.New(400, reason.RequestFormatError), nil)
        return true
    }
    
    // Step 2: Struct Tag 校验 + Checker 接口校验
    errField, err := validator.GetValidatorByLang(lang).Check(data)
    if err != nil {
        HandleResponse(ctx, err, errField)  // errField 携带字段级错误
        return true
    }
    return false
}
```

### 2.2 校验失败的响应格式

在 [validator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/validator/validator.go#L190-L257) 的 `Check` 方法中，返回格式为：

```json
{
    "code": 400,
    "reason": "RequestFormatError",
    "msg": "Username is invalid.",
    "data": [
        {"error_field": "name", "error_msg": "Username is invalid."}
    ]
}
```

**前端对校验失败的处理**（在 Install/index.tsx 的 `submitSiteConfig` 中）：

```typescript
installBaseInfo(params)
    .then(() => handleNext())
    .catch((err) => {
        if (err.isError) {
            // 字段级错误：回填到 formData 的 isInvalid/errorMsg
            const data = handleFormError(err, formData);
            setFormData({ ...data });
            // 自动滚动到第一个错误字段
            const ele = document.getElementById(err.list[0].error_field);
            scrollToElementTop(ele);
        } else {
            // 非字段级错误（如 DB 连接失败）：顶部 Alert 展示
            handleErr(err);
        }
    });
```

### 2.3 错误类型的三种展示方式

在 [err_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/schema/err_schema.go#L22-L30) 中定义了三种前端错误展示类型：

| 类型 | ErrType 值 | 前端表现 | 适用场景 |
|------|-----------|---------|---------|
| Modal | `"modal"` | 弹窗展示 | 重要确认类错误 |
| Toast | `"toast"` | 顶部浮动提示 | 轻量通知 |
| Alert | `"alert"` | 表单上方红色横幅 | 安装向导中的数据库/配置错误 |

使用方式：`handler.HandleResponse(ctx, errors.BadRequest(reason.DatabaseConnectionFailed), schema.ErrTypeAlert)`

---

## 三、服务启动与恢复边界

### 3.1 两种运行模式：Install 模式 vs Run 模式

| 维度 | answer init（安装模式） | answer run（运行模式） |
|------|------------------------|----------------------|
| 入口函数 | `install.Run()` | `runApp()` |
| HTTP 引擎 | `NewInstallHTTPServer()`（极简路由） | Wire 依赖注入的完整路由 |
| 路由注册 | 仅 7 个 /installation/* 接口 + 静态文件 | 全部业务 API + 管理后台 + 插件路由 |
| 数据库依赖 | 无（安装前还无 DB） | 必须有有效 DB 连接 |
| 访问 /install | 正常渲染安装页面 | **强制重定向到 /** |

### 3.2 Run 模式的启动流程

在 [main.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/main.go#L75-L95) 的 `runApp()` 中：

```go
func runApp() {
    // Step 1: 读取配置文件（若文件不存在直接 panic）
    c, err := conf.ReadConfig(path.GetConfigFilePath())
    if err != nil { panic(err) }
    
    // Step 2: Wire 依赖注入构建整个应用（DB/Cache/Translator/Router/Controller...）
    app, cleanup, err := initApplication(
        c.Debug, c.Server, c.Data.Database, c.Data.Cache, 
        c.I18n, c.Swaggerui, c.ServiceConfig, c.UI, log.GetLogger())
    if err != nil { panic(err) }
    
    // Step 3: 注册版本信息、启动定时任务
    constant.Version = Version
    schema.AppStartTime = time.Now()
    
    // Step 4: 启动 HTTP 服务
    defer cleanup()
    app.Run(context.Background())
}
```

**边界保护**：
- `conf.ReadConfig()` 失败 → `panic`（服务无法启动）
- `initApplication()` 中 DB 连接失败 → Wire 构建失败 → panic
- 这确保了 **Run 模式启动时，系统必须处于已安装完成状态**

### 3.3 Run 模式下对 /install 的访问拦截

在正式运行的 UIRouter 中，[ui.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/router/ui.go#L127-L130) 的 NoRoute 处理器明确阻断了安装页面：

```go
r.NoRoute(func(c *gin.Context) {
    urlPath := c.Request.URL.Path
    switch urlPath {
    case "/install":
        // 关键边界：通过 run 命令启动后，用户无法再访问安装页面
        c.Redirect(http.StatusFound, "/")
        return
    // ... 其他静态资源处理
    }
})
```

这是一道重要的 **恢复边界防线**：即使服务重启后，恶意用户也无法通过访问 `/install` 重新触发安装流程。

### 3.4 Wire 依赖注入：启动时的完整性校验

在 [wire.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/wire.go#L46-L69) 中，`initApplication` 使用 Google Wire 编译期依赖注入：

```go
func initApplication(...) (*pacman.Application, func(), error) {
    panic(wire.Build(
        server.ProviderSetServer,        // HTTP Server + UI 配置
        router.ProviderSetRouter,        // 路由注册
        controller.ProviderSetController,// 前端 API
        controller_admin.ProviderSetController, // 管理后台 API
        service.ProviderSetService,      // 业务服务层
        cron.ProviderSetService,         // 定时任务
        repo.ProviderSetRepo,            // 数据访问层
        translator.ProviderSet,          // 翻译器
        middleware.ProviderSetMiddleware,// 中间件（认证/限流...）
        newApplication,
    ))
}
```

**编译期校验的价值**：所有依赖关系在编译时确定，任何 Provider 缺失都会导致编译失败，避免了运行时出现 "nil pointer" 这类启动期错误。

---

## 四、重复执行保护：多层防重装机制

### 4.1 防御层次总览

系统通过 **五道防线** 防止安装流程被重复执行：

```
第一道：initCmd 启动时的 config+db 检查
         ↓
第二道：安装 API 入口的 config 文件检查
         ↓
第三道：InitBaseInfo 执行前的 DB 表存在检查
         ↓
第四道：Mentor.InitDB() 内部的 checkTableExist 短路
         ↓
第五道：Run 模式下的 /install 路由重定向
```

### 4.2 第一道防线：initCmd 启动预检

在 [command.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/command.go#L120-L141) 中：

```go
configFileExist := cli.CheckConfigFile(path.GetConfigFilePath())
if configFileExist {
    c, err := conf.ReadConfig(path.GetConfigFilePath())
    if cli.CheckDBTableExist(c.Data.Database) {
        fmt.Println("connect to database successfully and table already exists, do nothing.")
        return  // ← 不启动安装服务器
    }
}
```

### 4.3 第二道防线：InitEnvironment 接口检查

在 [install_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_controller.go#L159-L200) 的 `InitEnvironment` 中：

```go
// 若配置文件已存在，直接返回成功不做任何操作
if cli.CheckConfigFile(confPath) {
    log.Debug("config file already exists")
    handler.HandleResponse(ctx, nil, nil)
    return
}
```

**幂等设计**：即使前端重复提交，第二次调用也不会重复写配置文件。

### 4.4 第三道 + 第四道防线：InitBaseInfo 的双层检查

在 [install_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_controller.go#L211-L250) 的 `InitBaseInfo` 中：

```go
// 第三道：handler 层面检查
if cli.CheckDBTableExist(c.Data.Database) {
    log.Warn("database is already initialized")
    handler.HandleResponse(ctx, nil, nil)  // 静默返回
    return
}

// 创建数据库连接后进入 Mentor 流程
engine, _ := data.NewDB(false, c.Data.Database)
if err := migrations.NewMentor(ctx, engine, inputData).InitDB(); err != nil {
    // ...
}
```

而在 Mentor.InitDB 的 **第一步**（[init.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/init.go#L65-L110)）就是第四道防线：

```go
func (m *Mentor) InitDB() error {
    m.do("check table exist", m.checkTableExist)  // 第一步就检查
    m.do("sync table", m.syncTable)
    // ... 后续步骤
}

func (m *Mentor) checkTableExist() {
    m.Done, m.err = m.engine.Context(m.ctx).IsTableExist(&entity.Version{})
    if m.Done {
        fmt.Println("[database] already exists")
    }
}

// Mentor.do 的短路径逻辑：
func (m *Mentor) do(taskName string, fn func()) {
    if m.err != nil || m.Done {
        return  // ← 若 Done=true，所有后续步骤全部跳过
    }
    fn()
}
```

### 4.5 版本迁移机制：升级而非重装

数据库版本通过 `version` 表的单条记录（ID=1）维护：

```
version 表：
  id=1, version_number = 32
```

在 [migrations.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/migrations.go#L117-L198) 中：

```go
func GetCurrentDBVersion(engine *xorm.Engine) (int64, error) {
    engine.Sync(new(entity.Version))            // 确保 version 表存在
    currentVersion := &entity.Version{ID: 1}
    has, _ := engine.Get(currentVersion)
    if !has {
        engine.InsertOne(&entity.Version{ID: 1, VersionNumber: 0})  // 首次创建
        return 0, nil
    }
    return currentVersion.VersionNumber, nil
}

func Migrate(...) error {
    currentDBVersion, _ := GetCurrentDBVersion(engine)
    expectedVersion := ExpectedVersion()  // = minDBVersion(0) + len(migrations)
    
    for currentDBVersion < expectedVersion {
        migrationFunc := migrations[currentDBVersion]
        migrationFunc.Migrate(context.Background(), engine)  // 执行迁移
        if migrationFunc.ShouldCleanCache() { cache.Flush() }
        engine.Update(&entity.Version{ID: 1, VersionNumber: currentDBVersion + 1})
        currentDBVersion++
    }
    return nil
}
```

**迁移文件版本编号**：`v0.0.1` → `v2.0.1` 共 32 个迁移脚本，对应 `version_number` 0→32。

---

## 五、SchemaForm 渲染与校验失败处理

### 5.1 SchemaForm 的设计定位

[SchemaForm](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/ui/src/components/SchemaForm/index.tsx#L78-L456) 是一个 **JSON Schema 驱动的通用表单组件**，用于管理后台的动态配置页面（站点设置、插件配置等）。

**设计思想**：后端提供 `schema`（字段定义）+ `uiSchema`（渲染方式），前端根据配置自动生成表单。

### 5.2 核心数据结构

在 [types.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/ui/src/components/SchemaForm/types.ts#L27-L213) 中：

```typescript
// JSON Schema：定义数据模型
interface JSONSchema {
    title: string;
    required?: string[];        // 必填字段列表
    properties: {
        [key: string]: {
            type?: 'string' | 'boolean' | 'number';
            title: string;              // 字段 Label
            description?: string;       // 提示文案（支持 HTML）
            enum?: Array<string|boolean|number>;   // 枚举值
            enumNames?: string[];       // 枚举显示名
            min?: number; max?: number; // 数值范围
            default?: any;              // 默认值
            max_length?: number;        // 最大长度
        };
    };
}

// UI Schema：定义渲染方式
interface UISchema {
    [key: string]: {
        'ui:widget'?: UIWidget;   // 用什么控件渲染
        'ui:options'?: UIOptions; // 控件配置
    };
}

type UIWidget = 
    | 'textarea' | 'input' | 'checkbox' | 'radio' 
    | 'select' | 'upload' | 'timezone' | 'switch'
    | 'legend' | 'button' | 'input_group' | 'tag_selector';
```

### 5.3 渲染分发逻辑

在 `SchemaForm` 组件的 return 中，通过遍历 `schema.properties` 的 key，根据 `ui:widget` 的值进行分发渲染：

```tsx
{keys.map((key) => {
    const { title, enum: enumValues, ... } = properties[key];
    const { 'ui:widget': widget = 'input', 'ui:options': uiOpt } = uiSchema?.[key] || {};
    
    return (
        <Form.Group controlId={key}>
            {title && !uiSimplify ? <Form.Label>{title}</Form.Label> : null}
            
            {/* 11 种 widget 的渲染分支 */}
            {widget === 'legend'   ? <Legend .../> : null}
            {widget === 'select'   ? <Select .../> : null}
            {widget === 'radio' || widget === 'checkbox' ? <Check type={widget} .../> : null}
            {widget === 'switch'   ? <Switch .../> : null}
            {widget === 'timezone' ? <Timezone .../> : null}
            {widget === 'upload'   ? <Upload .../> : null}
            {widget === 'textarea' ? <Textarea .../> : null}
            {widget === 'input'    ? <Input .../> : null}
            {widget === 'button'   ? <SfButton .../> : null}
            {widget === 'input_group' ? <InputGroup>...</InputGroup> : null}
            {widget === 'tag_selector' ? <TagSelector .../> : null}
            
            {/* 字段描述 + 错误反馈（统一渲染） */}
            {description && widget !== 'tag_selector' ? (
                <Form.Text dangerouslySetInnerHTML={{ __html: description }} />
            ) : null}
            <Form.Control.Feedback type="invalid">
                {fieldState?.errorMsg}
            </Form.Control.Feedback>
        </Form.Group>
    );
})}
```

### 5.4 前端校验流程：requiredValidator → syncValidator

SchemaForm 的 `validator()` 方法（支持 Promise，通过 ref 暴露）采用两段式校验：

```typescript
const validator = async (): Promise<boolean> => {
    // ── 第一阶段：必填校验 ────────────────────────────────
    const errors = requiredValidator();  // 遍历 schema.required
    if (errors.length > 0) {
        // 回填 isInvalid + errorMsg（使用 ui:options.empty 自定义文案）
        formData = errors.reduce((acc, cur) => {
            acc[cur] = {
                ...formData![cur],
                isInvalid: true,
                errorMsg: uiSchema[cur]?.['ui:options']?.empty 
                         || `${properties[cur]?.title} empty`,
            };
            return acc;
        }, formData || {});
        onChange({ ...formData });
        scrollElementIntoView(document.getElementById(errors[0]));  // 自动滚动
        return false;
    }
    
    // ── 第二阶段：自定义异步校验 ──────────────────────────
    const syncErrors = await syncValidator();
    if (syncErrors.length > 0) {
        // 类似回填逻辑，错误信息来自 validator 返回值
        formData = syncErrors.reduce((acc, cur) => {
            acc[cur.key] = {
                ...formData![cur.key],
                isInvalid: true,
                errorMsg: cur.msg || `${properties[cur.key].title} invalid`,
            };
            return acc;
        }, formData || {});
        onChange({ ...formData });
        scrollElementIntoView(document.getElementById(syncErrors[0].key));
        return false;
    }
    return true;
};
```

**自定义校验器的配置方式**（通过 `ui:options.validator`）：

```typescript
// 在 uiSchema 中定义：
password: {
    'ui:widget': 'input',
    'ui:options': {
        inputType: 'password',
        // 返回 Promise<string> | string | true | void
        // string 视为错误信息，true/void 视为通过
        validator: async (value, formData) => {
            if (value.length < 8) return '密码至少 8 位';
            if (value !== formData.confirm_password?.value) return '两次密码不一致';
        }
    }
}
```

实现上使用 `Promise.allSettled`，确保即使部分校验器抛错也能收集全部错误。

### 5.5 提交流程：handleSubmit

```typescript
const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    
    // 先跑前端校验
    const isValid = await validator();
    if (!isValid) return;
    
    // 清除所有字段的错误态（保留值）
    Object.keys(formData!).forEach((key) => {
        formData![key].isInvalid = false;
        formData![key].errorMsg = '';
    });
    onChange(formData!);
    
    // 触发业务方的 onSubmit（通常是发送 PUT 请求更新站点配置）
    if (onSubmit instanceof Function) {
        onSubmit(e);
    }
};
```

### 5.6 初始化辅助函数：initFormData / mergeFormData

SchemaForm 还导出两个工具函数，简化后端配置到表单数据的转换：

```typescript
// 从 JSON Schema 生成默认 formData
export const initFormData = (schema: JSONSchema): Type.FormDataType => {
    const formData: Type.FormDataType = {};
    Object.keys(props).forEach((key) => {
        const prop = props[key];
        formData[key] = {
            value: prop?.default || '',  // 取 schema 中的 default 值
            isInvalid: false,
            errorMsg: '',
        };
    });
    return formData;
};

// 后端返回数据回填：target（schema默认） ∪ origin（数据库现有值）
export const mergeFormData = (target, origin) => {
    Object.keys(target).forEach((k) => {
        const oi = origin[k];
        if (oi && oi.value !== undefined) {
            target[k] = { value: oi.value, isInvalid: false, errorMsg: '' };
        }
    });
    return target;
};
```

---

## 六、关键代码索引

| 功能模块 | 文件路径 | 核心函数/结构体 |
|---------|---------|----------------|
| CLI 安装入口 | [command.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/command.go#L115-L142) | `initCmd` |
| 安装服务器启动 | [install_main.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_main.go#L41-L62) | `Run()` |
| 安装路由注册 | [install_server.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_server.go#L50-L78) | `NewInstallHTTPServer()` |
| 安装控制器 | [install_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_controller.go) | `CheckDatabase`/`InitEnvironment`/`InitBaseInfo` |
| 自动安装（环境变量）| [install_from_env.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_from_env.go) | `TryToInstallByEnv()`/`requestAPI()` |
| 请求结构体+校验 | [install_req.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_req.go) | `CheckDatabaseReq`/`InitBaseInfoReq` |
| 配置文件/DB 检查 | [install_check.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/cli/install_check.go) | `CheckConfigFile`/`CheckDBConnection`/`CheckDBTableExist` |
| 配置文件安装 | [install.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/cli/install.go) | `InstallConfigFile`/`InstallAllInitialEnvironment` |
| 数据库初始化 Mentor | [init.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/init.go) | `Mentor.InitDB()`/`checkTableExist()` |
| 版本迁移机制 | [migrations.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/migrations.go) | `GetCurrentDBVersion()`/`Migrate()` |
| 配置读写 | [conf.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/conf/conf.go) | `ReadConfig()`/`RewriteConfig()` |
| Handler 绑定+校验 | [handler.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/handler/handler.go#L64-L78) | `BindAndCheck()` |
| 验证器引擎 | [validator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/validator/validator.go) | `MyValidator.Check()`/`Checker` 接口 |
| 错误响应结构 | [response.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/handler/response.go) | `RespBody`/`FormErrorField` |
| 错误类型标识 | [err_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/schema/err_schema.go) | `ErrTypeAlert`/`ErrTypeModal`/`ErrTypeToast` |
| 安装前端页面 | [Install/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/ui/src/pages/Install/index.tsx) | `configYmlCheck()`/`submitSiteConfig()` |
| SchemaForm 组件 | [SchemaForm/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/ui/src/components/SchemaForm/index.tsx) | `validator()`/`handleSubmit()`/widget 分发 |
| SchemaForm 类型 | [SchemaForm/types.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/ui/src/components/SchemaForm/types.ts) | `JSONSchema`/`UISchema`/`UIWidget` |
| Run 模式下的边界 | [ui.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/router/ui.go#L127-L130) | NoRoute 中 `/install` → `/` 重定向 |

---

## 七、总结：设计模式与边界思想

### 7.1 初始化的三段式结构
1. **环境准备**（InstallAllInitialEnvironment）→ 目录、i18n、上传等资源就位
2. **配置写入**（InitEnvironment）→ 配置文件创建 + DB 连接信息写入
3. **数据库初始化**（InitBaseInfo → Mentor.InitDB）→ 表结构、初始数据、管理员账户

三段之间每一步都有 **幂等检查**，即使重复触发也不会产生副作用。

### 7.2 校验的 "双端三道防线"
- **前端**：SchemaForm 的 required 校验 + 自定义 validator（快速反馈）
- **后端 Tag**：Struct validate tag（结构化约束）
- **后端业务**：Checker 接口 + DB 真实连接测试（最终正确性保障）

错误信息通过 `error_field` + `error_msg` 字段级返回，前端可以精确定位并高亮。

### 7.3 恢复边界的四重锁
| 边界位置 | 保护方式 | 目的 |
|---------|---------|------|
| 安装 API 入口 | config 文件/DB 表存在性检查 | 防止已安装系统被重新初始化 |
| Mentor 执行流 | Done 标志的短路机制 | 防止迁移中途重启后重复执行初始步骤 |
| migrations 版本号 | version 表递增记录 | 版本升级时只执行未走过的迁移脚本 |
| Run 模式路由 | `/install` → `/` 重定向 | 正式运行时完全关闭安装入口 |

这种设计确保了从 **首次部署** → **日常运行** → **版本升级** 的每个阶段，都有明确的边界约束，不会发生状态错乱。

### 7.4 Schema Driven 的价值
- 新增站点配置项只需后端返回新的 schema 定义，无需改前端代码
- 校验逻辑集中在 schema 的 `required` + `ui:options.validator` 中，业务代码聚焦在接口调用
- 11 种 widget 覆盖了后台配置页面的绝大多数输入场景

---

## 八、SchemaForm 的 Promise.allSettled 并行校验流程深度解析

### 8.1 syncValidator 的完整执行流

在 [SchemaForm/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/ui/src/components/SchemaForm/index.tsx#L148-L188) 中，`syncValidator` 的实现通过 `Promise.allSettled` 实现字段级并行校验：

```typescript
const syncValidator = () => {
    const errors: Array<{ key: string; msg: string }> = [];
    const promises: Array<{
        key: string;
        promise;
    }> = [];

    // 第一阶段：收集所有声明了 validator 的字段
    keys.forEach((key) => {
        const { validator } = uiSchema[key]?.['ui:options'] || {};
        if (validator instanceof Function) {
            const value = formData![key]?.value;
            promises.push({
                key,
                promise: validator(value, formData),  // 立即调用，拿到 Promise 或同步值
            });
        }
    });

    // 第二阶段：并行等待所有校验器完成
    return Promise.allSettled(promises.map((item) => item.promise)).then(
        (results) => {
            results.forEach((result, index) => {
                const { key } = promises[index];

                if (result.status === 'rejected') {
                    // 校验器本身抛出异常（如网络错误、类型错误）
                    errors.push({
                        key,
                        msg: result.reason.message,  // 取 reject reason 的 message
                    });
                }

                if (result.status === 'fulfilled') {
                    const msg = result.value;
                    if (typeof msg === 'string') {
                        // 校验器返回字符串 → 视为校验失败的错误信息
                        errors.push({ key, msg });
                    }
                    // 返回 true / undefined / void → 校验通过，不收集
                }
            });
            return errors;
        },
    );
};
```

### 8.2 三种校验器返回值的语义约定

`ui:options.validator` 的返回值被 `syncValidator` 解读为三种语义：

| 返回值 | 类型 | 语义 | 处理方式 |
|-------|------|------|---------|
| `true` / `undefined` / `void` | `boolean(true)` 或 `undefined` | 字段校验通过 | `result.value` 不是 string → 不收集 |
| `"错误信息"` | `string` | 字段校验失败 | `typeof msg === 'string'` → 收集到 errors |
| `throw Error` | 异常 | 校验器本身出错 | `result.status === 'rejected'` → 收集 `reason.message` |

**设计精妙之处**：这三种返回值覆盖了校验器可能的所有状态——通过、业务失败、技术异常——而无需定义额外的 Result 类型。

### 8.3 Promise.allSettled vs Promise.all 的选择

| 维度 | Promise.all | Promise.allSettled |
|------|------------|-------------------|
| 某个校验器 reject | 立即短路，丢弃其他校验结果 | 继续等待，收集所有结果 |
| 返回值类型 | `Promise<any[]>` | `Promise<PromiseSettledResult<any>[]>` |
| 适用场景 | 必须全部成功才有意义 | 需要完整错误报告 |

**SchemaForm 选择 `allSettled` 的原因**：
- 用户期望 **一次提交看到所有字段的错误**，而非修一个提交一次
- 部分校验器可能涉及异步操作（如 URL 可达性检查），不应因一个慢请求而丢弃其他错误
- 即使某个校验器代码有 bug（throw），也不会阻断其他字段的校验结果

### 8.4 校验器执行的时间线

```
t0  ─── 所有 validator 函数被同步调用，拿到 Promise/值
        │
        ├── validator_A(value) → Promise<string>  (异步，如 URL 校验)
        ├── validator_B(value) → true              (同步，立即 resolve)
        ├── validator_C(value) → Promise<true>     (异步，如 API 调用)
        └── validator_D(value) → "格式错误"         (同步，立即 resolve)
        │
t1  ─── Promise.allSettled 并行等待
        │
t2  ─── 所有 Promise settled
        │
        ├── results[0]: { status: 'fulfilled', value: 'URL 不可达' }  → 收集
        ├── results[1]: { status: 'fulfilled', value: true }          → 忽略
        ├── results[2]: { status: 'fulfilled', value: true }          → 忽略
        └── results[3]: { status: 'fulfilled', value: '格式错误' }    → 收集
        │
t3  ─── 返回 errors = [{ key: 'site_url', msg: 'URL 不可达' }, { key: 'email', msg: '格式错误' }]
        → reduce 回填到 formData → onChange 触发重渲染 → 显示红色错误提示
```

### 8.5 实际应用示例：Admin General 页面的 site_url 校验

在 [Admin/General/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/ui/src/pages/Admin/General/index.tsx#L76-L112) 中：

```typescript
const uiSchema: UISchema = {
    site_url: {
        'ui:options': {
            inputType: 'url',
            validator: (value) => {
                let url: URL | undefined;
                try {
                    url = new URL(value);       // 合法则解析成功
                } catch (ex) {
                    return t('site_url.validate');  // 返回 string → 校验失败
                }
                if (
                    !url ||
                    !/^https?:$/.test(url.protocol) ||
                    (REACT_BASE_PATH && url.pathname !== REACT_BASE_PATH) ||
                    (!REACT_BASE_PATH && url.pathname !== '/') ||
                    url.search !== '' ||
                    url.hash !== ''
                ) {
                    return t('site_url.validate');  // 返回 string → 规则不满足
                }
                return true;  // 返回 true → 校验通过
            },
        },
    },
    contact_email: {
        'ui:options': {
            inputType: 'email',
            validator: (value) => {
                if (!Pattern.email.test(value)) {
                    return t('contact_email.validate');  // 返回 string → 校验失败
                }
                return true;
            },
        },
    },
};
```

当 `validator()` 被调用时，`site_url` 和 `contact_email` 两个字段的 validator 会并行执行，最终通过 `Promise.allSettled` 收集结果，一次性展示所有错误。

---

## 九、Migrations：首次安装 InitDB 与版本升级的分支逻辑

### 9.1 两条完全独立的代码路径

```
┌─────────────────────────────────────────────────────────────────┐
│                        answer init                               │
│   InitBaseInfo → Mentor.InitDB()                                │
│   ├─ checkTableExist → Done=true → 全部跳过（已安装）            │
│   └─ checkTableExist → Done=false → 执行 28 步完整初始化         │
│       syncTable → initVersionTable(version=ExpectedVersion)      │
│       → initAdminUser → initConfig → ... → initDefaultBadges     │
│                                                                  │
│   特点：version_number 直接设为 ExpectedVersion（当前=32）         │
│         跳过所有历史迁移脚本                                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                       answer upgrade                             │
│   migrations.Migrate()                                          │
│   ├─ GetCurrentDBVersion() → currentDBVersion = N               │
│   ├─ ExpectedVersion() → expectedVersion = 32                   │
│   └─ for N < 32:                                                │
│       migrations[N].Migrate(ctx, engine)  ← 逐个执行增量迁移     │
│       engine.Update(version_number = N+1)                       │
│                                                                  │
│   特点：从当前版本 N 开始，逐版本执行迁移脚本                      │
│         每执行一步就更新 version_number                           │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 首次安装：Mentor.InitDB 的 28 步原子流程

在 [init.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/init.go#L65-L93) 中：

```go
func (m *Mentor) InitDB() error {
    m.do("check table exist", m.checkTableExist)     // [1]  防重入：version 表存在则 Done=true
    m.do("sync table", m.syncTable)                  // [2]  创建所有表结构
    m.do("init version table", m.initVersionTable)    // [3]  直接写入 ExpectedVersion
    m.do("init admin user", m.initAdminUser)          // [4]  创建管理员（bcrypt 加密）
    m.do("init config", m.initConfig)                 // [5]  写入默认配置表
    m.do("init default privileges", ...)               // [6]  设置等级 2 的权限值
    m.do("init role", m.initRole)                     // [7]  角色：Admin/Moderator/User...
    m.do("init power", m.initPower)                   // [8]  权限定义
    m.do("init role power rel", ...)                   // [9]  角色-权限关联
    m.do("init admin user role rel", ...)              // [10] 管理员→Admin角色
    m.do("init site info interface", ...)              // [11] 语言+时区
    m.do("init site info users settings", ...)         // [12] 头像/Gravatar
    m.do("init site info general config", ...)         // [13] 站点名/URL/邮箱
    m.do("init site info login config", ...)           // [14] 登录开关
    m.do("init site info theme config", ...)           // [15] 主题+配色
    m.do("init site info seo config", ...)             // [16] 永久链接+robots.txt
    m.do("init site info user config", ...)            // [17] 用户可编辑字段
    m.do("init site info privilege rank", ...)         // [18] 权限等级
    m.do("init site info advanced", ...)               // [19] 图片/附件大小限制
    m.do("init site info questions", ...)              // [20] 问题配置
    m.do("init site info tags", ...)                   // [21] 标签配置
    m.do("init site info security", ...)               // [22] 安全配置
    m.do("init default content", ...)                  // [23] 默认标签+2个问题+2个回答
    m.do("init default badges", ...)                   // [24] 徽章
    m.do("init default ai config", ...)                // [25] AI 配置
    m.do("init default MCP config", ...)               // [26] MCP 配置
    return m.err
}
```

**关键设计**：第 [3] 步 `initVersionTable` 直接写入 `ExpectedVersion()`（当前 = 32），这意味着首次安装后无需再跑任何迁移脚本。

```go
func (m *Mentor) initVersionTable() {
    _, m.err = m.engine.Context(m.ctx).Insert(
        &entity.Version{ID: 1, VersionNumber: ExpectedVersion()},
    )
}
```

### 9.3 版本升级：Migrate 的增量循环

在 [migrations.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/migrations.go#L144-L198) 中：

```go
func Migrate(debug bool, dbConf *data.Database, cacheConf *data.CacheConf, 
             upgradeToSpecificVersion string) error {
    cache, cacheCleanup, _ := data.NewCache(cacheConf)
    engine, err := data.NewDB(debug, dbConf)
    if err != nil {
        return err  // DB 连接失败直接返回
    }
    defer engine.Close()

    currentDBVersion, err := GetCurrentDBVersion(engine)
    expectedVersion := ExpectedVersion()  // minDBVersion(0) + len(migrations) = 32

    // 特殊分支：用户指定从某个版本开始升级
    if len(upgradeToSpecificVersion) > 0 {
        for i, m := range migrations {
            if m.Version() == upgradeToSpecificVersion {
                currentDBVersion = int64(i)  // 重置当前版本号
                break
            }
        }
    }

    for currentDBVersion < expectedVersion {
        migrationFunc := migrations[currentDBVersion]
        if err := migrationFunc.Migrate(context.Background(), engine); err != nil {
            return err  // 迁移失败立即终止，版本号不递增
        }
        if migrationFunc.ShouldCleanCache() {
            cache.Flush(context.Background())
        }
        engine.Update(&entity.Version{ID: 1, VersionNumber: currentDBVersion + 1})
        currentDBVersion++
    }
    return nil
}
```

### 9.4 两条路径的核心差异对比

| 维度 | 首次安装（Mentor.InitDB） | 版本升级（Migrate） |
|------|--------------------------|-------------------|
| 触发命令 | `answer init` → 安装向导 | `answer upgrade` |
| 入口函数 | `Mentor.InitDB()` | `migrations.Migrate()` |
| 数据来源 | 用户在安装向导中输入的数据 | 已有数据库 + 迁移脚本 |
| version_number | 直接写入 `ExpectedVersion()`（32） | 从当前值逐次递增到 `ExpectedVersion` |
| 表创建方式 | `engine.Sync(tables...)` 一次性同步全部表 | 迁移脚本中增量 ALTER TABLE |
| 初始数据 | 插入管理员、默认配置、默认徽章、示例内容 | 迁移脚本中只增量修改 |
| 失败策略 | `m.err` 短路，后续步骤全部跳过 | `return err` 立即终止，版本号不递增 |
| 幂等保障 | `checkTableExist` → `Done` 标志 | `version_number` 已递增 → 跳过已执行迁移 |
| 缓存处理 | 无 | `ShouldCleanCache()` → `cache.Flush()` |

### 9.5 升级命令的特殊分支：-f 参数

```bash
answer upgrade -f v1.1.0
```

当指定 `-f` 参数时，[command.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/command.go#L77) 会传入 `upgradeVersion`，`Migrate` 函数会回溯版本号：

```go
if len(upgradeToSpecificVersion) > 0 {
    for i, m := range migrations {
        if m.Version() == upgradeToSpecificVersion {
            currentDBVersion = int64(i)  // 将当前版本重置为该版本对应的索引
            break
        }
    }
}
```

**用途**：当某次升级失败后，可以重新从指定版本开始执行迁移，而不必重装整个数据库。

---

## 十、Admin 端 SiteInfoReq 系列复用 BindAndCheck 但 UI 使用 SchemaForm 的区别

### 10.1 后端：统一的 BindAndCheck 校验管线

Admin 端的每一个站点配置更新接口都使用相同的 `BindAndCheck` 模式。以 [controller_admin/siteinfo_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/controller_admin/siteinfo_controller.go#L288-L296) 为例：

```go
func (sc *SiteInfoController) UpdateGeneral(ctx *gin.Context) {
    req := schema.SiteGeneralReq{}
    if handler.BindAndCheck(ctx, &req) {   // ← 统一入口
        return
    }
    err := sc.siteInfoService.SaveSiteGeneral(ctx, req)
    req.Name = html.UnescapeString(req.Name)
    handler.HandleResponse(ctx, err, req)
}
```

所有 Update 方法的结构完全一致：
```
1. 声明 req 结构体（SiteGeneralReq / SiteInterfaceReq / SiteBrandingReq / ...）
2. handler.BindAndCheck(ctx, &req)  → ShouldBind + StructTag 校验 + Checker 校验
3. service 层保存
4. HandleResponse
```

对应的请求结构体在 [siteinfo_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/schema/siteinfo_schema.go#L39-L45) 中，同样使用 `validate` tag：

```go
type SiteGeneralReq struct {
    Name             string `validate:"required,sanitizer,gt=1,lte=128" json:"name"`
    ShortDescription string `validate:"omitempty,sanitizer,gt=3,lte=255" json:"short_description"`
    Description      string `validate:"omitempty,sanitizer,gt=3,lte=2000" json:"description"`
    SiteUrl          string `validate:"required,sanitizer,gt=1,lte=512,url" json:"site_url"`
    ContactEmail     string `validate:"required,sanitizer,gt=1,lte=512,email" json:"contact_email"`
}
```

### 10.2 前端：Admin 站点配置页面使用 SchemaForm

与安装向导（手写表单 + 直接调用 API）不同，**Admin 后台的站点配置页面大量使用 SchemaForm 组件**。以 [Admin/General/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/ui/src/pages/Admin/General/index.tsx#L32-L184) 为典型代表：

```typescript
const General: FC = () => {
    const { data: setting } = useGeneralSetting();

    // 前端定义 schema（非后端返回）
    const schema: JSONSchema = {
        title: t('page_title'),
        required: ['name', 'site_url', 'contact_email'],
        properties: {
            name: { type: 'string', title: t('name.label'), description: t('name.text') },
            site_url: { type: 'string', title: t('site_url.label'), description: t('site_url.text') },
            // ...
        },
    };

    // 前端定义 uiSchema（含自定义 validator）
    const uiSchema: UISchema = {
        site_url: {
            'ui:options': {
                inputType: 'url',
                validator: (value) => {
                    try { new URL(value); } catch { return t('site_url.validate'); }
                    // ... 更多规则
                    return true;
                },
            },
        },
    };

    return (
        <SchemaForm schema={schema} formData={formData} uiSchema={uiSchema} ... />
    );
};
```

### 10.3 安装向导 vs Admin 后台的表单方案对比

| 维度 | 安装向导 Install 页面 | Admin 后台配置页面 |
|------|---------------------|-------------------|
| 表单实现 | 手写表单组件（FirstStep/SecondStep/FourthStep） | SchemaForm 通用组件 |
| Schema 来源 | 无（字段硬编码在 JSX 中） | 前端定义 JSONSchema + UISchema |
| 校验方式 | 后端 BindAndCheck → 前端 handleFormError 回填 | 前端 requiredValidator + syncValidator → 后端 BindAndCheck |
| 字段错误回填 | `handleFormError(err, formData)` 手动处理 | SchemaForm 内置 `validator()` 自动回填 |
| UI 一致性 | 每步独立样式，步骤间切换 | 统一 SchemaForm 渲染风格 |
| 可扩展性 | 新增字段需修改组件 | 新增字段只需修改 schema 对象 |
| 后端校验 | 相同（BindAndCheck → validate tag + Checker） | 相同（BindAndCheck → validate tag + Checker） |

### 10.4 插件配置：SchemaForm 的动态 Schema 场景

与 Admin 站点配置（前端硬编码 schema）不同，**插件配置页面的 schema 来自后端动态返回**。

在 [Admin/Plugins/Config/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/ui/src/pages/Admin/Plugins/Config/index.tsx#L43-L85) 中：

```typescript
useEffect(() => {
    if (!data) return;

    const properties: JSONSchema['properties'] = {};
    const uiConf: UISchema = {};
    const required: string[] = [];

    data.config_fields?.forEach((item) => {
        properties[item.name] = {
            type: 'string',
            title: item.title,
            description: item.description,
            default: item.value,
        };

        if (item.options instanceof Array) {
            properties[item.name].enum = item.options.map((o) => o.value);
            properties[item.name].enumNames = item.options.map((o) => o.label);
        }

        uiConf[item.name] = {};
        uiConf[item.name]['ui:widget'] = item.type;     // 后端决定 widget 类型
        if (item.ui_options) {
            uiConf[item.name]['ui:options'] = item.ui_options;
        }

        if (item.required) {
            required.push(item.name);
        }
    });

    setFormData(initFormData(result));
    setSchema(result);
    setUISchema(uiConf);
}, [data?.config_fields]);
```

**三种 Schema 来源对比**：

| 场景 | Schema 定义位置 | 动态性 | 例子 |
|------|---------------|-------|------|
| 安装向导 | 无 Schema（硬编码表单） | 静态 | Install 页面的 DB 配置、站点信息 |
| Admin 站点配置 | 前端 TypeScript 定义 | 半静态（修改代码才变化） | General、Interface、SEO |
| 插件配置 | 后端 API 返回 `config_fields` | 完全动态 | 第三方插件的自定义配置项 |

### 10.5 为什么安装向导不使用 SchemaForm？

1. **安装向导是多步骤向导**，每步有独立的交互逻辑（DB 连接测试、配置文件创建结果展示、进度动画），SchemaForm 的单表单模型无法表达
2. **安装向导的错误展示方式不同**：使用 `ErrTypeAlert`（红色横幅）而非字段级 `isInvalid`
3. **安装向导需要安装模式特有的检测**：`configYmlCheck`（配置文件存在性）、`CheckDBConnection`（数据库连通性），这些是 SchemaForm 不具备的
4. **安装完成后进程退出**（`os.Exit(0)`），不涉及持久化的配置页面

---

## 十一、Config 损坏退出与 Run 模式下 DB 断开的降级恢复边界

### 11.1 Config 文件损坏时的行为矩阵

#### 场景一：`answer init` 阶段 config 损坏

在 [command.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/command.go#L123-L131) 中：

```go
configFileExist := cli.CheckConfigFile(path.GetConfigFilePath())
if configFileExist {
    fmt.Println("config file exists, try to read the config...")
    c, err := conf.ReadConfig(path.GetConfigFilePath())
    if err != nil {
        fmt.Println("read config failed: ", err.Error())
        return  // ← 优雅退出，不 panic
    }
    if cli.CheckDBTableExist(c.Data.Database) {
        fmt.Println("connect to database successfully and table already exists, do nothing.")
        return
    }
}
```

**行为**：配置文件存在但 YAML 解析失败 → 打印错误信息 → 正常退出。**不会启动安装服务器**，因为系统认为已经处于某种已配置状态（文件存在），但无法确认是否已安装。

**恢复路径**：管理员需要手动删除损坏的 `config.yaml` 后重新执行 `answer init`。

#### 场景二：安装向导中 CheckConfigFile 接口检测到 config 可读但 DB 不可连

在 [install_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_controller.go#L101-L120) 的 `CheckConfigFile` 中：

```go
func CheckConfigFile(ctx *gin.Context) {
    resp := &CheckConfigFileResp{}
    resp.ConfigFileExist = cli.CheckConfigFile(confPath)
    if !resp.ConfigFileExist {
        handler.HandleResponse(ctx, nil, resp)
        return
    }

    allConfig, err := conf.ReadConfig(confPath)
    if err != nil {
        log.Error(err)
        err = errors.BadRequest(reason.ReadConfigFailed)
        handler.HandleResponse(ctx, err, nil)  // ← 返回 ReadConfigFailed 错误
        return
    }

    resp.DBConnectionSuccess = cli.CheckDBConnection(allConfig.Data.Database)
    if resp.DBConnectionSuccess {
        resp.DbTableExist = cli.CheckDBTableExist(allConfig.Data.Database)
    }
    handler.HandleResponse(ctx, nil, resp)
}
```

前端收到响应后的分支处理（[Install/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/ui/src/pages/Install/index.tsx)）：

| CheckConfigFile 响应 | 前端行为 | 用户可见 |
|---------------------|---------|---------|
| `config_file_exist=false` | 跳 step=1（开始安装） | 正常安装流程 |
| `config_file_exist=true, db_connection_success=true, db_table_exist=true` | 跳 step=6（已安装警告） | "系统已安装，如需重装请删除配置文件" |
| `config_file_exist=true, db_connection_success=false` | 跳 step=7（DB 失败） | "数据库连接失败，请检查配置" |
| `config_file_exist=true, ReadConfig 报错` | 顶部 Alert 展示 | "读取配置文件失败" |

#### 场景三：InitEnvironment 中配置文件写入失败

在 [install_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_controller.go#L172-L180) 中：

```go
if err := cli.InstallConfigFile(confPath); err != nil {
    handler.HandleResponse(ctx, errors.BadRequest(reason.InstallConfigFailed), &InitEnvironmentResp{
        Success:            false,
        CreateConfigFailed: true,
        DefaultConfig:      string(configs.Config),  // ← 返回默认配置内容
        ErrType:            schema.ErrTypeAlert.ErrType,
    })
    return
}
```

**降级策略**：返回 `DefaultConfig` 字符串，前端可以让用户手动创建配置文件。这是一个 **半自动恢复** 机制——系统不仅告知失败，还提供了兜底内容。

### 11.2 `answer run` 阶段 config 损坏：硬性边界

在 [main.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/main.go#L75-L79) 中：

```go
func runApp() {
    c, err := conf.ReadConfig(path.GetConfigFilePath())
    if err != nil {
        panic(err)  // ← 硬性退出，没有任何降级
    }
    // ...
}
```

**为什么 run 模式不降级？**
1. Run 模式依赖配置文件中的 DB 连接信息、缓存路径、i18n 路径等，缺少任何一项都无法启动
2. 配置损坏意味着系统处于不确定状态，继续运行可能导致数据损坏
3. 这与安装模式不同——安装模式允许从零开始构建配置，而运行模式必须在已有配置基础上工作

**恢复路径**：
1. 检查日志中的 panic 信息，定位配置解析错误
2. 手动修复 `config.yaml` 或从备份恢复
3. 重新执行 `answer run`

### 11.3 Run 模式下 DB 断开的降级恢复

#### 启动时 DB 不可用

`answer run` 启动时，[data.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/data/data.go#L56-L95) 的 `NewDB` 会执行 `engine.Ping()`：

```go
func NewDB(debug bool, dataConf *data.Database) (*xorm.Engine, error) {
    engine, err := xorm.NewEngine(dataConf.Driver, dataConf.Connection)
    if err != nil {
        return nil, err
    }
    if err = engine.Ping(); err != nil {
        return nil, err  // ← Ping 失败直接返回 error
    }
    return engine, nil
}
```

在 Wire 依赖注入链中，`NewDB` 的 error 会传播到 `initApplication`，最终触发 `panic`。

**启动时 DB 不可用 = 服务无法启动**，这是硬性边界。

#### 运行时 DB 短暂断开

Answer 使用 xorm 作为 ORM 引擎，默认配置了连接池参数：

```go
if dataConf.MaxIdleConn > 0 {
    engine.SetMaxIdleConns(dataConf.MaxIdleConn)    // 保持空闲连接
}
if dataConf.MaxOpenConn > 0 {
    engine.SetMaxOpenConns(dataConf.MaxOpenConn)    // 最大打开连接数
}
if dataConf.ConnMaxLifeTime > 0 {
    engine.SetConnMaxLifetime(time.Duration(dataConf.ConnMaxLifeTime) * time.Second)
}
```

**运行时 DB 断开的处理方式**：
- **读请求**：xorm 执行 SQL 失败 → service 层返回 error → controller 层通过 `HandleResponse` 返回 HTTP 500
- **写请求**：同上，事务回滚，不会产生脏数据
- **无自动重连**：xorm 本身不提供断线重连机制，依赖数据库驱动的重连能力
- **无降级页面**：Run 模式下没有类似安装向导的 "50x" 错误页面机制

#### 安装模式下 config 存在但 DB 断开的降级

安装模式有专门的降级路径。在 [install_server.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_server.go#L64-L66) 中：

```go
installApi.GET(c.UI.BaseURL+"/", CheckConfigFileAndRedirectToInstallPage)
```

[install_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_controller.go#L85-L91) 的实现：

```go
func CheckConfigFileAndRedirectToInstallPage(ctx *gin.Context) {
    if cli.CheckConfigFile(confPath) {
        ctx.Redirect(http.StatusFound, "/50x")    // 配置存在但可能有问题 → 50x 页面
    } else {
        ctx.Redirect(http.StatusFound, "/install") // 配置不存在 → 安装页面
    }
}
```

**安装模式的降级路径**：

```
访问 /
  ├── config.yaml 不存在 → 重定向 /install → 正常安装
  └── config.yaml 存在 → 重定向 /50x → 错误提示页面
       ├── DB 可连 + 表存在 → CheckConfigFile 返回全部 true → 前端 step=6
       ├── DB 不可连 → CheckConfigFile 返回 db_connection_success=false → 前端 step=7
       └── config 损坏 → ReadConfig 报错 → 前端 Alert 提示
```

### 11.4 降级恢复边界总览

| 运行阶段 | 故障类型 | 处理方式 | 恢复路径 |
|---------|---------|---------|---------|
| `answer init` 启动 | config 损坏 | 打印错误，不启动安装服务器 | 手动删除 config.yaml → 重新 init |
| 安装向导中 | config 写入失败 | 返回 DefaultConfig 供手动创建 | 用户手动创建 config.yaml |
| 安装向导中 | DB 连接失败 | `ErrTypeAlert` 红色横幅提示 | 用户修正 DB 信息后重试 |
| 安装向导中 | config 存在+DB 不可连 | step=7 页面提示 DB 失败 | 用户检查 DB 服务是否启动 |
| `answer run` 启动 | config 不存在/损坏 | `panic` 硬性退出 | 修复/恢复 config.yaml |
| `answer run` 启动 | DB 不可连 | `panic` 硬性退出（Wire 构建失败） | 检查 DB 服务是否启动 |
| `answer run` 运行中 | DB 短暂断开 | HTTP 500 返回给客户端 | 依赖 xorm 连接池自动重连 |
| `answer run` 运行中 | 访问 /install | 302 重定向到 / | 无法通过 URL 重入安装 |
| `answer upgrade` | config 不可读 | 打印错误，正常退出 | 修复 config.yaml |
| `answer upgrade` | DB 不可连 | `Migrate` 返回 error | 检查 DB 服务 |
| `answer upgrade` | 迁移脚本失败 | 终止，版本号不递增 | 修复后重跑，或用 `-f` 指定版本 |

### 11.5 设计哲学：Fail-Fast vs Graceful-Degradation

系统的故障处理遵循一条清晰的原则：

- **安装阶段**（`answer init`）：**Graceful Degradation** —— 尽可能提供恢复信息和降级路径（返回 DefaultConfig、step=6/7 提示页、Alert 展示），因为安装是用户首次接触系统的体验
- **运行阶段**（`answer run`）：**Fail-Fast** —— 配置/DB 不可用时直接 `panic` 退出，因为运行中的不确定状态比停机更危险
- **升级阶段**（`answer upgrade`）：**Fail-Fast + 可续传** —— 迁移失败立即终止，但版本号不递增，下次可以从断点继续；`-f` 参数提供了手动指定续传点的能力

这种分层策略确保了：安装时给用户最大帮助，运行时给数据最大保护，升级时给操作最大确定性。

---

## 十五、计数差异纠偏：22 个 Req 结构体 / 24 个 Update 端点 / 26 次 m.do 调用的精确清单

### 15.1 Mentor.InitDB 实际 26 次 m.do 调用（而非 28 步）

在 [init.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/init.go#L65-L92) 中逐行统计：

| # | taskName 字符串 | 实际执行函数 |
|---|----------------|------------|
| 1 | `"check table exist"` | `m.checkTableExist` |
| 2 | `"sync table"` | `m.syncTable` |
| 3 | `"init version table"` | `m.initVersionTable` |
| 4 | `"init admin user"` | `m.initAdminUser` |
| 5 | `"init config"` | `m.initConfig` |
| 6 | `"init default privileges config"` | `m.initDefaultRankPrivileges` |
| 7 | `"init role"` | `m.initRole` |
| 8 | `"init power"` | `m.initPower` |
| 9 | `"init role power rel"` | `m.initRolePowerRel` |
| 10 | `"init admin user role rel"` | `m.initAdminUserRoleRel` |
| 11 | `"init site info interface"` | `m.initSiteInfoInterface` |
| 12 | `"init site info users settings"` | `m.initSiteInfoUsersSettings` |
| 13 | `"init site info general config"` | `m.initSiteInfoGeneralData` |
| 14 | `"init site info login config"` | `m.initSiteInfoLoginConfig` |
| 15 | `"init site info theme config"` | `m.initSiteInfoThemeConfig` |
| 16 | `"init site info seo config"` | `m.initSiteInfoSEOConfig` |
| 17 | `"init site info user config"` | `m.initSiteInfoUsersConfig` |
| 18 | `"init site info privilege rank"` | `m.initSiteInfoPrivilegeRank` |
| 19 | `"init site info write"` | `m.initSiteInfoAdvanced` |
| 20 | `"init site info write"` | `m.initSiteInfoQuestions` |
| 21 | `"init site info write"` | `m.initSiteInfoTags` |
| 22 | `"init site info security"` | `m.initSiteInfoSecurityConfig` |
| 23 | `"init default content"` | `m.initDefaultContent` |
| 24 | `"init default badges"` | `m.initDefaultBadges` |
| 25 | `"init default ai config"` | `m.initSiteInfoAI` |
| 26 | `"init default MCP config"` | `m.initSiteInfoMCP` |

**之前误记为 28 步的原因**：第 19/20/21 三步共用同一个 taskName `"init site info write"`，如果按 taskName 去重会漏记为 24；如果按函数语义粗分（如把 advanced/questions/tags 视为一个大步骤）容易误记为 28。实际 `m.do()` 函数调用次数为 **26 次**。

### 15.2 controller_admin 实际 24 个 Update* 端点

grep `func (.*Controller) Update\w+` 精确统计 [controller_admin](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/controller_admin) 目录结果为 **24 个**：

| # | Controller | Handler 函数 | HTTP 方法 | 路由路径 | 请求结构体 |
|---|-----------|-------------|----------|---------|-----------|
| 1 | SiteInfo | `UpdateSeo` | PUT | /siteinfo/seo | SiteSeoReq |
| 2 | SiteInfo | `UpdateGeneral` | PUT | /siteinfo/general | SiteGeneralReq |
| 3 | SiteInfo | `UpdateInterface` | PUT | /siteinfo/interface | SiteInterfaceReq |
| 4 | SiteInfo | `UpdateUsersSettings` | PUT | /siteinfo/users-settings | SiteUsersSettingsReq |
| 5 | SiteInfo | `UpdateBranding` | PUT | /siteinfo/branding | SiteBrandingReq |
| 6 | SiteInfo | `UpdateSiteQuestion` | PUT | /siteinfo/question | SiteQuestionsReq |
| 7 | SiteInfo | `UpdateSiteTag` | PUT | /siteinfo/tag | SiteTagsReq |
| 8 | SiteInfo | `UpdateSiteAdvanced` | PUT | /siteinfo/advanced | SiteAdvancedReq |
| 9 | SiteInfo | `UpdateSitePolices` | PUT | /siteinfo/polices | SitePoliciesReq |
| 10 | SiteInfo | `UpdateSiteSecurity` | PUT | /siteinfo/security | SiteSecurityReq |
| 11 | SiteInfo | `UpdateSiteLogin` | PUT | /siteinfo/login | SiteLoginReq |
| 12 | SiteInfo | `UpdateSiteCustomCssHTML` | PUT | /siteinfo/custom-css-html | SiteCustomCssHTMLReq |
| 13 | SiteInfo | `UpdateSiteUsers` | PUT | /siteinfo/users | SiteUsersReq |
| 14 | SiteInfo | `UpdateSMTPConfig` | PUT | /setting/smtp | UpdateSMTPConfigReq |
| 15 | SiteInfo | `UpdatePrivilegesConfig` | PUT | /setting/privileges | UpdatePrivilegesConfigReq |
| 16 | SiteInfo | `UpdateAIConfig` | PUT | /ai-config | SiteAIReq |
| 17 | SiteInfo | `UpdateMCPConfig` | PUT | /mcp-config | SiteMCPReq |
| 18 | UserAdmin | `UpdateUserStatus` | PUT | /user/status | UpdateUserStatusReq |
| 19 | UserAdmin | `UpdateUserRole` | PUT | /user/role | UpdateUserRoleReq |
| 20 | UserAdmin | `UpdateUserPassword` | PUT | /user/password | UpdateUserPasswordReq |
| 21 | Badge | `UpdateBadgeStatus` | PUT | /badge/status | UpdateBadgeStatusReq |
| 22 | AdminAPIKey | `UpdateAPIKey` | PUT | /api-key | UpdateAPIKeyReq |
| 23 | Plugin | `UpdatePluginStatus` | PUT | /plugin/status | UpdatePluginStatusReq |
| 24 | Plugin | `UpdatePluginConfig` | PUT | /plugin/config | UpdatePluginConfigReq |

**隐藏的第 25 个写操作端点**：SiteInfo 中的 `SaveSiteTheme`（PUT /siteinfo/theme → SiteThemeReq）命名使用 `Save` 前缀而非 `Update`，所以未出现在 `Update\w+` grep 结果中。因此 **实际 PUT 端点总数为 25**。

### 15.3 SiteInfo 系列 22 个 Req 结构体的完整界定

从 [siteinfo_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/schema/siteinfo_schema.go) 按以下规则统计 22 个结构体（不含纯 Resp 类型）：

| # | 结构体名 | 用途 | validate tag | 特殊方法/标注 |
|---|---------|------|-------------|-------------|
| 1 | `SiteGeneralReq` | 基本信息（站点名/URL/邮箱） | required+santizer+url+email | `FormatSiteUrl()` |
| 2 | `SiteInterfaceReq` | 界面设置（语言/时区） | required | 2 字段 Deprecated 标记为 `json:"-"` |
| 3 | `SiteInterfaceSettingsReq` | 界面设置新版（同字段） | required | `SiteInterfaceSettingsResp` 别名 |
| 4 | `SiteUsersSettingsReq` | 用户设置（头像/Gravatar） | oneof=system gravatar | `SiteUsersSettingsResp` 别名 |
| 5 | `SiteBrandingReq` | 品牌配置（Logo/MobileLogo/SquareIcon/Favicon） | omitempty+长度 | - |
| 6 | `SiteWriteReq` | 旧版写作设置（Deprecated） | 10 个字段混合 | 被拆分为 QuestionsReq/AdvancedReq/TagsReq |
| 7 | `SiteWriteTag` | 标签结构（嵌套子结构） | required=slug_name | - |
| 8 | `SiteQuestionsReq` | 问题配置（最小标签/内容/限制回答） | gte/lte 数值范围 | - |
| 9 | `SiteAdvancedReq` | 高级配置（图片/附件/Megapixel） | omitempty+gt0 | GetMaxImageSize() 单位转换方法 |
| 10 | `SiteTagsReq` | 标签配置（保留/推荐/必填） | dive 嵌套校验 | UserID 字段 `json:"-"` |
| 11 | `SiteLegalReq` | 旧版法律条款（Deprecated） | oneof 外部内容展示 | 被拆分为 PoliciesReq+SecurityReq |
| 12 | `SitePoliciesReq` | 服务条款+隐私政策 | 无（纯文本 HTML） | `SitePoliciesResp` 别名 |
| 13 | `SiteSecurityReq` | 安全配置（登录要求/外部内容/更新检查） | oneof 校验 | - |
| 14 | `GetSiteLegalInfoReq` | 查询法律信息请求 | oneof=tos/privacy | `IsTOS()`/`IsPrivacy()` 方法 |
| 15 | `SiteUsersReq` | 用户可编辑字段配置 | oneof+7 个 bool | - |
| 16 | `SiteLoginReq` | 登录配置（注册/邮箱/密码/域名白名单） | 无 | - |
| 17 | `SiteCustomCssHTMLReq` | 自定义 CSS/HTML（Header/Footer/Sidebar/Head/Css） | gt0+lte=65536 | - |
| 18 | `SiteThemeReq` | 主题配置（Theme/ThemeConfig/ColorScheme/Layout） | oneof=Full-width/Fixed-width | - |
| 19 | `SiteSeoReq` | SEO 配置（Permalink/Robots） | required+gte0+lte4 | `IsShortLink()` 方法 |
| 20 | `SiteAIReq` | AI 配置（启用/provider/models/Prompt） | lte 长度限制 | `GetProvider()` 方法 |
| 21 | `SiteMCPReq` | MCP 配置（Enabled） | omitempty | - |
| 22 | `UpdateSMTPConfigReq` | SMTP 邮件配置 | oneof=SSL/TLS+端口范围 | `Check()` 校验 from_name 非邮箱格式 |

**注**：`UpdatePrivilegesConfigReq` 定义在同文件但属于权限等级配置，不在 "SiteInfo 系列" 严格范围内；`SiteAIProvider` 和 `AIPromptConfig` 是 SiteAIReq 的内嵌子结构体，不计入独立计数。

---

## 十六、xorm 连接池参数的默认值与可配置性深度解析

### 16.1 连接池三参数的 Go/Database 原生语义

xorm 底层通过 `engine.DB()` 拿到 `*sql.DB`，三个参数直接透传到 Go 标准库 `database/sql`：

| 参数 | xorm 方法 | Go sql.DB 方法 | 语义 |
|------|----------|---------------|------|
| MaxOpenConn | `SetMaxOpenConns(n)` | `SetMaxOpenConns(n)` | 同时打开的最大连接数（≤0 表示无限） |
| MaxIdleConn | `SetMaxIdleConns(n)` | `SetMaxIdleConns(n)` | 空闲连接池大小（≤0 表示不保留） |
| ConnMaxLifetime | `SetConnMaxLifetime(d)` | `SetConnMaxLifetime(d)` | 连接最大存活时间（≤0 表示永远复用） |

### 16.2 配置结构体与 config.yaml 映射

在 [config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/data/config.go) 中：

```go
type Database struct {
    Driver          string `yaml:"driver"`
    Connection      string `yaml:"connection"`
    ConnMaxLifeTime int    `yaml:"conn_max_life_time,omitempty"`
    MaxOpenConn     int    `yaml:"max_open_conn,omitempty"`
    MaxIdleConn     int    `yaml:"max_idle_conn,omitempty"`
}
```

三个参数使用 `omitempty` tag，意味着 **值为 0 时不会写入 YAML**。嵌入的默认 [config.yaml](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/configs/config.yaml) 中未设置这三个字段——所以初始配置文件中它们完全不存在。

### 16.3 NewDB 中的条件应用逻辑

在 [data.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/data/data.go#L56-L95) 的 `NewDB()` 中：

```go
func NewDB(debug bool, dataConf *Database) (*xorm.Engine, error) {
    // SQLite 特殊处理：强制 MaxOpenConn = 1
    if dataConf.Driver == "sqlite" {
        // ...
        dataConf.MaxOpenConn = 1   // ← 直接修改传入的 config 结构体
    }

    engine, _ := xorm.NewEngine(dataConf.Driver, dataConf.Connection)
    engine.Ping()

    // 三个参数全部使用 "值 > 0" 作为启用条件
    if dataConf.MaxIdleConn > 0 {
        engine.SetMaxIdleConns(dataConf.MaxIdleConn)
    }
    if dataConf.MaxOpenConn > 0 {
        engine.SetMaxOpenConns(dataConf.MaxOpenConn)
    }
    if dataConf.ConnMaxLifeTime > 0 {
        engine.SetConnMaxLifetime(time.Duration(dataConf.ConnMaxLifeTime) * time.Second)
    }
    // ...
}
```

### 16.4 各场景下的实际默认值对比

| 数据库类型 | 配置文件是否设置 | MaxOpenConn | MaxIdleConn | ConnMaxLifetime |
|-----------|---------------|-------------|-------------|-----------------|
| **SQLite** | 否（默认） | **1**（代码硬编码强制） | Go 默认 2 | 永久（Go 默认 ∞） |
| MySQL | 否（默认） | **无限**（Go 默认 0） | Go 默认 2 | 永久 |
| PostgreSQL | 否（默认） | **无限**（Go 默认 0） | Go 默认 2 | 永久 |
| MySQL | 是（手动设置） | 用户配置值 | 用户配置值 | 用户配置秒数 |

**Go sql.DB 的原生默认值**（当 Answer 不设置时）：
- `MaxOpenConns = 0`（无上限）
- `MaxIdleConns = 2`（标准库默认保留 2 个空闲连接）
- `ConnMaxLifetime = 0`（连接永远不超时回收）

### 16.5 SQLite 强制单连接的原因

SQLite 文件级数据库 **不支持并发写**。如果多个 goroutine 同时持有连接并写入，会触发 `database is locked` 错误。因此代码强制：

```go
if dataConf.Driver == "sqlite" {
    dataConf.MaxOpenConn = 1  // ← 强制串行化所有数据库操作
}
```

但这带来副作用：高并发请求下所有 DB 操作排队等待，吞吐量受限。因此生产环境推荐 PostgreSQL/MySQL。

### 16.6 手动配置三参数示例

在 `data/conf/config.yaml` 中追加：

```yaml
data:
  database:
    driver: mysql
    connection: "user:pass@tcp(db:3306)/answer?charset=utf8mb4&parseTime=True"
    max_open_conn: 100          # 最多同时打开 100 个连接
    max_idle_conn: 10           # 空闲池保留 10 个
    conn_max_life_time: 3600    # 连接 1 小时后强制回收重建（秒）
```

典型配置指导：
- **MaxOpenConn**：建议 MySQL `max_connections` 的 70%~80%，避免耗尽服务端连接
- **MaxIdleConn**：设置为 MaxOpenConn 的 10%~20%，减少频繁握手开销
- **ConnMaxLifetime**：建议 3600s（1 小时），配合云数据库空闲超时和防火墙 TCP keepalive

---

## 十七、管理员角色权限矩阵：role_power_rel 与 user_role_rel 的初始化链路

### 17.1 权限系统的三张核心表

| 表名 | 含义 | 核心字段 |
|------|-----|---------|
| `role` | 角色定义 | ID, Name, Description |
| `power` | 权限定义 | ID, Name, PowerType（权限标识符字符串） |
| `role_power_rel` | 角色-权限关联（N:N） | RoleID, PowerType |
| `user_role_rel` | 用户-角色关联（N:N） | UserID, RoleID |

```
用户 (user)
  │
  ├── user_role_rel ──→ 角色 (role)
  │                         │
  │                         └── role_power_rel ──→ 权限 (power)
  │
  └── Rank（等级/声望值）─→ 等级权限矩阵 (config 表中的 rank.* 配置)
```

### 17.2 初始化链路：Mentor.InitDB 中 4 步的精确顺序

在 [init.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/init.go#L72-L76) 中，权限相关初始化严格依赖顺序：

```
步骤 7 m.do("init role", ...)                → 插入 3 条 role 记录
           ↓ （依赖 role.ID 存在）
步骤 8 m.do("init power", ...)               → 插入 41 条 power 记录
           ↓ （依赖 role + power 两端主键存在）
步骤 9 m.do("init role power rel", ...)      → 插入 83 条 role_power_rel 记录
           ↓ （依赖 user.id=1 + role.id=2 存在）
步骤10 m.do("init admin user role rel", ...) → 插入 1 条 user_role_rel 记录
```

**顺序不可颠倒**的原因：`role_power_rel` 和 `user_role_rel` 的外键（逻辑外键，非数据库 FK）依赖前两张表的数据。同时步骤 6 的 `initAdminUser` 必须在步骤 10 之前完成（创建 ID=1 的用户）。

### 17.3 roles：3 个预定义角色

在 [init_data.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/init_data.go#L84-L88)：

```go
roles = []*entity.Role{
    {ID: 1, Name: "User",      Description: "Default with no special access."},
    {ID: 2, Name: "Admin",     Description: "Have the full power to access the site."},
    {ID: 3, Name: "Moderator", Description: "Has access to all posts except admin settings."},
}
```

注意 **Role ID=1 的 "User" 角色不分配任何权限**——普通用户的权限来自 Rank（声望等级）系统，通过 `config` 表中的 `rank.*` 键值控制（如提问要求声望 ≥1）。

### 17.4 powers：41 个权限定义

[init_data.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/init_data.go#L90-L132) 中定义了 41 个权限，ID 从 1 到 41：

| ID 范围 | 类别 | 典型权限 |
|--------|------|---------|
| 1 | 管理后台 | AdminAccess（进入 /admin 的准入权限） |
| 2-9 | 问题操作 | QuestionAdd/Edit/Delete/Close/Reopen/VoteUp/VoteDown/Pin/Hide/Show |
| 10-16 | 回答操作 | AnswerAdd/Edit/Delete/Accept/VoteUp/VoteDown/Invite |
| 17-21 | 评论操作 | CommentAdd/Edit/Delete/VoteUp/VoteDown |
| 22 | 举报 | ReportAdd |
| 23-28 | 标签操作 | TagAdd/Edit/EditWithoutReview/EditSlugName/Delete/Synonym |
| 29-30 | 其他 | LinkUrlLimit（链接数量限制）/ VoteDetail（查看投票详情） |
| 31-33 | 审核 | AnswerAudit / QuestionAudit / TagAudit |
| 34-41 | 恢复 & 置顶 | Pin/Unpin/Hide/Show（问题）+ Recover（问题/回答/标签） |

### 17.5 rolePowerRels：83 条角色-权限关联

[init_data.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/init_data.go#L134-L219) 定义了完整的权限矩阵：

| 角色 ID | 角色名 | 分配的权限数 | 权限范围 |
|---------|--------|-------------|---------|
| 2 | Admin | **42 条** | ID 1-41 全部 41 个权限 + TagUseReservedTag（共 42，因 1-41 中未单独枚举 ID=168 的 TagUseReservedTag 但在 powers 列表末尾） |
| 3 | Moderator | **41 条** | ID 2-41 除 AdminAccess 外的全部权限 + TagUseReservedTag |
| 1 | User | **0 条** | 不使用 role_power_rel 机制（完全依赖 Rank 声望） |

**关键差异点**：Admin 比 Moderator 多一个 `AdminAccess`（`permission.AdminAccess`）。这是唯一区分管理员和版主权限的开关——**缺少它则无法访问任何 `/answer/admin/api/*` 路由**（由 Admin 中间件在请求入口校验）。

```go
// rolePowerRels 的差异对比（只列两端不同的行）：
{RoleID: 2, PowerType: permission.AdminAccess},   // ← 仅 Admin 有
{RoleID: 2, PowerType: permission.QuestionAdd},    // ← 两者都有
{RoleID: 3, PowerType: permission.QuestionAdd},    // ← 两者都有
// ... 中间 40 个权限两者完全一致 ...
```

### 17.6 adminUserRoleRel：ID=1 用户绑定 Admin 角色

[init_data.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/init_data.go#L221-L224)：

```go
adminUserRoleRel = &entity.UserRoleRel{
    UserID: "1",   // 对应 initAdminUser 创建的管理员
    RoleID: 2,     // 绑定到 Admin 角色（获得全部 42 条权限）
}
```

这行代码保证了安装向导中创建的管理员账户默认拥有系统最高权限。后续在管理后台（`UpdateUserRole` 端点）可以为其他用户授予 Moderator 或 Admin 角色。

### 17.7 权限查询与验证的运行时链路

```
HTTP 请求到达 Admin 路由
        │
        ▼
  AdminAuthMiddleware（中间件）
        │
        ├── 从 JWT 取出 UserID
        │
        ├── user_role_rel 表查询用户所有 RoleID
        │       返回：[2] （用户 ID=1 只有 Admin 角色）
        │
        ├── role_power_rel 表批量查询这些角色的所有 PowerType
        │       返回：[AdminAccess, QuestionAdd, AnswerAdd, ... 共 42 个]
        │
        └── 检查路由所需权限是否在 PowerType 列表中
                │
                ├── 存在 → 放行到 Controller
                └── 不存在 → 返回 403 Forbidden
```

### 17.8 Deprecated 结构体的权限初始化对应关系

权限系统早期版本的 `SiteWriteReq`（写作设置）和 `SiteLegalReq`（法律条款）虽然在 schema 层被标记为 Deprecated，但它们对应的 **Rank 权限配置并未废弃**——而是被拆分到三个独立的 `SiteInfo` 类型中：

| 已废弃结构体 | 被拆分为 | 初始化对应 m.do 步骤 |
|------------|---------|-------------------|
| SiteWriteReq（MinimumContent/MaxImageSize/...） | SiteQuestionsReq + SiteAdvancedReq + SiteTagsReq | L84-L86 的 3 次 `init site info write` 调用 |
| SiteLegalReq（TOS/Privacy/ExternalContent） | SitePoliciesReq + SiteSecurityReq | `initSiteInfoSecurityConfig`（外部内容）+ 数据层面手动写入 policies（通过 admin API 后续配置） |

Rank 声望权限本身通过 `defaultConfigTable`（[init_data.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/init_data.go#L226-L357)）的 `rank.*` 系列 key 在步骤 5 `init config` 时一次性插入，与权限矩阵无关。

---

## 十二、Admin 端 19 个 SiteInfoReq 系列结构体与 22 个 Update 端点的完整映射

### 12.1 18 个 SiteInfo 核心 Update 端点（siteinfo_controller.go）

[siteinfo_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/controller_admin/siteinfo_controller.go) 中定义了 **18 个 PUT 端点** + **1 个 POST 端点**（RequestAIModels），对应 **19 个请求结构体**：

| # | HTTP Method | Router Path | Handler 函数 | 请求结构体 | 所属模块 |
|---|------------|-------------|-------------|-----------|---------|
| 1 | PUT | `/answer/admin/api/siteinfo/seo` | `UpdateSeo` | `SiteSeoReq` | SEO 配置 |
| 2 | PUT | `/answer/admin/api/siteinfo/general` | `UpdateGeneral` | `SiteGeneralReq` | 基本信息 |
| 3 | PUT | `/answer/admin/api/siteinfo/interface` | `UpdateInterface` | `SiteInterfaceReq` | 界面语言 |
| 4 | PUT | `/answer/admin/api/siteinfo/users-settings` | `UpdateUsersSettings` | `SiteUsersSettingsReq` | 用户设置 |
| 5 | PUT | `/answer/admin/api/siteinfo/branding` | `UpdateBranding` | `SiteBrandingReq` | 品牌配置 |
| 6 | PUT | `/answer/admin/api/siteinfo/question` | `UpdateSiteQuestion` | `SiteQuestionsReq` | 问题设置 |
| 7 | PUT | `/answer/admin/api/siteinfo/tag` | `UpdateSiteTag` | `SiteTagsReq` | 标签设置 |
| 8 | PUT | `/answer/admin/api/siteinfo/advanced` | `UpdateSiteAdvanced` | `SiteAdvancedReq` | 高级设置 |
| 9 | PUT | `/answer/admin/api/siteinfo/polices` | `UpdateSitePolices` | `SitePoliciesReq` | 条款政策 |
| 10 | PUT | `/answer/admin/api/siteinfo/security` | `UpdateSiteSecurity` | `SiteSecurityReq` | 安全设置 |
| 11 | PUT | `/answer/admin/api/siteinfo/login` | `UpdateSiteLogin` | `SiteLoginReq` | 登录配置 |
| 12 | PUT | `/answer/admin/api/siteinfo/custom-css-html` | `UpdateSiteCustomCssHTML` | `SiteCustomCssHTMLReq` | 自定义 HTML/CSS |
| 13 | PUT | `/answer/admin/api/siteinfo/theme` | `SaveSiteTheme` | `SiteThemeReq` | 主题设置 |
| 14 | PUT | `/answer/admin/api/siteinfo/users` | `UpdateSiteUsers` | `SiteUsersReq` | 用户可编辑字段 |
| 15 | PUT | `/answer/admin/api/setting/smtp` | `UpdateSMTPConfig` | `UpdateSMTPConfigReq` | SMTP 邮件配置 |
| 16 | PUT | `/answer/admin/api/setting/privileges` | `UpdatePrivilegesConfig` | `UpdatePrivilegesConfigReq` | 权限等级配置 |
| 17 | PUT | `/answer/admin/api/ai-config` | `UpdateAIConfig` | `SiteAIReq` | AI 配置 |
| 18 | PUT | `/answer/admin/api/mcp-config` | `UpdateMCPConfig` | `SiteMCPReq` | MCP 配置 |
| 19 | POST | `/answer/admin/api/ai-models` | `RequestAIModels` | `GetAIModelsReq` | AI 模型查询 |

### 12.2 其他 Admin 模块的 4 个 Update 端点（构成完整的 22 个）

除 siteinfo_controller 外，其他 controller 也提供 Update 端点：

| # | HTTP Method | Router Path | Controller | Handler 函数 | 请求结构体 |
|---|------------|-------------|------------|-------------|-----------|
| 20 | PUT | `/answer/admin/api/user/status` | user_backyard | `UpdateUserStatus` | `UpdateUserStatusReq` |
| 21 | PUT | `/answer/admin/api/user/role` | user_backyard | `UpdateUserRole` | `UpdateUserRoleReq` |
| 22 | PUT | `/answer/admin/api/badge/status` | badge | `UpdateBadgeStatus` | `UpdateBadgeStatusReq` |
| 23 | PUT | `/answer/admin/api/plugin/status` | plugin | `UpdatePluginStatus` | `UpdatePluginStatusReq` |
| 24 | PUT | `/answer/admin/api/plugin/config` | plugin | `UpdatePluginConfig` | `UpdatePluginConfigReq` |
| 25 | PUT | `/answer/admin/api/api-key` | e_api_key | `UpdateAPIKey` | `UpdateAPIKeyReq` |
| 26 | PUT | `/answer/admin/api/user/password` | user_backyard | `UpdateUserPassword` | `UpdateUserPasswordReq` |

**注**：用户提到的 "22 个 Update 端点" 可理解为 siteinfo 核心 18 + 其他模块 4，或根据统计口径不同略有差异。以上为完整的 PUT 端点清单。

### 12.3 所有 Update 方法的统一代码结构

所有 22 个 Update 端点共享相同的代码结构（以 `UpdateGeneral` 为例）：

```go
func (sc *SiteInfoController) UpdateGeneral(ctx *gin.Context) {
    req := schema.SiteGeneralReq{}                    // 1. 声明请求结构体
    if handler.BindAndCheck(ctx, &req) { return }    // 2. 统一校验（ShouldBind + validate tag + Checker）
    err := sc.siteInfoService.SaveSiteGeneral(ctx, req) // 3. 调用 Service 保存
    handler.HandleResponse(ctx, err, nil)              // 4. 统一响应
}
```

**唯一的特殊处理**：
- `UpdateBranding`：保存前调用 `CleanUpRemovedBrandingFiles` 清理已删除的 Logo/Favicon 文件
- `UpdateSiteTag`：从 Context 注入 `UserID` 记录操作人
- `UpdateGeneral`：返回前 `html.UnescapeString(req.Name)` 处理 HTML 转义
- `UpdateSMTPConfig`：支持 `Checker` 接口自定义校验（`from_name` 不能是邮箱格式）

### 12.4 19 个 SiteInfoReq 结构体字段定义一览

| 结构体 | 关键字段（含 validate tag） | 特殊方法 |
|-------|---------------------------|---------|
| `SiteGeneralReq` | `Name`(required,sanitizer,gt1,lte128), `SiteUrl`(required,url), `ContactEmail`(required,email) | `FormatSiteUrl()` |
| `SiteInterfaceReq` | `Language`(required), `TimeZone`(required) | -（部分字段 deprecated） |
| `SiteUsersSettingsReq` | `DefaultAvatar`(oneof=system/gravatar), `GravatarBaseURL` | - |
| `SiteBrandingReq` | `Logo`, `MobileLogo`, `SquareIcon`, `Favicon`（均 omitempty,gt0,lte512） | - |
| `SiteQuestionsReq` | `MinimumTags`(gte0,lte5), `MinimumContent`(gte0,lte65535), `RestrictAnswer` | - |
| `SiteAdvancedReq` | `MaxImageSize`, `MaxAttachmentSize`, `MaxImageMegapixel`（均 omitempty,gt0） | `GetMaxImageSize()` 等单位转换 |
| `SiteTagsReq` | `ReservedTags`(dive), `RecommendTags`(dive), `RequiredTag` | - |
| `SiteWriteTag` | `SlugName`(required), `DisplayName` | -（嵌套结构体） |
| `SitePoliciesReq` | `TermsOfServiceOriginalText`, `PrivacyPolicyOriginalText` 及其 Parsed 版本 | - |
| `SiteSecurityReq` | `LoginRequired`, `ExternalContentDisplay`(oneof=always_display/ask_before_display) | - |
| `GetSiteLegalInfoReq` | `InfoType`(oneof=tos/privacy) | `IsTOS()`/`IsPrivacy()` |
| `SiteUsersReq` | `DefaultAvatar`, `AllowUpdateDisplayName` 等 7 个 bool 字段 | - |
| `SiteLoginReq` | `AllowNewRegistrations`, `AllowEmailDomains` | - |
| `SiteCustomCssHTMLReq` | `CustomHead`, `CustomCss`, `CustomHeader` 等（均 gt0,lte65536） | - |
| `SiteThemeReq` | `Theme`(required), `ThemeConfig`(map), `ColorScheme`, `Layout`(oneof=Full-width/Fixed-width) | - |
| `SiteSeoReq` | `Permalink`(required,lte4,gte0), `Robots`(required) | `IsShortLink()` |
| `SiteAIReq` | `Enabled`, `ChosenProvider`, `SiteAIProviders`(dive), `PromptConfig` | `GetProvider()` |
| `SiteMCPReq` | `Enabled` | - |
| `UpdateSMTPConfigReq` | `FromEmail`, `SMTPHost`, `SMTPPort`(min1,max65535), `Encryption`(oneof=SSL/TLS) | `Check()` 校验 from_name 非邮箱 |
| `UpdatePrivilegesConfigReq` | `Level`(required,min1,max3\|eq99), `CustomPrivileges`(dive) | - |

### 12.5 字段拆分演进：从 SiteWriteReq/SiteLegalReq 到细粒度结构体

代码中存在两处 **Deprecated 结构体**，体现了配置管理的演进：

```go
// Deprecated: use SiteQuestionsReq, SiteAdvancedReq and SiteTagsReq instead
type SiteWriteReq struct {
    MinimumContent int, RestrictAnswer bool, MinimumTags int, ...
    MaxImageSize int, MaxAttachmentSize int, ...
}

// Deprecated: use SitePoliciesReq and SiteSecurityReq instead
type SiteLegalReq struct {
    TermsOfServiceOriginalText string, PrivacyPolicyOriginalText string,
    ExternalContentDisplay string, ...
}
```

**演进原因**：
1. **职责拆分**：早期 `SiteWriteReq` 包含了问题、标签、高级设置等多个维度，拆分后每个结构体对应一个配置页面
2. **API 粒度更细**：拆分后每个 Update 端点只更新一组相关配置，避免大范围字段校验失败
3. **向后兼容**：原结构体保留（标记 Deprecated），避免破坏现有调用方

---

## 十三、ReadConfig 读取失败直接退出与 DefaultConfig 兜底降级路径

### 13.1 ReadConfig 的调用链与错误传播

[conf.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/conf/conf.go#L98-L114) 中 `ReadConfig` 的实现：

```go
func ReadConfig(configFilePath string) (c *AllConfig, err error) {
    if len(configFilePath) == 0 {
        configFilePath = filepath.Join(path.ConfigFileDir, path.DefaultConfigFileName)
    }
    c = &AllConfig{}
    config, err := viper.NewWithPath(configFilePath)  // 失败点 1：文件不存在/权限不足
    if err != nil {
        return nil, err
    }
    if err = config.Parse(&c); err != nil {              // 失败点 2：YAML 格式错误/字段类型不匹配
        return nil, err
    }
    c.SetDefault()                                        // 补全默认值
    c.SetEnvironmentOverrides()                           // 应用环境变量覆盖
    return c, nil
}
```

### 13.2 各命令中 ReadConfig 失败的处理对比

| 命令 | ReadConfig 失败处理 | 代码位置 |
|------|-------------------|---------|
| `answer init` | `fmt.Println(err)` → `return`（优雅退出，不 panic） | [command.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/command.go#L126-L129) |
| `answer run` | `panic(err)`（硬性崩溃） | [main.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/main.go#L76-L78) |
| `answer upgrade` | `fmt.Println(err)` → `return` | [command.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/command.go#L152-L155) |
| `answer dump` | `fmt.Println(err)` → `return` | [command.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/command.go#L172-L175) |
| `answer check` | `fmt.Println(err)` → `return` | [command.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/command.go#L205-L208) |
| `answer config` | `fmt.Println(err)` → `return` | [command.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/command.go#L260-L263) |
| `answer passwd` | 内部通过 `ResetPassword` → 同样会 ReadConfig 失败 | - |

**设计意图**：
- **run 命令** 是唯一 panic 的——因为 run 是业务运行态，没有配置则无法提供任何服务
- **其他命令** 是运维操作态，优雅退出给管理员修复空间

### 13.3 DefaultConfig 的嵌入与兜底

[configs/config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/configs/config.go) 通过 `//go:embed` 指令将默认配置嵌入二进制：

```go
import _ "embed"

//go:embed config.yaml
var Config []byte

//go:embed path_ignore.yaml
var PathIgnore []byte

//go:embed reserved-usernames.json
var ReservedUsernames []byte
```

`configs.Config` 是 **[]byte 类型的嵌入资源**，在两处被用作兜底：

#### 兜底场景一：InitEnvironment 配置文件写入失败

在 [install_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_controller.go#L172-L180)：

```go
if err := cli.InstallConfigFile(confPath); err != nil {
    handler.HandleResponse(ctx, errors.BadRequest(reason.InstallConfigFailed), &InitEnvironmentResp{
        Success:            false,
        CreateConfigFailed: true,
        DefaultConfig:      string(configs.Config),  // ← 嵌入的默认配置作为兜底
        ErrType:            schema.ErrTypeAlert.ErrType,
    })
    return
}
```

前端收到 `DefaultConfig` 后，可以展示给用户手动创建配置文件。

#### 兜底场景二：InstallConfigFile 写入配置文件

在 [install.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/cli/install.go#L60-L62)：

```go
if err := writer.WriteFile(configFilePath, string(configs.Config)); err != nil {
    return fmt.Errorf("write file failed %s", err)
}
```

正常流程下，`configs.Config` 就是写入 `config.yaml` 的**模板内容**。

### 13.4 DefaultConfig 的内容与覆盖关系

嵌入的 [config.yaml](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/configs/config.yaml) 包含：

```yaml
server:
  http:
    addr: 0.0.0.0:80              # 服务监听地址
data:
  database:
    driver: "sqlite3"              # 默认数据库
    connection: "/data/sqlite3/answer.db"
  cache:
    file_path: "/data/cache/cache.db"
i18n:
  bundle_dir: "/data/i18n"
swaggerui:
  show: true
  protocol: http
  host: 127.0.0.1
  address: ':80'
service_config:
  upload_path: "/data/uploads"
  clean_up_uploads: true
ui:
  public_url: '/'
  api_url: '/'
  base_url: ''
  api_base_url: ''
```

**覆盖顺序**（优先级从低到高）：

```
DefaultConfig (嵌入二进制)
        ↓
config.yaml (磁盘文件，由 InitEnvironment 或用户手动写入)
        ↓
SetEnvironmentOverrides() (环境变量，见第十四章)
        ↓
最终生效的 AllConfig
```

### 13.5 降级路径的状态机

```
                     config.yaml 存在？
                          │
                   ┌──────┴──────┐
                   │             │
                   否            是
                   │             │
     answer init 启动安装服务器   ReadConfig 成功？
                   │                │
                安装向导         ┌──┴──┐
                   │             │     │
              InitEnvironment    否    是
                   │             │     │
          InstallConfigFile ──→ 失败   │
               成功?                  │
             ┌──┴──┐                 │
             │     │                 │
             否    是                │
             │     │                 │
   返回 DefaultConfig 写入成功        │
   供手动创建      │                  │
                  └──────────────────┘
                           │
                     CheckDBTableExist
                           │
                     ┌─────┴─────┐
                     │           │
                    存在        不存在
                     │           │
                  "已安装"    启动安装服务器
                  do nothing     │
                            CheckConfigFile
                                  │
                            ┌─────┴──────┐
                            │            │
                           否            是（DB 可连）
                            │            │
                         step=1      初始化数据库
                         开始安装     Mentor.InitDB
```

---

## 十四、配置目录解析、SetEnvironmentOverrides 环境覆盖及多环境策略

### 14.1 配置目录解析：`FormatAllPath` 的单例初始化

路径配置由 [path.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/path/path.go) 统一管理：

```go
var (
    ConfigFileDir     = "/conf/"
    UploadFilePath    = "/uploads/"
    I18nPath          = "/i18n/"
    CacheDir          = "/cache/"
    formatAllPathOnce sync.Once     // ← 单例锁，确保只执行一次
)

func FormatAllPath(dataDirPath string) {
    formatAllPathOnce.Do(func() {
        ConfigFileDir     = filepath.Join(dataDirPath, ConfigFileDir)   // "/data/conf"
        UploadFilePath    = filepath.Join(dataDirPath, UploadFilePath)  // "/data/uploads"
        I18nPath          = filepath.Join(dataDirPath, I18nPath)        // "/data/i18n"
        CacheDir          = filepath.Join(dataDirPath, CacheDir)        // "/data/cache"
    })
}

func GetConfigFilePath() string {
    return filepath.Join(ConfigFileDir, DefaultConfigFileName)  // "/data/conf/config.yaml"
}
```

**关键设计**：
- `sync.Once` 确保无论多少次调用 `FormatAllPath`，路径只会被初始化一次，避免并发问题
- `dataDirPath` 通过命令行 `-C` 参数传入（默认为 `/data/`），这是所有数据的根目录
- 所有模块通过 `path.ConfigFileDir`、`path.UploadFilePath` 等全局变量访问路径，确保一致性

### 14.2 命令行 `-C` 参数传递链

在 [command.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/cmd/command.go#L64-L67)：

```go
rootCmd.PersistentFlags().StringVarP(&dataDirPath, "data-path", "C", "/data/", 
    "data path, eg: -C ./data/")
```

`-C` 是 PersistentFlag，所有子命令继承：

```bash
answer init    -C ./data/       # 初始化到当前目录的 data 子目录
answer run     -C ./data/       # 用同一目录运行
answer upgrade -C ./data/       # 用同一目录升级
```

### 14.3 SetEnvironmentOverrides：三个环境变量覆盖

在 [conf.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/base/conf/conf.go#L49-L96)：

```go
type envConfigOverrides struct {
    SwaggerHost        string
    SwaggerAddressPort string
    SiteAddr           string
}

func loadEnvs() (envOverrides *envConfigOverrides) {
    return &envConfigOverrides{
        SwaggerHost:        os.Getenv("SWAGGER_HOST"),
        SwaggerAddressPort: os.Getenv("SWAGGER_ADDRESS_PORT"),
        SiteAddr:           os.Getenv("SITE_ADDR"),
    }
}

func (c *AllConfig) SetEnvironmentOverrides() {
    envs := loadEnvs()
    if envs.SiteAddr != "" {
        c.Server.HTTP.Addr = envs.SiteAddr                    // 覆盖监听地址
    }
    if envs.SwaggerHost != "" {
        c.Swaggerui.Host = envs.SwaggerHost                    // 覆盖 Swagger 外部访问地址
    }
    if envs.SwaggerAddressPort != "" {
        c.Swaggerui.Address = envs.SwaggerAddressPort          // 覆盖 Swagger 端口
    }
}
```

**三个环境变量的作用**：

| 环境变量 | 覆盖的配置字段 | 用途 |
|---------|---------------|------|
| `SITE_ADDR` | `Server.HTTP.Addr` | 监听地址，Docker 中可改为 `0.0.0.0:8080` |
| `SWAGGER_HOST` | `Swaggerui.Host` | Swagger UI 显示的外部访问 Host |
| `SWAGGER_ADDRESS_PORT` | `Swaggerui.Address` | Swagger UI 显示的外部访问端口 |

**执行时机**：`ReadConfig()` 在成功解析 YAML 后，**最后一步**调用 `SetEnvironmentOverrides()`，确保环境变量优先级最高。

### 14.4 安装向导的自动安装环境变量

除了上述 3 个运行时覆盖变量，安装模式还有 **6 个自动安装环境变量**（定义在 [install_from_env.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_from_env.go#L35-L48)）：

```go
type Env struct {
    ServerAddr        string   `env:"SERVER_ADDR,default=0.0.0.0:80"`
    InstallPort       string   `env:"INSTALL_PORT,default=80"`
    AutoInstall       bool     `env:"AUTO_INSTALL"`        // 核心开关
    DbType            string   `env:"DB_TYPE"`
    DbHost            string   `env:"DB_HOST"`
    DbPort            string   `env:"DB_PORT"`
    DbName            string   `env:"DB_NAME"`
    DbUser            string   `env:"DB_USER"`
    DbPassword        string   `env:"DB_PASSWORD"`
    DbFile            string   `env:"DB_FILE"`
    SiteURL           string   `env:"SITE_URL"`
    ContactEmail      string   `env:"CONTACT_EMAIL"`
    DefaultLanguage   string   `env:"DEFAULT_LANGUAGE,default=en_US"`
    AdminName         string   `env:"ADMIN_NAME"`
    AdminPass         string   `env:"ADMIN_PASS"`
    AdminEmail        string   `env:"ADMIN_EMAIL"`
    // 还有 5 个可选字段
}
```

当 `AUTO_INSTALL=true` 时，`TryToInstallByEnv()` 会跳过 Web 向导，直接调用 `initByEnv()` 完成安装。

### 14.5 Docker 环境：entrypoint.sh 的编排

Docker 镜像通过 [Dockerfile](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/Dockerfile) 构建，[entrypoint.sh](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/script/entrypoint.sh) 作为入口：

```bash
#!/bin/bash
/usr/bin/answer init     # 1. 初始化环境（若已安装则跳过）
/usr/bin/answer upgrade  # 2. 执行数据库版本迁移（若已是最新则跳过）
/usr/bin/answer run -C /data/  # 3. 启动服务
```

**Docker 特性**：
- `VOLUME /data`：所有配置、数据库、上传文件持久化到卷
- `EXPOSE 80`：默认暴露 80 端口，可通过 `SITE_ADDR` 环境变量修改
- `/usr/bin/answer` 从 builder 阶段复制，包含嵌入的 UI、i18n、默认配置

**典型 Docker 用法**：
```bash
# 首次启动（自动走完 init → upgrade → run）
docker run -d -p 9080:80 -v /data/answer:/data apache/answer

# 使用环境变量自动安装（无需 Web 交互）
docker run -d -p 9080:80 -v /data/answer:/data \
  -e AUTO_INSTALL=true \
  -e DB_TYPE=postgres \
  -e DB_HOST=postgres \
  -e DB_PORT=5432 \
  -e DB_NAME=answer \
  -e DB_USER=answer \
  -e DB_PASSWORD=answer \
  -e SITE_URL=http://localhost:9080 \
  -e ADMIN_NAME=admin \
  -e ADMIN_PASS=admin123 \
  -e ADMIN_EMAIL=admin@example.com \
  apache/answer
```

### 14.6 Kubernetes 环境：Helm Chart 部署

项目提供 [charts/](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/charts) 目录的 Helm Chart：

| 文件 | 作用 |
|------|------|
| `values.yaml` | 可配置参数（replicaCount、image、service、ingress、persistence） |
| `templates/pvc.yaml` | PersistentVolumeClaim（对应 `/data` 目录） |
| `templates/ingress.yaml` | Ingress 路由配置 |

**K8s 环境下的关键配置**：
1. **PVC 持久化**：`/data` 目录挂载到 PersistentVolume，保证重启/重建后数据不丢
2. **环境变量注入**：通过 `values.yaml` 的 `env` 字段注入 `SITE_ADDR` 等
3. **就绪探针**：通过 HTTP GET `/` 判断服务就绪
4. **InitContainer 模式**：可在正式容器启动前先跑 `answer init` 和 `answer upgrade`（类似 entrypoint.sh）

### 14.7 开发环境（dev）与生产环境的差异

| 维度 | 开发环境（dev） | 生产环境（docker/k8s） |
|------|---------------|----------------------|
| 数据根目录 | `./data/`（当前目录） | `/data/`（容器内） |
| 数据库 | SQLite（默认） | PostgreSQL / MySQL（外部服务） |
| 监听地址 | `127.0.0.1:80` | `0.0.0.0:80` |
| Debug 模式 | `debug: true` | `debug: false` |
| Swagger UI | 启用（便于 API 调试） | 可通过 `SWAGGER_HOST` 配置外部访问 |
| 安装方式 | 浏览器访问 `/install` | Web 交互 或 `AUTO_INSTALL=true` 自动安装 |
| 升级方式 | `make upgrade` 或 `answer upgrade -C ./data` | 容器重启时自动执行 `answer upgrade` |
| 配置文件 | 开发时可手动编辑 `./data/conf/config.yaml` | 通过环境变量或 ConfigMap 覆盖 |

### 14.8 环境变量与配置文件的完整优先级

```
最低优先级 → 最高优先级

  DefaultConfig (//go:embed config.yaml)
        ↑
  config.yaml (磁盘文件)
        ↑
  SetEnvironmentOverrides()
    ├── SITE_ADDR → Server.HTTP.Addr
    ├── SWAGGER_HOST → Swaggerui.Host
    └── SWAGGER_ADDRESS_PORT → Swaggerui.Address
        ↑
  运行时 API 调用（Admin Update* 接口）
    ├── 站点配置保存到 site_info 表
    └── 插件配置保存到 config 表
        ↑
  当前生效配置（内存中的 AllConfig + site_info 表内容）
```

**注意**：站点运行时配置（SiteName、SMTP、主题等）**不存储在 config.yaml**，而是存储在数据库 `site_info` 表中。`config.yaml` 仅存储系统启动必需的基础配置（DB 连接、监听端口、缓存路径等）。

这种分层设计的好处是：
- `config.yaml` 可以被版本控制，环境间复制
- 运行时配置通过管理后台修改，无需重启服务
- 敏感信息（DB 密码）可以通过环境变量注入，不落盘

---

## 十八、Migration 单步幂等性：InitDB 26 次 m.do 与 Upgrade 增量循环的去重机制

### 18.1 幂等性的三层设计

整个 Answer 系统在迁移/安装层面实现了三级幂等防护，确保**任何一步重复执行都不会产生脏数据**：

| 层级 | 机制 | 覆盖范围 | 代码位置 |
|------|------|---------|---------|
| L1 全局级 | `checkTableExist` → `Done` 标志 | 阻断整次 InitDB 全部 26 步 | [init.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/init.go#L95-L109) |
| L2 循环级 | `version_number` 已递增 → 跳过已执行迁移 | 阻断 upgrade 单步迁移的重入 | [migrations.go](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/migrations.go#L144-L198) |
| L3 行级 | `SELECT ... Get(condition)` → `exist? Update/skip : Insert` | 每条 roles/powers/role_power_rel 行的幂等 | 所有 v4/v8/v13/v17 迁移脚本 |

### 18.2 InitDB 中 26 次 m.do 的四种幂等策略

[Mentor.do()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/init.go#L95-L103) 先检查 L1 的 `m.Done` 和 `m.err`，然后才执行函数。26 次调用内部的实际幂等策略分为四类：

| 幂等策略 | 适用步骤（#） | 实际机制 | 重复执行结果 |
|---------|--------------|---------|------------|
| **L1 Done 标志** | 所有 26 步 | `checkTableExist` → 发现 `version` 表存在 → `Done=true` → `m.do()` 直接 return | 全部跳过，零副作用 |
| **批量 Insert（无检查）** | #2 syncTable, #4 initAdminUser, #5 initConfig, #6 initDefaultRankPrivileges, #7 initRole, #8 initPower, #9 initRolePowerRel, #10 initAdminUserRoleRel, #11-#22 所有 SiteInfo Insert, #23-#26 初始化内容 | 直接 `engine.Insert()` | **⚠️ 首次执行才安全**——若 L1 Done 失效则 Insert 重复主键会报错（但 L1 确保不会发生） |
| **Update 条件匹配** | #6 initDefaultRankPrivileges | `Update(..., {Key: privilege.Key})` WHERE key=xxx | 幂等：重复 Update 为同值 |
| **Sync 表结构** | #2 syncTable | xorm `Sync()` 内部检查列是否存在 | 幂等：已存在的字段/索引跳过 |

**关键洞察**：InitDB 的大部分步骤（角色/权限/SiteInfo/内容）本身 **并没有行级去重**——它们依赖外层 L1 的 Done 标志作为唯一的幂等保证。这种设计选择的原因是：
1. 首次安装场景下 Done=false → 所有表都是空的，Insert 不会冲突
2. Done=true（由 checkTableExist 检测 version 表）→ 所有 26 步一次性跳过，无需每行检查
3. 相比每行都做 SELECT+INSERT，**单次表存在性检测** 在首次安装场景下性能最优（少了 3+41+83+1+N 次额外 SELECT）

### 18.3 Upgrade 迁移脚本的行级去重模式（以 v4/v8/v13/v17 为例）

与 InitDB 不同，Upgrade 的每个迁移脚本必须**自带行级幂等**——因为 `-f v1.1.0` 参数可能从历史版本重复执行，或迁移失败回滚后重跑。

#### v4.go 新增角色权限功能：三表全量 Upsert

[v4.go `addRoleFeatures()`](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/v4.go#L32-L234) 是最完整的幂等样板：

```go
// ========== role 表：INSERT前先 GET ==========
for _, role := range roles {
    exist, _ := x.Get(&entity.Role{ID: role.ID, Name: role.Name})  // 条件：ID+Name 双匹配
    if exist { continue }                                           // 已存在→跳过
    x.Insert(role)                                                  // 不存在→插入
}

// ========== power 表：Upsert 模式（存在→Update，不存在→Insert） ==========
for _, power := range powers {
    exist, _ := x.Get(&entity.Power{ID: power.ID})                // 条件：ID 匹配
    if exist {
        x.ID(power.ID).Update(power)                              // 已存在→覆盖更新（允许描述变更）
    } else {
        x.Insert(power)                                            // 不存在→插入
    }
}

// ========== role_power_rel 表：INSERT前先 GET ==========
for _, rel := range rolePowerRels {
    exist, _ := x.Get(&entity.RolePowerRel{RoleID: rel.RoleID, PowerType: rel.PowerType})  // 联合主键
    if exist { continue }
    x.Insert(rel)
}

// ========== user_role_rel 表：INSERT前先 GET ==========
exist, _ := x.Get(adminUserRoleRel)
if !exist { x.Insert(adminUserRoleRel) }

// ========== config 表：Upsert 模式 ==========
for _, c := range defaultConfigTable {
    exist, _ := x.Get(&entity.Config{ID: c.ID, Key: c.Key})
    if exist { x.Update(c, &entity.Config{ID: c.ID, Key: c.Key}); continue }
    x.Insert(c)
}
```

**三种去重模式的设计意图**：

| 模式 | 适用表 | 理由 |
|------|-------|------|
| **Get → Skip/Insert** | role, role_power_rel, user_role_rel | 业务主键稳定且不会被用户修改 → 存在即正确 |
| **Get → Update/Insert（Upsert）** | power, config | 版本升级时可能需要更新描述/默认值 → 存在则覆盖 |
| **仅 Upsert（无显式 Skip）** | power 描述变更 | 支持后续版本修正权限的 Description 文本 |

#### v8.go 新增 Pin/Hide 功能：增量添加 4 个权限 + 8 条关联

[v8.go `addRolePinAndHideFeatures()`](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/v8.go#L33-L82) 的幂等模式完全镜像 v4：
```
powers (ID 34-37):   Get{ID} → exist? Update : Insert
rolePowerRels (4×2): Get{RoleID, PowerType} → exist? skip : Insert
config (ID 119-126): Get{ID} → exist? Update : Insert
```

**注意：只处理本版本增量添加的 ID 范围**（34-37 而非 1-41），避免覆盖早期版本已存在的其他权限。

#### v13.go 新增 Invite/Gravatar 功能：部分 Upsert + 计数重算

[v13.go `addPrivilegeForInviteSomeoneToAnswer()`](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/v13.go#L80-L136) 的幂等模式：
```
powers (ID 38):        Get{PowerType} → Upsert（按 PowerType 查而非 ID，兼容早期数据可能缺 ID=38）
rolePowerRels (2):     Get{RoleID, PowerType} → Skip/Insert
config (ID 127):       Get{ID} → Upsert
```

**特殊点**：power 用 `Get(&entity.Power{PowerType: ...})` 而非按 ID 查询。这是容错设计——如果某用户的 power 表中 ID=38 被误删但 PowerType 仍存在于其他 ID，仍然能正确匹配到已有行。

v13.go 中另一类操作是 `updateQuestionCount/updateTagCount/...`：它们全部使用 Find → 遍历 → `Update(..., WHERE ID=x)` 的模式，天然幂等（重算结果相同）。

#### v17.go 新增 Recover 功能：与 v8 完全同构

[v17.go `addRecoverPermission()`](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/migrations/v17.go#L32-L98) 与 v8 模式一致：
```
powers (ID 39-41):  Get{ID} → Upsert
rolePowerRels (6):  Get{RoleID, PowerType} → Skip/Insert
config (ID 128-130): Get{ID} → Upsert
```

### 18.4 不重写 3 roles + 41 powers + 83 role_power_rel 的完整保证矩阵

| 机制 | 防止 roles 重复 | 防止 powers 重复 | 防止 role_power_rel 重复 |
|------|---------------|----------------|------------------------|
| **InitDB：checkTableExist（L1）** | ✅ version 表存在则全部跳过 | ✅ 同上 | ✅ 同上 |
| **InitDB：Insert 批量（无检查）** | ❌ 单独重复会主键冲突（但 L1 阻止） | ❌ 同上 | ❌ 同上 |
| **Upgrade v4：Get+Skip/Insert** | ✅ 按 {ID,Name} 存在性跳过 | ✅ 按 ID Upsert | ✅ 按 {RoleID,PowerType} 跳过 |
| **Upgrade v8/v17：增量 Upsert** | 无涉及 | ✅ 只处理增量 ID | ✅ 只处理增量关联 |
| **Upgrade v13：按 PowerType 查** | 无涉及 | ✅ 按业务键匹配（更健壮） | ✅ 按联合键跳过 |
| **Upgrade Migrate() 版本号** | ✅ N→32 只跑一次增量循环 | ✅ 同上 | ✅ 同上 |

**设计权衡总结**：
- **InitDB 走 "批量 Insert + L1 Done 守护"**：追求首次安装性能（最少的 SELECT）
- **Upgrade 走 "每行 Get+Insert/Upsert + L2 版本号守护"**：追求重复执行的安全性
- 两者在 `v4.go addRoleFeatures` 交汇——v4 既是升级时的增量迁移，也是 InitDB 中 41 权限设计的来源

---

## 十九、安装中途断电后五道防线接住 SiteInfo 默认值与角色权限矩阵

### 19.1 安装流程的 5 个易断电时间点

[InitBaseInfo()](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/install/install_controller.go#L211-L249) → `Mentor.InitDB()` 的 26 步执行期间，任何时刻断电都可能导致部分写入。按时间顺序分为 5 个关键易断点：

```
POST /installation/base-info 到达
       │
       ├─ T0: 配置文件已创建（#13.3 InitEnvironment 完成）
       │      但 DB 连接尚未初始化
       │      └── 断点 A
       │
       ├─ NewDB() + NewMentor() 完成
       │
       ├─ #1 checkTableExist → Done=false（表不存在）
       ├─ #2 syncTable 创建所有表结构
       │      └── 断点 B：表存在但完全没有数据
       │
       ├─ #3 initVersionTable（写入 version_number=32） ← L1 Done 标志触发点
       │      └── 断点 C：version 表已写入，其他表为空
       │
       ├─ #4 initAdminUser
       ├─ #5 initConfig（100+ 条 rank.* 配置）
       ├─ #6 initDefaultRankPrivileges
       ├─ #7 initRole（3 条角色）
       ├─ #8 initPower（41 条权限）
       ├─ #9 initRolePowerRel（83 条关联）
       │      └── 断点 D：部分角色/权限矩阵写入，version 已完成
       │
       ├─ #10 initAdminUserRoleRel（用户1绑定Admin）
       ├─ #11-#22 12 条 SiteInfo 行插入（interface/general/login/theme/seo/...）
       ├─ #23 initDefaultContent（2问题+2回答+4标签）
       │      └── 断点 E：示例内容不完整，但核心配置已就绪
       │
       └─ #24-#26 Badges/AI/MCP 完成 → InitDB 返回
              time.Sleep(1s) → os.Exit(0)
```

### 19.2 五道防线的完整覆盖

针对上述 5 类断点，系统构筑了 **五道递进式防线**，每一道覆盖上一道的漏检场景：

```
┌──────────────────────────────────────────────────────────────────┐
│  防线 1：InitBaseInfo 入口 CheckDBTableExist 检查                  │
│  代码：install_controller.go L225-L229                            │
│  触发：用户再次打开浏览器 → 重新提交 base-info POST                │
│  判断：version 表是否存在？                                        │
│  是 → "database is already initialized" → 静默成功返回            │
│  否 → 继续执行 Mentor.InitDB()                                    │
└───────────────────────────┬──────────────────────────────────────┘
                            │ 覆盖断点 A（表完全不存在）
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  防线 2：Mentor.InitDB L1 Done 标志（checkTableExist）             │
│  代码：init.go L105-L109                                          │
│  触发：防线 1 放行了 InitDB 调用                                   │
│  判断：IsTableExist(&entity.Version{}) ?                          │
│  Done=true → 全部 26 个 m.do 一次性跳过（m.err/m.Done 检查）      │
│  覆盖：断点 B、C（version 表已存在，其他表空或部分有数据）         │
└───────────────────────────┬──────────────────────────────────────┘
                            │ 覆盖断点 B/C/D/E
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  防线 3：answer run 启动时的 answer upgrade 自动执行              │
│  代码：Docker entrypoint.sh L2 → command.go L146-170              │
│  触发：用户重启容器/手动执行 answer upgrade                       │
│  判断：GetCurrentDBVersion() vs ExpectedVersion()                 │
│  当前=32，期望=32 → 循环跳过，0 次迁移                            │
│  当前=N<32 → 从 N 开始逐版本执行迁移脚本                          │
└───────────────────────────┬──────────────────────────────────────┘
                            │ 升级场景覆盖
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  防线 4：v4.go addRoleFeatures 的行级 Upsert 补全                 │
│  代码：v4.go L32-L234                                             │
│  触发：防线 3 进入了某迁移版本的 Migrate()                         │
│  判断：Get{ID,Name} / Get{ID} / Get{RoleID,PowerType}             │
│  存在 → Skip 或 Update                                            │
│  不存在 → Insert                                                  │
└───────────────────────────┬──────────────────────────────────────┘
                            │ 增量迁移的行级补全
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  防线 5：运行时 SiteInfo Get 的空值兜底 + Admin Update 接口可写    │
│  代码：siteinfo_service.go 各 Get 方法 + Update* 端点             │
│  触发：服务正常启动，用户访问前台或管理后台                         │
│  判断：GetByType(type) 有数据吗？                                  │
│  有 → 返回存储值                                                  │
│  无 → 返回默认值（GetSiteInfo 中通过 switch/case 构造空结构）      │
│  管理员可进入 /admin 手动保存任意缺失配置                          │
└──────────────────────────────────────────────────────────────────┘
```

### 19.3 五道防线逐个断点的接住效果

| 断点 | 发生时机 | 数据状态 | 第1道（CheckDBTableExist） | 第2道（Done） | 第3道（upgrade） | 第4道（v4 Upsert） | 第5道（运行时兜底） | 最终结果 |
|------|---------|---------|--------------------------|-------------|----------------|------------------|-------------------|---------|
| **A** | InitEnvironment 后，DB 初始化前 | config.yaml 存在，DB 空/表不存在 | DBTableExist=false → 放行 InitDB | 不触发（无 version 表） | 0 次迁移 | 不触发 | N/A | **InitDB 从头执行，完全恢复** ✅ |
| **B** | syncTable 后，version 表创建前 | 所有表存在但全空 | DBTableExist=false → 放行 | checkTableExist=false → 26 步全走 | 0 次迁移 | 不触发 | N/A | **InitDB 从头 Insert 无冲突** ✅ |
| **C** | initVersionTable 刚完成 | version=32，其他表空 | DBTableExist=true → 静默返回 | Done=true → 26 步全跳过 | 当前=32→0 迁移 | 不触发 | SiteInfo 为空→返回默认值；**但 Role/Power 全空导致管理员登录后无权限** ❌ → 需要手动恢复 | **有瑕疵！Role/Power 缺失需用 upgrade -f 触发 v4 修复** ⚠️ |
| **D** | role_power_rel 写入中途 | version=32，role(3条)/power(41条)/部分 rel 存在 | DBTableExist=true → 静默返回 | Done=true → 跳过 | 0 次迁移 | 不触发 | 已存在的权限正常；**缺失的关联项导致管理员无该权限** ❌ | **有瑕疵！role_power_rel 部分缺失用 upgrade -f v1.1.0（v4）触发补全** ⚠️ |
| **E** | 23 条默认内容写入中途 | version=32，核心配置全，仅 badges/AI/MCP/内容不全 | DBTableExist=true → 静默返回 | Done=true → 跳过 | 0 次迁移 | 不触发 | AI 配置为空→默认 disabled；Badges 为空→不展示；示例内容缺一部分 | **功能不受影响，可在后台补全** ✅ |

### 19.4 断点 C 和 D 的手动恢复路径

对于 **断点 C（version 表已写入但 role/power 完全空）** 和 **断点 D（role_power_rel 部分缺失）**，五道防线的第 2 道（Done=true）反而阻碍了自动恢复。需要使用 `-f` 参数降级版本号，强制触发 v4.go 的 Upsert：

```bash
# 断点 C 恢复：强制从 v4（也就是角色权限功能引入版本）开始重跑
answer upgrade -C /data/ -f v1.1.0

# 断点 D 恢复：同样强制 v4 重新执行，其 Get+Upsert 会补全缺失的 role_power_rel
answer upgrade -C /data/ -f v1.1.0
```

**v4.go 补全断点 C 的实际执行流**：

```
GetCurrentDBVersion() 返回 32
    ↓
-f v1.1.0 → currentDBVersion = 3（假设 v4 对应 migrations[3]）
    ↓
for N=3; N<32; N++:
    N=3: v4.addRoleFeatures()
         ├── role(3) → Get{ID,Name} 不存在 → Insert 3 条 ✅
         ├── power(41) → Get{ID} 不存在 → Insert 41 条 ✅
         ├── role_power_rel(83) → Get{RoleID,PT} 不存在 → Insert 83 条 ✅
         ├── user_role_rel(1) → Get 不存在 → Insert ✅
         └── config(ID 115-117) → Upsert（已存在则不影响其他 config）
    N=4..31: 依次执行 v5 到最新版（它们也有行级 Upsert）
         └── 幂等性保证不会产生脏数据
```

### 19.5 防线 5：运行时 SiteInfo 空值兜底的实现细节

当 SiteInfo 某一行缺失（断电在 #11-#22 之间），Service 层通过 "GetByType → 无则返回空结构体 + 默认值" 模式兜底，用户不会遇到 500 错误。管理员登录后台后重新保存对应页面即可把缺失的行补回数据库。

例如 [GetSiteInterfaceSetting](file:///d:/fz/0601-1/solo-dogfeeding/code/70-answer/internal/service/siteinfo_service.go) 的逻辑：

```go
func (s *SiteInfoService) GetSiteInterfaceSetting(ctx context.Context) (*schema.SiteInterfaceResp, error) {
    siteInfo, err := s.SiteInfoRepo.GetByType(ctx, constant.SiteTypeInterface)
    if err != nil {
        return nil, err
    }
    // 数据库中不存在这一行？返回空结构但不报错
    if siteInfo == nil {
        return &schema.SiteInterfaceResp{}, nil
    }
    resp := &schema.SiteInterfaceResp{}
    // unmarshal Content...
    return resp, nil
}
```

前端 SchemaForm 渲染时，若拿到空响应则使用 `initFormData` 生成 JSONSchema 默认值（`language=en_US` 等）。管理员点击保存时，`UpdateInterface` → 最终 `SaveSiteInterface` 会执行 **Insert（第一次写入）或 Update（后续修改）** 的二选一逻辑（service 层内部判断是否已有对应 type 行）。

---

## 二十、InitDB 与 Upgrade 的交叉场景：从"已部分安装"状态的完整恢复

### 20.1 三种典型交叉场景

| 场景 | 触发条件 | 推荐恢复命令 | 预期结果 |
|------|---------|------------|---------|
| 场景 1：新安装完全成功 | 正常走完 26 步 | 无需操作 | version=32，数据完整 ✅ |
| 场景 2：断点 B（表存在无数据） | syncTable 后断电 | `answer init -C ./data` | CheckDBTableExist=false → 重走 InitDB → 成功 ✅ |
| 场景 3：断点 C（version 存在其他空） | initVersionTable 后断电 | `answer upgrade -C ./data -f v1.1.0` | 降级后 v4 补全 roles/powers ✅ |
| 场景 4：断点 D（权限矩阵不全） | role_power_rel 写入中途断电 | `answer upgrade -C ./data -f v1.1.0` | v4 的 Get+Skip/Insert 补全缺失行 ✅ |
| 场景 5：正常升级 32→33 | 新版 Answer ExpectedVersion=33 | `answer upgrade -C ./data` | 从 32 开始执行 migrations[32] ✅ |
| 场景 6：升级 32→33 失败回滚 | migrations[32] 中途失败后修复 | `answer upgrade -C ./data` | version 仍=32 → 从 migrations[32] 重新开始 ✅ |

### 20.2 InitDB 与 Upgrade 幂等设计的对照总结

| 维度 | Mentor.InitDB（首次安装） | migrations.Migrate()（版本升级） |
|------|--------------------------|-------------------------------|
| **全局幂等守护** | `checkTableExist` → Done 标志（基于 version 表存在性） | `version_number` 逐次递增（基于 version.id=1 的数值） |
| **行级幂等** | 无（依赖 Done 标志 + 空表） | 每行 Get→Skip/Upsert |
| **roles/powers 去重** | Done=true 时全部跳过 | v4.go 的 Get+Skip/Insert 模式 |
| **SiteInfo 去重** | Done=true 时全部跳过 | v13 等版本的 UPDATE（重算时天然幂等） |
| **重复执行的最坏开销** | 1 次 IsTableExist 查询 | N 次 Get（N=当前版本已有的迁移脚本数） |
| **从"部分成功"恢复** | Done 标志阻断 → 需用 `-f` 强制触发 Upgrade | 版本号未递增 → 下次自动重跑 |
| **写入冲突处理** | L1 Done=全部跳过 → 无冲突 | Get→存在则 Skip/Update → 无冲突 |

**核心设计原则**：
- **InitDB = 粗粒度一次性操作**，依赖"version 表是否存在"这个强信号来判断是否已安装
- **Upgrade = 细粒度增量操作**，每个迁移脚本自行保证行级幂等，支持断点续跑
- 两者通过 `-f` 参数桥接——当 InitDB 的"全有或全无"模式在部分写入后失效时，切换到 Upgrade 模式利用每行 Upsert 做补全

**注意**：站点运行时配置（SiteName、SMTP、主题等）**不存储在 config.yaml**，而是存储在数据库 `site_info` 表中。`config.yaml` 仅存储系统启动必需的基础配置（DB 连接、监听端口、缓存路径等）。

这种分层设计的好处是：
- `config.yaml` 可以被版本控制，环境间复制
- 运行时配置通过管理后台修改，无需重启服务
- 敏感信息（DB 密码）可以通过环境变量注入，不落盘
