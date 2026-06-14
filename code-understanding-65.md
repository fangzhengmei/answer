# 通知生成与发送系统代码分析

## 一、系统架构概览

通知系统采用 **事件驱动 + 双队列异步处理** 架构，主要包含以下核心组件：

```
业务操作 → 事件触发 → 队列投递 → 消费者处理 → 偏好检查 → 去重检查 → 通知存储/发送
```

### 核心模块关系图

| 模块 | 职责 | 关键文件 |
|------|------|----------|
| 事件触发层 | 业务操作完成后发送事件和通知消息 | [question_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/content/question_service.go)、[answer_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/content/answer_service.go)、[comment_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/comment/comment_service.go) |
| 事件队列 | 通用事件总线，处理徽章、审核等跨领域事件 | [event_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/eventqueue/event_queue.go) |
| 内部通知队列 | 站内通知（收件箱、成就）异步处理 | [notice_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/noticequeue/notice_queue.go) |
| 外部通知队列 | 邮件、插件通知等外部渠道异步处理 | [notice_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/noticequeue/notice_queue.go) |
| 通知处理核心 | 通知去重、存储、红点更新、粉丝推送 | [notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go) |
| 外部通知处理 | 邮件模板渲染、用户偏好检查 | [external_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/external_notification.go) |
| 用户通知配置 | 通知偏好设置管理 | [user_notification_config_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/user_notification_config/user_notification_config_service.go) |

---

## 二、事件触发机制

### 2.1 事件类型定义

