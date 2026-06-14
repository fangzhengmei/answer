# 提问发布与问题实体流转 — 端到端路径分析（修正版）

## 一、问题创建链路全景

### 1.1 前端入口 → API 调用

**提问页面**: [ui/src/pages/Questions/Ask/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/pages/Questions/Ask/index.tsx)

- 表单字段：`title` / `content` / `tags` / `answer_content`（自问自答） / `edit_summary`（编辑摘要）
- 提交分流：
  - 普通提问：`saveQuestion(params)` → `POST /answer/api/v1/question`
  - 自问自答：`saveQuestionWithAnswer(params)` → `POST /answer/api/v1/question/answer`
- 成功后跳转详情页 `pathFactory.questionLanding(id, url_title)`

**API 定义**: [ui/src/services/common.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/services/common.ts#L191-L336)

### 1.2 路由注册

**文件**: [internal/router/answer_api_router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/router/answer_api_router.go#L269-L277)

```go
r.POST("/question",       a.questionController.AddQuestion)          // 普通提问
r.POST("/question/answer", a.questionController.AddQuestionByAnswer)  // 自问自答
```

---

## 二、Controller 层：两条路径的差异

### 2.1 AddQuestion（普通提问）

**文件**: [internal/controller/question_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/controller/question_controller.go#L384-L487)

```go
canList, requireRanks, err := qc.rankService.CheckOperationPermissionsForRanks(ctx, req.UserID, []string{
    permission.QuestionAdd,       // [0] canAdd
    permission.QuestionEdit,      // [1] canEdit
    permission.QuestionDelete,    // [2] canDelete
    permission.QuestionClose,     // [3] canClose
    permission.QuestionReopen,    // [4] canReopen
    permission.TagUseReservedTag, // [5] canUseReservedTag
    permission.TagAdd,            // [6] canAddTag
    permission.LinkUrlLimit,      // [7] linkUrlLimitUser
})
```

**实传 8 项权限**，逐一赋值到 `req.QuestionPermission`：

| 索引 | 权限 | 赋值目标 |
|------|------|---------|
| 0 | QuestionAdd | req.CanAdd |
| 1 | QuestionEdit | req.CanEdit |
| 2 | QuestionDelete | req.CanDelete |
| 3 | QuestionClose | req.CanClose |
| 4 | QuestionReopen | req.CanReopen |
| 5 | TagUseReservedTag | req.CanUseReservedTag |
| 6 | TagAdd | req.CanAddTag |
| 7 | LinkUrlLimit | linkUrlLimitUser（本地变量，不进 req） |

**HasNewTag 二次拦截**（第 443-453 行）：

```go
hasNewTag, err := qc.questionService.HasNewTag(ctx, req.Tags)
if !req.CanAddTag && hasNewTag {
    // 用户无 TagAdd 权限但提交了新标签 → 403 Forbidden
    handler.HandleResponse(ctx, errors.Forbidden(reason.NoEnoughRankToOperate).WithMsg(msg), nil)
    return
}
```

这是一道**独立的二次拦截**：即使通过了 `CheckAddQuestion`，仍在此处先判断「用户能否创建新标签」；若 `CanAddTag=false` 且 `Tags` 中含有数据库不存在的标签名，直接 403 返回，不再进入 Service 层。

### 2.2 AddQuestionByAnswer（自问自答）

**文件**: [internal/controller/question_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/controller/question_controller.go#L499-L611)

```go
canList, err := qc.rankService.CheckOperationPermissions(ctx, req.UserID, []string{
    permission.QuestionAdd,       // [0]
    permission.QuestionEdit,      // [1]
    permission.QuestionDelete,    // [2]
    permission.QuestionClose,     // [3]
    permission.QuestionReopen,    // [4]
    permission.TagUseReservedTag, // [5]
    permission.LinkUrlLimit,      // [6]
})
```

**仅传 7 项权限**，缺少 `permission.TagAdd`：

| 索引 | 权限 | 赋值目标 |
|------|------|---------|
| 0 | QuestionAdd | req.CanAdd |
| 1 | QuestionEdit | req.CanEdit |
| 2 | QuestionDelete | req.CanDelete |
| 3 | QuestionClose | req.CanClose |
| 4 | QuestionReopen | req.CanReopen |
| 5 | TagUseReservedTag | req.CanUseReservedTag |
| — | ~~TagAdd~~ | **未查询，不存在** |
| 6 | LinkUrlLimit | linkUrlLimitUser |

**关键差异**：

1. 调用 `CheckOperationPermissions` 而非 `CheckOperationPermissionsForRanks`（后者额外返回 `requireRanks` 用于错误提示模板）
2. **没有 HasNewTag 二次拦截**——自问自答路径允许用户提交新标签名，即使没有 `TagAdd` 权限
3. 通过 `copier.Copy(questionReq, req)` 将 7 项权限的 `QuestionAddByAnswer` 复制为 `QuestionAdd`，然后调用同一个 `questionService.AddQuestion`

之后额外调用 `answerService.Insert` 创建回答，返回 `{"info": ..., "question": ...}` 复合结构。

---

## 三、唯一 ID 生成：uniqid 表自增，非雪花算法

**Entity**: [internal/entity/uniqid_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/entity/uniqid_entity.go)

```go
type Uniqid struct {
    ID         int64 `xorm:"not null pk autoincr BIGINT(20) id"`
    UniqidType int   `xorm:"not null default 0 INT(11) uniqid_type"`
}
```

**Repository**: [internal/repo/unique/uniqid_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/repo/unique/uniqid_repo.go#L47-L56)

```go
func (ur *uniqueIDRepo) GenUniqueIDStr(ctx context.Context, key string) (uniqueID string, err error) {
    objectType := constant.ObjectTypeStrMapping[key]       // "question" → 1
    bean := &entity.Uniqid{UniqidType: objectType}
    _, err = ur.data.DB.Context(ctx).Insert(bean)          // INSERT 得到自增 ID
    return fmt.Sprintf("1%03d%013d", objectType, bean.ID), nil
}
```

**机制解析**：
- 向 `uniqid` 表执行 INSERT，数据库 `autoincr` 自增得到 `bean.ID`
- 拼接格式：`1` + `三位对象类型` + `十三位自增ID`，如 `10010000000000042`
- 对象类型映射定义在 [internal/base/constant/object_type.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/constant/object_type.go#L34-L45)：`question=1, answer=2, tag=3, user=4, collection=6, comment=7`
- **不是雪花算法**，是单表自增 + 类型前缀拼接
- 外部可启用短 ID（Base62 编码），`uid.EnShortID` / `uid.DeShortID` 做长短互转

---

## 四、Review：委托插件审核，含直删档

**Service**: [internal/service/review/review_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/review/review_service.go#L112-L138)

```go
func (cs *ReviewService) AddQuestionReview(ctx context.Context,
    question *entity.Question, tags []*schema.TagItem, ip, ua string) (questionStatus int) {
    reviewContent := &plugin.ReviewContent{
        ObjectType: constant.QuestionObjectType,
        Title:      question.Title,
        Content:    question.ParsedText,
        IP:         ip,
        UserAgent:  ua,
    }
    for _, tag := range tags {
        reviewContent.Tags = append(reviewContent.Tags, tag.SlugName)
    }
    reviewContent.Author = cs.getReviewContentAuthorInfo(ctx, question.UserID)
    reviewStatus := cs.callPluginToReview(ctx, question.UserID, question.ID, reviewContent)
    switch reviewStatus {
    case plugin.ReviewStatusApproved:       // → Available
        questionStatus = entity.QuestionStatusAvailable
    case plugin.ReviewStatusNeedReview:     // → Pending
        questionStatus = entity.QuestionStatusPending
    case plugin.ReviewStatusDeleteDirectly: // → Deleted（直删档）
        questionStatus = entity.QuestionStatusDeleted
    default:
        questionStatus = entity.QuestionStatusAvailable
    }
    return questionStatus
}
```

**插件接口**: [plugin/reviewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/plugin/reviewer.go#L22-L80)

```go
type Reviewer interface {
    Base
    Review(content *ReviewContent) (result *ReviewResult)
}

type ReviewStatus string
const (
    ReviewStatusApproved       ReviewStatus = "approved"         // 放行
    ReviewStatusDeleteDirectly ReviewStatus = "delete_directly"  // 直删档
    ReviewStatusNeedReview     ReviewStatus = "need_review"      // 待人工审核
)

type ReviewResult struct {
    Approved     bool
    ReviewStatus ReviewStatus
    Reason       string
}
```

**审核流程详解**：

1. `callPluginToReview` 遍历所有注册的 `Reviewer` 插件
2. 默认状态为 `ReviewStatusApproved`（放行）
3. **短路逻辑**：一旦某个插件返回 `Approved=false`，后续插件不再执行（`if reviewStatus != plugin.ReviewStatusApproved { return nil }`）
4. 若插件返回 `ReviewStatusNeedReview`，会创建一条 `review` 记录入库，等待管理员审核
5. 若插件返回 `ReviewStatusDeleteDirectly`（**直删档**），问题状态直接设为 `QuestionStatusDeleted`

**在 AddQuestion 中的调用时序**：

```
question.Status = QuestionStatusPending  // 默认待审核
qs.questionRepo.AddQuestion(ctx, question)  // 先入库（Pending）

question.Status = qs.reviewService.AddQuestionReview(...)  // 委托插件判定
qs.questionRepo.UpdateQuestionStatus(ctx, question.ID, question.Status)  // 更新状态
```

问题先以 Pending 状态入库，再由插件审核决定最终状态。若插件判定直删档，则更新为 Deleted。

---

## 五、队列机制：进程内 Go Channel，非外部消息中间件

**底层实现**: [internal/base/queue/queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/queue/queue.go#L42-L130)

```go
type Queue[T any] struct {
    name    string
    queue   chan T               // Go channel
    handler func(ctx context.Context, msg T) error
    mu      sync.RWMutex
    closed  bool
    wg      sync.WaitGroup
}

func New[T any](name string, bufferSize int) *Queue[T] {
    q := &Queue[T]{
        name:  name,
        queue: make(chan T, bufferSize),  // 缓冲通道
    }
    q.startWorker()  // 启动单 goroutine 消费者
    return q
}
```

**四大队列**，均为 `queue.New[T](name, 128)` 创建的**进程内带缓冲 Go Channel**：

| 队列 | 类型 | 定义 | buffer |
|------|------|------|--------|
| Activity Queue | `Service[*schema.ActivityMsg]` | [activity_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/activityqueue/activity_queue.go#L29-L31) | 128 |
| Notification Queue | `Service[*schema.NotificationMsg]` | [notice_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/noticequeue/notice_queue.go#L27-L31) | 128 |
| External Notification Queue | `ExternalService[*schema.ExternalNotificationMsg]` | [notice_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/noticequeue/notice_queue.go#L33-L37) | 128 |
| Event Queue | `Service[*schema.EventMsg]` | [event_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/eventqueue/event_queue.go#L27-L31) | 128 |
| Vector Sync Queue | `Service[*vector_sync.Task]` | [vector_sync.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/vector_sync/vector_sync.go#L51-L57) | 128 |

**机制要点**：
- `Send()` 非阻塞写入 channel（队列满时阻塞或丢弃）
- `startWorker()` 启动**单个 goroutine** 消费者，顺序处理消息
- `Close()` 优雅关闭，等待 `WaitGroup` 处理完毕
- **无外部消息中间件**（非 Kafka/RabbitMQ），纯进程内异步

---

## 六、UpdateSearch 与 VectorSync：受插件控制

### 6.1 UpdateSearch：受 Search 插件控制

**文件**: [internal/repo/question/question_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/repo/question/question_repo.go#L584-L635)

```go
func (qr *questionRepo) UpdateSearch(ctx context.Context, questionID string) (err error) {
    var s plugin.Search
    _ = plugin.CallSearch(func(search plugin.Search) error {
        s = search
        return nil
    })
    if s == nil {
        return  // 没有注册 Search 插件 → 直接返回，什么都不做
    }
    // ... 构建 SearchContent，调用 s.UpdateContent(ctx, content)
}
```

**逻辑**：通过 `plugin.CallSearch` 查找是否注册了 `Search` 插件。若无插件则 `s == nil`，**静默返回**，不做任何搜索索引更新。

### 6.2 VectorSync：受 VectorSearch 插件控制

**文件**: [internal/service/vector_sync/vector_sync.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/vector_sync/vector_sync.go#L59-L85)

```go
func handle(ctx context.Context, data *data.Data, msg *Task) error {
    var vectorSearch plugin.VectorSearch
    _ = plugin.CallVectorSearch(func(vs plugin.VectorSearch) error {
        vectorSearch = vs
        return nil
    })
    if vectorSearch == nil {
        return nil  // 没有注册 VectorSearch 插件 → 静默返回
    }
    // ... 最多重试 3 次，调用 vectorSearch.UpdateContent / DeleteContent
}
```

**逻辑**：通过 `plugin.CallVectorSearch` 查找 `VectorSearch` 插件。无插件时静默跳过。有插件时最多重试 3 次。

**在 AddQuestion 中的调用链**：

```
AddQuestion
  ├─ qs.questionRepo.UpdateSearch(ctx, question.ID)
  │    └─ 检查 Search 插件 → 有则同步更新搜索索引
  └─ qs.vectorSyncService.Send(ctx, &vector_sync.Task{Action: ActionUpsert, ...})
       └─ 异步 Channel → handle() → 检查 VectorSearch 插件 → 有则更新向量索引
```

两者均受插件注册控制，**无插件时为空操作**。

---

## 七、UpdateQuestionLink：维护 question_link 表与 linked_count

**文件**: [internal/service/question_common/question.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/question_common/question.go#L703-L828)

```go
func (qs *QuestionCommon) UpdateQuestionLink(ctx context.Context,
    questionID, answerID, parsedText, originalText string) (string, error) {

    // 1. 删除当前问题已有的出站链接
    err := qs.questionRepo.RemoveQuestionLink(ctx, &entity.QuestionLink{
        FromQuestionID: uid.DeShortID(questionID),
        FromAnswerID:   uid.DeShortID(answerID),
    })

    // 2. 对被删链接的目标问题，重新计算 linked_count
    linkedQuestionIDs, _ := qs.questionRepo.GetLinkedQuestionIDs(ctx,
        uid.DeShortID(questionID), entity.QuestionLinkStatusDeleted)
    for _, id := range linkedQuestionIDs {
        qs.questionRepo.UpdateQuestionLinkCount(ctx, id)
    }

    // 3. 从 Markdown 原文提取 #questionID / #answerID 引用链接
    links := checker.GetQuestionLink(originalText)
    if len(links) == 0 {
        return parsedText, nil
    }

    // 4. 批量验证链接目标是否存在（查 question/answer 表）
    // ... 构建 answerCache / questionCache

    // 5. 生成新的 QuestionLink 记录，替换 parsedText 中的引用为 HTML 超链接
    for _, link := range links {
        newLink := &entity.QuestionLink{
            FromQuestionID: uid.DeShortID(questionID),
            FromAnswerID:   uid.DeShortID(answerID),
            ToQuestionID:   uid.DeShortID(link.QuestionID),
            ToAnswerID:     uid.DeShortID(link.AnswerID),
        }
        // 将 #123 替换为 <a href="/questions/123">#123</a>
        parsedText = strings.ReplaceAll(parsedText, "#"+link.QuestionID, htmlLink)
        // 排除自引用
        if newLink.FromQuestionID != newLink.ToQuestionID {
            validLinks = append(validLinks, newLink)
        }
    }

    // 6. 批量写入 question_link 表
    qs.questionRepo.LinkQuestion(ctx, validLinks...)

    // 7. 更新目标问题的 linked_count
    for _, link := range validLinks {
        qs.questionRepo.UpdateQuestionLinkCount(ctx, link.ToQuestionID)
    }

    return parsedText, nil
}
```

**数据流**：

| 步骤 | 操作 | 影响表 |
|------|------|--------|
| 删除旧链接 | `RemoveQuestionLink` | `question_link` |
| 回收计数 | `UpdateQuestionLinkCount` 对已删除目标 | `question.linked_count` |
| 提取引用 | `checker.GetQuestionLink(originalText)` | 无（纯解析） |
| 验证目标 | 查 `question` / `answer` 表确认存在 | `question`, `answer` |
| 写入新链接 | `LinkQuestion` | `question_link` |
| 更新计数 | `UpdateQuestionLinkCount` 对新目标 | `question.linked_count` |
| 替换文本 | `#123` → `<a href="/questions/123">#123</a>` | 返回值（后续写入 `question.parsed_text`） |

**在 AddQuestion 中的调用**（仅当 `Status == Available` 时）：

```go
if question.Status == entity.QuestionStatusAvailable {
    question.ParsedText, _ = qs.questioncommon.UpdateQuestionLink(
        ctx, question.ID, "", question.ParsedText, question.OriginalText)
    qs.questionRepo.UpdateQuestion(ctx, question, []string{"parsed_text"})
}
```

---

## 八、Service 层 AddQuestion 完整步骤（修正）

**文件**: [internal/service/content/question_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/content/question_service.go#L307-L469)

```
1.  前置校验（标签数 / 内容长度 / 推荐标签 / 保留标签）
2.  标签 SlugName 处理（空格→横线） + 查询标签实体
3.  构建 Question Entity，Status=Pending
4.  Repository.AddQuestion → INSERT question 表
    └─ GenUniqueIDStr → INSERT uniqid 表（autoincr）→ 拼接 "1"+type+id
5.  reviewService.AddQuestionReview → 委托插件审核
    ├─ 插件返回 Approved → Available
    ├─ 插件返回 NeedReview → Pending + 创建 review 记录
    └─ 插件返回 DeleteDirectly → Deleted（直删档）
6.  UpdateQuestionStatus → 更新 question.status
7.  若 Available：
    ├─ UpdateQuestionLink → 维护 question_link + linked_count + 替换 parsedText
    └─ UpdateQuestion(parsed_text)
8.  ChangeTag → 创建 tag_rel 关联
9.  UpdateSearch → 检查 Search 插件 → 有则同步搜索索引
10. AddRevision → 创建版本记录
11. 更新用户提问计数
12. Activity Queue.Send → 进程内 Channel
13. 若 Available → External Notification Queue.Send → 进程内 Channel
14. Event Queue.Send → 进程内 Channel
15. 若 Available → Vector Sync Queue.Send → 进程内 Channel
    └─ 异步 → handle() → 检查 VectorSearch 插件 → 有则更新
16. GetQuestion → 返回问题详情
```

---

## 九、端到端流转图

### 9.1 问题创建（AddQuestion 路径）

```
[前端 Ask 页面]
  │  saveQuestion(params)
  ▼
POST /answer/api/v1/question
  │
  ▼
[QuestionController.AddQuestion]
  ├─ BindAndCheckReturnErr → 参数绑定 + validate 校验
  ├─ DuplicateRequestRejection → 防重复提交
  ├─ CheckOperationPermissionsForRanks → 8 项权限
  │   [QuestionAdd, QuestionEdit, QuestionDelete,
  │    QuestionClose, QuestionReopen, TagUseReservedTag,
  │    TagAdd, LinkUrlLimit]
  ├─ ActionRecordVerifyCaptcha → 验证码（非管理员/非链接豁免）
  ├─ HasNewTag 二次拦截 → CanAddTag=false 且有新标签 → 403
  ├─ CheckAddQuestion → 业务校验
  └─ Service.AddQuestion
       │
       ▼
[QuestionService.AddQuestion]
  ├─ 前置校验（标签数/内容长度/推荐标签/保留标签）
  ├─ 构建 Entity (Status=Pending)
  ├─ Repo.AddQuestion
  │    └─ GenUniqueIDStr → INSERT uniqid(autoincr) → 拼接 "1"+type+id
  ├─ ReviewService.AddQuestionReview
  │    └─ plugin.CallReviewer → 插件返回 approved/need_review/delete_directly
  │         ├─ Approved → Available
  │         ├─ NeedReview → Pending + review 记录入库
  │         └─ DeleteDirectly → Deleted（直删档）
  ├─ Repo.UpdateQuestionStatus
  ├─ 若 Available:
  │    ├─ UpdateQuestionLink → question_link 表 + linked_count
  │    └─ Repo.UpdateQuestion(parsed_text)
  ├─ ChangeTag → tag_rel 关联
  ├─ Repo.UpdateSearch → 检查 Search 插件（无插件则跳过）
  ├─ RevisionService.AddRevision
  ├─ 更新用户提问计数
  ├─ ActivityQueue.Send → chan *ActivityMsg (buffer=128)
  ├─ 若 Available → ExternalNotificationQueue.Send → chan *ExternalNotificationMsg
  ├─ EventQueue.Send → chan *EventMsg
  ├─ 若 Available → VectorSyncQueue.Send
  │    └─ 异步 handle → 检查 VectorSearch 插件（无插件则跳过）
  └─ GetQuestion → 返回详情
```

### 9.2 问题创建（AddQuestionByAnswer 路径）

```
[前端 Ask 页面] (勾选"同时回答")
  │  saveQuestionWithAnswer(params)
  ▼
POST /answer/api/v1/question/answer
  │
  ▼
[QuestionController.AddQuestionByAnswer]
  ├─ CheckOperationPermissions → 仅 7 项权限（无 TagAdd）
  │   [QuestionAdd, QuestionEdit, QuestionDelete,
  │    QuestionClose, QuestionReopen, TagUseReservedTag,
  │    LinkUrlLimit]
  ├─ 无 HasNewTag 二次拦截
  ├─ copier.Copy → QuestionAdd
  ├─ CheckAddQuestion → 业务校验
  ├─ Service.AddQuestion → 同上完整流程
  └─ AnswerService.Insert → 额外创建回答
       └─ 返回 {info, question} 复合结构
```

### 9.3 问题展示

```
[前端列表页] useQuestionList → SWR → GET /answer/api/v1/question/page
[前端详情页] questionDetail  →     GET /answer/api/v1/question/info
  │
  ▼
[QuestionController]
  │
  ▼
[QuestionService.GetQuestionPage / GetQuestion]
  ├─ 权限判断（可见性、隐藏/删除状态）
  ├─ 条件构建（标签同义词、用户、排序、时间范围）
  ├─ Repo.GetQuestionPage / GetQuestion → XORM 查询
  ├─ FormatQuestionsPage / Info → 补充用户/标签/操作权限
  └─ 返回 QuestionPageResp / QuestionInfoResp
```

---

## 十、核心文件索引

| 层级 | 文件 | 核心职责 |
|------|------|---------|
| 前端页面 | [Ask/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/pages/Questions/Ask/index.tsx) | 提问表单 |
| 前端页面 | [Questions/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/pages/Questions/index.tsx) | 问题列表 |
| 前端页面 | [Detail/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/pages/Questions/Detail/index.tsx) | 问题详情 |
| 前端 API | [services/common.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/services/common.ts) | API 定义 |
| 前端 Hook | [client/question.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/services/client/question.ts) | SWR Hooks |
| 路由 | [answer_api_router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/router/answer_api_router.go) | 路由注册 |
| Controller | [question_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/controller/question_controller.go) | HTTP 接口 |
| Service | [question_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/content/question_service.go) | 业务逻辑 |
| Service | [question.go (common)](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/question_common/question.go) | UpdateQuestionLink 等 |
| Review | [review_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/review/review_service.go) | 插件审核（含直删档） |
| Plugin | [reviewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/plugin/reviewer.go) | Reviewer 插件接口 |
| Repository | [question_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/repo/question/question_repo.go) | 数据访问 |
| Repository | [uniqid_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/repo/unique/uniqid_repo.go) | ID 生成（自增） |
| Entity | [question_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/entity/question_entity.go) | 问题实体 |
| Entity | [uniqid_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/entity/uniqid_entity.go) | 自增 ID 实体 |
| Schema | [question_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/schema/question_schema.go) | 请求/响应 DTO |
| 队列 | [queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/queue/queue.go) | 进程内 Channel 队列 |
| 队列 | [activity_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/activityqueue/activity_queue.go) | Activity 队列 |
| 队列 | [notice_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/noticequeue/notice_queue.go) | Notification 队列 |
| 队列 | [event_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/eventqueue/event_queue.go) | Event 队列 |
| 向量同步 | [vector_sync.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/vector_sync/vector_sync.go) | VectorSearch 插件控制 |
| 搜索同步 | [search_sync.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/repo/search_sync/search_sync.go) | Search 插件控制 |
| 常量 | [object_type.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/constant/object_type.go) | 对象类型映射 |
| 校验器 | [validator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/validator/validator.go) | 参数校验 + Checker 接口 |
| 限流 | [rate_limit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/middleware/rate_limit.go) | 防重复提交 |
| 限流存储 | [limit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/repo/limit/limit.go) | Cache 存储限流键 |
| 版本 | [revision_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/repo/revision/revision_repo.go) | Revision 事务写入 |
| 标签 | [tag_common.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/tag_common/tag_common.go) | 标签双写 + rel 管理 |
| 版本实体 | [revision_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/entity/revision_entity.go) | Revision 实体 |

---

## 十一、QuestionAdd.Check：在 validator 链路隐式写 req.HTML

**文件**: [internal/schema/question_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/schema/question_schema.go#L78-L104)

```go
type QuestionAdd struct {
    Title   string     `validate:"required,notblank,gte=6,lte=150" json:"title"`
    Content string     `validate:"gte=0,lte=65535" json:"content"`
    HTML    string     `json:"-"`          // 注意：json:"-" 不参与反序列化
    Tags    []*TagItem `validate:"dive" json:"tags"`
    // ...
}

func (req *QuestionAdd) Check() (errFields []*validator.FormErrorField, err error) {
    req.HTML = converter.Markdown2HTML(req.Content)       // 隐式写 HTML
    for _, tag := range req.Tags {
        if len(tag.OriginalText) > 0 {
            tag.ParsedText = converter.Markdown2HTML(tag.OriginalText)  // 隐式写标签 HTML
        }
    }
    return nil, nil
}
```

**调用链路**（在 Controller 的 `BindAndCheckReturnErr` 中）：

1. `handler.BindAndCheckReturnErr(ctx, req)` → [handler.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/handler/handler.go#L63-L78)
2. 内部调用 `validator.GetValidatorByLang(lang).Check(data)` → [validator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/validator/validator.go#L190-L257)
3. `Check()` 方法先执行 `m.Validate.Struct(value)` 做结构化校验
4. **结构化校验失败则 Checker 短路（修订误读）**：若 `Validate.Struct(value)` 返回错误，代码直接 return，不执行 `Checker.Check()`：

```go
// [validator.go#L210-L242]
err = m.Validate.Struct(value)
if err != nil {
    // ... 组装 errFields ...
    if len(errFields) > 0 {
        // 直接 return，跳过下面的 Checker 接口调用！
        return errFields, myErrors.BadRequest(reason.RequestFormatError).WithMsg(errMsg)
    }
}

// 只有 Struct 校验通过，才会走到 Checker 接口调用
if v, ok := value.(Checker); ok {
    errFields, err = v.Check()
}
```

5. 只有结构化校验通过后，才检测 value 是否实现 `Checker` 接口（第 244 行）并调用 `v.Check()`
6. `QuestionAdd.Check()` 将 `req.Content`（Markdown）转为 HTML 写入 `req.HTML`

**关键点**：
- `HTML` 字段标记 `json:"-"`，前端不会传入，也不会被 `ShouldBind` 反序列化
- `req.HTML` 的值**完全由 Check() 方法在 validator 链路中隐式赋值**
- Controller 后续代码直接使用 `req.HTML`，但赋值时机隐藏在 validator 的 `Checker` 接口调用中，不在显式业务流程里
- **结构化校验失败时 `req.HTML` 不会被赋值**（`Checker` 被短路跳过），此时 Controller 会因错误直接 return，使用不到 HTML 字段
- `QuestionAddByAnswer.Check()` 同理，额外将 `req.AnswerContent` 转为 `req.AnswerHTML`
- Tag 的 `ParsedText` 也是在此处隐式写入：`converter.Markdown2HTML(tag.OriginalText)`

---

## 十二、AddRevision：事务内 INSERT + UPDATE 原子操作，含免审语义

**Repository**: [internal/repo/revision/revision_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/repo/revision/revision_repo.go#L57-L85)

```go
func (rr *revisionRepo) AddRevision(ctx context.Context, revision *entity.Revision, autoUpdateRevisionID bool) (err error) {
    objectTypeNumber, err := obj.GetObjectTypeNumberByObjectID(revision.ObjectID)
    if err != nil {
        return errors.BadRequest(reason.ObjectNotFound)
    }
    revision.ObjectType = objectTypeNumber
    if !rr.allowRecord(revision.ObjectType) {
        return nil   // 不允许记录的对象类型，静默跳过
    }
    _, err = rr.data.DB.Transaction(func(session *xorm.Session) (any, error) {
        session = session.Context(ctx)
        // 步骤 1: INSERT revision 记录
        _, err = session.Insert(revision)
        if err != nil {
            _ = session.Rollback()
            return nil, errors.InternalServer(reason.DatabaseError).WithError(err).WithStack()
        }
        // 步骤 2: 若 autoUpdateRevisionID=true，UPDATE 目标对象的 revision_id
        if autoUpdateRevisionID {
            err = rr.UpdateObjectRevisionId(ctx, revision, session)
            if err != nil {
                _ = session.Rollback()
                return nil, err
            }
        }
        return nil, nil
    })
    return err
}
```

`UpdateObjectRevisionId` 实现（第 88-102 行）：

```go
func (rr *revisionRepo) UpdateObjectRevisionId(ctx context.Context, revision *entity.Revision, session *xorm.Session) (err error) {
    tableName, _ := obj.GetObjectTypeStrByObjectID(revision.ObjectID) // "question" / "answer" / "tag"
    _, err = session.Table(tableName).Where("id = ?", revision.ObjectID).Cols("`revision_id`").Update(struct {
        RevisionID string `xorm:"revision_id"`
    }{
        RevisionID: revision.ID,
    })
    return err
}
```

**原子性**：INSERT revision + UPDATE question.revision_id 在**同一个 XORM Transaction** 中完成，任一步失败即 Rollback。

**autoUpdateRevisionID 仅控制 revision_id 指针回写，与审核无关**：

- `AddRevision` 的第二个参数 `autoUpdateRevisionID` 的唯一语义：是否在同一事务内 `UPDATE object.revision_id = revision.ID`
- `true` → INSERT revision 后立即 UPDATE 目标对象的 revision_id 字段
- `false` → 仅 INSERT revision，不更新 revision_id
- **该参数与"免审"完全无关**

**审核语义由 revisionDTO.Status 决定**（非 autoUpdateRevisionID）：

真正控制审核状态的是 `schema.AddRevisionDTO.Status` 字段：

```go
// 编辑场景（需审核时）：
if !canUpdate {
    revisionDTO.Status = entity.RevisionUnreviewedStatus  // =1 待审核
} else {
    revisionDTO.Status = entity.RevisionReviewPassStatus  // =2 审核通过
}
```

- **AddQuestion 场景**（[question_service.go#L417-L426](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/content/question_service.go#L417-L426)）：`revisionDTO.Status` 未显式赋值，零值为 0
- **编辑免审场景**（[question_service.go#L1039-L1043](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/content/question_service.go#L1039-L1043)）：显式设为 `RevisionReviewPassStatus(2)`，并先 UPDATE question 本体
- **编辑待审核场景**：显式设为 `RevisionUnreviewedStatus(1)`，不更新 question 本体

**Revision 状态常量**：[revision_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/entity/revision_entity.go#L27-L49)

```go
const (
    RevisionNormalStatus       = 0  // 默认值（零值，创建时使用）
    RevisionUnreviewedStatus   = 1  // 待审核
    RevisionReviewPassStatus   = 2  // 审核通过
    RevisionReviewRejectStatus = 3  // 审核拒绝
)

type Revision struct {
    Status int `xorm:"not null default 1 INT(11) status"`  // 数据库默认值也是 1（待审核）
}
```

**实体默认值 vs Go 零值注意**：Revision 实体 `Status` 字段 XORM tag 写的是 `default 1`，但 Go struct 零值为 0。实际插入时使用的是 Go 零值 0，因为 Insert 时并未显式指定状态字段。

**allowRecord 白名单**（第 188-199 行）：只有 question、answer、tag 三种对象类型允许记录版本，其他类型静默返回 nil。

---

## 十三、ChangeTag：新标签双写与旧 rel 删除

**调用入口**：[question_service.go#L1160-L1166](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/content/question_service.go#L1160-L1166)

```go
func (qs *QuestionService) ChangeTag(ctx context.Context, objectTagData *schema.TagChange) (...) {
    minimumTags, _ := qs.tagCommon.GetMinimumTags(ctx)
    return qs.tagCommon.ObjectChangeTag(ctx, objectTagData, minimumTags)
}
```

**核心实现**：[tag_common.go#L661-L864](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/tag_common/tag_common.go#L661-L864)

### 13.1 新标签双写：tag 表 + revision + activity

```go
// 比对提交的 tag 名称与数据库已有标签，找出新增标签
addTagList := make([]*entity.Tag, 0)
for _, tag := range objectTagData.Tags {
    _, ok := tagInDbMapping[strings.ToLower(tag.SlugName)]
    if ok {
        continue   // 已存在的标签，跳过
    }
    // 构建新标签实体
    item := &entity.Tag{}
    item.SlugName = strings.ReplaceAll(tag.SlugName, " ", "-")
    item.DisplayName = tag.DisplayName
    item.OriginalText = tag.OriginalText
    item.ParsedText = tag.ParsedText      // 已在 Check() 中转为 HTML
    item.Status = entity.TagStatusAvailable
    item.UserID = objectTagData.UserID
    addTagList = append(addTagList, item)
}

if len(addTagList) > 0 {
    // 双写第一写：批量 INSERT tag 表
    err = ts.tagCommonRepo.AddTagList(ctx, addTagList)

    for _, tag := range addTagList {
        // 双写第二写：为每个新标签创建 revision 记录
        revisionDTO := &schema.AddRevisionDTO{
            UserID:   objectTagData.UserID,
            ObjectID: tag.ID,
            Title:    tag.SlugName,
        }
        tagInfoJson, _ := json.Marshal(tag)
        revisionDTO.Content = string(tagInfoJson)
        revisionID, _ := ts.revisionService.AddRevision(ctx, revisionDTO, true)  // 免审

        // 双写第三写：发送 Activity 事件（TagCreated）
        ts.activityQueueService.Send(ctx, &schema.ActivityMsg{
            UserID:           objectTagData.UserID,
            ObjectID:         tag.ID,
            OriginalObjectID: tag.ID,
            ActivityTypeKey:  constant.ActTagCreated,
            RevisionID:       revisionID,
        })
    }
}
```

**双写总结**：对每个新标签，依次执行三写操作：
1. **tag 表** INSERT（`AddTagList`）
2. **revision 表** INSERT + tag.revision_id UPDATE（`AddRevision(ctx, dto, true)`，事务原子，免审）
3. **Activity 队列** Send（`ActTagCreated`）

### 13.2 旧 rel 删除：CreateOrUpdateTagRelList

```go
err = ts.CreateOrUpdateTagRelList(ctx, objectTagData.ObjectID, thisObjTagIDList)
```

**实现**：[tag_common.go#L799-L864](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/tag_common/tag_common.go#L799-L864)

```go
func (ts *TagCommonService) CreateOrUpdateTagRelList(ctx context.Context, objectId string, tagIDs []string) (err error) {
    addTagIDMapping := make(map[string]struct{})
    for _, t := range tagIDs {
        addTagIDMapping[t] = struct{}{}       // 新标签 ID 集合
    }

    // 1. 获取当前对象的所有旧 tag_rel
    oldTagRelList, _ := ts.tagRelRepo.GetObjectTagRelList(ctx, objectId)

    // 2. 比对：旧 rel 中不在新集合中的 → 加入删除列表
    var deleteTagRel []int64
    for _, rel := range oldTagRelList {
        if _, ok := addTagIDMapping[rel.TagID]; !ok {
            deleteTagRel = append(deleteTagRel, rel.ID)
            needRefreshTagIDs = append(needRefreshTagIDs, rel.TagID)
        }
    }

    // 3. 对新标签：不存在 rel 则创建，已存在但被移除过则重新启用
    addTagRelList := make([]*entity.TagRel, 0)
    enableTagRelList := make([]int64, 0)
    defaultTagRelStatus, _ := ts.tagRelRepo.GetTagRelDefaultStatusByObjectID(ctx, objectId)
    for _, tagID := range tagIDs {
        rel, exist, _ := ts.tagRelRepo.GetObjectTagRelWithoutStatus(ctx, objectId, tagID)
        if !exist {
            addTagRelList = append(addTagRelList, &entity.TagRel{  // 新建 rel
                TagID: tagID, ObjectID: objectId, Status: defaultTagRelStatus,
            })
        }
        if exist && rel.Status != entity.TagRelStatusAvailable && rel.Status != entity.TagRelStatusHide {
            enableTagRelList = append(enableTagRelList, rel.ID)   // 重新启用被删除的 rel
        }
    }

    // 4. 执行删除
    if len(deleteTagRel) > 0 {
        ts.tagRelRepo.RemoveTagRelListByIDs(ctx, deleteTagRel)
    }
    // 5. 执行新增
    if len(addTagRelList) > 0 {
        ts.tagRelRepo.AddTagRelList(ctx, addTagRelList)
    }
    // 6. 执行重新启用
    if len(enableTagRelList) > 0 {
        ts.tagRelRepo.EnableTagRelByIDs(ctx, enableTagRelList, ...)
    }
    // 7. 刷新所有受影响标签的 question_count
    ts.RefreshTagQuestionCount(ctx, needRefreshTagIDs)
}
```

**rel 管理的三种操作**：

| 操作 | 条件 | 方法 |
|------|------|------|
| 删除旧 rel | 旧 rel 的 TagID 不在新 tagIDs 中 | `RemoveTagRelListByIDs` |
| 新增 rel | objectId+tagID 组合不存在 | `AddTagRelList` |
| 重新启用 rel | rel 已存在但状态为非 Available/Hide | `EnableTagRelByIDs` |

**defaultTagRelStatus 真实逻辑（修订误读）**：[tag_rel_repo.go#L195-L207](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/repo/tag/tag_rel_repo.go#L195-L207)

```go
func (tr *tagRelRepo) GetTagRelDefaultStatusByObjectID(ctx context.Context, objectID string) (status int, err error) {
    question := entity.Question{}
    exist, err := tr.data.DB.Context(ctx).ID(objectID).Cols("show", "status").Get(&question)
    if exist && (question.Show == entity.QuestionHide || question.Status == entity.QuestionStatusDeleted) {
        return entity.TagRelStatusHide, nil
    }
    return entity.TagRelStatusAvailable, nil
}
```

只有当问题的 **`Show == QuestionHide`（被隐藏）** 或 **`Status == QuestionStatusDeleted`（已删除）** 时，才返回 Hide。**与"待审核(Pending)还是已通过(Available)"完全无关**，Pending 状态同样返回 Available。

**SlugName 规范化（先 ToLower 再转 Dash）**：[tag_common.go#L674-L707](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/tag_common/tag_common.go#L674-L707)

```go
// 第一轮：查询已有标签前，先 ToLower
for _, t := range objectTagData.Tags {
    t.SlugName = strings.ToLower(t.SlugName)           // 先转小写
    thisObjTagNameList = append(thisObjTagNameList, t.SlugName)
}
// 第二轮：发现是新标签，ToLower 后再 ReplaceAll 空格→横线
for _, tag := range objectTagData.Tags {
    _, ok := tagInDbMapping[strings.ToLower(tag.SlugName)]
    if ok { continue }
    item := &entity.Tag{}
    item.SlugName = strings.ReplaceAll(tag.SlugName, " ", "-")  // 再转 Dash
    // ...
}
```

注意**顺序不可逆**：先 ToLower 统一大小写做匹配，匹配不上确认是新标签后，再 ReplaceAll 空格→横线生成入库的 SlugName。

最后 `RefreshTagQuestionCount` 重新计算所有受影响标签的 `question_count`（含新增和删除的标签）。

---

## 十四、Queue Worker 用 context.TODO 丢弃请求 ctx

**文件**: [internal/base/queue/queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/queue/queue.go#L114-L130)

```go
func (q *Queue[T]) processMessage(msg T) {
    q.mu.RLock()
    handler := q.handler
    q.mu.RUnlock()

    if handler == nil {
        log.Warnf("[%s] no handler registered, dropping message: %+v", q.name, msg)
        return
    }

    // Use background context for async processing
    // TODO: Consider adding timeout or using a derived context
    if err := handler(context.TODO(), msg); err != nil {
        log.Errorf("[%s] handler error: %v", q.name, err)
    }
}
```

**问题分析**：

1. `processMessage` 在 worker goroutine 中被调用，与原始 HTTP 请求完全解耦
2. `handler(context.TODO(), msg)` 传入 `context.TODO()`，**不是**原始请求的 `ctx`
3. **后果**：
   - 请求级别的 `ctx` 中的 traceID、language、user-agent 等上下文信息**全部丢失**
   - 请求超时/取消信号无法传递到异步处理中
   - 数据库操作使用的是 `context.TODO()`，无法被请求生命周期管控
4. 源码中的 TODO 注释也承认了这一点：`// TODO: Consider adding timeout or using a derived context`
5. `Send()` 方法虽然接收 `ctx context.Context` 参数（第 63 行），但仅用于 channel 写入时的 context cancel 检测，并未将 ctx 传递到消息中

**Handler 未注册则 msg 静默丢弃（修订误读）**：

第 120-123 行检查 `handler == nil`（未注册）时：
```go
if handler == nil {
    log.Warnf("[%s] no handler registered, dropping message: %+v", q.name, msg)
    return
}
```
消息被直接 `return` 丢弃，只有一条 `log.Warnf` 日志记录。没有重试、没有死信队列、没有 panic。若队列初始化顺序错误导致 handler 注册晚于消息投递，消息会静默丢失。

---

## 十五、DuplicateRequestRejection：MD5 键 + TTL + defer Clear

**中间件**: [internal/base/middleware/rate_limit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/middleware/rate_limit.go#L47-L72)

```go
func (rm *RateLimitMiddleware) DuplicateRequestRejection(ctx *gin.Context, req any) (reject bool, key string) {
    userID := GetLoginUserIDFromContext(ctx)
    fullPath := ctx.FullPath()
    reqJson, _ := json.Marshal(req)
    key = encryption.MD5(fmt.Sprintf("%s:%s:%s", userID, fullPath, string(reqJson)))
    var err error
    reject, err = rm.limitRepo.CheckAndRecord(ctx, key)
    if err != nil {
        log.Errorf("check and record rate limit error: %s", err.Error())
        return false, key
    }
    if !reject {
        return false, key
    }
    log.Debugf("duplicate request: [%s] %s", fullPath, string(reqJson))
    handler.HandleResponse(ctx, errors.BadRequest(reason.DuplicateRequestError), nil)
    return true, key
}

func (rm *RateLimitMiddleware) DuplicateRequestClear(ctx *gin.Context, key string) {
    err := rm.limitRepo.ClearRecord(ctx, key)
    if err != nil {
        log.Errorf("clear rate limit error: %s", err.Error())
    }
}
```

**Cache 存储**: [internal/repo/limit/limit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/repo/limit/limit.go#L46-L65)

```go
func (lr *LimitRepo) CheckAndRecord(ctx context.Context, key string) (limit bool, err error) {
    _, exist, err := lr.data.Cache.GetString(ctx, constant.RateLimitCacheKeyPrefix+key)
    if err != nil {
        return false, errors.InternalServer(reason.DatabaseError).WithError(err).WithStack()
    }
    if exist {
        return true, nil        // 键已存在 → 判定为重复请求
    }
    err = lr.data.Cache.SetString(ctx, constant.RateLimitCacheKeyPrefix+key,
        fmt.Sprintf("%d", time.Now().Unix()), constant.RateLimitCacheTime)
    if err != nil {
        return false, errors.InternalServer(reason.DatabaseError).WithError(err).WithStack()
    }
    return false, nil           // 首次写入 → 非重复请求
}

func (lr *LimitRepo) ClearRecord(ctx context.Context, key string) error {
    return lr.data.Cache.Del(ctx, constant.RateLimitCacheKeyPrefix+key)
}
```

**Get + Set 非原子（修订误读）**：

`CheckAndRecord` 先执行 `Cache.GetString`（读），再执行 `Cache.SetString`（写），是两个独立的缓存操作，中间存在竞态窗口：

```
请求 A: GetString(key) → false（不存在）
请求 B: GetString(key) → false（不存在） ← 同时进入
请求 A: SetString(key)  ✓
请求 B: SetString(key)  ✓ ← 两个请求都通过，防重失效
```

未使用 Redis 的 `SETNX` 或 `SET ... NX` 原子命令，也没有分布式锁。在并发相同请求下防重可能被突破。

**常量**: [cache_key.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/constant/cache_key.go#L51-L52)

```go
RateLimitCacheKeyPrefix = "answer:rate-limit:"
RateLimitCacheTime      = 5 * time.Minute
```

**在 Controller 中的使用**：[question_controller.go#L390-L399](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/controller/question_controller.go#L390-L399)

```go
reject, rejectKey := qc.rateLimitMiddleware.DuplicateRequestRejection(ctx, req)
if reject {
    return
}
defer func() {
    // If status is not 200 means that the bad request has been returned,
    // so the record should be cleared
    if ctx.Writer.Status() != http.StatusOK {
        qc.rateLimitMiddleware.DuplicateRequestClear(ctx, rejectKey)
    }
}()
```

**完整机制**：

| 环节 | 实现 | 说明 |
|------|------|------|
| **键生成** | `MD5(userID:fullPath:reqJson)` | 用户 + 路由 + 请求体 三要素哈希 |
| **键前缀** | `answer:rate-limit:` + MD5值 | Cache 命名空间隔离 |
| **TTL** | `5 * time.Minute` | 5 分钟自动过期 |
| **判定逻辑** | Cache.Get 存在 → 重复；不存在 → 写入 Cache.Set | 即写即查模式 |
| **写入值** | `time.Now().Unix()` 时间戳 | 仅用于标记存在性，值无业务含义 |
| **defer Clear** | `ctx.Writer.Status() != 200` 时清除 | 业务失败（校验不通过等）不应占用防重配额 |
| **不 Clear** | `Status == 200` 时保留 | 成功请求保留 5 分钟防重锁，避免重复提交 |

**defer Clear 的语义**：
- 防重复提交的本质是防止**成功请求**被重复执行
- 如果请求因校验失败返回 400/403，说明请求根本没生效，此时应**清除锁键**，允许用户修正后重新提交
- 如果请求成功返回 200，锁键保留 5 分钟自动过期，期间相同内容的重复请求被拦截
- `defer` 确保 Clear 逻辑在函数返回时一定执行，无论成功还是 panic

**匿名用户退化（补充）**：[auth.go#L270-L275](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/middleware/auth.go#L270-L275)

`GetLoginUserIDFromContext(ctx)` 在用户未登录时返回空字符串 `""`，此时防重键退化为：

```
MD5("" + ":" + fullPath + ":" + reqJson)
```

所有匿名用户共享同一个防重键，任意匿名用户提交后，5 分钟内**所有其他匿名用户**提交相同内容都会被判定为重复请求。这是一个设计缺陷：匿名防重退化为全局防重，而非按用户隔离。

---

## 十六、Cache 后端默认是 memory in-process，多副本失效

**初始化**: [internal/base/data/data.go#L97-L137](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/data/data.go#L97-L137)

```go
func NewCache(c *CacheConf) (cache.Cache, func(), error) {
    // 第一步：尝试加载 Cache 插件
    var pluginCache plugin.Cache
    _ = plugin.CallCache(func(fn plugin.Cache) error {
        pluginCache = fn
        return nil
    })
    if pluginCache != nil {
        return pluginCache, func() {}, nil  // 有插件 → 用插件（如 Redis）
    }

    // 第二步：无插件 → 使用进程内内存缓存
    memCache := memory.NewCache()       // pacman/contrib/cache/memory

    // 可选：从文件加载缓存快照（冷启动加速）
    if len(c.FilePath) > 0 {
        cacheFileDir := filepath.Dir(c.FilePath)
        if err := memory.Load(memCache, c.FilePath); err != nil {
            // ... load 失败忽略 ...
        }
    }

    // 可选：定时持久化缓存到文件（默认 30s 一次）
    if len(c.FilePath) > 0 {
        ticker := time.NewTicker(30 * time.Second)
        go func() {
            for range ticker.C {
                memory.Save(memCache, c.FilePath)
            }
        }()
        // shutdown 时也 save 一次
    }
    return memCache, cleanup, nil
}
```

**关键事实**：

1. **默认无插件时走 memory**：`memory.NewCache()` 返回的是基于 `sync.Map` 的纯内存实现，**每个进程一份独立副本**
2. **插件优先**：若注册了 `plugin.Cache`（如 Redis 插件），则使用插件，此时多副本共享同一缓存
3. **文件持久化是兜底**：`FilePath` 配置项只是为了重启后恢复缓存快照，不是分布式方案
4. **多副本部署失效**：默认配置下，每个节点各自维护独立 Cache，防重锁、频控、会话等依赖 Cache 的逻辑在多节点间**互不感知**
5. **DuplicateRequestRejection 受影响最大**：用户两次提交被分发到不同 pod → 各自 Cache 都查不到 → 防重完全失效

**memory cache 实现**：来自 `github.com/segmentfault/pacman/contrib/cache/memory`，内部是 Go `sync.Map` + TTL 轮询清理。

---

## 十七、Captcha 豁免：须 isAdmin 且 linkUrlLimitUser 双满足

**代码**: [internal/controller/question_controller.go#L417-L427](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/controller/question_controller.go#L417-L427)

```go
isAdmin := middleware.GetUserIsAdminModerator(ctx)
if !isAdmin || !linkUrlLimitUser {
    captchaPass := qc.actionService.ActionRecordVerifyCaptcha(
        ctx, entity.CaptchaActionQuestion, req.UserID, req.CaptchaID, req.CaptchaCode)
    if !captchaPass {
        // 返回验证码错误
        handler.HandleResponse(ctx, errors.BadRequest(reason.CaptchaVerificationFailed), errFields)
        return
    }
}
```

**逻辑分析**：

条件 `!isAdmin || !linkUrlLimitUser` 等价于 `NOT (isAdmin && linkUrlLimitUser)`（德摩根定律）。

即：**只有 isAdmin=true 且 linkUrlLimitUser=true 同时满足时，才豁免验证码校验**。只要任一不满足，就必须验证。

| isAdmin | linkUrlLimitUser | 结果 |
|---------|------------------|------|
| false | false | 需验证码 |
| false | true  | 需验证码 |
| true  | false | 需验证码 |
| true  | true  | **豁免** |

**`linkUrlLimitUser` 的来源**：从 8 项（或 7 项）权限检查的 `LinkUrlLimit` 结果中来：

```go
linkUrlLimitUser := canList[7]  // 第 8 项权限（AddQuestion 路径）
linkUrlLimitUser := canList[6]  // 第 7 项权限（AddQuestionByAnswer 路径）
```

语义是「用户是否在链接数量限制白名单内」——高信誉或特殊权限用户免链接限制，同时也免验证码。

**其他场景的验证码豁免规则**：

| 操作 | 豁免条件 |
|------|---------|
| AddQuestion | `isAdmin && linkUrlLimitUser` |
| AddQuestionByAnswer | `isAdmin && linkUrlLimitUser` |
| UpdateQuestion | `isAdmin && linkUrlLimitUser` |
| DeleteQuestion | 仅 `isAdmin`（单条件） |
| InvitationAnswer | 仅 `isAdmin`（单条件） |

删除和邀请回答的验证码豁免门槛更低——只要是管理员/版主就豁免，不需要同时满足 linkUrlLimitUser。

---

## 十八、RefreshTagQuestionCount 失败只 log，错误被吞噬

**定义**: [internal/service/tag_common/tag_common.go#L749-L762](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/tag_common/tag_common.go#L749-L762)

```go
func (ts *TagCommonService) RefreshTagQuestionCount(ctx context.Context, tagIDs []string) (err error) {
    for _, tagID := range tagIDs {
        count, err := ts.tagRelRepo.CountTagRelByTagID(ctx, tagID)
        if err != nil {
            return err
        }
        err = ts.tagCommonRepo.UpdateTagQuestionCount(ctx, tagID, int(count))
        if err != nil {
            return err
        }
    }
    return nil
}
```

函数本身正确返回 error，但**调用方**多处只 log 不向上传递：

**调用点 1**：`CreateOrUpdateTagRelList` 末尾 [tag_common.go#L859-L862](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/tag_common/tag_common.go#L859-L862)

```go
err = ts.RefreshTagQuestionCount(ctx, needRefreshTagIDs)
if err != nil {
    log.Error(err)    // ← 只 log，不 return
}
```

**调用点 2**：删除问题后 [question_service.go#L634-L637](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/content/question_service.go#L634-L637)

```go
err = qs.tagCommon.RefreshTagQuestionCount(ctx, tagIDs)
if err != nil {
    log.Error("efreshTagQuestionCount error", err.Error())  // ← 只 log，不 return
}
```

**调用点 3**：回答被接受/取消时更新 tag 计数 [question_service.go#L774-L776](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/content/question_service.go#L774-L776)

```go
if err = qs.tagCommon.RefreshTagQuestionCount(ctx, tagIDs); err != nil {
    log.Errorf("update tag's question count failed, %v", err)  // ← 只 log，不 return
}
```

**后果**：标签的 `question_count` 可能与真实数据不一致，但主流程不受影响。属于「数据统计可降级」的设计权衡。

**例外**：`MergeTag` 场景 [tag_service.go#L489-L492](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/tag/tag_service.go#L489-L492) 是 return error 的，因为标签合并是强一致性操作。

---

## 十九、Queue Send：buffer 满 + ctx 取消 + 另有 3 条 drop 路径

**Send 方法**: [internal/base/queue/queue.go#L61-L78](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/queue/queue.go#L61-L78)

```go
func (q *Queue[T]) Send(ctx context.Context, msg T) {
    q.mu.RLock()
    defer q.mu.RUnlock()

    if q.closed {
        log.Warnf("[%s] queue is closed, dropping message", q.name)
        return   // ← drop 路径 1：队列已关闭
    }

    select {
    case q.queue <- msg:
        log.Debugf("[%s] enqueued message: %+v", q.name, msg)
    case <-ctx.Done():
        log.Warnf("[%s] context cancelled while sending message", q.name)
        // ← drop 路径 2：ctx 取消（请求超时/断开等），消息不进队列
    }
}
```

**第三条 drop 路径**：Handler 未注册时 worker 丢弃 [queue.go#L120-L123](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/queue/queue.go#L120-L123)

```go
if handler == nil {
    log.Warnf("[%s] no handler registered, dropping message: %+v", q.name, msg)
    return   // ← drop 路径 3：handler 未注册，worker 收到消息后直接丢
}
```

**三条 drop 路径汇总**：

| 路径 | 位置 | 触发条件 | 日志级别 |
|------|------|---------|---------|
| 1 | Send() | `q.closed == true` | Warn |
| 2 | Send() | `ctx.Done()` 先于 channel 写入 | Warn |
| 3 | processMessage() | `handler == nil` 未注册 | Warn |

**关于 buffer 满**：

`Send` 没有 default 分支，当 channel buffer 满且 ctx 未取消时，**Send 会阻塞**，直到有空位或 ctx 取消。不是非阻塞丢弃，而是可能阻塞调用者 goroutine（即 HTTP 请求 goroutine）。

这意味着在高并发写入场景下，如果消费速度跟不上，buffer 满会反压到 HTTP handler，导致请求处理时间变长。

**关于 context.TODO**：worker 消费时用 `context.TODO()`，丢弃了原始请求的 ctx，因此即使发送时 ctx 还在，消费时也已经和请求生命周期无关。
