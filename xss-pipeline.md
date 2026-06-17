# 问答正文 XSS 防护管道分析

本文档梳理问答正文从作者输入到 Markdown 渲染再到前端展示的完整数据流，重点分析白名单收紧、属性清理与扩展点机制如何逐层降险。

---

## 一、整体数据流概览

```
用户输入(Markdown原文)
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│  前端编辑器（CodeMirror + 工具栏）                         │
│  - 用户输入 Markdown                                       │
│  - 预览时调用 POST /answer/api/v1/post/render              │
└──────────────────────────────────────────────────────────┘
    │
    ▼ 提交时 Content 字段
┌──────────────────────────────────────────────────────────┐
│  后端 Schema 层（参数校验 Check 方法）                      │
│  - AnswerAddReq.Check()   [answer_schema.go:63-72]         │
│  - QuestionAdd.Check()    [question_schema.go:96-104]      │
│  - AddCommentReq.Check()  [comment_schema.go:59-68]        │
│  调用 converter.Markdown2HTML(Content) → req.HTML          │
└──────────────────────────────────────────────────────────┘
    │
    ▼ req.Content(原文) + req.HTML(转后HTML)
┌──────────────────────────────────────────────────────────┐
│  Markdown2HTML 核心管道   [pkg/converter/markdown.go]     │
│  ┌─ 第1层: Goldmark 解析 + DangerousHTMLRenderer          │
│  │    内含 1a: RawHTML → bluemonday 严格 UGCPolicy 逐段过滤│
│  │    内含 1b: HTMLBlock → SecureWrite HTML实体转义        │
│  │    内含 1c: Link/AutoLink → URL合法性 + 危险协议拦截     │
│  └─ 第2层: Bluemonday 放宽版 UGCPolicy 全局二次过滤         │
└──────────────────────────────────────────────────────────┘
    │
    ▼ entity.OriginalText + entity.ParsedText
┌──────────────────────────────────────────────────────────┐
│  Service 层                                                │
│  - Insert/AddQuestion/AddComment 写入 DB                  │
│  - ReviewService 调用审核插件（业务安全）                   │
│  - UpdateQuestionLink 做链接替换（二次净化）                │
│    [question_common/question.go:703-828]                   │
└──────────────────────────────────────────────────────────┘
    │
    ▼ 存储到 DB
┌──────────────────────────────────────────────────────────┐
│  前端展示层                                                │
│  - dangerouslySetInnerHTML={{ __html: data.html }}        │
│  - htmlRender() UX 增强   [Editor/utils/index.ts:40-124]  │
│  - useRenderHtmlPlugin 插件渲染钩子                        │
└──────────────────────────────────────────────────────────┘
```

---

## 二、第 1 层：Goldmark 解析阶段 —— 节点级过滤

**代码位置**：[markdown.go:79-189](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L79-L189)

通过自定义扩展 `DangerousHTMLFilterExtension` 将 `DangerousHTMLRenderer` 注入 goldmark 渲染管线，优先级为 1（最高）。该渲染器针对 4 种 AST 节点做了特殊处理。

注意：`DangerousHTMLRenderer` 内部持有一个 `Filter` 字段，它是 **默认 UGCPolicy**（第 86 行 `bluemonday.UGCPolicy()`），没有经过后续的放宽调整。因此第 1 层内部存在**两道不同严格度的 bluemonday 策略**：

| 阶段 | 策略 | 严格度 |
|------|------|--------|
| 1a RawHTML 逐段过滤 | `DangerousHTMLRenderer.Filter`（默认 UGCPolicy） | 更严格：nofollow=true，RequireParseableURLs=true |
| 2 全局二次过滤 | `Markdown2HTML` 局部 filter（放宽版 UGCPolicy） | 较宽松：nofollow=false，RequireParseableURLs=false |

### 2.1 RawHTML（内联原始 HTML）
函数：[renderRawHTML](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L104-L119)

