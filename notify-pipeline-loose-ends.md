# 通知管线细节扩写

## 一、取消采纳中用户标识与问题标识等值比较的判定

### 1.1 代码位置

[sendCancelAcceptAnswerNotification](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/activity/answer_repo.go#L331-L349)

```go
func (ar *AnswerActivityRepo) sendCancelAcceptAnswerNotification(
    ctx context.Context, op *schema.AcceptAnswerOperationInfo) {
    for _, act := range op.Activities {
        msg := &schema.NotificationMsg{
            TriggerUserID:  act.TriggerUserID,
            ReceiverUserID: act.ActivityUserID,
            Type:           schema.NotificationTypeAchievement,
            ObjectID:       op.AnswerObjectID,
        }
        // 🔴 可疑比较：用户ID vs 对象ID
        if act.ActivityUserID == op.QuestionObjectID {
            msg.ObjectType = constant.QuestionObjectType
        } else {
            msg.ObjectType = constant.AnswerObjectType
        }
        if msg.TriggerUserID != msg.ReceiverUserID {
            ar.notificationQueueService.Send(ctx, msg)
        }
    }
}
```

### 1.2 上游数据结构

[AcceptAnswerOperationInfo](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/schema/answer_activity_schema.go#L23-L32)

```go
type AcceptAnswerOperationInfo struct {
    TriggerUserID    string   // 采纳/取消采纳操作者（=问题作者）
    QuestionObjectID string   // 问题对象ID（如 "1aBc"，短编码）
    QuestionUserID   string   // 问题作者用户ID（如 "xyz123"，UUID风格）
    AnswerObjectID   string   // 回答对象ID
    AnswerUserID     string   // 回答作者用户ID
    Activities       []*AcceptAnswerActivity
}
```

**关键事实**：`QuestionObjectID` 和 `QuestionUserID` 是**两个完全不同的字段**：
- `QuestionUserID`：用户ID（如 "3XkEexP28rqgBkE9Yp9Ttc"，24位UUID编码）
- `QuestionObjectID`：问题对象ID（如 "3aU1M73P2S2O"，相同编码风格但标识不同实体）

### 1.3 上游调用链

[updateAnswerRank](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/content/answer_service.go#L516-L534) → [CancelAcceptAnswer](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/activity/answer_activity_service.go#L66-L71)

```go
// answer_service.go L521-L522
err := as.answerActivityService.CancelAcceptAnswer(ctx, userID,
    questionInfo.AcceptedAnswerID,   // → answerObjID
    questionInfo.ID,                  // → questionObjID  ✅ 传入的是问题对象ID
    questionInfo.UserID,              // → questionUserID  ✅ 传入的是问题作者用户ID
    oldAnswerInfo.UserID)             // → answerUserID
```

上游正确地将 `questionInfo.ID` 和 `questionInfo.UserID` 分别传入了 `QuestionObjectID` 和 `QuestionUserID`。

### 1.4 判定结论：字段复用的意图，但写法存在混淆

**不是上游 bug**：上游参数传递是正确的，`QuestionObjectID` 和 `QuestionUserID` 各司其职。

**但写法存在混淆（属于"字段复用不规范"而非 bug）**：

```go
// sendCancelAcceptAnswerNotification 中
if act.ActivityUserID == op.QuestionObjectID {  // 用户ID 比较 问题对象ID
    msg.ObjectType = constant.QuestionObjectType
}
```

这段代码的**真实意图**是：
- 如果声望收益人（`ActivityUserID`）== 问题作者用户ID → 说明这条声望是"问题被采纳"的声望 → 成就通知的 `ObjectType` 应该是 `QuestionObjectType`（去重 key 用问题ID）
- 否则 → 声望收益人是回答作者 → `ObjectType` 应该是 `AnswerObjectType`（去重 key 用回答ID）

**但写错了比较目标**：代码拿 `ActivityUserID`（用户ID）去和 `op.QuestionObjectID`（问题对象ID）比较。

**问题严重性分析**：
- 用户ID 和对象ID 虽然都是短编码字符串，但**编码空间几乎无重叠概率极低**。ID 生成为 `uid.EnShortID(uuid.NewString())`，UUID 碰撞概率可忽略。因此 `ActivityUserID == op.QuestionObjectID` 实际上**永远为 false**。
- 结果：**所有取消采纳的成就通知 ObjectType 都被设置为 `AnswerObjectType`**，问题作者的取消采纳声望通知也会用回答ID作为 `ObjectID` 去重。
- 实际影响**有限但存在正确性问题**：
  - 如果问题作者同时也是回答作者（自问自答后采纳再取消），其成就通知的 ObjectType 应该是 QuestionObjectType，但实际被设置为 AnswerObjectType。但 `ObjectID` 传的是 `op.AnswerObjectID`，所以数据库中去重查询时用的是回答ID，声望 Sum 查询可能对不上。
  - 成就去重 key 是 `(userID, objectID, type)`，其中 `objectID` 由 `NotificationMsg.ObjectID` 传入，与 `ObjectType` **无关**（ObjectType 不参与去重查询，只影响 `objectInfoService.GetInfo` 的解析）。因此成就去重不受影响，受影响的是后续 `GetInfo` 时对 ObjectType 的判断——不过成就类型的通知不经过 `objectInfoService.GetInfo`（成就走的是另一分支），所以实际上这段比较**几乎没有效果**。

**正确写法应该是**：

```go
// 正确写法：比较用户ID
if act.ActivityUserID == op.QuestionUserID {
    msg.ObjectType = constant.QuestionObjectType
} else {
    msg.ObjectType = constant.AnswerObjectType
}
```

对比采纳路径的写法（[sendAcceptAnswerNotification](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/activity/answer_repo.go#L301-L314)）：

```go
// 采纳的成就通知循环——没有设置 ObjectType，使用了默认值！
for _, act := range op.Activities {
    msg := &schema.NotificationMsg{
        Type:           schema.NotificationTypeAchievement,
        ObjectID:       op.AnswerObjectID,
        ReceiverUserID: act.ActivityUserID,
        TriggerUserID:  act.TriggerUserID,
    }
    msg.ObjectType = constant.AnswerObjectType  // ✅ 这里统一设为 AnswerObjectType
    ...
}
```

采纳路径**所有成就通知统一使用 `AnswerObjectType`**，没有按受益人区分。这进一步说明取消采纳路径的那个 if-else 分支"区分受益人设置 ObjectType"的意图本身就与采纳路径不一致，加上比较写错，形成了"意图存疑+实现错误"的双重混淆。

**总结判定**：

| 维度 | 结论 |
|------|------|
| 上游传参 | ✅ 正确，无 bug |
| 比较目标 | ❌ 写错，应该是 `op.QuestionUserID` 而不是 `op.QuestionObjectID` |
| 实际影响 | ⚠️ 极低（成就去重不依赖 ObjectType，且成就通知不走 objectInfoService） |
| 设计意图 | 存疑（采纳路径统一用 AnswerObjectType，取消采纳却尝试区分，与采纳路径不一致） |
| 定性 | **疑似字段混淆 + 设计不一致**，非严重 bug，属代码质量问题 |

---

## 二、动作映射落零：所有成就通知映射后都落零 + 徽章动作不在映射键里

### 2.1 映射表全貌

[NotificationMsgTypeMapping](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/base/constant/notification.go#L82-L102)

```go
var NotificationMsgTypeMapping = map[string]int{
    // 1 = 帖子类
    NotificationUpdateQuestion:         1,
    NotificationAnswerTheQuestion:      1,
    NotificationUpdateAnswer:           1,
    NotificationAcceptAnswer:           1,
    NotificationCommentQuestion:        1,
    NotificationCommentAnswer:          1,
    NotificationReplyToYou:             1,
    NotificationMentionYou:             1,
    NotificationYourQuestionIsClosed:   1,
    NotificationYourQuestionWasDeleted: 1,
    NotificationYourAnswerWasDeleted:   1,
    NotificationYourCommentWasDeleted:  1,

    // 2 = 投票类
    NotificationUpVotedTheQuestion:     2,
    NotificationDownVotedTheQuestion:   2,
    NotificationUpVotedTheAnswer:       2,
    NotificationDownVotedTheAnswer:     2,
    NotificationUpVotedTheComment:      2,

    // 3 = 邀请类
    NotificationInvitedYouToAnswer:     3,
}
```

**未包含的 NotificationAction**：

| 动作常量 | 值 | 是否在映射表 |
|----------|---|:-----------:|
| `NotificationEarnedBadge` | `"notification.action.earned_badge"` | ❌ 不在 |
| 所有投票成就通知（VoteUp/VotedUp 等） | （无 NotificationAction，为空字符串） | ❌ 不在 |
| 采纳成就通知 | （无 NotificationAction，为空字符串） | ❌ 不在 |
| 取消采纳成就通知 | （无 NotificationAction，为空字符串） | ❌ 不在 |
| 修订审核成就通知 | （无 NotificationAction，为空字符串） | ❌ 不在 |

### 2.2 所有成就通知映射后都落零

成就类通知（`Type == Achievement (2)`）的构建代码中，**几乎都没有设置 `NotificationAction`**：

| 成就来源 | NotificationAction 设置 | 映射结果 MsgType |
|---------|------------------------|:----------------:|
| 投票获得/损失声望 | 不设置（空字符串） | **0** |
| 投票消耗声望 | 不设置（空字符串） | **0** |
| 采纳获得声望 | 不设置（空字符串） | **0** |
| 取消采纳损失声望 | 不设置（空字符串） | **0** |
| 修订审核通过声望 | 不设置（空字符串） | **0** |
| 徽章授予 | `NotificationEarnedBadge`（显式设置） | **0**（不在映射表） |

**映射代码**（[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L193-L198)）：

```go
info.MsgType = 0  // Go int 零值
_, ok := constant.NotificationMsgTypeMapping[req.NotificationAction]
if ok {
    info.MsgType = constant.NotificationMsgTypeMapping[req.NotificationAction]
}
```

因为成就通知的 `NotificationAction` 要么是空字符串，要么是 `NotificationEarnedBadge`（不在映射表），所以 `ok` 永远为 `false`，`info.MsgType` 永远落在 Go 的零值 **0** 上。

### 2.3 徽章动作不在映射键里的单独说明

徽章授予是成就通知中**唯一显式设置了 NotificationAction** 的场景：

[badge_award_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/badge/badge_award_service.go#L180-L182)

```go
notificationMsg.NotificationAction = constant.NotificationEarnedBadge
```

但 `NotificationEarnedBadge` 虽然定义在常量中（[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/base/constant/notification.go#L60)），却**没有被加入 `NotificationMsgTypeMapping`**：

```go
// 常量定义了
NotificationEarnedBadge = "notification.action.earned_badge"

// 但映射表没包含
var NotificationMsgTypeMapping = map[string]int{
    // ... 19 个条目 ...
    // ❌ 缺 NotificationEarnedBadge
}
```

### 2.4 MsgType=0 的影响矩阵

| 通知类型 | MsgType 值 | 影响范围 |
|---------|:----------:|---------|
| Inbox（大部分） | 1/2/3（正常映射） | ✅ posts/votes/invites 子分类筛选正常 |
| Inbox（罕见动作） | 0（未映射） | ❌ 仅在 inbox_type=all 可见，子分类筛选丢失 |
| Achievement（全部） | 0 | ⚠️ 成就标签页没有 inbox_type 子分类，不影响筛选 |
| Achievement（徽章） | 0 | ⚠️ 同上，且徽章有独立的 badge 弹窗缓存 |

**总结**：成就通知 MsgType 全落零是**设计选择而非缺陷**，因为成就标签页没有子分类筛选的概念。但徽章动作的显式设置却没有加入映射表，属于**不一致的代码质量问题**（可以加但没必要，因为不造成实际影响）。

---

## 三、通知入口往对象映射塞问题、回答、评论三标识的过程及前端跳转

### 3.1 塞入过程

代码位置：[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L131-L144)

```go
objInfo, err = ns.objectInfoService.GetInfo(ctx, req.ObjectInfo.ObjectID)
if err != nil {
    log.Error(err)
    return err
} else {
    req.ObjectInfo.Title = objInfo.Title
    questionID = objInfo.QuestionID
    objectMap := make(map[string]string)
    objectMap["question"] = uid.DeShortID(objInfo.QuestionID)
    objectMap["answer"]   = uid.DeShortID(objInfo.AnswerID)
    objectMap["comment"]  = objInfo.CommentID
    req.ObjectInfo.ObjectMap = objectMap
}
```

这 3 个字段的**值来源**依赖于 `objectInfoService.GetInfo()` 的对象类型解析：

[GetInfo](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/object_info/object_info.go#L187-L299) 根据 ObjectID 的前缀判断对象类型，分别走 Question/Answer/Comment/Tag 四条分支。

### 3.2 各对象类型的三标识填充结果

| 通知 ObjectID 类型 | `objectMap["question"]` | `objectMap["answer"]` | `objectMap["comment"]` | 示例场景 |
|-------------------|------------------------|----------------------|-----------------------|---------|
| **Question** | `uid.DeShortID(questionInfo.ID)` → 问题ID（长） | `""`（`objInfo.AnswerID` 默认为空） | `""`（`objInfo.CommentID` 默认为空） | 问题被更新、问题被关闭 |
| **Answer** | `uid.DeShortID(answerInfo.QuestionID)` → 所属问题ID（长） | `uid.DeShortID(answerInfo.ID)` → 回答ID（长） | `""` | 回答被采纳、回答被评论 |
| **Comment** | `uid.DeShortID(commentInfo.QuestionID)` → 所属问题ID（长） | `uid.DeShortID(answerInfo.ID)` → 所属回答ID（长，评论对象是回答时） | `commentInfo.ID` → 评论ID（短，不解码） | 评论问题/回答、回复评论 |
| **Tag** | `""`（questionID 为空） | `""` | `""` | 无（标签不产生通知） |
| **BadgeAward** | 不走 `objectInfoService.GetInfo`，直接塞 `{"badge_id": ...}` | — | — | 徽章授予 |

**注意 `uid.DeShortID` 的作用**：
- `objInfo.QuestionID`、`objInfo.AnswerID` 存储的是**短编码**（如 "3aU1M73P2S2O"）
- `uid.DeShortID()` 将短编码还原为**长编码**（原始UUID的短编码 → 标准UUID字符串）
- `objInfo.CommentID` 是评论ID，评论ID本身就是短编码且**不调用 DeShortID**，直接原样存储

### 3.3 NotificationContent 序列化后的 ObjectMap 结构

写入数据库的 `notification.content` 是 JSON 序列化的 [NotificationContent](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/schema/notification_schema.go)：

```json
{
  "object_info": {
    "title": "如何理解 Go 的泛型？",
    "object_id": "3aU1M73P2S2O",
    "object_map": {
      "question": "7f9c3d2e-5b1a-4f8e-9c6d-2e7f1a3b5c8d",
      "answer": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "comment": "3XkEexP28r"
    },
    "object_type": "answer"
  },
  "user_info": {
    "id": "...",
    "username": "...",
    "display_name": "..."
  },
  "notification_action": "notification.action.comment_answer",
  "rank": 0,
  "type": 1
}
```

### 3.4 前端跳转使用三字段

前端收到通知列表后，根据 `ObjectMap` 中的三个字段构建跳转 URL：

```typescript
// 伪代码：前端根据 ObjectMap 构建跳转链接
function buildNotificationUrl(objectMap, objectType, notificationAction) {
    const { question, answer, comment } = objectMap;

    // 评论通知 → 跳转到评论锚点
    if (comment) {
        if (answer) {
            return `/questions/${question}#answer-${answer}-comment-${comment}`;
        }
        return `/questions/${question}#comment-${comment}`;
    }

    // 回答通知 → 跳转到回答锚点
    if (answer) {
        return `/questions/${question}#answer-${answer}`;
    }

    // 问题通知 → 直接跳转到问题页
    if (question) {
        return `/questions/${question}`;
    }

    // 徽章通知 → 跳转到徽章详情
    if (objectMap.badge_id) {
        return `/badges/${objectMap.badge_id}`;
    }
}
```

**三字段的精确作用**：

| 字段 | 前端用途 | 缺失时的降级策略 |
|------|---------|----------------|
| `objectMap["question"]` | 所有跳转的基础路径（`/questions/{question}`） | 通知无法跳转（徽章除外） |
| `objectMap["answer"]` | 回答页锚点（`#answer-{answer}`）+ 回答评论定位 | 降级为问题页顶部 |
| `objectMap["comment"]` | 评论锚点（`#comment-{comment}` 或 `#answer-X-comment-Y`） | 降级为回答锚点或问题页顶部 |

**三字段冗余设计**：即使通知对象本身是 Comment，也同时填入 question 和 answer，确保前端无论如何都能定位到正确的页面层级（问题 → 回答 → 评论）。

---

## 四、关注者查询只取问题关注者、标签关注者走外部新问题广播、两条人群不重叠

### 4.1 内部通知关注者推送：只取问题关注者

[SendNotificationToAllFollower](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L327-L357)

```go
func (ns *NotificationCommon) SendNotificationToAllFollower(ctx context.Context,
    msg *schema.NotificationMsg, questionID string) {

    if msg.NoNeedPushAllFollow || len(questionID) == 0 {
        return
    }

    // 确定查询对象ID
    condObjectID := msg.ObjectID
    if len(questionID) > 0 {
        condObjectID = uid.DeShortID(questionID)  // ✅ 优先使用问题ID
    }

    // 查询关注者
    userIDs, err := ns.followRepo.GetFollowUserIDs(ctx, condObjectID)
    // ...
}
```

**关键点**：`condObjectID` **永远是问题ID**（当 questionID 非空时）。即使 `msg.ObjectID` 是回答ID或评论ID，也会被覆盖为问题ID。

[GetFollowUserIDs](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/activity_common/follow.go#L93-L116) 的实现：

```go
func (ar *FollowRepo) GetFollowUserIDs(ctx context.Context, objectID string) (userIDs []string, err error) {
    // 1. 根据 objectID 判断对象类型
    objectTypeStr, err := obj.GetObjectTypeStrByObjectID(objectID)

    // 2. 根据对象类型获取 "follow" 动作的 activity_type ID
    activityType, err := ar.activityRepo.GetActivityTypeByObjectType(ctx, objectTypeStr, "follow")

    // 3. 查询 activity 表中该对象的所有关注用户
    session.Where("object_id = ?", objectID)
    session.Where("activity_type = ?", activityType)
    session.Where("cancelled = 0")
    session.Find(&userIDs)
}
```

查询维度是 `(object_id, activity_type)`，而 activity_type 是按对象类型动态解析的。因此：
- 传入问题ID → 查询"关注问题"的 activity_type → 返回**问题关注者**
- 传入回答ID → 查询"关注回答"的 activity_type → 但回答没有关注功能（UI无入口），返回空
- 传入标签ID → 查询"关注标签"的 activity_type → 返回**标签关注者**（但内部推送不传入标签ID）

### 4.2 外部新问题广播：标签关注者 + 全量订阅者

[handleNewQuestionNotification](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/new_question_notification.go#L35-L71) → [getNewQuestionSubscribers](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/new_question_notification.go#L74-L197)

```go
func (ns *ExternalNotificationService) getNewQuestionSubscribers(ctx context.Context,
    msg *schema.ExternalNotificationMsg) (subscribers []*NewQuestionSubscriber, err error) {

    subscribersMapping := make(map[string]*NewQuestionSubscriber)

    // ① 标签关注者：遍历问题的每个标签，查该标签的关注用户
    for _, tagID := range msg.NewQuestionTemplateRawData.TagIDs {
        userIDs, err := ns.followRepo.GetFollowUserIDs(ctx, tagID)
        // ... 查询这些用户的 AllNewQuestionForFollowingTagsSource 偏好
        for _, config := range userNotificationConfigs {
            if _, ok := subscribersMapping[config.UserID]; ok {
                continue  // 去重
            }
            subscribersMapping[config.UserID] = &NewQuestionSubscriber{
                notificationType: constant.NotificationNewQuestionFollowedTag,
                userID:           config.UserID,
            }
        }
    }

    // ② 全量新问题订阅者：查 AllNewQuestionSource 偏好
    notificationConfigs, err := ns.userNotificationConfigRepo.GetBySource(
        ctx, constant.AllNewQuestionSource)
    for _, config := range notificationConfigs {
        if _, ok := subscribersMapping[config.UserID]; ok {
            continue  // 去重：如果该用户同时是标签关注者，不重复添加
        }
        subscribersMapping[config.UserID] = &NewQuestionSubscriber{
            notificationType: constant.NotificationNewQuestion,
            userID:           config.UserID,
        }
    }

    // ③ 排除问题作者
    delete(subscribersMapping, msg.NewQuestionTemplateRawData.QuestionAuthorUserID)

    return subscribers, nil
}
```

### 4.3 两条人群的来源与不重叠论证

| 人群来源 | 查询方式 | 数据来源 | 标识类型 | 触发场景 |
|---------|---------|---------|---------|---------|
| **内部关注者推送** | `GetFollowUserIDs(questionID)` | activity 表 + activity_type="follow"+问题类型 | 问题ID | 回答创建、采纳、问题更新、回答更新 |
| **外部标签关注者** | `GetFollowUserIDs(tagID)` 循环 + 偏好过滤 | activity 表 + activity_type="follow"+标签类型 | 标签ID | 新问题创建（含审核通过后） |
| **外部全量订阅者** | `GetBySource(AllNewQuestionSource)` | user_notification_config 表 | 用户ID | 新问题创建（含审核通过后） |

**不重叠的两层含义**：

**第一层：内部推送 vs 外部广播——人群完全不重叠**

| 维度 | 内部关注者推送 | 外部新问题广播 |
|------|-------------|-------------|
| 触发场景 | 问题更新 / 回答创建 / 回答更新 / 采纳 | 新问题发布 |
| 查询对象 | 问题ID（question） | 标签ID（tag）+ 用户偏好（config） |
| 人群交集 | ❌ 无交集。内部推送的是"关注某个具体问题的人"；外部广播的是"关注某个标签或订阅所有新问题的人"。一个用户可以同时在两个集合中，但两个集合的**来源实体完全不同**（问题 vs 标签/偏好）。 |
| 去重处理 | 不需要。两个队列独立处理，内部推送写站内通知库，外部广播发邮件，互不影响。 |

**第二层：外部广播内部——标签关注者与全量订阅者可能重叠，有去重**

```
subscribersMapping = { userID -> subscriber }

for 标签关注者:
    if userID already in subscribersMapping → skip  // ✅ 去重1
    else → add with NotificationNewQuestionFollowedTag

for 全量订阅者:
    if userID already in subscribersMapping → skip  // ✅ 去重2（与标签关注者去重）
    else → add with NotificationNewQuestion
```

如果一个用户 **既关注了某标签又开启了全量新问题订阅**，优先使用标签关注者的 `NotificationNewQuestionFollowedTag` 类型（先遍历标签，先到先得）。

### 4.4 为什么设计成两条独立通道

| 设计考量 | 说明 |
|---------|------|
| **关注粒度不同** | 关注问题 → 跟踪特定问题的动态（更新、新回答）；关注标签 → 发现领域内的新问题 |
| **通知频率不同** | 关注问题 → 高频（可能有多条回答、评论）；关注标签 → 低频（每天几条新问题） |
| **通知渠道不同** | 关注问题 → 站内收件箱（实时）；关注标签 → 邮件（异步，每日有限额） |
| **偏好检查不同** | 关注问题 → 无条件写入站内通知；关注标签 → 检查用户通知偏好是否启用邮件 |
| **触发动作不同** | UpdateQuestion / AnswerTheQuestion / UpdateAnswer / AcceptAnswer → 内部队列；新问题创建 → 外部队列 |

**总结**：内部关注者推送和外部标签/全量订阅广播是两条**完全独立**的通道，查询的对象实体不同（问题 vs 标签），使用的队列不同（notificationQueue vs externalNotificationQueue），写入的目标不同（站内通知 DB vs 邮件/插件），不存在人群重叠需要去重的场景。
