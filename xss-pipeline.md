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
│  └─ 第2层: Bluemonday UGCPolicy 白名单过滤                 │
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

通过自定义扩展 `DangerousHTMLFilterExtension` 将 `DangerousHTMLRenderer` 注入 goldmark 渲染管线，优先级为 1（最高）。该渲染器针对 4 种 AST 节点做了特殊处理：

### 2.1 RawHTML（内联原始 HTML）
函数：[renderRawHTML](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L104-L119)

```go
if string(segment) == "<kbd>" || string(segment) == "</kbd>" {
    // <kbd> 标签直接放行（用于键盘按键样式）
    w.Write(segment.Value(source))
} else {
    // 其余原始 HTML 片段，用 bluemonday.UGCPolicy() 逐段过滤
    w.Write(r.Filter.SanitizeBytes(segment.Value(source)))
}
```

### 2.2 HTMLBlock（HTML 块级内容）
函数：[renderHTMLBlock](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L121-L134)

使用 `r.Writer.SecureWrite` 写入，这是 goldmark 内置的安全写入方法，会对内容做 HTML 转义。

### 2.3 Link / AutoLink（链接）
函数：[renderLink](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L136-L158)、[renderAutoLink](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L160-L183)

```go
// 只在 URL 合法时才渲染 <a> 标签
if entering && r.renderLinkIsUrl(string(n.Destination)) {
    // 使用 util.EscapeHTML + util.URLEscape 双重转义 href
    w.Write(util.EscapeHTML(util.URLEscape(n.Destination, true)))
    goldmarkHTML.RenderAttributes(w, n, goldmarkHTML.LinkAttributeFilter)
}
```

URL 合法性校验：[renderLinkIsUrl](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L185-L189)
- 必须是合法 URL（govalidator.IsURL）
- 或者是以 `/` 开头的站内相对路径

---

## 三、第 2 层：Bluemonday 白名单 —— 全局 HTML 净化

**代码位置**：[markdown.go:39-66](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L39-L66)

Goldmark 输出完整 HTML 后，使用 `bluemonday.UGCPolicy()`（User Generated Content 策略）作为基线，在此基础上做白名单的**收紧**与**定向扩展**：

### 3.1 策略基线（UGCPolicy）
bluemonday 内置的 UGC 策略默认允许：
- 标签：`a, b, blockquote, br, caption, cite, code, col, colgroup, dd, del, dfn, dl, dt, em, figcaption, figure, h1-h6, hr, i, img, ins, kbd, li, mark, ol, p, pre, q, rp, rt, ruby, s, samp, small, strike, strong, sub, sup, table, tbody, td, tfoot, th, thead, tr, u, ul`
- 自动移除 `script, iframe, form, object, embed` 等危险标签
- 移除 `on*` 事件处理器
- 默认对所有链接加 `rel="nofollow"`

### 3.2 收紧配置
| 配置 | 作用 |
|------|------|
| `RequireNoFollowOnLinks(false)` | 不强制所有链接加 nofollow（交给前端 htmlRender 按外链判断） |
| `RequireParseableURLs(false)` | 不强制 URL 必须可解析（允许相对路径） |
| `RequireNoFollowOnFullyQualifiedLinks(false)` | 不强制绝对链接加 nofollow |

### 3.3 扩展配置
| 配置 | 作用 |
|------|------|
| `AllowStyling()` | 允许 `style` 属性（用于 Markdown 扩展的行内样式） |
| `AllowElements("kbd")` | 额外放行 `<kbd>` 标签 |
| `AllowAttrs("title").Matching(...).Globally()` | 全局允许 `title` 属性，值必须匹配正则 `^[\p{L}\p{N}\s\-_',\[\]!\./\\\(\)]*$` 或 `^@embed?$`（后者用于 Embed 嵌入插件的标记） |
| `AllowAttrs("start").OnElements("ol")` | 允许 `<ol start="...">` 用于有序列表起始编号 |