事件类型采用 `对象.动作` 命名方式，定义在 [event.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/base/constant/event.go#L45-L75)：

```go
// 问题相关事件
EventQuestionCreate  // 问题创建
EventQuestionUpdate  // 问题更新
EventQuestionDelete  // 问题删除
EventQuestionVote    // 问题投票
EventQuestionAccept  // 问题采纳答案

// 回答相关事件
EventAnswerCreate    // 回答创建
EventAnswerUpdate    // 回答更新
EventAnswerDelete    // 回答删除
EventAnswerVote      // 回答投票

// 评论相关事件
EventCommentCreate   // 评论创建
EventCommentUpdate   // 评论更新
EventCommentDelete   // 评论删除
EventCommentVote     // 评论投票
```

### 2.2 事件触发流程

#### 新问题创建触发

在 [question_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/content/question_service.go#L450-L465) 中：

```go
// 1. 发送外部通知（邮件给关注标签用户和订阅用户）
qs.externalNotificationQueueService.Send(ctx,
    schema.CreateNewQuestionNotificationMsg(question.ID, question.Title, question.UserID, newTags))

// 2. 发送事件到事件总线
qs.eventQueueService.Send(ctx, schema.NewEvent(constant.EventQuestionCreate, req.UserID).
    TID(question.ID).QID(question.ID, question.UserID))
```

#### 新回答创建触发

在 [answer_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/content/answer_service.go#L341-L345) 中：

```go
// 1. 发送活动队列消息
as.activityQueueService.Send(ctx, &schema.ActivityMsg{...})

// 2. 发送事件到事件总线
as.eventQueueService.Send(ctx, schema.NewEvent(constant.EventAnswerCreate, req.UserID).
    TID(insertData.ID).AID(insertData.ID, insertData.UserID))

// 3. 发送内部通知给问题作者（在 notificationAnswerTheQuestion 方法中）
as.notificationQueueService.Send(ctx, msg)

// 4. 发送外部邮件通知
as.externalNotificationQueueService.Send(ctx, externalNotificationMsg)
```

#### 新评论创建触发

在 [comment_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/comment/comment_service.go#L574-L612) 中：

```go
// 1. 发送内部通知给问题/回答作者
msg := &schema.NotificationMsg{
    ReceiverUserID: questionUserID,  // 或 answerUserID
    TriggerUserID:  commentUserID,
    Type:           schema.NotificationTypeInbox,
    ObjectID:       commentID,
}
cs.notificationQueueService.Send(ctx, msg)

// 2. 发送外部邮件通知
externalNotificationMsg := &schema.ExternalNotificationMsg{
    ReceiverUserID: receiverUserInfo.ID,
    ReceiverEmail:  receiverUserInfo.EMail,
    ReceiverLang:   receiverUserInfo.Language,
    NewCommentTemplateRawData: rawData,
}
cs.externalNotificationQueueService.Send(ctx, externalNotificationMsg)
```

### 2.3 消息结构

#### EventMsg - 事件总线消息

定义在 [event_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/schema/event_schema.go#L28-L44)：

```go
type EventMsg struct {
    EventType       constant.EventType  // 事件类型
    UserID          string              // 触发用户ID
    TriggerObjectID string              // 触发对象ID
    QuestionID      string              // 关联问题ID
    QuestionUserID  string              // 问题作者ID
    AnswerID        string              // 关联回答ID
    AnswerUserID    string              // 回答作者ID
    CommentID       string              // 关联评论ID
    CommentUserID   string              // 评论作者ID
    ExtraInfo       map[string]string   // 额外信息
}
```

#### NotificationMsg - 内部通知消息

定义在 [notification_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/schema/notification_schema.go#L76-L95)：

```go
type NotificationMsg struct {
    TriggerUserID       string              // 触发者用户ID
    ReceiverUserID      string              // 接收者用户ID
    Type                int                 // 1=收件箱 2=成就
    Title               string              // 通知标题
    ObjectID            string              // 关联对象ID
    ObjectType          string              // 关联对象类型
    NotificationAction  string              // 通知动作类型
    NoNeedPushAllFollow bool                // 是否不需要推送给所有关注者
    ExtraInfo           map[string]string   // 额外信息
}
```

#### ExternalNotificationMsg - 外部通知消息

定义在 [new_question_queue_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/schema/new_question_queue_schema.go#L27-L36)：

```go
type ExternalNotificationMsg struct {
    ReceiverUserID                  string                          // 接收者用户ID
    ReceiverEmail                   string                          // 接收者邮箱
    ReceiverLang                    string                          // 接收者语言
    NewAnswerTemplateRawData        *NewAnswerTemplateRawData      // 新回答模板数据
    NewInviteAnswerTemplateRawData  *NewInviteAnswerTemplateRawData // 邀请回答模板数据
    NewCommentTemplateRawData       *NewCommentTemplateRawData     // 新评论模板数据
    NewQuestionTemplateRawData      *NewQuestionTemplateRawData    // 新问题模板数据
}
```

---

## 三、接收者计算逻辑

### 3.1 新问题通知接收者

在 [new_question_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/new_question_notification.go#L74-L137) 的 `getNewQuestionSubscribers` 方法中：

```go
func (ns *ExternalNotificationService) getNewQuestionSubscribers(ctx context.Context, msg *schema.ExternalNotificationMsg) (
    subscribers []*NewQuestionSubscriber, err error) {
    
    subscribersMapping := make(map[string]*NewQuestionSubscriber)

    // 1. 获取所有关注该问题标签的用户
    for _, tagID := range msg.NewQuestionTemplateRawData.TagIDs {
        userIDs, err := ns.followRepo.GetFollowUserIDs(ctx, tagID)
        // ...去重处理
    }
    // 查询这些用户的通知偏好配置
    userNotificationConfigs, err := ns.userNotificationConfigRepo.GetByUsersAndSource(
        ctx, tagsFollowerIDs, constant.AllNewQuestionForFollowingTagsSource)

    // 2. 获取所有订阅"全部新问题"的用户
    notificationConfigs, err := ns.userNotificationConfigRepo.GetBySource(
        ctx, constant.AllNewQuestionSource)

    // 3. 排除问题作者自己
    delete(subscribersMapping, msg.NewQuestionTemplateRawData.QuestionAuthorUserID)
    
    // 4. 发送频率限制检查
    if ns.checkSendNewQuestionNotificationEmailLimit(ctx, notificationConfig.UserID) {
        continue  // 超过当日发送限制则跳过
    }
    
    return subscribers, nil
}
```

### 3.2 新回答通知接收者

在 [answer_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/content/answer_service.go#L763-L800) 的 `notificationAnswerTheQuestion` 方法中：

```go
func (as *AnswerService) notificationAnswerTheQuestion(ctx context.Context,
    questionUserID, questionID, answerID, answerUserID, questionTitle, answerSummary string) {
    
    // 如果是自问自答，不发送通知
    if questionUserID == answerUserID {
        return
    }
    
    // 接收者 = 问题作者
    msg.ReceiverUserID = questionUserID
    msg.TriggerUserID = answerUserID
    msg.NotificationAction = constant.NotificationAnswerTheQuestion
    
    as.notificationQueueService.Send(ctx, msg)
    // ... 发送外部邮件通知
}
```

### 3.3 新评论通知接收者

在 [comment_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/comment/comment_service.go#L568-L661) 中：

```go
// 评论问题时：接收者 = 问题作者
func (cs *CommentService) notificationQuestionComment(ctx context.Context, questionUserID, ...) {
    if questionUserID == commentUserID {
        return  // 自己评论自己的问题不通知
    }
    msg.ReceiverUserID = questionUserID
    msg.NotificationAction = constant.NotificationCommentQuestion
    // ...
}

// 评论回答时：接收者 = 回答作者
func (cs *CommentService) notificationAnswerComment(ctx context.Context, answerUserID, ...) {
    if answerUserID == commentUserID {
        return  // 自己评论自己的回答不通知
    }
    msg.ReceiverUserID = answerUserID
    msg.NotificationAction = constant.NotificationCommentAnswer
    // ...
}

// 回复评论时：接收者 = 被回复的评论作者
func (cs *CommentService) notificationCommentReply(ctx context.Context, replyUserID, ...) {
    if replyUserID == commentUserID {
        return  // 自己回复自己不通知
    }
    msg.ReceiverUserID = replyUserID
    msg.NotificationAction = constant.NotificationReplyToYou
    // ...
}
```

### 3.4 @提及通知接收者

在评论中 @ 某用户时，接收者为被 @ 的用户，通知动作为 `NotificationMentionYou`。

### 3.5 关注者推送

在 [notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L327-L357) 的 `SendNotificationToAllFollower` 方法中：

```go
func (ns *NotificationCommon) SendNotificationToAllFollower(ctx context.Context, 
    msg *schema.NotificationMsg, questionID string) {
    
    // 仅对特定动作推送给关注者
    if msg.NotificationAction != constant.NotificationUpdateQuestion &&
        msg.NotificationAction != constant.NotificationAnswerTheQuestion &&
        msg.NotificationAction != constant.NotificationUpdateAnswer &&
        msg.NotificationAction != constant.NotificationAcceptAnswer {
        return
    }
    
    // 获取关注该问题的所有用户
    userIDs, err := ns.followRepo.GetFollowUserIDs(ctx, condObjectID)
    
    // 给每个关注者发送通知
    for _, userID := range userIDs {
        t := &schema.NotificationMsg{}
        _ = copier.Copy(t, msg)
        t.ReceiverUserID = userID
        t.NoNeedPushAllFollow = true  // 防止递归推送
        ns.notificationQueueService.Send(ctx, t)
    }
}
```

---

## 四、队列系统详解

### 4.1 队列基础架构

队列实现在 [queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/base/queue/queue.go) 中，采用泛型设计的内存队列：

```go
type Queue[T any] struct {
    name    string
    queue   chan T                        // 带缓冲的channel
    handler func(ctx context.Context, msg T) error
    mu      sync.RWMutex
    closed  bool
    wg      sync.WaitGroup
}

// 后台worker协程处理消息
func (q *Queue[T]) startWorker() {
    q.wg.Add(1)
    go func() {
        defer q.wg.Done()
        for msg := range q.queue {
            q.processMessage(msg)  // 调用注册的handler处理
        }
    }()
}
```

### 4.2 三种队列实例

#### (1) 事件队列 (eventqueue)

定义在 [event_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/eventqueue/event_queue.go)：

```go
func NewService() Service {
    return queue.New[*schema.EventMsg]("event", 128)  // 缓冲大小128
}
```

**注册的处理器**：
- 徽章事件处理器：[badge_event_handler.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/badge/badge_event_handler.go#L60)

#### (2) 内部通知队列 (noticequeue)

定义在 [notice_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/noticequeue/notice_queue.go#L27-L31)：

```go
type Service queue.Service[*schema.NotificationMsg]

func NewService() Service {
    return queue.New[*schema.NotificationMsg]("notification", 128)
}
```

**注册的处理器**：
- 通知核心处理器：在 [notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L96) 中注册 `AddNotification` 方法

#### (3) 外部通知队列 (noticequeue.ExternalService)

定义在 [notice_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/noticequeue/notice_queue.go#L33-L37)：

```go
type ExternalService queue.Service[*schema.ExternalNotificationMsg]

func NewExternalService() ExternalService {
    return queue.New[*schema.ExternalNotificationMsg]("external_notification", 128)
}
```

**注册的处理器**：
- 外部通知处理器：在 [external_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/external_notification.go#L70) 中注册 `Handler` 方法

### 4.3 队列处理流程

#### 内部通知处理流程

在 [notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L100-L220) 的 `AddNotification` 方法中：

```go
func (ns *NotificationCommon) AddNotification(ctx context.Context, msg *schema.NotificationMsg) (err error) {
    
    // 1. 成就类型通知：如果启用了Rank Agent则跳过
    if msg.Type == schema.NotificationTypeAchievement && plugin.RankAgentEnabled() {
        return nil
    }
    
    // 2. 构建通知内容
    req := &schema.NotificationContent{
        TriggerUserID:      msg.TriggerUserID,
        ReceiverUserID:     msg.ReceiverUserID,
        ObjectInfo:         schema.ObjectInfo{...},
        NotificationAction: msg.NotificationAction,
        Type:               msg.Type,
    }
    
    // 3. 获取对象详细信息（标题、关联ID等）
    objInfo, err = ns.objectInfoService.GetInfo(ctx, req.ObjectInfo.ObjectID)
    
    // 4. 成就通知去重检查
    if msg.Type == schema.NotificationTypeAchievement {
        // 查询是否已有相同用户+相同对象+相同类型的通知
        notificationInfo, exist, err := ns.notificationRepo.GetByUserIdObjectIdTypeId(
            ctx, req.ReceiverUserID, req.ObjectInfo.ObjectID, req.Type)
        
        if exist {
            // 已存在：更新内容而非新增
            updateContent.Rank = rank
            content, _ := json.Marshal(updateContent)
            notificationInfo.Content = string(content)
            err = ns.notificationRepo.UpdateNotificationContent(ctx, notificationInfo)
            return nil
        }
    }
    
    // 5. 新增通知记录
    info := &entity.Notification{
        UserID:    req.ReceiverUserID,
        Type:      req.Type,
        IsRead:    schema.NotificationNotRead,
        Status:    schema.NotificationStatusNormal,
        ObjectID:  req.ObjectInfo.ObjectID,
        Content:   string(content),  // JSON序列化后的通知内容
        MsgType:   constant.NotificationMsgTypeMapping[req.NotificationAction],
    }
    err = ns.notificationRepo.AddNotification(ctx, info)
    
    // 6. 更新红点计数
    err = ns.addRedDot(ctx, info.UserID, msg.Type)
    
    // 7. 异步推送给所有关注者
    go ns.SendNotificationToAllFollower(ctx, msg, questionID)
    
    // 8. 同步通知到插件
    if msg.Type == schema.NotificationTypeInbox {
        ns.syncNotificationToPlugin(ctx, objInfo, msg)
    }
    
    return nil
}
```

#### 外部通知处理流程

在 [external_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/external_notification.go#L74-L97) 的 `Handler` 方法中：

```go
func (ns *ExternalNotificationService) Handler(ctx context.Context, msg *schema.ExternalNotificationMsg) error {
    
    // 根据消息类型分发给不同的处理器
    if msg.NewQuestionTemplateRawData != nil {
        return ns.handleNewQuestionNotification(ctx, msg)
    }
    if msg.NewCommentTemplateRawData != nil {
        return ns.handleNewCommentNotification(ctx, msg)
    }
    if msg.NewAnswerTemplateRawData != nil {
        return ns.handleNewAnswerNotification(ctx, msg)
    }
    if msg.NewInviteAnswerTemplateRawData != nil {
        return ns.handleInviteAnswerNotification(ctx, msg)
    }
    return nil
}
```

以新评论通知为例，在 [new_comment_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/new_comment_notification.go#L32-L53)：

```go
func (ns *ExternalNotificationService) handleNewCommentNotification(ctx context.Context,
    msg *schema.ExternalNotificationMsg) error {
    
    // 1. 查询用户通知偏好配置
    notificationConfig, exist, err := ns.userNotificationConfigRepo.GetByUserIDAndSource(
        ctx, msg.ReceiverUserID, constant.InboxSource)
    
    if !exist {
        return nil  // 没有配置则不发送
    }
    
    // 2. 检查各渠道是否启用
    channels := schema.NewNotificationChannelsFormJson(notificationConfig.Channels)
    for _, channel := range channels {
        if !channel.Enable {
            continue
        }
        if channel.Key == constant.EmailChannel {
            // 3. 检查用户状态（是否被封禁等）
            if unavailable := ns.checkUserStatusBeforeNotification(ctx, userID); unavailable {
                return
            }
            // 4. 渲染邮件模板并发送
            ns.sendNewCommentNotificationEmail(ctx, ...)
        }
    }
    return nil
}
```

---

## 五、去重机制

### 5.1 成就通知去重

在 [notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L147-L173) 中：

```go
if msg.Type == schema.NotificationTypeAchievement {
    // 按 (用户ID, 对象ID, 通知类型) 唯一键查询
    notificationInfo, exist, err := ns.notificationRepo.GetByUserIdObjectIdTypeId(
        ctx, req.ReceiverUserID, req.ObjectInfo.ObjectID, req.Type)
    
    rank, err := ns.activityRepo.GetUserIDObjectIDActivitySum(
        ctx, req.ReceiverUserID, req.ObjectInfo.ObjectID)
    req.Rank = rank
    
    if exist {
        // 已存在：更新Rank值而非创建新通知
        updateContent := &schema.NotificationContent{}
        json.Unmarshal([]byte(notificationInfo.Content), updateContent)
        updateContent.Rank = rank
        content, _ := json.Marshal(updateContent)
        notificationInfo.Content = string(content)
        err = ns.notificationRepo.UpdateNotificationContent(ctx, notificationInfo)
        return nil  // 直接返回，不创建新通知
    }
}
```

**数据库查询**在 [notification_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/notification/notification_repo.go#L99-L106)：

```go
func (nr *notificationRepo) GetByUserIdObjectIdTypeId(ctx context.Context, 
    userID, objectID string, notificationType int) (*entity.Notification, bool, error) {
    
    info := &entity.Notification{}
    exist, err := nr.data.DB.Context(ctx).
        Where("user_id = ?", userID).
        And("object_id = ?", objectID).
        And("type = ?", notificationType).
        Get(info)
    return info, exist, err
}
```

### 5.2 新问题通知接收者去重

在 [new_question_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/new_question_notification.go#L76-L129) 中使用 `subscribersMapping` map 去重：

```go
subscribersMapping := make(map[string]*NewQuestionSubscriber)

// 1. 从标签关注者添加
for _, userNotificationConfig := range userNotificationConfigs {
    if _, ok := subscribersMapping[userNotificationConfig.UserID]; ok {
        continue  // 已存在，跳过
    }
    subscribersMapping[userNotificationConfig.UserID] = ...
}

// 2. 从全部新问题订阅者添加
for _, notificationConfig := range notificationConfigs {
    if _, ok := subscribersMapping[notificationConfig.UserID]; ok {
        continue  // 已存在，跳过（防止同一用户同时在两个来源中）
    }
    subscribersMapping[notificationConfig.UserID] = ...
}

// 3. 排除问题作者
delete(subscribersMapping, msg.NewQuestionTemplateRawData.QuestionAuthorUserID)
```

### 5.3 关注者推送防递归

在 [notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L328-L356) 中：

```go
func (ns *NotificationCommon) SendNotificationToAllFollower(...) {
    if msg.NoNeedPushAllFollow || len(questionID) == 0 {
        return  // 标记为不需要推送，防止递归
    }
    // ...
    for _, userID := range userIDs {
        t := &schema.NotificationMsg{}
        _ = copier.Copy(t, msg)
        t.ReceiverUserID = userID
        t.NoNeedPushAllFollow = true  // 设置标记，下游处理时不会再次推送给关注者
        ns.notificationQueueService.Send(ctx, t)
    }
}
```

### 5.4 自我触发过滤

在所有通知触发点都有自我过滤：

```go
// 新回答通知
if questionUserID == answerUserID {
    return  // 自问自答不通知
}

// 新评论通知
if questionUserID == commentUserID {
    return  // 自己评论自己不通知
}

// 回复通知
if replyUserID == commentUserID {
    return  // 自己回复自己不通知
}
```

### 5.5 发送频率限制

新问题邮件通知有每日发送限制，在 [new_question_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/new_question_notification.go#L140-L160)：

```go
func (ns *ExternalNotificationService) checkSendNewQuestionNotificationEmailLimit(
    ctx context.Context, userID string) bool {
    
    key := constant.NewQuestionNotificationLimitCacheKeyPrefix + userID
    old, exist, err := ns.data.Cache.GetInt64(ctx, key)
    
    if exist && old >= constant.NewQuestionNotificationLimitMax {
        return true  // 超过限制，不发送
    }
    // 计数+1，缓存24小时
    if !exist {
        err = ns.data.Cache.SetInt64(ctx, key, 1, constant.NewQuestionNotificationLimitCacheTime)
    } else {
        _, err = ns.data.Cache.Increase(ctx, key, 1)
    }
    return false
}
```

---

## 六、用户通知偏好设置

### 6.1 偏好配置结构

定义在 [user_notification_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/schema/user_notification_schema.go#L56-L60)：

```go
type NotificationConfig struct {
    // 收件箱通知（回答、评论、@等）的邮件通知开关
    Inbox                          NotificationChannelConfig `json:"inbox"`
    
    // 所有新问题的邮件通知开关
    AllNewQuestion                 NotificationChannelConfig `json:"all_new_question"`
    
    // 关注标签的新问题邮件通知开关
    AllNewQuestionForFollowingTags NotificationChannelConfig `json:"all_new_question_for_following_tags"`
}

type NotificationChannelConfig struct {
    Key    constant.NotificationChannelKey `json:"key"`    // 渠道类型，如"email"
    Enable bool                            `json:"enable"` // 是否启用
}
```

### 6.2 通知来源类型

定义在 [notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/base/constant/notification.go#L66-L70)：

```go
const (
    InboxSource                          NotificationSource = "inbox"
    AllNewQuestionSource                 NotificationSource = "all_new_question"
    AllNewQuestionForFollowingTagsSource NotificationSource = "all_new_question_for_following_tags"
)
```

### 6.3 默认配置

用户注册时设置默认配置，在 [user_notification_config_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/user_notification_config/user_notification_config_service.go#L93-L97)：

```go
func (us *UserNotificationConfigService) SetDefaultUserNotificationConfig(
    ctx context.Context, userIDs []string) (err error) {
    
    // 默认启用收件箱的邮件通知
    return us.userNotificationConfigRepo.Add(ctx, userIDs,
        string(constant.InboxSource), `[{"key":"email","enable":true}]`)
}
```

### 6.4 偏好检查时机

#### 外部通知发送前检查

在各通知处理器中查询用户偏好：

```go
// 新回答/新评论/邀请回答：查询 InboxSource 配置
notificationConfig, exist, err := ns.userNotificationConfigRepo.GetByUserIDAndSource(
    ctx, msg.ReceiverUserID, constant.InboxSource)

// 新问题：查询 AllNewQuestionSource 和 AllNewQuestionForFollowingTagsSource 配置
userNotificationConfigs, err := ns.userNotificationConfigRepo.GetByUsersAndSource(
    ctx, tagsFollowerIDs, constant.AllNewQuestionForFollowingTagsSource)

notificationConfigs, err := ns.userNotificationConfigRepo.GetBySource(
    ctx, constant.AllNewQuestionSource)
```

#### 渠道启用检查

```go
channels := schema.NewNotificationChannelsFormJson(notificationConfig.Channels)
for _, channel := range channels {
    if !channel.Enable {
        continue  // 渠道未启用，跳过
    }
    if channel.Key == constant.EmailChannel {
        // 发送邮件
    }
}
```

---

## 七、完整数据流

### 7.1 新回答通知完整流程

```
用户A回答用户B的问题
    ↓
[answer_service.go] 创建回答成功
    ├─→ 发送 ActivityMsg 到 activityQueue（用于声望计算）
    ├─→ 发送 EventMsg 到 eventQueue（EventAnswerCreate，用于徽章等）
    └─→ 调用 notificationAnswerTheQuestion()
          │
          ├─→ 检查：如果自问自答，直接返回
          ├─→ 构造 NotificationMsg {
          │     ReceiverUserID: 用户B（问题作者）
          │     TriggerUserID:  用户A（回答者）
          │     Type:           1（收件箱）
          │     ObjectID:       回答ID
          │     NotificationAction: "notification.action.answer_the_question"
          │   }
          ├─→ 发送到 notificationQueue（内部通知队列）
          │
          └─→ 构造 ExternalNotificationMsg {
                ReceiverUserID: 用户B
                ReceiverEmail:  用户B的邮箱
                ReceiverLang:   用户B的语言
                NewAnswerTemplateRawData: { ... }
              }
              发送到 externalNotificationQueue（外部通知队列）
```

### 7.2 内部通知队列处理

```
notificationQueue 收到 NotificationMsg
    ↓
[notification.go] AddNotification() 处理
    ├─→ 成就类型检查：如果启用Rank Agent则跳过
    ├─→ 获取对象信息（标题、问题ID、回答ID等）
    ├─→ 如果是成就通知：
    │     └─→ 查询是否已有相同(用户,对象,类型)的通知
    │         ├─ 存在：更新Rank值，直接返回（去重）
    │         └─ 不存在：继续创建
    ├─→ 获取触发用户信息
    ├─→ 保存 Notification 到数据库
    ├─→ 更新用户红点计数（Redis缓存）
    ├─→ 异步调用 SendNotificationToAllFollower()
    │     └─→ 给所有关注该问题的用户发送通知（标记 NoNeedPushAllFollow=true）
    └─→ 同步通知到插件（如飞书、钉钉等）
```

### 7.3 外部通知队列处理

```
externalNotificationQueue 收到 ExternalNotificationMsg
    ↓
[external_notification.go] Handler() 分发
    ↓
根据消息类型调用对应处理器（如 handleNewAnswerNotification）
    ├─→ 查询用户通知偏好配置（InboxSource）
    │     └─ 不存在配置：直接返回
    ├─→ 遍历渠道配置
    │     ├─ 渠道未启用：跳过
    │     └─ 邮件渠道启用：
    │          ├─→ 检查用户状态（是否可用）
    │          ├─→ 按用户语言渲染邮件模板
    │          └─→ 发送邮件并保存退订码
    └─→ 返回
```

### 7.4 新问题通知完整流程

```
用户A发布新问题（带标签 "golang"、"java"）
    ↓
[question_service.go] 创建问题成功
    ├─→ 构造 ExternalNotificationMsg {
    │     NewQuestionTemplateRawData: {
    │       QuestionID: 问题ID
    │       QuestionTitle: 问题标题
    │       QuestionAuthorUserID: 用户A
    │       TagIDs: [tag-golang-id, tag-java-id]
    │       Tags: ["golang", "java"]
    │     }
    │   }
    ├─→ 发送到 externalNotificationQueue
    └─→ 发送 EventMsg 到 eventQueue（EventQuestionCreate）

externalNotificationQueue 处理
    ↓
[new_question_notification.go] handleNewQuestionNotification()
    ├─→ 调用 getNewQuestionSubscribers() 计算接收者
    │     ├─→ 获取标签 "golang" 的关注者列表 [U1, U2, U3]
    │     ├─→ 获取标签 "java" 的关注者列表 [U2, U4]
    │     ├─→ 合并去重 [U1, U2, U3, U4]
    │     ├─→ 查询这些用户的 AllNewQuestionForFollowingTagsSource 偏好
    │     ├─→ 获取 AllNewQuestionSource 的订阅用户 [U5, U6]
    │     ├─→ 合并去重 [U1, U2, U3, U4, U5, U6]
    │     ├─→ 排除作者用户A
    │     ├─→ 检查每个用户的发送频率限制
    │     └─→ 返回最终接收者列表
    ├─→ 遍历每个接收者
    │     └─→ 遍历渠道
    │          ├─ 邮件渠道启用：
    │          │    ├─→ 检查用户状态
    │          │    ├─→ 渲染邮件模板（新问题通知）
    │          │    └─→ 发送邮件
    │          └─ 其他渠道：调用对应插件
    └─→ 同步通知到插件（如飞书机器人等）
```

---

## 八、关键设计要点

### 8.1 异步解耦

所有通知操作都通过队列异步处理，不阻塞主业务流程。使用带缓冲的 channel 作为队列，支持高并发场景。

### 8.2 幂等性设计

- 成就通知通过 `(user_id, object_id, type)` 唯一键实现幂等更新
- 关注者推送通过 `NoNeedPushAllFollow` 标记防止递归
- 新问题接收者通过 map 去重

### 8.3 可扩展性

- 队列采用泛型设计，易于扩展新的消息类型
- 外部通知支持插件扩展，可添加新的通知渠道（如短信、Webhook等）
- 用户配置支持多渠道配置

### 8.4 用户体验

- 支持红点提醒，未读消息计数使用 Redis 缓存
- 支持邮件退订功能，每封邮件包含退订链接
- 发送频率限制防止骚扰用户

### 8.5 容错设计

- 队列处理失败仅打日志，不影响主流程
- 用户状态检查避免给无效用户发送通知
- 插件通知失败不影响主流程

---

## 九、通知类型与动作映射

### 9.1 通知动作常量

定义在 [notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/base/constant/notification.go#L22-L61)：

| 常量 | 说明 | 消息类型 |
|------|------|----------|
| `NotificationUpdateQuestion` | 问题被更新 | 1（帖子） |
| `NotificationAnswerTheQuestion` | 问题被回答 | 1（帖子） |
| `NotificationUpVotedTheQuestion` | 问题被点赞 | 2（投票） |
| `NotificationDownVotedTheQuestion` | 问题被踩 | 2（投票） |
| `NotificationUpdateAnswer` | 回答被更新 | 1（帖子） |
| `NotificationAcceptAnswer` | 回答被采纳 | 1（帖子） |
| `NotificationUpVotedTheAnswer` | 回答被点赞 | 2（投票） |
| `NotificationDownVotedTheAnswer` | 回答被踩 | 2（投票） |
| `NotificationCommentQuestion` | 问题被评论 | 1（帖子） |
| `NotificationCommentAnswer` | 回答被评论 | 1（帖子） |
| `NotificationUpVotedTheComment` | 评论被点赞 | 2（投票） |
| `NotificationReplyToYou` | 收到回复 | 1（帖子） |
| `NotificationMentionYou` | 被@提及 | 1（帖子） |
| `NotificationYourQuestionIsClosed` | 问题被关闭 | 1（帖子） |
| `NotificationYourQuestionWasDeleted` | 问题被删除 | 1（帖子） |
| `NotificationYourAnswerWasDeleted` | 回答被删除 | 1（帖子） |
| `NotificationYourCommentWasDeleted` | 评论被删除 | 1（帖子） |
| `NotificationInvitedYouToAnswer` | 被邀请回答 | 3（邀请） |

### 9.2 消息类型分组

通过 `NotificationMsgTypeMapping` 映射到不同的收件箱分类：

```go
var NotificationMsgTypeMapping = map[string]int{
    // 1 = 帖子类（Posts）
    NotificationUpdateQuestion:         1,
    NotificationAnswerTheQuestion:      1,
    NotificationUpdateAnswer:           1,
    NotificationAcceptAnswer:           1,
    NotificationCommentQuestion:        1,
    NotificationCommentAnswer:          1,
    NotificationReplyToYou:             1,
    NotificationMentionYou:             1,
    // ...
    
    // 2 = 投票类（Votes）
    NotificationUpVotedTheQuestion:     2,
    NotificationDownVotedTheQuestion:   2,
    NotificationUpVotedTheAnswer:       2,
    NotificationDownVotedTheAnswer:     2,
    NotificationUpVotedTheComment:      2,
    
    // 3 = 邀请类（Invites）
    NotificationInvitedYouToAnswer:     3,
}
```

---

## 十、核心协作关系总结

### 10.1 组件协作图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          业务操作层                                  │
│  [question_service]  [answer_service]  [comment_service]  [vote...] │
└────────┬──────────────────────────────┬─────────────────────────────┘
         │ 发送 EventMsg                │ 发送 NotificationMsg
         │                              │ 发送 ExternalNotificationMsg
         ↓                              ↓
┌───────────────────┐        ┌───────────────────────────┐
│   eventqueue      │        │   noticequeue             │
│   (事件总线)      │        │   ┌─────────────────────┐ │
└─────────┬─────────┘        │   │  notification     │ │
          │ 消费事件         │   │  (站内通知队列)   │ │
          ↓                  │   └─────────┬──────────┘ │
┌───────────────────┐        │             │ 消费        │
│  badge_event_     │        │             ↓            │
│  handler          │        │   ┌─────────────────────┐ │
│  (徽章处理)       │        │   │ external_notification│ │
└───────────────────┘        │   │ (外部通知队列)      │ │
                             │   └─────────┬──────────┘ │
                             │             │ 消费        │
                             └─────────────┼────────────┘
                                           ↓
                             ┌───────────────────────────┐
                             │   notification_common     │
                             │   ├─ AddNotification()   │
                             │   │  · 去重检查          │
                             │   │  · 保存通知          │
                             │   │  · 更新红点          │
                             │   │  · 推送关注者        │
                             │   │  · 同步插件          │
                             │   └─ SendNotificationToAllFollower() │
                             └─────────────┬───────────────┘
                                           │
                                           ↓
                             ┌───────────────────────────┐
                             │   user_notification_      │
                             │   config_service          │
                             │   (偏好设置管理)          │
                             └───────────────────────────┘
```

### 10.2 关键协作流程

1. **事件触发**：业务服务完成操作后，同时发送事件到事件总线和通知到通知队列
2. **接收者计算**：
   - 一对一通知（回答、评论）：直接指定接收者为内容作者
   - 广播通知（新问题）：通过标签关注 + 全量订阅计算接收者
   - 关注者推送：查询问题的关注用户列表
3. **偏好过滤**：外部通知发送前查询用户偏好配置，仅发送给启用对应渠道的用户
4. **去重处理**：
   - 成就通知按唯一键更新而非新增
   - 多来源接收者通过 map 去重
   - 自我触发直接过滤
5. **队列消费**：
   - 内部通知：保存到数据库，更新红点，推送给关注者
   - 外部通知：检查偏好，渲染模板，发送邮件/调用插件
6. **限流保护**：新问题通知有每日发送上限，防止骚扰用户
