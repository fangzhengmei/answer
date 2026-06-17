# 问答正文 XSS 防护管道分析

本文档梳理问答正文从作者输入到 Markdown 渲染再到前端展示的完整数据流，重点分析白名单放宽与收紧、属性清理与扩展点机制如何逐层降险。

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
- **默认 `RequireNoFollowOnLinks(true)`**：所有 `<a>` 自动加 `rel="nofollow"`（由 `AllowStandardURLs()` 设置）
- **默认 `RequireParseableURLs(true)`**：URL 属性值必须可被 `net/url.Parse` 解析（由 `AllowStandardURLs()` 设置）
- **默认 `AllowURLSchemes("mailto", "http", "https")`**：仅允许这三个 URL scheme（由 `AllowStandardURLs()` 设置）
- **默认 `AllowRelativeURLs(true)`**：允许相对 URL（由 `AllowStandardURLs()` 设置）
- **默认不允许 `class` 属性**：UGCPolicy 源码注释明确写 `"class" is not permitted as we are not allowing users to style their own content`。`AllowStandardAttributes()` 只放行 `dir`、`lang`、`id`、`title`，不含 `class`
- **默认不允许 `style` 属性**：UGCPolicy 源码注释明确写 `"style" is not permitted as we are not yet sanitising CSS and it is an XSS attack vector`。未调用 `AllowStyles()` 或 `AllowAttrs("style")`
- **`AllowLists()` 只允许 `ol`/`ul` 的 `type`、`li` 的 `value`**：未允许 `start` 属性

> **class/style 澄清**：UGCPolicy 默认**同时禁止** `class` 和 `style` 属性。项目代码调用 `AllowStyling()` 后，**仅放行 `class`**（受 `SpaceSeparatedTokens` 正则约束），`style` 仍被移除。goldmark 的 `GlobalAttributeFilter` 虽然包含 `class` 和 `style`（允许 AST 节点属性输出），但 bluemonday 第 2 层会过滤掉不被允许的属性。

### 3.2 放宽配置（在 UGCPolicy 基线上退让）

| 配置 | UGCPolicy 默认值 | 项目设置 | 效果 |
|------|------------------|---------|------|
| `RequireNoFollowOnLinks(false)` | true（所有链接加 nofollow） | false | **取消**对所有链接强制加 nofollow。后端输出的 `<a>` 不带 nofollow，外链 nofollow 由前端 `htmlRender` 补上 |
| `RequireNoFollowOnFullyQualifiedLinks(false)` | true（绝对链接加 nofollow） | false | **取消**对绝对链接强制加 nofollow |
| `RequireParseableURLs(false)` | true（URL 必须可解析 + scheme 白名单 `mailto`/`http`/`https`） | false | **同时取消**两项校验：(1) URL 可解析性（`net/url.Parse`）；(2) URL scheme 白名单。bluemonday `sanitizeAttrs` 中整个 URL 校验块包裹在 `if p.requireParseableURLs { ... }` 内，置 false 后 `AllowURLSchemes("mailto","http","https")` 设置的 scheme 白名单**不再生效**，`javascript:`/`data:` 等协议的 href 在第 2 层不会被拦截。**但实际无影响**：异常 href 到不了第 2 层（详见 3.5 节"异常链接过滤边界"分析） |

### 3.3 扩展配置（在 UGCPolicy 基线上新增能力）

