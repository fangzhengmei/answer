# 问答社区多实体全文搜索代码链路分析

## 一、整体架构概览

### 1.1 架构设计

搜索系统采用**插件化架构**设计，核心通过 `plugin.Search` 接口定义搜索能力，支持两种运行模式：

- **系统内置搜索**：基于数据库 `LIKE` 查询实现，无需额外依赖
- **插件搜索**：通过注册 `Search` 插件（如 Elasticsearch、Meilisearch 等）提供专业搜索引擎能力

多实体支持：**问题(question)** 和 **回答(answer)** 两种实体的联合搜索。

### 1.2 核心模块分层

```
┌─────────────────────────────────────────────────────┐
│                     Controller                       │
│              search_controller.go                   │
├─────────────────────────────────────────────────────┤
│                     Service                          │
│  search_service.go  │  search_parser.go             │
├─────────────────────────────────────────────────────┤
│                     Repository                       │
│  search_repo.go  │  search_sync.go                  │
├─────────────────────────────────────────────────────┤
│                     Plugin Interface                 │
│  search.go (Search / SearchSyncer)                  │
├─────────────────────────────────────────────────────┤
│                     Data Layer                       │
│  data.go (DB + Cache)                               │
└─────────────────────────────────────────────────────┘
```

### 1.3 核心文件清单

| 文件 | 职责 |
|------|------|
| [plugin/search.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/plugin/search.go) | 搜索插件接口定义 |
| [internal/controller/search_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/controller/search_controller.go) | 搜索 HTTP 入口 |
| [internal/service/content/search_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/service/content/search_service.go) | 搜索业务逻辑编排 |
| [internal/service/search_parser/search_parser.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/service/search_parser/search_parser.go) | 查询语法解析器 |
| [internal/repo/search_common/search_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/search_common/search_repo.go) | 内置搜索实现（数据库查询） |
| [internal/repo/search_sync/search_sync.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/search_sync/search_sync.go) | 搜索数据同步器 |
| [internal/schema/search_schema.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/schema/search_schema.go) | 搜索数据结构定义 |

---

## 二、索引写入流程

### 2.1 增量索引更新

索引更新采用**触发式**设计，在实体增删改操作后主动更新搜索索引。

#### 2.1.1 触发点详细分类

索引更新触发点按操作语义可以分为以下几类，其中**置顶(pin)、展示(show)属性不同步**，**创建后的补写在 service 层触发**而非 repo 层。

##### 问题实体 — 创建与内容变更

| 操作 | 触发位置 | 调用时机 | 行号 | 说明 |
|------|----------|----------|------|------|
| **创建后补写** | question_service.go | 问题入库 + 标签关联完成后 | L415 | `AddQuestion()` 创建时 repo 层**不触发**，由 service 层在 `ChangeTag` 之后补调 `UpdateSearch`，确保标签已写入后再建索引 |
| 更新问题内容 | question_repo.go | 标题/正文等字段更新后 | L101 | `UpdateQuestion()` 直接触发 |
| 更新问题标签 | question_service.go | 标签增删改后（服务层多处） | L415 等 | 标签变更由 service 层统一触发 |

##### 问题实体 — 状态与计数变更

| 操作 | 函数 | 行号 | 说明 |
|------|------|------|------|
| 浏览量 +1 | `UpdatePvCount` | L105-L114 | 每次访问问题页面时触发，**存在空 ID 边界问题** |
| 回答数变更 | `UpdateAnswerCount` | L116-L126 | 回答增删后触发，**存在空 ID 边界问题** |
| 状态变更（软删除/恢复等） | `UpdateQuestionStatus` | L150-L158 | 含状态变更时触发 |
| 状态变更（不更新时间） | `UpdateQuestionStatusWithOutUpdateTime` | L160-L168 | 仅更新 status 字段也触发 |
| 恢复删除 | `RecoverQuestion` | L195-L203 | 将 status 从删除态恢复为可用 |
| **置顶(pin) / 展示(show)** | `UpdateQuestionOperation` | L205-L212 | **不触发** UpdateSearch！这两个操作属性不同步到索引 |
| 采纳答案变更 | `UpdateAccepted` | L214-L222 | `accepted_answer_id` 变更时触发，影响 `HasAccepted` 字段 |
| 最后回答 ID 变更 | `UpdateLastAnswer` | L224-L232 | 新回答或回答删除时更新 |
| 批量软删除用户所有问题 | `RemoveAllUserQuestion` | L664-L666 | 循环逐个触发 |
| 物理永久删除 | `DeletePermanentlyQuestions` | L170-L192 | **不触发**，直接 DELETE 已删除状态的数据 |