```go
if string(segment) == "<kbd>" || string(segment) == "</kbd>" {
    w.Write(segment.Value(source))
} else {
    w.Write(r.Filter.SanitizeBytes(segment.Value(source)))
}
```

- `<kbd>` / `</kbd>` 原样放行（键盘按键样式）
- 其余所有内联 HTML 片段经 `r.Filter.SanitizeBytes()` 过滤——此 `r.Filter` 是**默认 UGCPolicy**，会自动为 `<a>` 加 `rel="nofollow"`、校验 URL 可解析性
- 放行的 `<kbd>` 在第 2 层还会被 `AllowElements("kbd")` 保留

### 2.2 HTMLBlock（HTML 块级内容）
函数：[renderHTMLBlock](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L121-L134)

```go
r.Writer.SecureWrite(w, line.Value(source))
```

`SecureWrite` 是 goldmark `Writer` 接口的方法，官方定义为 **"writes the given source to writer with replacing insecure characters"**（见 goldmark v1.7.4 `renderer/html` 包）。它将 `<`、`>`、`&`、`"` 等字符替换为 HTML 实体（`&lt;`、`&gt;`、`&amp;`、`&quot;`）。

**效果**：所有块级原始 HTML 被彻底**文本化**。例如用户写 `<div class="xss">` 会输出 `&lt;div class=&quot;xss&quot;&gt;`，在浏览器中显示为可见文本而非 DOM 元素。此内容进入第 2 层 bluemonday 时已不含任何真实 HTML 标签，不会被解析执行。

**与 goldmark 默认行为的对比**：
- goldmark 默认 `renderHTMLBlock`：`Unsafe=false` 时输出 `<!-- raw HTML omitted -->` 注释（完全移除）
- 项目自定义 `renderHTMLBlock`：始终 `SecureWrite`（转义为可见文本）

项目选择"转义为可见文本"而非"完全移除"，用户能看到自己写了什么，同时保证安全。

### 2.3 Link（Markdown 链接）
函数：[renderLink](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L136-L158)

```go
if entering && r.renderLinkIsUrl(string(n.Destination)) {
    _, _ = w.WriteString("<a href=\"")
    if r.Unsafe || !goldmarkHTML.IsDangerousURL(n.Destination) {
        _, _ = w.Write(util.EscapeHTML(util.URLEscape(n.Destination, true)))
    }
    _ = w.WriteByte('"')
    // ... title、attributes
}
```

三道防线：
1. **`renderLinkIsUrl`**：URL 必须通过 `govalidator.IsURL` 或以 `/` 开头
2. **`IsDangerousURL`**：拦截 `javascript:`、`vbscript:`、`data:` 等危险协议（`r.Unsafe` 默认 false）
3. **`util.EscapeHTML(util.URLEscape(...))`**：对 `href` 值做 HTML 实体转义 + URL 编码

注意源码中被注释掉的一行 `// _, _ = w.WriteString("<a test=\"1\" rel=\"nofollow\" href=\"")`——项目**故意不在 goldmark 层添加 nofollow**，将 nofollow 逻辑交给前端。

### 2.4 AutoLink（自动链接）
函数：[renderAutoLink](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L160-L183)

