# 通知管线修订分析

## 一、动作映射与触发用户基本信息拉取前移到写库前

### 1.1 执行顺序修订

[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L175-L199)

**正确顺序（写库前完成所有准备）：**

```go
// 1. 构建实体基础字段
info := &entity.Notification{}
info.UserID = req.ReceiverUserID
info.Type = req.Type
info.IsRead = schema.NotificationNotRead
info.Status = schema.NotificationStatusNormal
info.ObjectID = req.ObjectInfo.ObjectID

// 2. 拉取触发用户基本信息（前移到写库前）
userBasicInfo, exist, err := ns.userCommon.GetUserBasicInfoByID(ctx, req.TriggerUserID)
if err != nil { return fmt.Errorf("get user basic info error: %w", err) }
if !exist { return fmt.Errorf("user not exist: %s", req.TriggerUserID) }
req.UserInfo = userBasicInfo

// 3. 序列化通知内容（含用户信息）
content, _ := json.Marshal(req)

// 4. MsgType 动作映射（前移到写库前）
info.MsgType = 0  // Go 零值默认
_, ok := constant.NotificationMsgTypeMapping[req.NotificationAction]
if ok {
    info.MsgType = constant.NotificationMsgTypeMapping[req.NotificationAction]
}

// 5. 内容赋值
info.Content = string(content)

// 6. 最后才写库
err = ns.notificationRepo.AddNotification(ctx, info)
```

**设计意图**：
- `GetUserBasicInfoByID` 和 `NotificationMsgTypeMapping` 是 **纯读操作**，一旦失败应直接返回，不留下半条记录
- 写入数据库的 `info.Content` 包含用户基本信息，必须在写库前准备完毕
- `MsgType` 是数据库字段，也必须在写库前确定

### 1.2 MsgType 映射与内容序列化的关系

- `info.MsgType` 是独立数据库字段，用于前端子分类筛选（posts=1, votes=2, invites=3）
- `info.Content` 是 JSON 文本，包含 `NotificationAction` 字符串，前端展示时再做 i18n 翻译
- 两者在写库前**并行计算**，互不依赖

---

## 二、徽章二级提醒入主流程

[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L207-L212)

徽章二级提醒不再是"额外"逻辑，而是 **主流程的一部分**，紧跟在 `addRedDot` 之后执行：

```go
// 主流程顺序:
err = ns.notificationRepo.AddNotification(ctx, info)     // 1. 写库
err = ns.addRedDot(ctx, info.UserID, msg.Type)          // 2. 增加红点计数

// 3. 徽章二级提醒（主流程内嵌）
if req.ObjectInfo.ObjectType == constant.BadgeAwardObjectType {
    err = ns.AddBadgeAwardAlertCache(ctx, info.UserID, info.ID, req.ObjectInfo.ObjectMap["badge_id"])
    // 失败只打日志，不中断主流程
}

// 4. 关注者推送
go ns.SendNotificationToAllFollower(ctx, msg, questionID)

// 5. 插件同步
if msg.Type == schema.NotificationTypeInbox {
    ns.syncNotificationToPlugin(ctx, objInfo, msg)
}
```

**并入主流程的意义**：
- 徽章授予是一个完整的通知事件，需要同时更新"成就红点计数"和"徽章弹窗缓存"
- 两者都是写入 Redis，但键和值格式完全不同，是两个独立的副作用
- 放在 addRedDot 之后语义上是"先更新计数，再更新弹窗详情"

---

## 三、补两条触发路径

### 3.1 取消采纳路径

