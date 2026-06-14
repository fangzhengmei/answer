# 通知管线全路径分析

## 一、五条补全触发路径

### 1.1 投票触发路径

**事件源**：[vote_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/activity/vote_repo.go#L72-L129)

投票操作同时投递 **两条独立消息** 到内部通知队列：

```
Vote() 事务完成后
  ├─→ 对每个 Rank != 0 的 Activity:
  │     sendAchievementNotification(activityUserID, objectCreatorUserID, objectID)
  │     → NotificationMsg {
  │         Type: Achievement (2)
  │         ReceiverUserID: 被投票对象作者（获得声望者）
  │         TriggerUserID:  被投票对象作者（自身，因为成就通知触发者就是获得者）
  │         ObjectID:       被投票对象
  │         ObjectType:     自动推断 (question/answer/comment)
  │       }
  │     → 发送到 notificationQueue
  │
  └─→ sendVoteInboxNotification(triggerUserID, receiverUserID, objectID, upvote)
        → NotificationMsg {
            Type: Inbox (1)
            TriggerUserID:  投票者
            ReceiverUserID: 被投票对象作者
            ObjectID:       被投票对象
            NotificationAction:
              question + upvote   → NotificationUpVotedTheQuestion
              question + downvote → NotificationDownVotedTheQuestion
              answer + upvote     → NotificationUpVotedTheAnswer
              answer + downvote   → NotificationDownVotedTheAnswer
              comment + upvote    → NotificationUpVotedTheComment
              comment + downvote  → (不发送，评论踩无 Inbox 通知)
          }
        → 仅在 NotificationAction 非空时发送到 notificationQueue
```

**接收者**：被投票对象的作者（`objectCreatorUserID`）。自我投票被过滤（`triggerUserID == receiverUserID` 时 return）。

**注意**：投票仅走内部通知队列，**不发送外部通知**（无邮件、无 ExternalNotificationMsg）。

---

### 1.2 采纳触发路径

**事件源**：[answer_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/activity/answer_repo.go#L301-L329)

采纳操作由 [answer_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/content/answer_service.go#L447-L514) 的 `AcceptAnswer` 触发，通过 `answerActivityService.AcceptAnswer()` 进入活动层，最终调用 `sendAcceptAnswerNotification`，投递 **两条独立消息**：

```
AcceptAnswer()
  → eventQueueService.Send(EventQuestionAccept)  // 事件总线
  → updateAnswerRank()
      → answerActivityService.AcceptAnswer()
          → sendAcceptAnswerNotification()

sendAcceptAnswerNotification() 投递:
  ├─→ 成就通知（循环每个 Activity）:
  │     NotificationMsg {
  │       Type: Achievement (2)
  │       ReceiverUserID: act.ActivityUserID (获得声望者)
  │       TriggerUserID:  act.TriggerUserID
  │       ObjectType:     AnswerObjectType
  │     }
  │     → 仅在 triggerUserID != receiverUserID 时发送
  │
  └─→ 收件箱通知（循环每个 Activity）:
        NotificationMsg {
          Type: Inbox (1)
          ReceiverUserID: act.ActivityUserID
          TriggerUserID:  op.TriggerUserID (采纳操作者)
          ObjectID:       op.AnswerObjectID
          ObjectType:     AnswerObjectType
          NotificationAction: NotificationAcceptAnswer
        }
        → 仅在 act.ActivityUserID != op.QuestionUserID 时发送
          (即不通知问题作者自己"你的回答被采纳"，因为问题作者=采纳者)
```

**接收者**：回答作者获得成就通知；除问题作者外的人获得 Inbox 通知。采纳仅走内部通知队列，**不发送外部通知**。

---

### 1.3 修订审核触发路径

**事件源**：[revision_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/content/revision_service.go#L168-L179)

修订审核通过时投递 **成就通知**：

```
RevisionService 审核通过后:
  NotificationMsg {
    Type: Achievement (2)
    TriggerUserID:  req.UserID (审核者)
    ReceiverUserID: revisioninfo.UserID (修订提交者)
    ObjectID:       revisioninfo.ObjectID
    ObjectType:     推断出的对象类型
  }
  → 发送到 notificationQueue
```

**回答修订审核通过后额外发送 Inbox 通知**（[revision_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/content/revision_service.go#L270-L278)）：

```
NotificationMsg {
  Type: Inbox (1)
  TriggerUserID:  revisionitem.UserID (修订提交者)
  ReceiverUserID: questionInfo.UserID (问题作者)
  ObjectID:       answerinfo.ID
  ObjectType:     AnswerObjectType
  NotificationAction: NotificationUpdateAnswer
}
```

**接收者**：成就通知 → 修订提交者；Inbox 通知 → 问题作者（回答编辑审核通过时）。仅走内部通知队列。

---

### 1.4 徽章授予触发路径

**事件源**：[badge_award_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/badge/badge_award_service.go#L148-L191)

```
BadgeAwardService.Award():
  NotificationMsg {
    TriggerUserID:      badgeAward.UserID (自己给自己)
    ReceiverUserID:     badgeAward.UserID
    Type:               Achievement (2)
    ObjectID:           badgeAward.ID (授予记录ID，非徽章ID)
    ObjectType:         BadgeAwardObjectType
    Title:              badgeData.Name (徽章名称的 i18n key)
    ExtraInfo:          {"badge_id": badgeData.ID}
    NotificationAction: NotificationEarnedBadge
  }
  → 发送到 notificationQueue
```

**接收者**：被授予徽章的用户自己。仅走内部通知队列。徽章授予在 `AddNotification` 中会额外写入 **徽章二级提醒缓存**（详见第六节）。

---

### 1.5 审核分流触发路径

**事件源**：[review_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/review/review_service.go#L270-L389)

审核通过（`UpdateReview` → `updateObjectStatus`）根据对象类型走不同分支：

#### 问题审核通过

```
→ 更新问题状态为 Available
→ externalNotificationQueueService.Send(
      CreateNewQuestionNotificationMsg(questionInfo.ID, ...))
  // 走外部通知队列，触发新问题邮件通知给关注标签/订阅用户
→ vectorSyncService.Send (索引更新)
```

**注意**：问题审核通过 **只走外部通知队列**，不走内部 notificationQueue。问题本身无 "你的问题已通过" 的 Inbox 通知。

#### 回答审核通过

```
→ 更新回答状态为 Available
→ notificationAnswerTheQuestion(questionUserID, questionID, answerID, ...)
  // 走内部通知队列：NotificationAnswerTheQuestion 给问题作者
  // 同时走外部通知队列：邮件给问题作者
→ vectorSyncService.Send (索引更新)
```

**接收者**：问题作者。与正常回答路径相同。

#### 评论审核通过

```
→ 更新评论状态为 Available
→ notificationCommentOnTheQuestion(comment)
  // 根据评论对象类型分发：
  //   对问题的评论 → notificationQuestionComment → 问题作者
  //   对回答的评论 → notificationAnswerComment → 回答作者
  //   回复评论     → notificationCommentReply → NotificationReplyToYou
  // 走内部通知队列 + 外部通知队列
```

**接收者**：被评论对象作者 / 被回复评论者。与正常评论路径相同。

---

## 二、外部 Handler 展开：NewComment 与 InviteAnswer

### 2.1 NewComment 分发分支

[external_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/external_notification.go#L86-L88)

```
Handler() 检测 msg.NewCommentTemplateRawData != nil
  → handleNewCommentNotification(ctx, msg)
```

[new_comment_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/new_comment_notification.go#L32-L53) 展开流程：

```
1. 查询用户偏好: userNotificationConfigRepo.GetByUserIDAndSource(
     ctx, msg.ReceiverUserID, constant.InboxSource)
   - 不存在 → 直接返回（不发送邮件）

2. 解析渠道配置: channels = NewNotificationChannelsFormJson(config.Channels)

3. 遍历渠道:
   - channel.Enable == false → 跳过
   - channel.Key == EmailChannel →
       sendNewCommentNotificationEmail(userID, email, lang, rawData)
         ├─ checkUserStatusBeforeNotification(userID)
         │    → 用户不存在/已封禁 → 返回
         ├─ 按用户语言渲染邮件模板:
         │    title, body = emailService.NewCommentTemplate(ctx, rawData)
         └─ 发送邮件:
              emailService.SendAndSaveCodeWithTime(
                ctx, userID, email, title, body,
                rawData.UnsubscribeCode,    // 退订码
                codeContent.ToJSONString(), // 含 InboxSource 退订来源
                1*24*time.Hour)             // 退订码有效期24小时
```

**NewCommentTemplateRawData 包含**：
- `QuestionTitle` / `QuestionID` — 关联问题
- `AnswerID` — 关联回答（评论回答时）
- `CommentID` / `CommentSummary` — 评论内容
- `CommentUserDisplayName` — 评论者昵称
- `UnsubscribeCode` — 退订码

### 2.2 InviteAnswer 分发分支

[external_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/external_notification.go#L92-L94)

```
Handler() 检测 msg.NewInviteAnswerTemplateRawData != nil
  → handleInviteAnswerNotification(ctx, msg)
```

[invite_answer_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/invite_answer_notification.go#L32-L82) 展开流程：

```
1. 查询用户偏好: userNotificationConfigRepo.GetByUserIDAndSource(
     ctx, msg.ReceiverUserID, constant.InboxSource)
   - 不存在 → 直接返回

2. 解析渠道配置并遍历:
   - EmailChannel 启用 →
       sendInviteAnswerNotificationEmail(userID, email, lang, rawData)
         ├─ checkUserStatusBeforeNotification(userID)
         ├─ 按用户语言渲染邮件模板:
         │    title, body = emailService.NewInviteAnswerTemplate(ctx, rawData)
         └─ 发送邮件:
              emailService.SendAndSaveCodeWithTime(
                ctx, userID, email, title, body,
                rawData.UnsubscribeCode,
                codeContent.ToJSONString(), // 含 InboxSource 退订来源
                1*24*time.Hour)
```

**NewInviteAnswerTemplateRawData 包含**：
- `InviterDisplayName` — 邀请者昵称
- `QuestionTitle` / `QuestionID` — 关联问题
- `UnsubscribeCode` — 退订码

**关键差异**：NewComment 和 InviteAnswer 都使用 `InboxSource` 查偏好；而 NewQuestion 使用 `AllNewQuestionSource` 和 `AllNewQuestionForFollowingTagsSource` 两个来源查偏好。

---

## 三、红点机制

### 3.1 Redis 键结构

定义在 [cache_key.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/base/constant/cache_key.go#L53-L54)：

```go
RedDotCacheKey  = "answer:red-dot:%s:%s"   // 格式: answer:red-dot:{type}:{userID}
RedDotCacheTime = 30 * 24 * time.Hour       // TTL: 30天
```

### 3.2 三类红点键

| Redis 键 | 类型参数 | 通知分类 | 值类型 | 用途 |
|----------|---------|---------|--------|------|
| `answer:red-dot:inbox:{userID}` | `NotificationTypeInbox` ("inbox") | 收件箱 | int64 (未读数) | 站内收件箱红点 |
| `answer:red-dot:achievement:{userID}` | `NotificationTypeAchievement` ("achievement") | 成就 | int64 (未读数) | 站内成就红点 |
| `answer:red-dot:badge:{userID}` | `NotificationTypeBadgeAchievement` ("badge") | 徽章授予 | string (JSON) | 徽章二级提醒弹窗 |

**键生成逻辑**在 [notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L222-L244)：

```go
// Inbox 红点
key = fmt.Sprintf("answer:red-dot:inbox:%s", userID)
// Achievement 红点
key = fmt.Sprintf("answer:red-dot:achievement:%s", userID)
// Badge 弹窗
key = fmt.Sprintf("answer:red-dot:badge:%s", userID)
```

### 3.3 Inbox 与 Achievement 分类

通知实体 `Type` 字段（[notification_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/entity/notification_entity.go#L32)）：

| Type 值 | 含义 | 红点键 | 说明 |
|---------|------|--------|------|
| 1 | Inbox（收件箱） | `answer:red-dot:inbox:{userID}` | 包含帖子、投票、邀请等交互通知 |
| 2 | Achievement（成就） | `answer:red-dot:achievement:{userID}` | 声望变化、投票获得的成就 |

前端分类（`NotificationType` 映射）：
- `inbox` → Type=1 → 红点键用 `inbox`
- `achievement` → Type=2 → 红点键用 `achievement`

### 3.4 红点操作

| 操作 | 方法 | 行为 |
|------|------|------|
| **新增通知时** | [addRedDot](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L222-L244) | 键存在 → `Increase(ctx, key, 1)`；键不存在 → `SetInt64(ctx, key, 1, 30天)` |
| **标记已读时** | [DecreaseRedDot](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L246-L268) | 键存在 → `Decrease(ctx, key, 1)`；减到 ≤0 → `DeleteRedDot` |
| **清除全部** | [DeleteRedDot](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L270-L282) | 直接 `Cache.Del(ctx, key)` |

### 3.5 前端获取与清空入口

[notification_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/controller/notification_controller.go)

| API 路由 | 方法 | 功能 |
|----------|------|------|
| `GET /answer/api/v1/notification/status` | [GetRedDot](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/controller/notification_controller.go#L58-L77) | 获取三类红点状态（Inbox数、Achievement数、徽章弹窗、审核数） |
| `PUT /answer/api/v1/notification/status` | [ClearRedDot](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/controller/notification_controller.go#L89-L110) | 清空指定类型的红点（传 `type: "inbox"` 或 `"achievement"`） |
| `PUT /answer/api/v1/notification/read/state/all` | [ClearUnRead](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/controller/notification_controller.go#L122-L130) | 批量标记某类型全部已读 + 清红点 |
| `PUT /answer/api/v1/notification/read/state` | [ClearIDUnRead](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/controller/notification_controller.go#L142-L150) | 标记单条通知已读 → DecreaseRedDot → 移除徽章弹窗缓存 |
| `GET /answer/api/v1/notification/page` | [GetList](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/controller/notification_controller.go#L165-L173) | 分页查询通知列表（type=inbox/achievement, inbox_type=all/posts/votes/invites） |

**GetRedDot 返回结构**（[RedDot](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/schema/notification_schema.go#L104-L110)）：

```json
{
  "inbox": 5,          // 收件箱未读数 (int64)
  "achievement": 3,    // 成就未读数 (int64)
  "revision": 2,       // 待审核数量 (int64, 仅管理员/审核员可见)
  "can_revision": true, // 是否有审核权限
  "badge_award": {     // 徽章授予弹窗 (仅最近一个)
    "notification_id": "...",
    "badge_id": "...",
    "name": "金徽章",
    "icon": "url",
    "level": 3
  }
}
```

---

## 四、插件同步仅对收件箱类型生效

[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L216-L219)

```go
if msg.Type == schema.NotificationTypeInbox {
    ns.syncNotificationToPlugin(ctx, objInfo, msg)
}
```

**原因**：成就类型通知（Type=2）是系统内部的声望/投票统计，无业务语义，不应推送到飞书、钉钉等外部系统。只有收件箱类型（回答、评论、@、邀请等交互行为）才对第三方插件有意义。

**syncNotificationToPlugin 流程**（[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L359-L445)）：

```
1. 获取站点信息（general + seo + interface）
2. 构建 plugin.NotificationMessage:
   - Type:           plugin.NotificationType(msg.NotificationAction)
   - ReceiverUserID: msg.ReceiverUserID
   - TriggerUserID:  msg.TriggerUserID
   - QuestionTitle:  objInfo.Title
   - QuestionUrl / AnswerUrl / CommentUrl (根据 permalink 配置)
3. 补充 TriggerUser 信息 (DisplayName, UserUrl)
4. 补充 ReceiverLang (用户语言 → 站点默认语言)
5. 补充 ReceiverExternalID (用户 SSO 外部ID, 优先匹配插件 slug)
6. plugin.CallNotification(fn -> fn.Notify(pluginNotificationMsg))
```

---

## 五、徽章二级提醒缓存

### 5.1 写入时机

[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L207-L212)

```go
if req.ObjectInfo.ObjectType == constant.BadgeAwardObjectType {
    err = ns.AddBadgeAwardAlertCache(ctx, info.UserID, info.ID, req.ObjectInfo.ObjectMap["badge_id"])
}
```

仅在通知对象类型为 `BadgeAwardObjectType` 时写入，与成就通知的 RedDot 是 **两套独立缓存**。

### 5.2 缓存结构

[AddBadgeAwardAlertCache](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L284-L306)

```
Redis 键: answer:red-dot:badge:{userID}
Redis 值: JSON 格式的 RedDotBadgeAwardCache

RedDotBadgeAwardCache {
  BadgeAwardList: map[string]*RedDotBadgeAward {
    "{notificationID}": {
      NotificationID: "...",
      BadgeID:        "...",
      Name:           "",  // 前端展示时才填充
      Icon:           "",  // 前端展示时才填充
      Level:          0    // 前端展示时才填充
    }
  }
}
```

**设计意图**：用户可能同时被授予多个徽章，缓存使用 map 按 notificationID 存储，支持多个待弹窗徽章。前端 GetRedDot 时只展示排序后最早的一个（[GetBadgeAward](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/schema/notification_schema.go#L132-L142)），用户标记已读后移除该条目。

### 5.3 读取与清除

- **读取**：[getBadgeAward](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/notification_service.go#L99-L128) — GetRedDot 时读取，补充 Name/Icon/Level
- **移除单条**：[RemoveBadgeAwardAlertCache](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L308-L325) — ClearIDUnRead 时调用，删除指定 notificationID 的条目；如果 BadgeAwardList 为空则删除整个键
- **标记已读**：ClearIDUnRead 同时 DecreaseRedDot + RemoveBadgeAwardAlertCache

---

## 六、动作映射表未命中时 MsgType 落零的副作用

### 6.1 映射逻辑

[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L194-L198)

```go
info.MsgType = 0  // Go 零值
_, ok := constant.NotificationMsgTypeMapping[req.NotificationAction]
if ok {
    info.MsgType = constant.NotificationMsgTypeMapping[req.NotificationAction]
}
```

[NotificationMsgTypeMapping](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/base/constant/notification.go#L82-L102) 覆盖的动作：

| 映射值 | 包含的 NotificationAction |
|--------|--------------------------|
| 1 (帖子) | UpdateQuestion, AnswerTheQuestion, UpdateAnswer, AcceptAnswer, CommentQuestion, CommentAnswer, ReplyToYou, MentionYou, YourQuestionIsClosed, YourQuestionWasDeleted, YourAnswerWasDeleted, YourCommentWasDeleted |
| 2 (投票) | UpVotedTheQuestion, DownVotedTheQuestion, UpVotedTheAnswer, DownVotedTheAnswer, UpVotedTheComment |
| 3 (邀请) | InvitedYouToAnswer |

**未覆盖的动作**：`NotificationEarnedBadge`（徽章授予）不在映射表中。

### 6.2 副作用分析

**实体定义**（[notification_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/entity/notification_entity.go#L33)）：

```go
MsgType int `xorm:"not null default 0 INT(11) msg_type"`
```

数据库默认值为 0。当 `NotificationMsgTypeMapping` 未命中时，Go 零值 0 直接写入数据库。

**副作用**：

1. **前端分类查询失效**：`GetNotificationPage` 按 InboxType 过滤时，`MsgType=0` 的通知不属于任何子分类（posts=1, votes=2, invites=3），在 `inbox_type=all` 之外的所有筛选条件（posts/votes/invites）中 **不可见**，只能通过 `all` 才能看到。

2. **徽章授予通知的 MsgType=0**：`NotificationEarnedBadge` 不在映射表中，所以所有徽章授予通知的 MsgType 都是 0。这意味着在收件箱页面的"帖子"/"投票"/"邀请"筛选中都 **看不到** 徽章通知——尽管徽章通知类型是 Achievement（Type=2），出现在"成就"标签页中，但成就标签页没有 `inbox_type` 子筛选，所以实际影响有限。

3. **成就通知通用问题**：所有成就类通知（投票获得声望、修订审核通过、采纳获得声望等）都没有设置 `NotificationAction`，因此 `MsgType` 也是 0。但成就通知出现在 Achievement 标签页而非 Inbox 标签页，Inbox 的子分类筛选不影响它们。

**结论**：MsgType 落零的最大影响是徽章授予通知如果在 Inbox 标签页中展示，则无法通过子分类筛选到。当前架构下，成就类通知在独立标签页展示，影响有限，但如果未来在 Inbox 中增加新的通知动作且忘记更新映射表，则会产生前端子分类丢失的问题。

---

## 七、双队列独立投递协作图

### 7.1 核心事实

业务服务在完成操作后，**同时独立投递** 两个队列：

- **notificationQueue**（内部通知队列）：处理站内通知 → 数据库写入 + 红点 + 关注者推送 + 插件同步
- **externalNotificationQueue**（外部通知队列）：处理邮件/插件通知 → 偏好检查 → 邮件发送

两个队列 **互不依赖**，各自有独立的 handler，各自独立消费。不存在"内部通知成功后再发外部通知"的串行依赖。

### 7.2 协作图

```
                              ┌──────────────────────────────────────┐
                              │           业务操作层                   │
                              │                                      │
                              │  question_service                    │
                              │  answer_service                      │
                              │  comment_service                     │
                              │  vote_repo                           │
                              │  answer_activity_repo                │
                              │  badge_award_service                 │
                              │  revision_service                    │
                              │  review_service                      │
                              └─────┬────────────────────┬───────────┘
                                    │                    │
                     同时独立投递    │                    │  同时独立投递
                     (1)内部通知     │                    │  (2)外部通知
                                    │                    │
                                    ▼                    ▼
              ┌──────────────────────────────┐  ┌──────────────────────────────────┐
              │   notificationQueue          │  │   externalNotificationQueue      │
              │   (缓冲128, 泛型 NotificationMsg)│  │   (缓冲128, 泛型 ExternalNotificationMsg)│
              │                              │  │                                  │
              │   Handler:                   │  │   Handler:                       │
              │   NotificationCommon.        │  │   ExternalNotificationService.   │
              │     AddNotification()        │  │     Handler()                    │
              └──────────────┬───────────────┘  └───────────────┬──────────────────┘
                             │                                  │
                             ▼                                  ▼
              ┌──────────────────────────────┐  ┌──────────────────────────────────┐
              │  AddNotification 处理流程:    │  │  Handler 分发:                   │
              │                              │  │                                  │
              │  1. RankAgent 跳过检查       │  │  NewQuestionTemplateRawData?     │
              │  2. 获取对象信息             │  │    → handleNewQuestion           │
              │  3. 成就去重:                │  │       → getNewQuestionSubscribers│
              │     (userID,objectID,type)   │  │       → 标签关注者+全量订阅者    │
              │     存在→更新Rank返回        │  │       → 频率限制检查             │
              │     不存在→继续创建          │  │       → 渠道偏好检查 → 发邮件    │
              │  4. 写入 DB (notification)   │  │                                  │
              │  5. MsgType 映射(可能落零)   │  │  NewCommentTemplateRawData?      │
              │  6. addRedDot (Redis计数)    │  │    → handleNewComment            │
              │  7. 异步 SendToAllFollower   │  │       → InboxSource偏好检查      │
              │     (NoNeedPushAllFollow=true│  │       → 渠道偏好检查 → 发邮件    │
              │      防递归)                 │  │                                  │
              │  8. syncNotificationToPlugin │  │  NewAnswerTemplateRawData?       │
              │     (仅 Inbox 类型)          │  │    → handleNewAnswer             │
              │                              │  │       → InboxSource偏好检查      │
              │  额外:                       │  │       → 渠道偏好检查 → 发邮件    │
              │  · BadgeAwardObjectType →    │  │                                  │
              │    AddBadgeAwardAlertCache   │  │  NewInviteAnswerTemplateRawData? │
              │    (徽章二级提醒)            │  │    → handleInviteAnswer          │
              └──────────────────────────────┘  │       → InboxSource偏好检查      │
                                                │       → 渠道偏好检查 → 发邮件    │
                                                └──────────────────────────────────┘
```

### 7.3 各事件源的双队列投递矩阵

| 触发事件 | 内部通知队列 | 外部通知队列 | eventQueue |
|----------|:-----------:|:-----------:|:----------:|
| **新问题** | — | ✅ `CreateNewQuestionNotificationMsg` | ✅ EventQuestionCreate |
| **新回答** | ✅ NotificationAnswerTheQuestion → 问题作者 | ✅ NewAnswerTemplateRawData → 问题作者 | ✅ EventAnswerCreate |
| **回答更新** | ✅ NotificationUpdateAnswer → 问题作者 | — | ✅ EventAnswerUpdate |
| **采纳回答** | ✅ Achievement + NotificationAcceptAnswer | — | ✅ EventQuestionAccept |
| **评论问题** | ✅ NotificationCommentQuestion → 问题作者 | ✅ NewCommentTemplateRawData → 问题作者 | — |
| **评论回答** | ✅ NotificationCommentAnswer → 回答作者 | ✅ NewCommentTemplateRawData → 回答作者 | — |
| **回复评论** | ✅ NotificationReplyToYou → 被回复者 | ✅ NewCommentTemplateRawData → 被回复者 | — |
| **@提及** | ✅ NotificationMentionYou → 被@者 | — | — |
| **邀请回答** | ✅ NotificationInvitedYouToAnswer → 被邀请者 | ✅ NewInviteAnswerTemplateRawData → 被邀请者 | — |
| **投票** | ✅ Inbox(投票通知) + Achievement(声望) | — | ✅ EventQuestionVote / EventAnswerVote |
| **徽章授予** | ✅ Achievement(NotificationEarnedBadge) | — | — |
| **修订审核通过** | ✅ Achievement + 可能的 Inbox(回答编辑) | — | — |
| **审核通过-问题** | — | ✅ CreateNewQuestionNotificationMsg | — |
| **审核通过-回答** | ✅ NotificationAnswerTheQuestion | ✅ NewAnswerTemplateRawData | — |
| **审核通过-评论** | ✅ Comment/Reply 通知 | ✅ NewCommentTemplateRawData | — |
| **问题/回答/评论删除** | ✅ NotificationYourXxxWasDeleted → 作者 | — | — |
| **问题关闭** | ✅ NotificationYourQuestionIsClosed → 作者 | — | — |

**图例说明**：
- ✅ = 投递到该队列
- — = 不投递到该队列
- 内部通知队列和外部通知队列是 **独立并行** 投递的
- eventQueue 是独立的事件总线，与通知无直接依赖关系

### 7.4 关键协作约束

1. **双队列独立投递**：业务代码在同一个方法中分别构造 `NotificationMsg` 和 `ExternalNotificationMsg`，各自调用 `Send()`。两个队列各自消费，不存在先后依赖。

2. **关注者推送仅走内部队列**：`SendNotificationToAllFollower` 仅向 `notificationQueue` 投递复制的消息（`NoNeedPushAllFollow=true`），不触发外部邮件。

3. **插件同步双通道**：
   - 内部队列的 `syncNotificationToPlugin` 仅对 Inbox 类型生效，推送站内交互事件到插件
   - 外部队列的 `handleNewQuestionNotification` 也会调用 `syncNewQuestionNotificationToPlugin`，独立推送新问题事件到插件

4. **偏好检查仅在外部队列**：用户的通知偏好（email 开关等）仅在外部通知消费时检查。内部通知 **无条件写入数据库**，用户可在前端查看。

5. **去重仅对成就类型**：内部队列的 `AddNotification` 仅对 Type=Achievement 做 `(userID, objectID, type)` 去重；Inbox 类型不做去重，每次操作都产生新记录。
