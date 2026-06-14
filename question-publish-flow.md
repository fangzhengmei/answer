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