**事件源**：[answer_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/activity/answer_repo.go#L98-L143)

```
AnswerService.AcceptAnswer(req={AnswerID: ""})  // 取消采纳（传空 AnswerID）
  → updateAnswerRank()
      → answerActivityService.CancelAcceptAnswer()
          → SaveCancelAcceptAnswerActivity()
              │ 事务内:
              │   1. 查询现有活动记录
              │   2. cancelActivities() - 标记活动为已取消
              │   3. rollbackUserRank() - 回滚声望
              └─→ sendCancelAcceptAnswerNotification()  // 投递成就通知

sendCancelAcceptAnswerNotification() 投递:
  循环每个 Activity:
    NotificationMsg {
      Type: Achievement (2)
      TriggerUserID:  act.TriggerUserID (采纳操作者/取消操作者)
      ReceiverUserID: act.ActivityUserID (声望收益人)
      ObjectID:       op.AnswerObjectID
      ObjectType:     act.ActivityUserID == op.QuestionObjectID
                         ? QuestionObjectType  // 问题作者取消采纳自己问题的声望
                         : AnswerObjectType    // 回答作者取消采纳声望
    }
    → 仅在 TriggerUserID != ReceiverUserID 时发送到 notificationQueue
```

**接收者**：所有因采纳操作获得过声望的用户（包括问题作者和回答作者），ObjectType 区分是问题声望还是回答声望。

**注意**：取消采纳**只投成就通知**，没有 Inbox 通知，没有外部邮件通知。去重逻辑会更新已有成就通知的 Rank 值（扣减声望）。

### 3.2 外部新问题插件同步路径

**事件源**：[new_question_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/new_question_notification.go#L70-L270)

```
handleNewQuestionNotification()
  ├─→ getNewQuestionSubscribers()  // 计算邮件接收者
  │    └─→ 遍历接收者发邮件（每个接收者独立渠道检查）
  │
  └─→ syncNewQuestionNotificationToPlugin(ctx, msg)  // 独立的插件同步分支

syncNewQuestionNotificationToPlugin() 流程:
  ├─→ plugin.CallNotification(fn -> ...)
  │
  ├─→ 1. 计算插件接收者（与邮件接收者独立计算）:
  │    subscribersMapping = { userID -> notificationType }
  │    ├─→ 遍历问题标签:
  │    │    GetFollowUserIDs(tagID) → 标记为 NotificationNewQuestionFollowedTag
  │    ├─→ fn.GetNewQuestionSubscribers() → 标记为 NotificationNewQuestion
  │    └─→ delete(subscribersMapping, QuestionAuthorUserID)  // 排除作者
  │
  ├─→ 2. 构建通用插件消息 newPluginQuestionNotification():
  │    - 获取站点信息 (general + seo + interface)
  │    - 获取作者信息 (DisplayName, UserUrl)
  │    - 构建 QuestionUrl / QuestionTitle / Tags
  │
  └─→ 3. 遍历 subscribersMapping 逐个发送:
       newMsg = copier.Copy(pluginNotificationMsg)
       newMsg.ReceiverUserID = subscriberUserID
       newMsg.Type = notificationType
       ├─→ 补充 ReceiverLang (用户语言)
       ├─→ 补充 ReceiverExternalID (优先匹配当前插件 slug 的 SSO 绑定，fallback 到任意 SSO)
       └─→ fn.Notify(newMsg)
```

**关键差异**（插件同步 vs 邮件发送）：

| 维度 | 邮件发送 | 插件同步 |
|------|---------|---------|
| 接收者计算 | 查 `user_notification_config` 偏好表 | 标签关注者 + 插件自定义 `GetNewQuestionSubscribers()` |
| 偏好检查 | 按用户渠道配置过滤 | 不检查用户通知偏好，插件自行决定 |
| 渠道遍历 | 按用户启用的渠道（email 等） | 按注册的 notification 插件 |
| 外部ID | 无（用邮箱） | 优先匹配当前插件 slug 的 SSO 绑定 |
| 频率限制 | 每日上限 50 封 | 无限制 |

**双通道插件同步**：
- `syncNotificationToPlugin`（内部队列，仅 Inbox）：处理回答、评论、@、邀请等站内交互
- `syncNewQuestionNotificationToPlugin`（外部队列，新问题）：独立处理新问题广播，不经过内部 notificationQueue

---

## 四、成就去重时红点、徽章提醒、关注者推送、插件同步连带跳过

### 4.1 去重提前返回的完整跳过链

[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L147-L173)

```go
if msg.Type == schema.NotificationTypeAchievement {
    notificationInfo, exist, err := ns.notificationRepo.GetByUserIdObjectIdTypeId(
        ctx, req.ReceiverUserID, req.ObjectInfo.ObjectID, req.Type)

    rank, err := ns.activityRepo.GetUserIDObjectIDActivitySum(
        ctx, req.ReceiverUserID, req.ObjectInfo.ObjectID)
    req.Rank = rank

    if exist {
        // 仅更新 Content 中的 Rank 值
        updateContent := &schema.NotificationContent{}
        json.Unmarshal([]byte(notificationInfo.Content), updateContent)
        updateContent.Rank = rank
        content, _ := json.Marshal(updateContent)
        notificationInfo.Content = string(content)
        err = ns.notificationRepo.UpdateNotificationContent(ctx, notificationInfo)

        // ⚠️  直接 return，跳过 ALL 后续步骤
        return nil
    }
}

// ↓↓↓ 以下全部跳过 ↓↓↓
// 1. info.UserID/Type/ObjectID 等字段赋值       — 跳过
// 2. GetUserBasicInfoByID (触发用户信息拉取)   — 跳过
// 3. MsgType 映射                             — 跳过
// 4. AddNotification (写库)                    — 跳过（改为 UpdateNotificationContent）
// 5. addRedDot (增加红点计数)                  — 跳过
// 6. AddBadgeAwardAlertCache (徽章二级提醒)     — 跳过
// 7. SendNotificationToAllFollower (关注者推送) — 跳过
// 8. syncNotificationToPlugin (插件同步)       — 跳过（且成就类型本就不触发）
```

### 4.2 跳过的设计意图

| 操作 | 跳过原因 |
|------|---------|
| **addRedDot** | 成就通知是对"已有声望对象"的更新，不是新事件，不应增加红点。用户已经看到过这条通知，红点不应重复计数 |
| **徽章提醒** | 只有新授予徽章时才弹弹窗。去重说明用户已经获得过该徽章（或同一对象的成就），不应重复弹窗 |
| **关注者推送** | 成就通知针对具体用户的声望变化，不具有"动态更新"语义，无需推送给关注者 |
| **插件同步** | 成就类型本身就不触发插件同步（仅 Inbox 触发） |
| **MsgType 映射** | 成就通知没有 NotificationAction（投票成就无动作值），映射无意义 |

**副作用**：去重时 `addRedDot` 被跳过，但 `UpdateNotificationContent` 更新了 `notification.updated_at`。前端按创建时间排序时位置不变，但按更新时间排序会上浮。红点计数不变。

---

## 五、红点新增只走收件箱与成就而徽章键由二级提醒以 JSON 写入

### 5.1 红点分类写入规则

[addRedDot](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L222-L244)

```go
func (ns *NotificationCommon) addRedDot(ctx context.Context, userID string, noticeType int) error {
    var key string
    if noticeType == schema.NotificationTypeInbox {
        // inbox 键: answer:red-dot:inbox:{userID}
        key = fmt.Sprintf(constant.RedDotCacheKey, constant.NotificationTypeInbox, userID)
    } else {
        // achievement 键: answer:red-dot:achievement:{userID}
        key = fmt.Sprintf(constant.RedDotCacheKey, constant.NotificationTypeAchievement, userID)
    }
    // Increase 或 SetInt64，值为 int64 未读计数
}
```

**addRedDot 只处理两类键**：
- `answer:red-dot:inbox:{userID}` → 值：int64 未读数
- `answer:red-dot:achievement:{userID}` → 值：int64 未读数

### 5.2 徽章键完全独立写入

徽章键 **不经过 addRedDot**，完全由 `AddBadgeAwardAlertCache` 独立写入：

[AddBadgeAwardAlertCache](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L284-L306)

```go
func (ns *NotificationCommon) AddBadgeAwardAlertCache(ctx context.Context, 
    userID, notificationID, badgeID string) (err error) {
    
    // 徽章键: answer:red-dot:badge:{userID}
    key := fmt.Sprintf(constant.RedDotCacheKey, constant.NotificationTypeBadgeAchievement, userID)
    
    cacheData, exist, err := ns.data.Cache.GetString(ctx, key)
    if !exist {
        c := schema.NewRedDotBadgeAwardCache()
        c.AddBadgeAward(&schema.RedDotBadgeAward{
            NotificationID: notificationID,
            BadgeID:        badgeID,
        })
        // 值为 JSON 字符串，不是 int64！
        return ns.data.Cache.SetString(ctx, key, c.ToJSON(), constant.RedDotCacheTime)
    }
    
    c := schema.NewRedDotBadgeAwardCache()
    c.FromJSON(cacheData)
    c.AddBadgeAward(&schema.RedDotBadgeAward{
        NotificationID: notificationID,
        BadgeID:        badgeID,
    })
    return ns.data.Cache.SetString(ctx, key, c.ToJSON(), constant.RedDotCacheTime)
}
```

**三类键的本质差异**：

| 键模式 | 值类型 | 写入时机 | 操作 |
|--------|--------|---------|------|
| `answer:red-dot:inbox:{userID}` | int64 | 新增 Inbox 通知且未去重时 | addRedDot → Increase/SetInt64 |
| `answer:red-dot:achievement:{userID}` | int64 | 新增 Achievement 通知且未去重时 | addRedDot → Increase/SetInt64 |
| `answer:red-dot:badge:{userID}` | JSON string | 新增 BadgeAward 类型通知且未去重时 | AddBadgeAwardAlertCache → SetString |

**设计意图**：
- 前两类是"计数型"红点，语义是"你有 N 条未读消息"
- 第三类是"详情型"红点，语义是"你获得了新徽章 X，点击查看详情"，需要存储 badgeID 等元数据
- 两者使用相同的 `RedDotCacheKey` 前缀，但值类型完全不同，是**并行写入**的两套独立缓存

---

## 六、前端红点接口待审核计数与审核权限来源与门禁

### 6.1 GetRedDot 接口权限门禁

[notification_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/controller/notification_controller.go#L58-L77)

```go
func (nc *NotificationController) GetRedDot(ctx *gin.Context) {
    req := &schema.GetRedDot{}
    req.UserID = middleware.GetLoginUserIDFromContext(ctx)

    // 🔒 门禁 1: 检查审核权限
    canList, err := nc.rankService.CheckOperationPermissions(ctx, req.UserID, []string{
        permission.QuestionAudit,  // "question.audit"
        permission.AnswerAudit,    // "answer.audit"
        permission.TagAudit,       // "tag.audit"
    })
    req.CanReviewQuestion = canList[0]
    req.CanReviewAnswer = canList[1]
    req.CanReviewTag = canList[2]

    // 🔒 门禁 2: 检查是否管理员/审核员
    req.IsAdmin = middleware.GetUserIsAdminModerator(ctx)

    resp, err := nc.notificationService.GetRedDot(ctx, req)
    handler.HandleResponse(ctx, err, resp)
}
```

**权限常量**定义在 [permission_name.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/permission/permission_name.go#L59-L61)：
```go
AnswerAudit   = "answer.audit"
QuestionAudit = "question.audit"
TagAudit      = "tag.audit"
```

### 6.2 待审核计数组成

[countAllReviewAmount](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/notification_service.go#L130-L164)

```go
func (ns *NotificationService) countAllReviewAmount(ctx context.Context, req *schema.GetRedDot) (amount int64) {
    // 1. 待审核队列数（仅管理员可见）
    if req.IsAdmin {
        reviewCount, err := ns.reviewService.GetReviewPendingCount(ctx)
        amount += reviewCount
    }

    // 2. 举报待处理数（仅管理员可见）
    if req.IsAdmin {
        reportCount, err := ns.reportRepo.GetReportCount(ctx)
        amount += reportCount
    }

    // 3. 待审核修订数（根据权限过滤）
    countUnreviewedRevision, err := ns.revisionService.GetUnreviewedRevisionCount(ctx, &schema.RevisionSearch{
        CanReviewQuestion: req.CanReviewQuestion,  // 有问题审核权限才计入
        CanReviewAnswer:   req.CanReviewAnswer,    // 有回答审核权限才计入
        CanReviewTag:      req.CanReviewTag,       // 有标签审核权限才计入
        UserID:            req.UserID,
    })
    amount += countUnreviewedRevision

    return amount
}
```

### 6.3 权限与计数的对应关系

| 计数项 | 权限要求 | 说明 |
|--------|---------|------|
| 待审核队列数 | `IsAdmin == true` | 管理员才能看到所有待审核内容（question+answer+tag） |
| 举报待处理数 | `IsAdmin == true` | 管理员才能处理举报 |
| 待审核修订数 | 对应权限为 true | 问题审核员→计入问题修订数；回答审核员→计入回答修订数；标签审核员→计入标签修订数 |

**GetRedDot 完整返回**（[notification_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/notification_service.go#L80-L97)）：

```go
func (ns *NotificationService) GetRedDot(ctx context.Context, req *schema.GetRedDot) (resp *schema.RedDot, err error) {
    // 1. 读取 inbox 红点计数（Redis）
    inboxKey := fmt.Sprintf("answer:red-dot:inbox:%s", req.UserID)
    redBot.Inbox, _, _ = ns.data.Cache.GetInt64(ctx, inboxKey)

    // 2. 读取 achievement 红点计数（Redis）
    achievementKey := fmt.Sprintf("answer:red-dot:achievement:%s", req.UserID)
    redBot.Achievement, _, _ = ns.data.Cache.GetInt64(ctx, achievementKey)

    // 3. 有权限才计算待审核数
    if req.CanReviewAnswer || req.CanReviewQuestion || req.CanReviewTag {
        redBot.CanRevision = true
        redBot.Revision = ns.countAllReviewAmount(ctx, req)
    }

    // 4. 读取徽章弹窗（Redis JSON）
    redBot.BadgeAward = ns.getBadgeAward(ctx, req.UserID)

    return redBot, nil
}
```

---

## 七、关注者推送每条通知后都靠对象 ID 与防递归双闸过滤

### 7.1 双闸过滤逻辑

[SendNotificationToAllFollower](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L327-L357)

```go
func (ns *NotificationCommon) SendNotificationToAllFollower(ctx context.Context,
    msg *schema.NotificationMsg, questionID string) {

    // 🚧 闸 1: 防递归标记 + 对象 ID 检查
    if msg.NoNeedPushAllFollow || len(questionID) == 0 {
        return
    }

    // 🚧 闸 2: 动作白名单过滤
    if msg.NotificationAction != constant.NotificationUpdateQuestion &&
        msg.NotificationAction != constant.NotificationAnswerTheQuestion &&
        msg.NotificationAction != constant.NotificationUpdateAnswer &&
        msg.NotificationAction != constant.NotificationAcceptAnswer {
        return
    }

    // 确定关注的对象 ID
    condObjectID := msg.ObjectID
    if len(questionID) > 0 {
        condObjectID = uid.DeShortID(questionID)  // 优先用问题 ID 查询关注者
    }

    // 查询关注该对象的用户
    userIDs, err := ns.followRepo.GetFollowUserIDs(ctx, condObjectID)

    // 给每个关注者发送通知
    for _, userID := range userIDs {
        t := &schema.NotificationMsg{}
        _ = copier.Copy(t, msg)
        t.ReceiverUserID = userID
        t.TriggerUserID = msg.TriggerUserID
        t.NoNeedPushAllFollow = true  // ✅ 标记下游不再推送，防递归
        ns.notificationQueueService.Send(ctx, t)
    }
}
```

### 7.2 双闸协同机制

```
原始通知入队 → AddNotification 处理
  │
  ├─→ 写库 + addRedDot + 徽章提醒
  │
  └─→ go SendNotificationToAllFollower(msg, questionID)
        │
        ├─→ 闸1检查: NoNeedPushAllFollow == false AND questionID != ""
        │    → 不满足直接 return
        │
        ├─→ 闸2检查: NotificationAction 在白名单(UpdateQuestion/AnswerTheQuestion/UpdateAnswer/AcceptAnswer)
        │    → 不满足直接 return
        │
        ├─→ GetFollowUserIDs(condObjectID)  // 查询关注者
        │
        └─→ 循环每个关注者:
              t = copy(msg)
              t.ReceiverUserID = userID
              t.NoNeedPushAllFollow = true  // 设置闸1标记
              notificationQueue.Send(t)
              
              ↓ 下游 AddNotification 处理 t 时:
                 SendNotificationToAllFollower(t, questionID)
                   → 闸1: NoNeedPushAllFollow == true → return  ✓ 防递归成功
```

**设计意图**：
- **闸1（`NoNeedPushAllFollow`）**：防止无限递归。复制的消息自带标记，下游处理时直接跳过推送
- **闸2（动作白名单）**：只有"动态更新"类动作才值得推送给关注者。投票、评论等动作已经有单独的通知路径，不需要通过关注者推送重复发送
- **对象 ID 优先问题 ID**：关注者是关注"问题"的，不是关注"回答"的，所以用 questionID 查询

---

## 八、投票成就声望增减对象不等于被投票对象作者

### 8.1 投票活动的双向声望设计

[getActivities](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/content/vote_service.go#L258-L299)

一次投票操作产生 **两条独立的 Activity**，对应两个不同的 `ActivityUserID`（声望增减对象）：

```go
func (vs *VoteService) getActivities(ctx context.Context, op *schema.VoteOperationInfo) (
    activities []*schema.VoteActivity) {

    // 根据对象类型和投票方向确定动作
    var actions []string
    switch op.ObjectType {
    case constant.QuestionObjectType:
        if op.VoteUp {
            actions = []string{
                activity_type.QuestionVoteUp,      // 投票者消耗声望
                activity_type.QuestionVotedUp,     // 被投票者获得声望
            }
        } else {
            actions = []string{
                activity_type.QuestionVoteDown,    // 投票者消耗声望
                activity_type.QuestionVotedDown,   // 被投票者损失声望
            }
        }
    // Answer 和 Comment 同理
    }

    for _, action := range actions {
        t := &schema.VoteActivity{}
        cfg, _ := vs.configService.GetConfigByKey(ctx, action)
        t.ActivityType, t.Rank = cfg.ID, cfg.GetIntValue()

        // 🔑 关键: 根据 action 类型区分声望对象
        if strings.Contains(action, "voted") {
            // "voted" 系列: 被投票的对象作者获得/损失声望
            t.ActivityUserID = op.ObjectCreatorUserID  // 被投票对象作者
            t.TriggerUserID = op.OperatingUserID       // 投票者
        } else {
            // "vote" 系列: 投票者本人消耗声望
            t.ActivityUserID = op.OperatingUserID      // 投票者本人
            t.TriggerUserID = "0"                      // 无触发者
        }
        activities = append(activities, t)
    }
    return activities
}
```

### 8.2 两类 Activity 的差异

| action 类型 | ActivityUserID | TriggerUserID | Rank 符号 | 说明 |
|------------|----------------|---------------|-----------|------|
| `QuestionVoteUp` | 投票者 | "0" | 负（消耗声望） | 投票需要消耗自己的声望 |
| `QuestionVotedUp` | 被投票对象作者 | 投票者 | 正（获得声望） | 被投票者获得声望 |
| `QuestionVoteDown` | 投票者 | "0" | 负（消耗声望） | 踩也需要消耗声望 |
| `QuestionVotedDown` | 被投票对象作者 | 投票者 | 负（损失声望） | 被踩者损失声望 |

### 8.3 成就通知接收者推导

[sendAchievementNotification](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/activity/vote_repo.go#L456-L470)

```go
func (vr *VoteRepo) sendAchievementNotification(ctx context.Context,
    activityUserID, objectUserID, objectID string) {

    msg := &schema.NotificationMsg{
        ReceiverUserID: activityUserID,  // 🔑 接收者是 ActivityUserID
        TriggerUserID:  objectUserID,    // 触发者是被投票对象作者
        Type:           schema.NotificationTypeAchievement,
        ObjectID:       objectID,
        ObjectType:     objectType,
    }
    vr.notificationQueueService.Send(ctx, msg)
}
```

**调用处**（[vote_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/activity/vote_repo.go#L119-L124)）：

```go
for _, activity := range op.Activities {
    if activity.Rank == 0 {
        continue
    }
    // activity.ActivityUserID = 声望增减对象
    // op.ObjectCreatorUserID = 被投票对象作者
    vr.sendAchievementNotification(ctx, activity.ActivityUserID, op.ObjectCreatorUserID, op.ObjectID)
}
```

**实际接收者**：

| Activity 类型 | ActivityUserID | 接收者 | TriggerUserID |
|--------------|----------------|--------|---------------|
| QuestionVoteUp | 投票者 | 投票者本人 | 被投票对象作者 |
| QuestionVotedUp | 被投票对象作者 | 被投票对象作者 | 被投票对象作者（自己） |
| QuestionVoteDown | 投票者 | 投票者本人 | 被投票对象作者 |
| QuestionVotedDown | 被投票对象作者 | 被投票对象作者 | 被投票对象作者（自己） |

**关键修正**：之前误判"接收者都是被投票对象作者"，实际**投票者本人也会收到成就通知**（通知他"你消耗了 X 声望投票"）。`TriggerUserID` 在"voted"系列中等于 `ReceiverUserID`，但投票路径有 `TriggerUserID != ReceiverUserID` 的过滤吗？

查看 [sendAchievementNotification](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/activity/vote_repo.go#L456-L470) —— **没有过滤**！投票者和被投票者都会收到成就通知，哪怕 `TriggerUserID == ReceiverUserID`。这与采纳路径的过滤逻辑不同。

---

## 九、采纳跳过条件改为受益人不为问题作者

### 9.1 采纳 Inbox 通知跳过条件

[sendAcceptAnswerNotification](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/activity/answer_repo.go#L316-L328)

```go
for _, act := range op.Activities {
    msg := &schema.NotificationMsg{
        ReceiverUserID: act.ActivityUserID,  // 受益人
        Type:           schema.NotificationTypeInbox,
        ObjectID:       op.AnswerObjectID,
        TriggerUserID:  op.TriggerUserID,    // 采纳操作者 = 问题作者
    }
    // ✅ 跳过条件: 受益人 != 问题作者
    if act.ActivityUserID != op.QuestionUserID {
        msg.ObjectType = constant.AnswerObjectType
        msg.NotificationAction = constant.NotificationAcceptAnswer
        ar.notificationQueueService.Send(ctx, msg)
    }
}
```

**修正前（错误描述）**："不通知问题作者自己'你的回答被采纳'"

**修正后（正确逻辑）**：`act.ActivityUserID != op.QuestionUserID`

即：**受益人不是问题作者时才发送 Inbox 通知**。

### 9.2 受益人构成分析

采纳操作产生的 `Activities` 包括：

| Activity 类型 | ActivityUserID（受益人） | 说明 |
|--------------|-------------------------|------|
| `ActAnswerAccepted` | 回答作者 | 回答被采纳获得声望 |
| `ActQuestionAccepted` | 问题作者 | 问题采纳了回答获得声望 |

所以：
- 回答作者（`act.ActivityUserID = 回答作者ID`）≠ 问题作者（`op.QuestionUserID`）→ **发送** Inbox 通知
- 问题作者（`act.ActivityUserID = 问题作者ID`）= 问题作者 → **跳过**，不通知

**设计意图**：问题作者就是采纳操作者，不需要通知自己"你采纳了一个回答"。只有回答作者需要收到"你的回答被采纳了"的通知。

### 9.3 采纳成就通知的不同跳过条件

[sendAcceptAnswerNotification](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/activity/answer_repo.go#L301-L314)

```go
// 成就通知循环
for _, act := range op.Activities {
    msg := &schema.NotificationMsg{
        Type:           schema.NotificationTypeAchievement,
        ReceiverUserID: act.ActivityUserID,
        TriggerUserID:  act.TriggerUserID,
        ObjectType:     constant.AnswerObjectType,
    }
    // ✅ 跳过条件: TriggerUserID != ReceiverUserID
    if msg.TriggerUserID != msg.ReceiverUserID {
        ar.notificationQueueService.Send(ctx, msg)
    }
}
```

成就通知的跳过条件是**触发者≠接收者**，这是为了避免"自己给自己发成就通知"。问题作者采纳自己问题下的回答时，`TriggerUserID = 问题作者ID`，而 `ActQuestionAccepted` 的 `ActivityUserID = 问题作者ID`，所以这条会被过滤。只有回答作者的成就通知会发送。

### 9.4 采纳两条通知的跳过条件对比

| 通知类型 | 跳过条件 | 过滤掉的接收者 | 最终接收者 |
|---------|---------|--------------|-----------|
| **成就通知** | `TriggerUserID != ReceiverUserID` | 问题作者（自己采纳自己问题的声望） | 回答作者 |
| **Inbox 通知** | `act.ActivityUserID != op.QuestionUserID` | 问题作者（不需要通知自己） | 回答作者 |

虽然最终接收者相同（都是回答作者），但过滤逻辑完全不同：
- 成就通知：防止自我触发（自己给自己发声望通知）
- Inbox 通知：防止无意义通知（不需要通知操作者自己）

---

## 十、修订后的主流程时序图

```
AddNotification(msg) 入口
  │
  ├─→ RankAgent 检查: Type==Achievement && RankAgentEnabled → return
  │
  ├─→ 构建 NotificationContent req
  │    ├─→ BadgeAwardObjectType? → 从 msg.ExtraInfo 取 badge_id
  │    └─→ 其他类型 → objectInfoService.GetInfo() 拉取对象信息
  │
  ├─→ Type==Achievement?
  │    │
  │    ├─→ 是:
  │    │    ├─→ 查询 (userID, objectID, type) 是否已存在
  │    │    ├─→ 查询当前 Rank = ActivitySum
  │    │    ├─→ 已存在?
  │    │    │   ├─→ 是: UpdateNotificationContent (仅更新 Content 中的 Rank)
  │    │    │   │   └─→ return nil  ⚠️  跳过 ALL 后续步骤
  │    │    │   │                        (addRedDot/徽章提醒/关注者推送/插件同步全部跳过)
  │    │    │   └─→ 否: 继续
  │    │    └─→ 继续
  │    └─→ 否: 继续
  │
  ├─→ 构建 Notification info 基础字段 (UserID/Type/ObjectID/IsRead/Status)
  │
  ├─→ ✅ 写库前准备:
  │    ├─→ userCommon.GetUserBasicInfoByID(TriggerUserID) → req.UserInfo
  │    ├─→ json.Marshal(req) → info.Content (含用户信息)
  │    └─→ NotificationMsgTypeMapping[Action] → info.MsgType (可能落零)
  │
  ├─→ notificationRepo.AddNotification(info)  ✍️ 写库
  │
  ├─→ addRedDot(UserID, Type)  🔴 仅 inbox/achievement 两类键，int64 计数
  │
  ├─→ ✅ 徽章二级提醒（主流程内嵌）:
  │    ObjectType==BadgeAwardObjectType?
  │      └─→ 是: AddBadgeAwardAlertCache() → answer:red-dot:badge:{userID}，JSON 写入
  │
  ├─→ go SendNotificationToAllFollower(msg, questionID)
  │    ├─→ 闸1: NoNeedPushAllFollow || questionID=="" → return
  │    ├─→ 闸2: Action in 白名单? → return
  │    ├─→ GetFollowUserIDs(condObjectID)
  │    └─→ 循环: 复制消息 + NoNeedPushAllFollow=true → 入队
  │
  └─→ Type==Inbox?
       └─→ 是: syncNotificationToPlugin() → 推送到第三方插件
```