### 3.4 Markdown2BasicHTML（更严格的子集）
**代码位置**：[markdown.go:68-77](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/pkg/converter/markdown.go#L68-L77)

用于用户简介（Bio）等场景，白名单进一步收紧到：
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

`data.html` 来自后端数据库中的 `ParsedText` 字段，是已经经过多层净化的受信 HTML。

### 6.2 htmlRender UX 增强
**代码位置**：[Editor/utils/index.ts:40-124](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/ui/src/components/Editor/utils/index.ts#L40-L124)

对已渲染的 DOM 做纯前端优化（**不涉及安全边界扩展**）：

| 处理 | 作用 |
|------|------|
| LaTeX `<br>` 替换 | 修复公式块内 `<br>` 导致的渲染异常 |
| 表格样式包装 | 给 `<table>` 加 Bootstrap `.table .table-bordered` 并套 `.table-responsive` |
| 外链加 nofollow | 判断 `a.href` origin 与当前站点不同则加 `rel="nofollow"` |
| 代码块复制按钮 | 给 `<pre>` 加复制按钮和 Tooltip |

### 6.3 前端插件渲染钩子
**代码位置**：[pluginKit/index.ts:328-366](file:///d:/fz/0601-2/solo-dogfeeding/code/16-answer/ui/src/utils/pluginKit/index.ts#L328-L366)

```typescript
const useRenderHtmlPlugin = (element) => {
    plugins.getPlugins()
        .filter(p => p.activated && p.hooks?.useRender
            && (p.info.type === PluginType.Editor || p.info.type === PluginType.Render))
        .forEach(p => p.hooks.useRender.forEach(hook => hook(element, request)));
};
```

- 插件只能操作**已渲染完成、已净化**的 DOM 节点
- 插件不参与 HTML 生成阶段，无法绕过后端净化管道

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
| `useRender` / `useRenderHtmlPlugin` | 对已渲染 DOM 增强 | 只读/修改已净化 DOM，不接触原始 HTML 生成 |
| 工具栏 PluginSlot | 编辑器工具栏扩展 | 仅注入 UI 按钮，不绕过 Markdown 解析 |

### 7.3 扩展点设计原则
1. **HTML 生成权完全在后端**：前端预览也走后端 `Markdown2HTML` 接口，不存在两套渲染逻辑
2. **插件不介入 HTML 生成核心管道**：Filter/Parser 虽定义接口但当前 Markdown2HTML 主流程未直接调用，避免插件引入 XSS 漏洞
3. **插件可被禁用隔离**：除 Base 插件外，所有功能插件均可被管理员关闭
4. **title 属性的 @embed? 约定**：为嵌入插件保留的最小化后门，值被严格正则约束，不能执行脚本

---

## 八、逐层降险总表

| 层级 | 位置 | 防护机制 | 降低的风险 |
|------|------|---------|-----------|
| 1 | Goldmark `DangerousHTMLRenderer` | RawHTML 逐段 Sanitize、URL 合法性校验、SecureWrite | 内联 `<script>`、`javascript:` 协议、恶意事件属性 |
| 2 | Bluemonday UGCPolicy + 自定义规则 | 标签白名单、属性白名单 + 正则约束、style 受控放行 | 危险标签（iframe/form/object）、危险属性（on*/srcdoc）、非法 title 值 |
| 3 | Service `UpdateQuestionLink` | 验证 ID 存在后才替换链接、href 由服务端格式化构造 | 伪造的站内链接 ID、href 注入 |
| 4 | ReviewService + Reviewer 插件 | 内容审核打标，非 Approved 不展示 | 垃圾内容、违规内容、未过审内容对外暴露 |
| 5 | 前端 `dangerouslySetInnerHTML` | 只渲染后端 ParsedText，不渲染用户原文 | 前端自行拼接 HTML 导致的注入 |
| 6 | 前端 `htmlRender` | 外链自动加 nofollow、DOM 结构受控改造 | 外链权重流失、UX 问题（非安全，但为防御性设计） |
| 7 | 前端 `useRenderHtmlPlugin` | 插件只读/改已净化 DOM | 恶意插件扩展 XSS 面 |

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