```go
if !entering || !r.renderLinkIsUrl(string(n.URL(source))) {
    return ast.WalkContinue, nil
}
_, _ = w.WriteString(`<a href="`)
_, _ = w.Write(util.EscapeHTML(util.URLEscape(url, false)))
```

与 `renderLink` 的区别：**没有 `IsDangerousURL` 检查**。但 `renderLinkIsUrl` 要求 `govalidator.IsURL` 返回 true，该函数对 URL 格式有较严格校验（需要合法 host），`javascript:alert(1)` 等无 host 的危险协议无法通过。此外 `util.EscapeHTML` 会转义引号和尖括号，阻止属性注入。

---

## 三、第 2 层：Bluemonday 白名单 —— 全局 HTML 净化

**代码位置**：[markdown.go:39-66](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L39-L66)

Goldmark 输出完整 HTML 后，使用 `bluemonday.UGCPolicy()` 作为基线，在此基础上做**放宽**与**定向扩展**：

### 3.1 策略基线（UGCPolicy 默认值）
bluemonday 内置的 UGC 策略默认配置：
- 允许标签：`a, b, blockquote, br, caption, cite, code, col, colgroup, dd, del, dfn, dl, dt, em, figcaption, figure, h1-h6, hr, i, img, ins, kbd, li, mark, ol, p, pre, q, rp, rt, ruby, s, samp, small, strike, strong, sub, sup, table, tbody, td, tfoot, th, thead, tr, u, ul`
- 移除标签：`script, iframe, form, object, embed, style` 等
- 移除所有 `on*` 事件处理器
- **默认 `RequireNoFollowOnLinks(true)`**：所有 `<a>` 自动加 `rel="nofollow"`
- **默认 `RequireParseableURLs(true)`**：URL 属性值必须可被 `net/url.Parse` 解析
- **默认不允许 `style` 属性**

### 3.2 放宽配置（在 UGCPolicy 基线上退让）

| 配置 | UGCPolicy 默认值 | 项目设置 | 效果 |
|------|------------------|---------|------|
| `RequireNoFollowOnLinks(false)` | true（所有链接加 nofollow） | false | **取消**对所有链接强制加 nofollow。后端输出的 `<a>` 不带 nofollow，外链 nofollow 由前端 `htmlRender` 补上 |
| `RequireNoFollowOnFullyQualifiedLinks(false)` | true（绝对链接加 nofollow） | false | **取消**对绝对链接强制加 nofollow |
| `RequireParseableURLs(false)` | true（URL 必须可解析） | false | **取消** URL 可解析性校验。不仅允许相对路径，也允许任何无法被 `net/url.Parse` 解析的 `href` 值通过。风险在于 bluemonday 不再拦截格式异常的 URL，但 goldmark 层已对链接做了 `govalidator.IsURL` + `EscapeHTML` 双重防护，所以对 Markdown 链接无实际影响；对 RawHTML 中的链接，第 1 层的默认 UGCPolicy 仍然校验 URL |

### 3.3 扩展配置（在 UGCPolicy 基线上新增能力）

| 配置 | 效果 | 风险评估 |
|------|------|---------|
| `AllowStyling()` | **仅放行 `class` 属性**（全局，值须匹配 `SpaceSeparatedTokens`）。**不放行 `style`** | 常见误读：`AllowStyling()` 不等于允许 `style` 属性。bluemonday v1.0.27 的 `AllowStyling()` 实现只调用 `AllowAttrs("class").Matching(SpaceSeparatedTokens).Globally()`，源码注释明确说"当 bluemonday 内置 CSS 解析器后才会允许受控的 style 属性"。因此 `style` 属性在 UGCPolicy 与本项目策略下**均被移除**，用户无法通过 `style` 注入 CSS |
| `AllowElements("kbd")` | 额外放行 `<kbd>` | 低风险。kbd 标签无安全隐患 |
| `AllowAttrs("title").Matching(regex).Globally()` | 全局允许 `title` 属性，值必须匹配 `^[\p{L}\p{N}\s\-_',\[\]!\./\\\(\)]*$` 或 `^@embed?$` | **叠加规则**。UGCPolicy 通过 `AllowStandardAttributes()` 已允许 `title`（含自己的正则约束 `Paragraph`）。此调用在 bluemonday 中是**追加**而非替换——`title` 值只要匹配 UGCPolicy 原有正则**或**此新增正则即可通过。新增正则的真正作用是放行 `@embed?` 值供 Embed 插件使用 |
| `AllowAttrs("start").OnElements("ol")` | 允许 `<ol start="...">`，**未调用 `.Matching()`，bluemonday 不校验 start 值** | 实际安全。详见下方"属性值约束"分析 |

### 3.4 Markdown2BasicHTML（更严格的子集）
**代码位置**：[markdown.go:68-77](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L68-L77)

用于用户简介（Bio）等场景，先用 `Markdown2HTML` 转换，再用全新的空白策略二次过滤：
- 仅允许标签：`p, b, br, strong, em`
- 仅允许 `img[src]`
- 剥离标签时自动加空格避免文字粘连

---

## 四、第 3 层：Service 层链接替换 —— 结构性二次净化

**代码位置**：[question_common/question.go:703-828](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/service/question_common/question.go#L703-L828)

当问答审核通过（Status=Available）后，Service 层会调用 `UpdateQuestionLink` 对 ParsedText 做进一步处理：

### 4.1 流程
1. 从 **OriginalText（Markdown 原文）** 中提取链接引用：
   - `/questions/{questionID}[/{answerID}]` 形式的 URL
   - `#{questionID}` 或 `#{answerID}` 形式的 ID 引用
2. 通过 [checker.GetQuestionLink](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/checker/question_link.go#L39-L62) 解析出 QuestionID / AnswerID
3. 查询数据库验证这些 ID **真实存在**
4. 将 ParsedText 中的 `#ID` 文本，替换为安全构造的 `<a href="/questions/...">#ID</a>` 链接
5. 建立 QuestionLink 关联记录

### 4.2 安全意义
- 避免用户输入 `#123` 被其他机制误渲染为指向不存在页面的链接
- 链接的 `href` 完全由服务端通过格式化字符串构造，不从用户输入继承
- 只在审核通过后执行，确保待处理内容已经过前两层净化

---

## 五、第 4 层：Review 审核 —— 业务级内容安全

**代码位置**：[review_service.go:206-243](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/service/review/review_service.go#L206-L243)

`callPluginToReview` 遍历所有已注册的 `Reviewer` 插件：

```go
_ = plugin.CallReviewer(func(reviewer plugin.Reviewer) error {
    if reviewStatus != plugin.ReviewStatusApproved {
        return nil
    }
    if result := reviewer.Review(reviewContent); !result.Approved {
        reviewStatus = result.ReviewStatus
        r.Reason = result.Reason
    }
    return nil
})
```

- 任一插件返回不通过，内容即进入 Pending / Rejected 状态
- 只有状态为 Available 的内容才会对外展示并做链接替换

---

## 六、前端展示层

### 6.1 HTML 渲染
**代码位置**：
- 问题：[Question/index.tsx:160-166](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/ui/src/pages/Questions/Detail/components/Question/index.tsx#L160-L166)
- 回答：[Answer/index.tsx:133-138](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/ui/src/pages/Questions/Detail/components/Answer/index.tsx#L133-L138)

```tsx
<article
    className="fmt text-break text-wrap"
    dangerouslySetInnerHTML={{ __html: data?.html }}
/>
```

`data.html` 来自后端数据库中的 `ParsedText` 字段，是已经过多层净化的受信 HTML。

### 6.2 htmlRender UX 增强
**代码位置**：[Editor/utils/index.ts:40-124](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/ui/src/components/Editor/utils/index.ts#L40-L124)

对已渲染的 DOM 做纯前端优化：

| 处理 | 作用 | 安全边界影响 |
|------|------|-------------|
| LaTeX `<br>` 替换 | 修复公式块内 `<br>` 导致的渲染异常 | 使用 `p.innerHTML` 赋值，但操作对象是公式段落内的已知标签 |
| 表格样式包装 | 给 `<table>` 加 Bootstrap class 并套 `.table-responsive` | 使用 `createElement` + `replaceChild`，不涉及 innerHTML 注入 |
| 外链加 nofollow | 判断 `a.href` origin 与当前站点不同则加 `rel="nofollow"` | **这是后端取消 nofollow 后的安全兜底**——只有外链加 nofollow，站内链接不加 |
| 代码块复制按钮 | 给 `<pre>` 加复制按钮和 Tooltip | 使用 `codeTool.innerHTML` 插入按钮 HTML，但内容为硬编码模板字符串 |

### 6.3 前端插件渲染钩子
**代码位置**：[pluginKit/index.ts:328-366](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/ui/src/utils/pluginKit/index.ts#L328-L366)

```typescript
const useRenderHtmlPlugin = (element: HTMLElement | RefObject<HTMLElement> | null) => {
    plugins.getPlugins()
        .filter(p => p.activated && p.hooks?.useRender
            && (p.info.type === PluginType.Editor || p.info.type === PluginType.Render))
        .forEach(p => p.hooks.useRender.forEach(hook => hook(element, request)));
};
```

插件接收的是**已渲染完成、已净化的 DOM 元素引用**，但此引用是可写的——插件完全可以调用 `element.innerHTML = '...'` 注入任意 HTML，或 `document.createElement('script')` 插入脚本。

**安全边界不是技术性的，而是信任性的**：
- 插件代码由站点管理员安装，属于受信代码
- 插件处于 `activated` 状态才被调用，管理员可随时禁用
- 攻击面 = 恶意插件 或 被入侵的插件源

---

## 七、扩展点机制与安全边界

### 7.1 后端插件接口（plugin/ 目录）

| 接口 | 方法签名 | 安全角色 |
|------|---------|---------|
| [Filter](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/plugin/filter.go) | `FilterText(text string) error` | 文本级过滤钩子（当前主流程未直接调用，预留扩展点） |
| [Parser](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/plugin/parser.go) | `Parse(text string) (string, error)` | 文本解析/转换钩子（当前主流程未直接调用，预留扩展点） |
| [Reviewer](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/plugin/reviewer.go) | `Review(content) ReviewResult` | 业务审核层，可将内容打为待审/拒绝 |
| [Embed](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/plugin/embed.go) | `GetEmbedConfigs()` | 影响 `title` 属性白名单中的 `@embed?` 标记 |

**关键**：`MakePlugin[PluginType](super)` 的 `super` 参数决定插件是否可被管理员禁用：
- `super=true`：基础插件，始终启用（如 `CallBase`）
- `super=false`：功能插件，受 `StatusManager` 控制（如 Filter、Parser、Reviewer、Embed）

### 7.2 前端插件接口

| 钩子 | 作用 | 安全边界 |
|------|------|---------|
| `EditorReplacement` | 替换整个编辑器组件 | 仍通过 `/answer/api/v1/post/render` 获取预览 HTML |
| `useRender` / `useRenderHtmlPlugin` | 对已渲染 DOM 增强 | 接收 DOM 元素引用，**技术上可修改 innerHTML、创建 script 等**；安全性依赖插件受信 |
| 工具栏 PluginSlot | 编辑器工具栏扩展 | 仅注入 UI 按钮，不绕过 Markdown 解析 |

### 7.3 扩展点设计原则
1. **HTML 生成权完全在后端**：前端预览也走后端 `Markdown2HTML` 接口，不存在两套渲染逻辑
2. **插件不介入 HTML 生成核心管道**：Filter/Parser 虽定义接口但当前 Markdown2HTML 主流程未直接调用，避免插件引入 XSS 漏洞
3. **插件可被禁用隔离**：除 Base 插件外，所有功能插件均可被管理员关闭
4. **title 属性的 @embed? 约定**：为嵌入插件保留的最小化附加规则，值被正则 `^@embed?$` 严格约束为固定字面量，不可注入脚本
5. **两道 bluemonday 策略隔离**：第 1 层内嵌的 RawHTML 过滤使用严格默认 UGCPolicy，第 2 层全局过滤使用放宽版 UGCPolicy——即使第 2 层放宽了 URL 校验，第 1 层的 RawHTML 仍受严格约束

---

## 八、逐层降险总表

| 层级 | 位置 | 防护机制 | 降低的风险 | 引入的放宽 |
|------|------|---------|-----------|-----------|
| 1a | `renderRawHTML` | 默认 UGCPolicy 逐段过滤（nofollow=true, ParseableURLs=true） | 内联 `<script>`、危险标签、危险属性、不可解析 URL | `<kbd>` 原样放行（低风险） |
| 1b | `renderHTMLBlock` | `SecureWrite` HTML 实体转义 | 块级原始 HTML 完全文本化，不可能执行 | 无 |
| 1c | `renderLink` | IsURL + IsDangerousURL + EscapeHTML + URLEscape | `javascript:` 等危险协议、href 注入 | 无 |
| 1d | `renderAutoLink` | IsURL + EscapeHTML + URLEscape（无 IsDangerousURL） | 格式非法 URL、href 注入 | 缺少 IsDangerousURL，由 govalidator.IsURL 兜底 |
| 2 | Bluemonday 放宽版 UGCPolicy | 标签白名单 + 属性白名单 + 正则约束 | 漏过第 1 层的危险标签/属性 | 取消 nofollow、取消 URL 解析校验、允许 style 属性 |
| 3 | `UpdateQuestionLink` | 验证 ID 存在后才替换链接、href 服务端格式化 | 伪造站内链接 ID、href 注入 | 无 |
| 4 | ReviewService | 内容审核打标，非 Approved 不展示 | 垃圾/违规内容对外暴露 | 无 |
| 5 | `dangerouslySetInnerHTML` | 只渲染后端 ParsedText | 前端自行拼接 HTML 导致的注入 | 无 |
| 6 | `htmlRender` | 外链自动加 nofollow（补后端放宽的缺口） | 外链权重流失 | 无（非安全层，UX 增强） |
| 7 | `useRenderHtmlPlugin` | 插件接收已净化 DOM | — | **插件可写 DOM，安全性依赖信任边界** |

---

## 九、关键文件索引

| 文件 | 职责 |
|------|------|
| [pkg/converter/markdown.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go) | Markdown→HTML 转换核心，含 1~2 层防护 |
| [pkg/checker/question_link.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/checker/question_link.go) | 从原文提取问答链接引用 |
| [internal/schema/answer_schema.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/schema/answer_schema.go) | 回答请求 DTO，Check() 中触发 Markdown2HTML |
| [internal/schema/question_schema.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/schema/question_schema.go) | 问题请求 DTO，Check() 中触发 Markdown2HTML |
| [internal/schema/comment_schema.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/schema/comment_schema.go) | 评论请求 DTO，Check() 中触发 Markdown2HTML |
| [internal/service/content/answer_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/service/content/answer_service.go) | 回答写入、审核、链接替换 |
| [internal/service/content/question_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/service/content/question_service.go) | 问题写入、审核、链接替换 |
| [internal/service/question_common/question.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/service/question_common/question.go#L703-L828) | UpdateQuestionLink 链接替换实现（第 3 层） |
| [internal/service/review/review_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/service/review/review_service.go#L206-L243) | 审核插件调用（第 4 层） |
| [plugin/plugin.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/plugin/plugin.go) | MakePlugin 插件注册/调度框架 |
| [plugin/filter.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/plugin/filter.go) | Filter 插件接口定义 |
| [plugin/parser.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/plugin/parser.go) | Parser 插件接口定义 |
| [plugin/embed.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/plugin/embed.go) | Embed 插件接口定义 |
| [internal/controller/upload_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/controller/upload_controller.go#L98-L114) | 前端预览接口 PostRender |
| [ui/src/components/Editor/Viewer.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/ui/src/components/Editor/Viewer.tsx) | 编辑器预览组件 |
| [ui/src/components/Editor/utils/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/ui/src/components/Editor/utils/index.ts#L40-L124) | htmlRender 前端 DOM 增强 |
| [ui/src/pages/Questions/Detail/components/Question/index.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/ui/src/pages/Questions/Detail/components/Question/index.tsx) | 问题正文渲染 |
| [ui/src/pages/Questions/Detail/components/Answer/index.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/ui/src/pages/Questions/Detail/components/Answer/index.tsx) | 回答正文渲染 |
| [ui/src/utils/pluginKit/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/ui/src/utils/pluginKit/index.ts#L328-L366) | 前端 useRenderHtmlPlugin 钩子 |