| 配置 | 效果 | 风险评估 |
|------|------|---------|
| `AllowStyling()` | **相对 UGCPolicy 默认新增放行 `class` 属性**（全局，值须匹配 `SpaceSeparatedTokens` 正则 `^([\s\p{L}\p{N}_-]+)$`）。**仍不放行 `style`** | ① bluemonday v1.0.27 `AllowStyling()` 实现只调用 `AllowAttrs("class").Matching(SpaceSeparatedTokens).Globally()`，源码注释明确写"class is not permitted as we are not allowing users to style their own content"、"style is not permitted as we are not yet sanitising CSS"。② **放行 class 的实际用途**：goldmark `renderFencedCodeBlock`（html.go）对 \`\`\`go 代码块生成 `<code class="language-go">`，前端高亮插件据此 class 添加语法高亮。③ **两层 class 处理交互**：第 1a 层 RawHTML 的默认 UGCPolicy 不放行 class，内联原始 HTML 中的 class 属性全被剔除；第 2 层放宽版 UGCPolicy 新增放行 class，但此时 HTML 已经是 goldmark 生成的（非用户原始 HTML），class 值受 `SpaceSeparatedTokens` 正则约束。④ `style` 属性在 UGCPolicy 与本项目策略下**均被移除**，用户无法通过 `style` 注入 CSS |
| `AllowElements("kbd")` | 额外放行 `<kbd>` | 低风险。kbd 标签无安全隐患 |
| `AllowAttrs("title").Matching(regex).Globally()` | 全局允许 `title` 属性，值必须匹配 `^[\p{L}\p{N}\s\-_',\[\]!\./\\\(\)]*$` 或 `^@embed?$` | **叠加规则**。UGCPolicy 通过 `AllowStandardAttributes()` 已允许 `title`（含自己的正则约束 `Paragraph`）。此调用在 bluemonday 中是**追加**而非替换——`title` 值只要匹配 UGCPolicy 原有正则**或**此新增正则即可通过。新增正则的真正作用是放行 `@embed?` 值供 Embed 插件使用 |
| `AllowAttrs("start").OnElements("ol")` | 允许 `<ol start="...">`，**未调用 `.Matching()`，bluemonday 不校验 start 值** | 实际安全。详见 3.4 节"属性值约束"分析 |

### 3.4 属性值约束分析：start 属性是否受数值正则限制

**结论：`AllowAttrs("start").OnElements("ol")` 没有数值正则约束，但 `start` 值在所有输入路径上实际都被约束为整数。**

**bluemonday 行为**：从 bluemonday v1.0.27 `policy.go` 的 `OnElements` 实现看，未调用 `.Matching()` 时 `attrPolicyBuilder.regexp` 为 nil，生成的 `attrPolicy.regexp` 也为 nil。`sanitize.go` 的 `sanitizeAttrs` 中：

```go
if ap.regexp != nil {
    if ap.regexp.MatchString(htmlAttr.Val) { ... }
} else {
    cleanAttrs = append(cleanAttrs, htmlAttr)  // 直接保留，不检查值
}
```

所以第 2 层对 `<ol start="任意字符串">` 都会放行。

**但 `start` 值受上游约束，不会出现非数字值**：

| 输入路径 | 第 1 层约束 | 到达第 2 层时的状态 | 第 2 层行为 |
|---------|---------|---------|---------|
| Markdown 有序列表 `3. item` | goldmark `renderList`：**`if n.IsOrdered() && n.Start != 1`** 才输出 `start` 属性（`fmt.Fprintf(w, " start=\"%d\"", n.Start)`），`n.Start` 是 `int` 类型，`%d` 格式化强制为整数（见 goldmark v1.7.4 `renderer/html/html.go`） | 纯数字 start 值（Start≠1）或无 start 属性（Start==1） | `AllowAttrs("start").OnElements("ol")` 未调用 `.Matching()`，`ap.regexp == nil`，**直接保留不校验值**（但值已被上游约束为纯整数） |
| RawHTML `<ol start="abc">` | 第 1a 层默认 UGCPolicy 的 `AllowLists()` 只允许 `ol`/`ul` 的 `type` 属性、`li` 的 `value` 属性，**未允许 `start`**——start 属性被移除 | 属性已不存在 | 无需处理 |
| HTMLBlock `<ol start="abc">` | 第 1b 层 `SecureWrite` 将整个内容转义为文本 `&lt;ol start=&quot;abc&quot;&gt;` | 文本化，无真实标签 | 无需处理 |

### 3.5 异常链接过滤边界：javascript: 等危险协议能否绕过多层过滤

**结论：异常 href（`javascript:`、`data:`、`vbscript:` 等）无法绕过多层过滤到达可执行 DOM。**

异常 href 的三条输入路径分析：

**路径 A：Markdown 链接 `[text](javascript:alert(1))`**
- goldmark [renderLink](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L136-L158) 调用 `renderLinkIsUrl("javascript:alert(1)")`
- `govalidator.IsURL("javascript:alert(1)")` 返回 **false**，三步校验均不通过：
  - **第一步（前置过滤）**：`str == ""` → false；`strings.HasPrefix(str, ".")` → false；`http//`/`https//`/`ftp//` 等前缀黑名单不匹配 → 通过
  - **第二步（url.Parse 探测）**：`url.Parse("javascript:alert(1)")` 解析出 scheme=`javascript`、opaque=`alert(1)`，host 为空；由于 `strTemp` 含 `:` 但不含 `://`，会被前缀 `http://` 后再解析，host 仍为空（`u.Host == ""` 且 `u.Path` 不含 `.` → false）
  - **第三步（rxURL 正则匹配）**：`URLScheme = ((ftp|tcp|udp|wss?)://)` 只允许这些 scheme 后跟 `://`，`javascript:` 不匹配 → **false**
- 不以 `/` 开头
- `renderLinkIsUrl` 返回 false → **不渲染 `<a>` 标签**，输出纯文本 `[text](javascript:alert(1))`
- 到第 2 层时不含任何 `<a>`，bluemonday 无需处理

**路径 B：RawHTML `<a href="javascript:alert(1)">click</a>`**
- 第 1a 层默认 UGCPolicy（`requireParseableURLs=true` + `AllowURLSchemes("mailto","http","https")`）
- `sanitizeAttrs` 进入 `if p.requireParseableURLs { ... }` 块，调用 `validURL("javascript:alert(1)")`
- `url.Parse("javascript:alert(1)")` 解析出 scheme=`javascript`，但 `javascript` 不在 `allowURLSchemes` 白名单中
- `validURL` 返回 false → **href 属性被移除**（`<a>` 标签保留但无 href）
- 输出：`<a rel="nofollow">click</a>`（默认 UGCPolicy 加 nofollow）
- 第 2 层放宽版 UGCPolicy：`<a>` 仍被允许，但 href 已不存在，无法再生

**路径 C：HTMLBlock `<a href="javascript:alert(1)">click</a>`**
- 第 1b 层 [renderHTMLBlock](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L121-L134) 调用 `SecureWrite`
- 整个内容转义为 `&lt;a href=&quot;javascript:alert(1)&quot;&gt;click&lt;/a&gt;`
- **完全文本化**，浏览器显示为可见文本而非 DOM 元素

**补充：goldmark `renderLink` 的第二道防线**
即使 `renderLinkIsUrl` 通过了（例如某些构造的 URL 绕过 `govalidator.IsURL`），[renderLink](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L141-L143) 还有 `IsDangerousURL` 检查：

```go
if r.Unsafe || !goldmarkHTML.IsDangerousURL(n.Destination) {
    _, _ = w.Write(util.EscapeHTML(util.URLEscape(n.Destination, true)))
}
```

`IsDangerousURL` 拦截 `javascript:`、`vbscript:`、`data:` 等危险协议。注意 `renderAutoLink` 没有此检查，但其 `renderLinkIsUrl` 同样会拦截无 host 的 `javascript:` 协议。

**异常 href 多层过滤汇总表**：

| 输入路径 | 第 1 层拦截点 | 拦截机制 | 到达第 2 层时的状态 | 第 2 层是否需处理 |
|---------|---------|---------|---------|---------|
| 路径 A：Markdown `[text](javascript:...)` | `renderLinkIsUrl` → `govalidator.IsURL` | rxURL 正则不匹配 `javascript:` scheme | 不生成 `<a>` 标签，输出纯文本 | 否（无 `<a>` 可处理） |
| 路径 B：RawHTML `<a href="javascript:...">` | `sanitizeAttrs` → `validURL` + `allowURLSchemes` | `javascript` 不在 `mailto`/`http`/`https` 白名单 | href 属性被移除，`<a>` 无 href | 否（href 已不存在） |
| 路径 C：HTMLBlock `<a href="javascript:...">` | `SecureWrite` | HTML 实体转义 | 完全文本化，无 DOM 元素 | 否（无标签可处理） |

**防御纵深关键**：第 2 层 `RequireParseableURLs(false)` 同时取消了 URL 可解析性校验和 scheme 白名单，理论上存在缺口。但异常 href 在三条输入路径中均被第 1 层拦截（路径 A/C 在 goldmark 层、路径 B 在第 1a 层），**到达第 2 层时已不含可执行的异常 href**，因此放宽不会产生实际风险。

### 3.6 放宽限制的实际风险总结

| 放宽项 | 理论风险 | 实际是否受控 | 受控原因 |
|--------|---------|------------|---------|
| `RequireNoFollowOnLinks(false)` | 外链丢失 nofollow，SEO 权重流失 | 是（非安全问题） | 前端 [htmlRender](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/ui/src/components/Editor/utils/index.ts#L74-L81) 对外链补 nofollow |
| `RequireParseableURLs(false)` | 第 2 层不再校验 URL scheme，`javascript:` 等 href 理论上可通过 | 是 | 异常 href 在 goldmark 层（路径 A/C）或第 1a 层（路径 B）已被拦截，到不了第 2 层 |
| `AllowStyling()` 放行 `class` | 用户可注入任意 class 名 | 是 | class 值受 `SpaceSeparatedTokens` 正则约束（`^([\s\p{L}\p{N}_-]+)$`），且 `style` 属性仍被移除 |
| `AllowAttrs("start")` 无 Matching | `<ol start="恶意值">` 理论上可通过第 2 层 | 是 | Markdown 语法路径受 goldmark int 类型约束；RawHTML 路径被第 1a 层移除 |

### 3.7 Markdown2BasicHTML（更严格的子集）
**代码位置**：[markdown.go:68-77](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L68-L77)

用于用户简介（Bio）等场景，先用 `Markdown2HTML` 转换，再用全新的空白策略二次过滤：
- 仅允许标签：`p, b, br, strong, em`
- 仅允许 `img[src]`
- 剥离标签时自动加空格避免文字粘连

---



### 3.8 图片 `<img src>` 安全流程

图片在 Markdown 中的语法为 `![alt](src "title")`，处理链路为 **goldmark 渲染器 → bluemonday 两层过滤**，与链接 `<a href>` 流程类似但少了 govalidator.IsURL 校验。

#### 3.8.1 第 1 层：goldmark `renderImage`（项目未覆盖，走 goldmark 默认渲染器）

项目 `DangerousHTMLRenderer` 的 `RegisterFuncs` 只注册了 `HTMLBlock`/`RawHTML`/`Link`/`AutoLink` **四种**，**没有覆盖 `ast.KindImage`**，因此图片走 goldmark v1.7.4 的 [renderImage](https://github.com/yuin/goldmark/blob/v1.7.4/renderer/html/html.go) 默认实现：

```go
func (r *Renderer) renderImage(..., entering bool) ... {
    if !entering { return ast.WalkContinue, nil }
    n := node.(*ast.Image)
    _, _ = w.WriteString("<img src=\"")
    if r.Unsafe || !IsDangerousURL(n.Destination) {
        _, _ = w.Write(util.EscapeHTML(util.URLEscape(n.Destination, true)))
    }
    _, _ = w.WriteString(`\" alt=\"`)
    r.renderAttribute(w, source, n)   // alt 文本走 SecureWrite
    ...
    if n.Attributes() != nil {
        RenderAttributes(w, n, ImageAttributeFilter)
    }
}
```

关键约束：
- **`IsDangerousURL` 检查**（同 `renderLink`）：拦截 `javascript:`/`vbscript:`/`data:` 等危险协议
- **`URLEscape(n.Destination, true)`**：URL 百分号编码
- **`EscapeHTML`**：HTML 实体转义
- **无 `govalidator.IsURL` 检查**（不像 `renderLink`/`renderAutoLink` 有 `renderLinkIsUrl` 守卫）——格式不合法但非危险协议的 URL（如相对路径）会被直接渲染
- `ImageAttributeFilter` 扩展了 `GlobalAttributeFilter`，放行 `align`/`border`/`crossorigin`/`height`/`loading`/`srcset`/`usemap`/`width` 等，但这些属性在 bluemonday 第 2 层会被进一步过滤

#### 3.8.2 第 1a 层：RawHTML 中的 `<img>`（内嵌严格 UGCPolicy）

如果作者直接在内联原始 HTML 中写 `<img src="...">`（如 `<img src="javascript:alert(1)">`），会走 [renderRawHTML](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L104-L118) 的 bluemonday 默认 UGCPolicy：
- **第 1 关**：`requireParseableURLs=true` + scheme 白名单 `mailto`/`http`/`https`，`validURL` 校验不通过的 `src` 被移除
- **第 2 关**：`AllowStyling()` 未被调用，`class`/`style` 属性全被移除
- **结果**：`<img src="http://example.com/a.png">` 保留；`<img src="javascript:...">` 被移除 src

#### 3.8.3 第 1b 层：HTMLBlock 中的 `<img>`

整个块被 `SecureWrite` 转义为纯文本 `&lt;img ...&gt;`，与其他 HTMLBlock 内容一致。

#### 3.8.4 第 2 层：放宽版 UGCPolicy 的 img 策略

第 2 层 [Markdown2HTML](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L56-L64) 虽然调用了 `RequireParseableURLs(false)`（取消 URL scheme 白名单），但 UGCPolicy 的 `AllowImages()` 已经定义好：

```go
// bluemonday helpers.go AllowImages()
func (p *Policy) AllowImages() {
    p.AllowAttrs("align").Matching(ImageAlign).OnElements("img")
    p.AllowAttrs("alt").Matching(Paragraph).OnElements("img")
    p.AllowStandardURLs()           // 把 img.src 加入 URL 属性清单
    p.AllowAttrs("src").OnElements("img")
}
```

`AllowStandardURLs()` 的作用是把 `img.src` 加入 `sanitize.go` 中 `requireParseableURLs` 块处理的 URL 属性清单（sanitize.go 注释明列 `- img.src`）。但由于第 2 层设了 `RequireParseableURLs(false)`，**整个 URL 校验块被跳过**，`img.src` 的 `validURL`/scheme 白名单校验都不执行，`ap.regexp == nil` 直接保留值。

**但这并不意味着风险**：
- Markdown 语法路径的图片 src 在 goldmark `renderImage` 中已经过 `IsDangerousURL` 拦截，危险协议到不了第 2 层
- RawHTML 路径的 `<img>` 已被 1a 层处理完毕

#### 3.8.5 图片 src 多层过滤汇总表

| 输入路径 | 第 1 层拦截点 | 拦截机制 | 到达第 2 层时的状态 | 第 2 层行为 |
|---------|---------|---------|---------|---------|
| Markdown `![alt](src)` | goldmark `renderImage` → `IsDangerousURL` + URLEscape + EscapeHTML | 拦截 `javascript:`/`vbscript:`/`data:`，值被 URL 编码/HTML 转义 | `<img src="经编码的值" alt="...">`（无危险协议） | `ap.regexp == nil` 直接保留 src；`srcset`/`style`/`class` 等非白名单属性被移除 |
| RawHTML `<img src="...">` | `sanitizeAttrs` → `validURL`（1a 层） | `javascript:` 等 scheme 不通过白名单，src 被移除 | src 合法则保留，否则 src 被移除 | 合法 src 继续保留 |
| HTMLBlock `<img src="...">` | `SecureWrite` | HTML 实体转义 | 完全文本化 | 无需处理 |

### 3.9 评论正文写入流程

评论与问答正文不同：**问答正文的 Markdown 渲染在前端完成**（前端发请求提交时即携带 `parsed_text`），而**评论正文的 Markdown→HTML 转换在后端 schema 校验阶段完成**。

#### 3.9.1 写入链路全景

```
CommentController.AddComment
  └─ gin 绑定 & 校验 schema.AddCommentReq.Check()
       └─ converter.Markdown2HTML(req.OriginalText)  →  req.ParsedText
  └─ commentService.AddComment(ctx, req)
       ├─ copier.Copy(comment, req)        # OriginalText / ParsedText 一起复制
       ├─ comment.Status = CommentStatusAvailable
       ├─ objectInfoService.GetInfo(...)   # 校验对象存在且未被删
       ├─ (reply) commentCommonRepo.GetComment + 设 ReplyUserID/ReplyCommentID
       ├─ commentRepo.AddComment(ctx, comment)   # 入库，写入 original_text & parsed_text
       └─ reviewService.AddCommentReview(...)     # 审核打标，可能改 status
```

#### 3.9.2 关键代码证据

**Schema 校验阶段渲染 HTML**（[comment_schema.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/schema/comment_schema.go#L59-L68)）：

```go
func (req *AddCommentReq) Check() (errFields []*validator.FormErrorField, err error) {
    req.ParsedText = converter.Markdown2HTML(req.OriginalText)  // 与问答共用同一套 Markdown2HTML
    if req.ParsedText == "" {
        return ... errors.BadRequest(reason.CommentContentCannotEmpty)
    }
    return nil, nil
}
```

**评论长度约束**：`OriginalText` 的 validate tag 为 `gte=2,lte=600`，即评论最多 600 字符（问答正文无此硬上限）。

**Service 层写入**（[comment_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/service/comment/comment_service.go#L133-L177)）：
- 使用 `copier.Copy` 把请求体的 `OriginalText` 和 `ParsedText` 原样复制到 entity
- `commentRepo.AddComment` 直接写入 DB，**不会再次调用 bluemonday 净化**
- `reviewService.AddCommentReview` 只打审核状态标签（`Pending`/`Available`），不改 ParsedText 内容

**更新评论时**（`UpdateComment`，[comment_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/service/comment/comment_service.go#L302-L340)）：前端同样提交 `OriginalText` + `ParsedText`，由 controller 层的 schema `Check()` 重新调用 `Markdown2HTML`。

#### 3.9.3 评论 vs 问答正文：流程差异

| 维度 | 问答正文（Question/Answer） | 评论（Comment） |
|------|--------------------------|--------------|
| Markdown 渲染位置 | 前端（前端生成 `parsed_text` 提交） | 后端（schema.Check 调用 `Markdown2HTML`） |
| 二次净化 | 后端不重跑 Markdown，只接受前端提交的 `parsed_text` | 后端生成 `parsed_text`，使用同一套 `Markdown2HTML` |
| XSS 管道一致性 | **一致**：最终都经由 `Markdown2HTML` 的 goldmark+bluemonday 两层 | **一致**：经由同一 `Markdown2HTML` |
| 评论专属 | — | AddCommentReq.Check 后还会走审核服务（`AddCommentReview`），非 Approved 不对外展示 |
| 长度硬上限 | 无 | OriginalText `lte=600` |
| 链接替换（UpdateQuestionLink） | Question/Answer 写入时调用 `UpdateQuestionLink` 校验 `#ID`/站内链接并替换 href | **不调用**：评论不做站内链接检测与替换 |

> **结论**：评论正文走的是与问答正文**完全相同**的 Markdown→HTML 管道（同一 [Markdown2HTML](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L40-L66)），防护层完全复用。差异点在于评论在 schema 校验阶段就完成了渲染，且额外经过审核状态打标，**不参与 UpdateQuestionLink 的站内链接替换**。

### 3.10 无效站内链接处理：UpdateQuestionLink 链路

站内问答中引用其他问答（以 URL 形式或 `#ID` 形式）需要被校验为"有效"后才能真正写入，否则保持原状、不建立关联。这一处理不影响 HTML 安全性（不引入新的 XSS 风险），但影响 SEO 链接图谱和导航正确性。

#### 3.10.1 调用位置

`UpdateQuestionLink` 只在以下四个入口被调用（[question_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/service/content/question_service.go#L398) / [answer_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/service/content/answer_service.go#L283)）：
- 新增 Question（`questionService.AddQuestion`）
- 更新 Question（`questionService.UpdateQuestion`）
- 新增 Answer（`answerService.AddAnswer`）
- 更新 Answer（`answerService.UpdateAnswer`）

**评论不调用此函数**（见 3.9 节）。

#### 3.10.2 `GetQuestionLink`：从正文提取链接（[question_link.go](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/checker/question_link.go#L39-L62)）

扫描 `originalText`（Markdown 原文，不是 HTML）中的两种模式：

| 模式 | 扫描方式 | 解析结果 |
|------|---------|---------|
| URL 路径 `/questions/xxx[/yyy]` | 逐字符匹配字符串 `/questions/`（**不是正则**，也不检查完整 URL），匹配后读取连续数字/字母为 QuestionID，可选 `/` 后再读为 AnswerID | `QuestionLinkTypeURL`，QuestionID + 可选 AnswerID |
| ID 形式 `#ID` | 遇到字符 `#`，读取后续连续数字/字母 | `QuestionLinkTypeID`，先判断是 Question 还是 Answer |

**注意**：`GetQuestionLink` **扫描的是 `originalText`（Markdown 原文）**，不是 parsed HTML。URL 协议、域名等部分**完全忽略**——只要包含 `/questions/1234` 字符串段就会被匹配（如 `example.com/questions/1234`、`/questions/1234`、`https://a.b/questions/1234` 都能命中）。

ID 合法性在 `addUniqueID` 中通过 `obj.GetObjectTypeStrByObjectID(uid.DeShortID(id))` 校验（[question_link.go L99-L130](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/checker/question_link.go#L99-L130)）：
- ID 首位（DeShortID 后）必须符合 Question/Answer 对象前缀规则（见单测 step 14 `Error id`，ID `10110000000000060` 首位 `101` 非问答类型 → 返回空）
- 不能重复添加（`uniqueIDs` map 去重）

#### 3.10.3 ID 有效性二次校验（DB 查询）

`GetQuestionLink` 只做对象类型前缀校验，`UpdateQuestionLink` 随后做 DB 查询（[question_common.go L723-L807](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/service/question_common/question.go#L723-L807)）：

```go
links := checker.GetQuestionLink(originalText)
// 1. answerRepo.GetByIDs → answerCache
// 2. questionRepo.FindByID  → questionCache

for _, link := range links {
    // QuestionID 不在 questionCache 中 → skip（无效链接，不替换）
    if _, exists := questionCache[linkQuestionID]; linkQuestionID != "0" && !exists {
        continue
    }
    // AnswerID 不在 answerCache 中 → skip
    if linkAnswerID != "0" {
        if _, exists := answerCache[linkAnswerID]; !exists {
            continue
        }
    }
    // 构建有效链接，并替换 parsedText 中的 #ID → <a href="/questions/xxx">#ID</a>
}
```

#### 3.10.4 替换机制：只替换 `#ID`，不替换 URL

注意替换逻辑只操作 `parsedText` 中 `#` 前缀的 ID 字符串（[L793-L801](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/internal/service/question_common/question.go#L793-L801)）：

```go
if link.QuestionID != "" {
    htmlLink := fmt.Sprintf("<a href=\"/questions/%s\">#%s</a>", link.QuestionID, link.QuestionID)
    parsedText = strings.ReplaceAll(parsedText, "#"+link.QuestionID, htmlLink)
}
```

**对 URL 形式的站内链接（如 `https://host/questions/1234`）不做任何改写**——它们在 Markdown→HTML 转换阶段已经由 goldmark 渲染为 `<a href="https://host/questions/1234">`，UpdateQuestionLink 只负责：
1. 记录 QuestionLink 关联（用于站内链接图、被引用计数）
2. ID 形式 `#1234` 的内联引用转成可点击 `<a>`

#### 3.10.5 无效链接的三条路径处理汇总

| 输入示例 | GetQuestionLink 是否提取 | DB 是否存在 | 行为 | 对用户可见效果 |
|---------|----------------------|-----------|------|-------------|
| `#10010000000000060`（有效 QuestionID） | 是 | 是 | 替换为 `<a href="/questions/xxx">#xxx</a>`，建立 QuestionLink | 可点击，计入反向引用计数 |
| `#10020000000000060`（有效 AnswerID） | 是 | 是 | 替换为 `<a href="/questions/qid/aid">#aid</a>`，建立 QuestionLink | 可点击，计入反向引用计数 |
| `#99999999999999999`（不存在） | 是（如果前缀合法） | 否 | skip（不替换、不入库） | 原样保留为纯文本 `#9999...` |
| `/questions/invalid` | 否（"invalid" 非 Question 前缀 ID） | — | skip | Markdown 中的 URL 由 goldmark 正常渲染，不建立关联 |
| `https://host/questions/10010000000000060` | 是 | 是 | 不替换 parsedText，只建立 QuestionLink 关联 | URL 链接可点击（goldmark 已渲染），计入反向引用计数 |
| `#10110000000000060`（前缀 101 非问答类型，见单测） | 否（GetQuestionLink 的 `addUniqueID` 提前过滤） | — | skip | 原样保留为纯文本 |
| `javascript:alert(1)` | 否（不含 `/questions/` 或 `#`） | — | skip | 由 Markdown 渲染管道正常拦截（见 3.5 节） |

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
| 1a | `renderRawHTML` | 默认 UGCPolicy 逐段过滤（nofollow=true, ParseableURLs=true, scheme 白名单, **不允许 class/style**） | 内联 `<script>`、危险标签、`class`/`style` 等非白名单属性、不可解析 URL、`javascript:` 等 scheme | `<kbd>` 原样放行（低风险） |
| 1b | `renderHTMLBlock` | `SecureWrite` HTML 实体转义 | 块级原始 HTML 完全文本化，不可能执行 | 无 |
| 1c | `renderLink` | IsURL + IsDangerousURL + EscapeHTML + URLEscape | `javascript:` 等危险协议、href 注入 | 无 |
| 1d | `renderAutoLink` | IsURL + EscapeHTML + URLEscape（无 IsDangerousURL） | 格式非法 URL、href 注入 | 缺少 IsDangerousURL，由 govalidator.IsURL 兜底 |
| 1e | `renderImage`（goldmark 默认） | IsDangerousURL + URLEscape + EscapeHTML | `javascript:`/`data:` 等危险协议、src 注入 | 无 govalidator.IsURL，但危险协议被 IsDangerousURL 拦截；格式不合法但非危险的相对路径会被保留（第 2 层作最终过滤） |
| 2 | Bluemonday 放宽版 UGCPolicy | 标签白名单 + 属性白名单 + 正则约束 | 漏过第 1 层的危险标签/属性 | 取消 nofollow（前端补）、取消 URL 解析+scheme 校验（异常 href 到不了第 2 层）、**新增放行 `class`**（供 goldmark 代码块语言标记；RawHTML 的 class 已被 1a 层移除；**仍不放行 `style`**） |
| 3 | `UpdateQuestionLink` + Comment Schema.Check | ID 存在才替换；评论在 schema.Check 重调 Markdown2HTML 生成 ParsedText | 伪造站内链接 ID；评论正文跳过前端直接提交 parsed_text 的可能 | 评论不参与站内链接替换与反向计数 |
| 4 | ReviewService | 内容审核打标，非 Approved 不展示 | 垃圾/违规内容对外暴露 | 无 |
| 5 | `dangerouslySetInnerHTML` | 只渲染后端 ParsedText | 前端自行拼接 HTML 导致的注入 | 无 |
| 6 | `htmlRender` | 外链自动加 nofollow（补后端放宽的缺口） | 外链权重流失 | 无（非安全层，UX 增强） |
| 7 | `useRenderHtmlPlugin` | 插件接收已净化 DOM | — | **插件可写 DOM，安全性依赖信任边界** |
| 8 | `AddCommentReview` 评论审核 | 内容行为打标，非 Approved 不对外展示 | 评论正文含有伤害内容对外曝露 | 不改变 parsed_text，只影响过滤条件 |

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