> **关键差异 1：pin/show 属性不同步**
> [UpdateQuestionOperation](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/question/question_repo.go#L205-L212) 只更新 `pin` 和 `show` 两列，但没有调用 `UpdateSearch`。这意味着搜索索引中不区分置顶和普通问题，搜索结果不会因置顶而加权。
>
> **关键差异 2：创建后补写在 service 层触发**
> 问题创建流程中，[question_repo.AddQuestion](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/question/question_repo.go#L66-L79) 本身不触发搜索更新，而是在 [question_service.AddQuestion](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/service/content/question_service.go#L415) 的 `ChangeTag` 之后补调，确保标签关联已写入。
>
> **边界问题：浏览量/回答数更新传递空对象 ID**
> `UpdatePvCount` 和 `UpdateAnswerCount` 存在隐蔽的 Bug：方法内部创建了新的空 `Question{}` 对象，DB 更新后未回填 ID，导致传给 `UpdateSearch(ctx, question.ID)` 的是**空字符串**。由于 `UpdateSearch` 内有 `!exist` 检查会静默返回，这两个高频操作实际上**不会触发任何索引更新**。详见 [2.1.3 空 ID 边界问题分析](#213-空-id-边界问题分析)。

##### 回答实体

| 操作 | 函数 | 行号 | 说明 |
|------|------|------|------|
| 创建回答 | `AddAnswer` | L83 | 创建后触发 `updateSearch` |
| 软删除回答 | `RemoveAnswer` | L96 | 软删除（status → deleted）后触发 |
| 恢复回答 | `RecoverAnswer` | L109 | 从删除态恢复后触发 |
| 批量软删除用户所有回答 | `RemoveAllUserAnswer` | L141-L143 | 循环逐个触发 |
| 更新回答内容 | `UpdateAnswer` | L155 | 字段更新后触发 |
| 回答状态变更 | `UpdateAnswerStatus` | L165 | 审核/删除等状态变更 |
| 采纳状态变更 | `UpdateAnswerAccepted` | L254 | 问题采纳/取消采纳该回答时触发 |

所有回答实体的索引更新都通过私有方法 [updateSearch](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/answer/answer_repo.go#L462) 统一调用。

#### 2.1.2 更新流程

以问题为例，核心方法 `UpdateSearch`（[question_repo.go#L584-L635](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/question/question_repo.go#L584-L635)）：

```go
func (qr *questionRepo) UpdateSearch(ctx context.Context, questionID string) (err error) {
    // 1. 检查搜索插件是否启用
    var s plugin.Search
    _ = plugin.CallSearch(func(search plugin.Search) error {
        s = search
        return nil
    })
    if s == nil {
        return  // 未启用搜索插件，直接返回
    }

    // 2. 从数据库获取完整问题数据
    question, exist, err := qr.GetQuestion(ctx, questionID)

    // 3. 获取问题关联的标签 ID 列表
    tagListList := make([]*entity.TagRel, 0)
    // ... 查询 tag_rel 表

    // 4. 组装 SearchContent 对象
    content := &plugin.SearchContent{
        ObjectID:    questionID,
        Title:       question.Title,
        Type:        constant.QuestionObjectType,
        Content:     question.OriginalText,  // 原始 Markdown 文本
        Answers:     int64(question.AnswerCount),
        Status:      plugin.SearchContentStatus(question.Status),
        Tags:        tags,                    // 标签 ID 列表
        QuestionID:  questionID,
        UserID:      question.UserID,
        Views:       int64(question.ViewCount),
        Created:     question.CreatedAt.Unix(),
        Active:      question.UpdatedAt.Unix(),
        Score:       int64(question.VoteCount),
        HasAccepted: question.AcceptedAnswerID != "",
    }

    // 5. 调用搜索插件更新索引
    err = s.UpdateContent(ctx, content)
    return
}
```

**关键设计要点：**

- **容错性**：使用 `_ = qr.UpdateSearch(...)` 忽略错误，索引更新失败不影响主流程
- **插件优先**：只有当搜索插件注册时才执行索引更新，内置搜索直接查库无需索引
- **完整数据**：每次更新都重新读取完整数据，保证索引数据一致性
- **标签展开**：问题的标签通过 `tag_rel` 关联表查询，存储标签 ID 列表

### 2.2 全量索引同步

通过 `SearchSyncer` 接口实现全量数据同步，用于搜索引擎首次部署或重建索引场景。

#### 2.2.1 同步器实现

[PluginSyncer](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/search_sync/search_sync.go#L38-L141) 实现了 `plugin.SearchSyncer` 接口：

```go
type SearchSyncer interface {
    GetAnswersPage(ctx context.Context, page, pageSize int) ([]*SearchContent, error)
    GetQuestionsPage(ctx context.Context, page, pageSize int) ([]*SearchContent, error)
}
```

#### 2.2.2 注册时机

在 [plugin_common_service.go#L101-L106](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/service/plugin_common/plugin_common_service.go#L101-L106) 中，当搜索插件配置更新时注册同步器：

```go
_ = plugin.CallSearch(func(search plugin.Search) error {
    if search.Info().SlugName == req.PluginSlugName {
        search.RegisterSyncer(ctx, search_sync.NewPluginSyncer(ps.data))
    }
    return nil
})
```

向量搜索插件则在系统启动时自动注册（[plugin_common_service.go#L187-L190](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/service/plugin_common/plugin_common_service.go#L187-L190)）。

### 2.3 索引数据结构

[SearchContent](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/plugin/search.go#L33-L48) 定义了索引文档的完整字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| ObjectID | string | 内容唯一 ID（问题 ID 或回答 ID） |
| Title | string | 标题（回答复用问题标题） |
| Type | string | 实体类型："question" / "answer" |
| Content | string | 正文内容（原始文本） |
| Answers | int64 | 回答数（问题有效） |
| Status | SearchContentStatus | 状态：1=可用, 10=已删除 |
| Tags | []string | 标签 ID 列表 |
| QuestionID | string | 关联问题 ID |
| UserID | string | 作者用户 ID |
| Views | int64 | 浏览量 |
| Created | int64 | 创建时间戳 |
| Active | int64 | 最后活跃时间戳 |
| Score | int64 | 投票得分 |
| HasAccepted | bool | 是否有采纳答案 |

> **索引字段注意**：问题的 `pin`（置顶）和 `show`（展示）两个属性**不写入搜索索引**，因为 `UpdateQuestionOperation` 方法不触发 `UpdateSearch`。这意味着搜索结果中置顶问题不会获得加权，展示属性也不会作为过滤条件。如果需要按置顶排序，必须额外回库查询 `pin` 字段再进行二次排序。

#### 2.3.1 索引字段与数据库字段映射

| SearchContent 字段 | 问题实体 (entity.Question) | 回答实体 (entity.Answer) |
|-------------------|---------------------------|--------------------------|
| ObjectID | question.ID | answer.ID |
| Title | question.Title | 关联 question.Title |
| Type | constant.QuestionObjectType ("question") | constant.AnswerObjectType ("answer") |
| Content | question.OriginalText | answer.OriginalText |
| Answers | question.AnswerCount | 0 |
| Status | question.Status (1=可用, 10=已删除) | answer.Status |
| Tags | tag_rel 查询标签 ID 列表 | (空，回答本身无标签) |
| QuestionID | question.ID | answer.QuestionID |
| UserID | question.UserID | answer.UserID |
| Views | question.ViewCount | (无，回答不统计浏览量) |
| Created | question.CreatedAt.Unix() | answer.CreatedAt.Unix() |
| Active | question.UpdatedAt.Unix() | answer.UpdatedAt.Unix() |
| Score | question.VoteCount | answer.VoteCount |
| HasAccepted | question.AcceptedAnswerID != "" | answer.Adopted |

---

## 三、查询解析流程

### 3.1 请求入口

搜索请求从 [SearchController.Search](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/controller/search_controller.go#L64-L93) 进入：

1. 参数绑定与校验（`SearchDTO`）
2. 验证码校验（非管理员用户）
3. 调用 `searchService.Search(ctx, &dto)`

### 3.2 查询预处理

在 [SearchDTO.Check()](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/schema/search_schema.go#L41-L48) 中进行预处理：

```go
func (s *SearchDTO) Check() (errField []*validator.FormErrorField, err error) {
    replacedContent, patterns := ReplaceSearchContent(s.Query)
    s.Query = strings.Join(append(patterns, replacedContent), " ")
    return nil, nil
}
```

[ReplaceSearchContent](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/schema/search_schema.go#L50-L70) 函数的作用：

1. 提取 `key:value` 模式和 `[tag]` 模式保存到 patterns
2. 将特殊字符 `+#.<>\-_()*` 替换为空格
3. 最终将 patterns 和处理后的普通文本重新拼接

**示例：**
```
输入: "user:aaa-sss [tag1] ssssfdfdf-as#fsadf [tag2] score:3"
输出: "user:aaa-sss score:3 [tag1] [tag2] ssssfdfdf as fsadf"
```

### 3.3 查询结构解析

核心解析器 [SearchParser.ParseStructure](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/service/search_parser/search_parser.go#L50-L106) 负责将查询字符串解析为结构化的 `SearchCondition`。

#### 3.3.1 解析顺序

解析器按以下顺序逐一提取查询修饰符，每匹配一个就从 query 中移除：

| 顺序 | 解析项 | 语法 | 说明 |
|------|--------|------|------|
| 1 | 标签 | `[tagname]` | 支持多个标签分组 |
| 2 | 用户 | `user:username` / `user:me` | 指定作者 |
| 3 | 投票数 | `score:N` | 最低投票数 |
| 4 | 精确短语 | `"hello world"` | 引号内作为整体 |
| 5 | 未采纳 | `hasaccepted:no` | 仅问题 |
| 6 | 浏览量 | `views:N` | 最低浏览量 |
| 7 | 回答数 | `answers:N` | 最低回答数 |
| 8 | 已采纳回答 | `isaccepted:yes` | 仅回答 |
| 9 | 指定问题内 | `inquestion:ID` | 仅回答 |
| 10 | 类型限定 | `is:question` / `is:answer` | 实体类型 |

#### 3.3.2 目标类型推断

解析器会根据使用的修饰符自动推断搜索目标类型：

- 出现 `hasaccepted:no`、`views:N`、`answers:N`、`is:question` → 搜索问题
- 出现 `isaccepted:yes`、`inquestion:ID`、`is:answer` → 搜索回答
- 都没出现 → 搜索全部

#### 3.3.3 标签解析细节

[parseTags](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/service/search_parser/search_parser.go#L109-L152) 方法：

- 匹配 `[...]` 格式的标签名
- 通过 `tagCommonService.GetTagBySlugName` 按 slug 查找标签
- 展开主标签（MainTag）和同义词标签
- 每个标签组是一个 OR 条件，多个标签组之间是 AND 关系
- 最多支持 5 个标签组

#### 3.3.4 关键词限制

最终提取的关键词最多保留 **5 个**（[search_parser.go#L102-L104](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/service/search_parser/search_parser.go#L102-L104)）。

### 3.4 SearchCondition 结构

[SearchCondition](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/schema/search_schema.go#L72-L93) 是解析后的结构化查询条件：

```go
type SearchCondition struct {
    TargetType   string     // 目标类型: ""=全部, "question", "answer"
    UserID       string     // 用户 ID
    VoteAmount   int        // 最低投票数 (-1 表示不限)
    NotAccepted  bool       // 未采纳问题
    Views        int        // 最低浏览量 (-1 表示不限)
    AnswerAmount int        // 最低回答数 (-1 表示不限)
    Accepted     bool       // 已采纳回答
    QuestionID   string     // 指定问题 ID
    Tags         [][]string // 标签 ID 组列表
    Words        []string   // 搜索关键词列表
}
```

---

## 四、搜索执行流程

### 4.1 搜索服务编排

[SearchService.Search](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/service/content/search_service.go#L47-L85) 是搜索的核心编排方法：

```go
func (ss *SearchService) Search(ctx context.Context, dto *schema.SearchDTO) (resp *schema.SearchResp, err error) {
    // 1. 解析查询结构
    cond := ss.searchParser.ParseStructure(ctx, dto)

    // 2. 检查是否有搜索插件
    var finder plugin.Search
    _ = plugin.CallSearch(func(search plugin.Search) error {
        finder = search
        return nil
    })

    // 3. 无插件 → 使用内置搜索
    if finder == nil {
        switch {
        case cond.SearchAll():
            resp.SearchResults, resp.Total, err =
                ss.searchRepo.SearchContents(ctx, cond.Words, cond.Tags, ...)
        case cond.SearchQuestion():
            resp.SearchResults, resp.Total, err =
                ss.searchRepo.SearchQuestions(ctx, cond.Words, cond.Tags, ...)
        case cond.SearchAnswer():
            resp.SearchResults, resp.Total, err =
                ss.searchRepo.SearchAnswers(ctx, cond.Words, cond.Tags, ...)
        }
        return
    }

    // 4. 有插件 → 使用插件搜索
    return ss.searchByPlugin(ctx, finder, cond, dto)
}
```

### 4.2 内置搜索实现

内置搜索基于数据库 `LIKE` 查询实现，核心在 [search_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/search_common/search_repo.go)。

#### 4.2.1 联合搜索（SearchContents）

[SearchContents](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/search_common/search_repo.go#L102-L242) 同时搜索问题和回答，使用 `UNION ALL` 合并结果：

```go
// 问题查询
b = builder.MySQL().Select(qfs...).From("`question`")
// 回答查询
ub = builder.MySQL().Select(afs...).From("`answer`").
    LeftJoin("`question`", "`question`.id = `answer`.question_id")

// 合并
sql := fmt.Sprintf("(%s UNION ALL %s)", bSQL, ubSQL)
```

**查询字段定义：**

- `qFields`（问题字段）：id, title, parsed_text, created_at, user_id, vote_count, answer_count, accepted, status, post_update_time
- `aFields`（回答字段）：id, question_id, title(问题标题), parsed_text(回答内容), created_at, user_id, vote_count, answer_count=0, adopted, status, post_update_time

#### 4.2.2 过滤条件

内置搜索支持的过滤条件：

| 条件 | 问题搜索 | 回答搜索 | 联合搜索 |
|------|----------|----------|----------|
| 关键词 LIKE | ✓ (title + original_text) | ✓ (original_text) | ✓ |
| 标签过滤 | ✓ (INNER JOIN tag_rel) | ✓ (INNER JOIN tag_rel) | ✓ |
| 用户过滤 | ✓ | ✓ | ✓ |
| 投票数过滤 | ✓ | ✓ | ✓ |
| 未采纳过滤 | ✓ | - | - |
| 浏览量过滤 | ✓ | - | - |
| 回答数过滤 | ✓ | - | - |
| 采纳回答过滤 | - | ✓ | - |
| 指定问题 ID | - | ✓ | - |
| 状态过滤 | ✓ (未删除+已展示) | ✓ (未删除+已展示) | ✓ |

#### 4.2.3 标签过滤实现

多组标签使用多次 INNER JOIN 实现 AND 关系：

```go
for ti, tagID := range tagIDs {
    ast := "tag_rel" + strconv.Itoa(ti)
    b.Join("INNER", "tag_rel as "+ast, "question.id = "+ast+".object_id").
        And(builder.Eq{ast + ".status": entity.TagRelStatusAvailable}).
        And(builder.In(ast+".tag_id", tagID))
}
```

每组标签内部是 OR 关系（`In(tag_id, tagID...)`），组之间是 AND 关系（多次 INNER JOIN）。

### 4.3 插件搜索流程

当有搜索插件时，走插件搜索路径 [searchByPlugin](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/service/content/search_service.go#L87-L104)：

```go
func (ss *SearchService) searchByPlugin(ctx context.Context, finder plugin.Search,
    cond *schema.SearchCondition, dto *schema.SearchDTO) (resp *schema.SearchResp, err error) {

    var res []plugin.SearchResult
    // 根据类型调用不同的插件方法
    switch {
    case cond.SearchAll():
        res, resp.Total, err = finder.SearchContents(ctx, cond.Convert2PluginSearchCond(...))
    case cond.SearchQuestion():
        res, resp.Total, err = finder.SearchQuestions(ctx, cond.Convert2PluginSearchCond(...))
    case cond.SearchAnswer():
        res, resp.Total, err = finder.SearchAnswers(ctx, cond.Convert2PluginSearchCond(...))
    }

    // 将插件返回的 ID 列表转换为完整数据
    resp.SearchResults, err = ss.searchRepo.ParseSearchPluginResult(ctx, res, cond.Words)
    return resp, err
}
```

[ParseSearchPluginResult](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/search_common/search_repo.go#L467-L491) 根据插件返回的 ID 回库查询完整数据，保证与数据库状态一致。

---

## 五、排序加权机制

### 5.1 排序模式

系统支持四种排序方式，在 [parseOrder](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/search_common/search_repo.go#L450-L464) 中定义：

| 排序方式 | 排序字段 | 说明 |
|----------|----------|------|
| newest | created_at DESC | 按创建时间倒序 |
| active | post_update_time DESC | 按最后活跃时间倒序 |
| score | vote_count DESC | 按投票得分倒序 |
| relevance | relevance DESC | 按相关性倒序（默认） |

### 5.2 相关性排序算法

当排序方式为 `relevance` 且有关键词时，通过 [addRelevanceField](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/search_common/search_repo.go#L578-L609) 函数动态计算相关性分数。

#### 5.2.1 算法原理

基于**字符替换法**计算词频相关性：

```
relevance = Σ (LENGTH(field) - LENGTH(REPLACE(field, word, '')))
```

**原理说明：**
- 原始字段长度减去替换关键词后的长度 = 关键词出现的总字符数
- 关键词出现次数越多、越长，差值越大，相关性越高
- 多个字段的相关性相加得到总分

#### 5.2.2 实现代码

```go
func addRelevanceField(searchFields, words, fields []string) (res []string, args []any) {
    relevanceRes := []string{}

    for _, searchField := range searchFields {
        // 对每个字段，逐层嵌套 REPLACE 替换所有关键词
        var replaced string
        for i, word := range words {
            if i == 0 {
                replaced = fmt.Sprintf("REPLACE(%s, ?, '')", searchField)
            } else {
                replaced = fmt.Sprintf("REPLACE(%s, ?, '')", replaced)
            }
            args = append(args, word)
        }
        // 计算长度差
        relevance := fmt.Sprintf("(LENGTH(%s) - LENGTH(%s))", searchField, replaced)
        relevanceRes = append(relevanceRes, relevance)
    }

    // 多个字段相加
    res = append(fields, "("+strings.Join(relevanceRes, " + ")+") as relevance")
    return
}
```

#### 5.2.3 字段权重

- **问题搜索**：`title` + `original_text` 两个字段参与计算
- **回答搜索**：仅 ``answer`.original_text`` 一个字段参与计算

> 注意：这是一种简化的相关性算法，不涉及 TF-IDF 或 BM25 等高级排序模型。专业搜索由插件提供。

### 5.3 结果处理

#### 5.3.1 搜索结果解析

[parseResult](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/repo/search_common/search_repo.go#L494-L576) 将数据库原始结果转换为前端所需格式：

1. 提取 question_id 和 user_id 列表
2. 转换短 ID（如果启用）
3. 生成 URL 标题（[UrlTitle](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/pkg/htmltext/htmltext.go#L67-L77)）
4. 生成搜索摘要（[FetchMatchedExcerpt](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/pkg/htmltext/htmltext.go#L174-L190)）
5. 批量获取标签信息（`BatchGetObjectTag`）
6. 批量获取用户信息（`BatchUserBasicInfoByID`）

#### 5.3.2 搜索摘要生成

[FetchMatchedExcerpt](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/pkg/htmltext/htmltext.go#L174-L190) 生成高亮摘要：

1. 找到第一个匹配关键词的位置
2. 以匹配位置为中心，前后各取指定长度（默认 100 字符）
3. 超出部分用 `...` 标记

---

## 六、缓存协作机制

### 6.1 缓存架构

缓存通过 [plugin.Cache](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/plugin/cache.go) 插件接口提供，支持两种模式：

- **内存缓存**：默认实现，基于 `memory.NewCache()`
- **插件缓存**：如 Redis 缓存插件，通过 `plugin.CallCache` 注册

### 6.2 缓存初始化

在 [data.go](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/base/data/data.go#L98-L138) 的 `NewCache` 函数中初始化：

```go
func NewCache(c *CacheConf) (cache.Cache, func(), error) {
    // 优先使用插件缓存
    var pluginCache plugin.Cache
    _ = plugin.CallCache(func(fn plugin.Cache) error {
        pluginCache = fn
        return nil
    })
    if pluginCache != nil {
        return pluginCache, func() {}, nil
    }

    // 降级为内存缓存
    memCache := memory.NewCache()
    // ... 持久化配置
    return memCache, cleanup, nil
}
```

### 6.3 缓存与搜索的协作

搜索系统通过 `data.Cache` 间接使用缓存，主要场景：

1. **标签信息缓存**：`tagCommon.BatchGetObjectTag` 可能缓存标签数据
2. **用户信息缓存**：`userCommon.BatchUserBasicInfoByID` 批量获取用户时使用缓存
3. **搜索结果无直接缓存**：搜索请求本身不做结果缓存，每次实时查询

> 注意：搜索结果的缓存策略由搜索插件自行实现，核心系统不提供搜索结果缓存。

### 6.4 插件配置缓存

在 [plugin_common_service.go#L180-L183](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/internal/service/plugin_common/plugin_common_service.go#L180-L183) 中，当缓存插件启用时会替换 `data.Cache`：

```go
_ = plugin.CallCache(func(cache plugin.Cache) error {
    ps.data.Cache = cache
    return nil
})
```

这意味着缓存插件可以热替换整个系统的缓存层。

---

## 七、向量搜索扩展

除了传统全文搜索，系统还提供 [VectorSearch](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/plugin/vector_search.go) 插件接口用于语义搜索。

### 7.1 向量搜索接口

```go
type VectorSearch interface {
    Base
    Description() VectorSearchDesc
    RegisterSyncer(ctx context.Context, syncer VectorSearchSyncer)
    SearchSimilar(ctx context.Context, query string, topK int) ([]VectorSearchResult, error)
    UpdateContent(ctx context.Context, content *VectorSearchContent) error
    DeleteContent(ctx context.Context, objectID string) error
}
```

### 7.2 向量数据结构

[VectorSearchContent](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/plugin/vector_search.go#L44-L55) 相比全文搜索增加了：

- 聚合内容（问题+回答+评论）用于生成 embedding
- Metadata 存储结构化元数据（问题 ID、回答 ID、评论 ID 等）

### 7.3 内置 Embedding 工具

插件包提供了 [GenerateEmbedding](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/plugin/vector_search.go#L140-L174) 工具函数，支持 OpenAI 兼容的 embedding API。

---

## 八、关键设计模式与特性

### 8.1 插件模式

使用 [MakePlugin](file:///d:/fz/0601-2/solo-dogfeeding/code/35-answer/plugin/plugin.go#L147) 泛型函数创建插件栈：

- 注册函数（register）：供插件 init() 调用
- 调用函数（CallXxx）：供业务代码遍历调用所有插件
- 支持启用/禁用状态管理

### 8.2 容错设计

- 索引更新失败不影响主流程（`_ = UpdateSearch(...)`）
- 搜索插件不可用时降级为内置搜索
- 缓存插件不可用时降级为内存缓存

### 8.3 同步与异步

- 增量索引更新：**同步**调用，在数据库事务外执行
- 全量索引同步：插件自行决定异步调度
- 缓存持久化：**异步**定时保存（每分钟一次）

### 8.4 数据一致性

- 每次索引更新都从数据库重新读取完整数据
- 插件搜索结果需要回库校验状态（`ParseSearchPluginResult`）
- 软删除数据通过 status 字段过滤

---

## 九、代码调用链总结

### 9.1 索引写入调用链

```
question_service.UpdateQuestion
  → questionRepo.UpdateQuestion
    → questionRepo.UpdateSearch
      → plugin.CallSearch (检查插件)
        → searchPlugin.UpdateContent
```

### 9.2 搜索查询调用链

```
search_controller.Search
  → searchService.Search
    → searchParser.ParseStructure
      → parseTags → tagCommonService.GetTagBySlugName
      → parseUserID → userCommon.GetUserBasicInfoByUserName
      → parseVotes / parseViews / parseAnswers ...
    → 判断是否有搜索插件
    ├─ 有插件 → plugin.Search.SearchContents
    │           → searchRepo.ParseSearchPluginResult
    │             → tagCommon.BatchGetObjectTag
    │             → userCommon.BatchUserBasicInfoByID
    └─ 无插件 → searchRepo.SearchContents (UNION ALL 查询)
                → parseResult
                  → htmltext.FetchMatchedExcerpt
                  → tagCommon.BatchGetObjectTag
                  → userCommon.BatchUserBasicInfoByID
```

### 9.3 全量同步调用链

```
plugin_common_service.UpdatePluginConfig
  → searchPlugin.RegisterSyncer(syncer)
    ↓ (插件内部异步调用)
  → searchSyncer.GetQuestionsPage / GetAnswersPage
    → 数据库分页查询
    → 转换为 SearchContent
```
