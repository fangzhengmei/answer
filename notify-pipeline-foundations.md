# 通知管线基础细节补全

## 一、RankAgent 开关为真时所有成就通知整条跳过的源头与影响面

### 1.1 开关定义源头

[plugin/user_center.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/plugin/user_center.go#L53-L54)

```go
type UserCenterDesc struct {
    RankAgentEnabled bool `json:"rank_agent_enabled"`
}
```

RankAgent 是 **UserCenter 插件** 的一个布尔开关，表示"声望（Rank）是否由外部用户中心统一管理"。启用后，系统内置的声望计算、存储、通知全部旁路。

开关读取函数：

[RankAgentEnabled](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/plugin/user_center.go#L112-L118)

```go
func RankAgentEnabled() (enabled bool) {
    _ = CallUserCenter(func(fn UserCenter) error {
        enabled = fn.Description().RankAgentEnabled
        return nil
    })
    return
}
```

无插件注册时返回 `false`（Go 零值），即默认使用系统内置声望。

### 1.2 四层门控位置

RankAgent 在整条声望链路有 **4 处独立门控**，覆盖用户声望仓储、用户服务、声望服务、通知系统：

#### 门控 1：声望仓储 — 不执行用户声望变更

[user_rank_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/rank/user_rank_repo.go#L87-L88)

```go
func (ur *UserRankRepo) ChangeUserRank(ctx context.Context, session *xorm.Session,
    userID string, userCurrentScore, deltaRank int) (err error) {
    // IMPORTANT: If user center enabled the rank agent, then we should not change user rank.
    if plugin.RankAgentEnabled() || deltaRank == 0 {
        return nil  // ✅ 直接返回，不更新 user.rank 字段
    }
    // ... 实际 UPDATE `user` SET `rank` = `rank` + deltaRank WHERE `id` = ?
}
```

**影响面**：用户表的 `rank` 字段不再变化。声望完全由外部 UserCenter 维护。

#### 门控 2：用户服务 — 声望榜和投票榜返回空

[user_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/content/user_service.go#L878-L901)

```go
func (us *UserService) getActivityUserRankStat(ctx context.Context, ...) (...) {
    if plugin.RankAgentEnabled() {
        return make([]*entity.ActivityUserRankStat, 0), make([]string, 0), nil
    }
    // ... 查询 activity 表聚合声望榜
}

func (us *UserService) getActivityUserVoteStat(ctx context.Context, ...) (...) {
    if plugin.RankAgentEnabled() {
        return make([]*entity.ActivityUserVoteStat, 0), make([]string, 0), nil
    }
    // ... 查询 activity 表聚合投票榜
}
```

**影响面**：用户首页 / 排行榜的声望榜和投票榜返回空数组，前端不展示。

#### 门控 3：声望服务 — 个人声望页返回空

[rank_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/rank/rank_service.go#L265-L267)

```go
func (rs *RankService) GetRankPersonalPage(ctx context.Context, req *schema.GetRankPersonalWithPageReq) (...) {
    if plugin.RankAgentEnabled() {
        return pager.NewPageModel(0, []string{}), nil
    }
    // ... 查询用户个人声望记录
}
```

**影响面**：用户个人中心 "我的声望" 页面返回空分页。

#### 门控 4：通知系统 — 所有成就通知整条跳过

[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L110-L112)

```go
func (ns *NotificationCommon) AddNotification(ctx context.Context, msg *schema.NotificationMsg) (err error) {
    if msg.Type == schema.NotificationTypeAchievement && plugin.RankAgentEnabled() {
        return nil  // ✅ 整条 return，所有后续步骤全部跳过
    }
    // ↓↓↓ 以下全部跳过 ↓↓↓
    // objectInfoService.GetInfo()
    // GetByUserIdObjectIdTypeId() 成就去重查询
    // GetUserIDObjectIDActivitySum() 声望汇总查询
    // notificationRepo.AddNotification() 写库
    // addRedDot() 增加成就红点
    // AddBadgeAwardAlertCache() 徽章弹窗（⚠️ 徽章弹窗也受影响！）
    // SendNotificationToAllFollower() 关注者推送
    // syncNotificationToPlugin() 插件同步
}
```

**影响面**：
- ✅ 投票声望通知：不写库，不弹红点
- ✅ 采纳声望通知：不写库，不弹红点
- ✅ 修订审核声望通知：不写库，不弹红点
- ✅ 取消采纳声望通知：根本就不进入 AddNotification（上游无变化）
- ⚠️ **徽章授予通知也被跳过**：因为徽章的 `msg.Type` 也是 `NotificationTypeAchievement (2)`，`AddBadgeAwardAlertCache` 在 RankAgent 为真时完全不会执行 → **徽章弹窗彻底失效**。这是 RankAgent 设计上的副作用——徽章与声望被绑定在同一通知类型下，启用外部声望管理会连带关闭徽章弹窗提醒。

#### 额外：用户仓储 — 登录同步时用外部声望覆盖本地

[user_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/user/user_repo.go#L377-L379)

```go
// If plugin enable rank agent, use rank from user center.
if plugin.RankAgentEnabled() {
    original.Rank = ucUser.Rank  // 每次登录同步时，本地 rank 被外部值覆盖
}
```

### 1.3 门控汇总表

| 门控位置 | 模块 | 方法 | 行为 |
|---------|------|------|------|
| 1 | 声望仓储 | `ChangeUserRank` | 不更新 `user.rank` 字段 |
| 2 | 用户服务 | `getActivityUserRankStat` / `getActivityUserVoteStat` | 返回空排行榜 |
| 3 | 声望服务 | `GetRankPersonalPage` | 返回空分页 |
| 4 | 通知系统 | `AddNotification` | Type=Achievement 直接 return，成就/徽章全部跳过 |
| 5（额外） | 用户仓储 | 登录同步 | 用外部 `ucUser.Rank` 覆盖本地 |

---

## 二、Rank 由活动汇总函数对该用户该对象未取消活动 rank 列累加的语义

### 2.1 查询语句

[activity_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/repo/activity_common/activity_repo.go#L123-L136)

```go
func (ar *ActivityRepo) GetUserIDObjectIDActivitySum(ctx context.Context,
    userID, objectID string) (int, error) {
    sum := &entity.ActivityRankSum{}
    _, err := ar.data.DB.Context(ctx).Table(entity.Activity{}.TableName()).
        Select("sum(`rank`) as `rank`").
        Where("user_id = ?", userID).
        And("object_id = ?", objectID).
        And("cancelled = 0").  // ✅ 只统计未取消的活动
        Get(sum)
    return sum.Rank, nil
}
```

**SQL 语义**：

```sql
SELECT SUM(`rank`) AS `rank`
FROM `activity`
WHERE `user_id` = ?
  AND `object_id` = ?
  AND `cancelled` = 0;
```

### 2.2 维度说明

| 维度 | 含义 | 示例 |
|------|------|------|
| **user_id** | 声望归属用户 | "3XkEexP28rqgBkE9Yp9Ttc"（某用户ID） |
| **object_id** | 关联的具体对象 | "3aU1M73P2S2O"（某回答ID） |
| **cancelled = 0** | 活动未被取消/回滚 | 取消采纳时，之前的采纳活动被标记 `cancelled=1`，就不再计入 |
| **SUM(rank)** | rank 列的代数和（有负有正） | 投票声望 +10，被踩声望 -2，合计 +8 |

### 2.3 设计意图

该函数用于"对某用户在某具体对象上累计获得的净声望"实时计算：

- 采纳某回答 → 回答作者获得 +15 声望 → 活动表插入 rank=15 的活动
- 取消采纳 → 该活动 `cancelled=1` → SUM 时自动排除 → 相当于声望 -15
- 回答被投票 +10 × 3 次 → 三条 rank=10 活动 → SUM = 30
- 回答被踩 -2 次 → 一条 rank=-2 活动 → SUM = 28

这是成就通知去重时更新 `req.Rank` 的基础数据来源。成就通知 Content JSON 中的 `Rank` 字段永远是这个 SUM 的实时值，而不是增量值。

### 2.4 调用链

```
AddNotification(msg.Type=Achievement)
  └─→ 成就去重分支
       ├─→ GetByUserIdObjectIdTypeId(userID, objectID, type=2)  // 查是否已有成就通知
       └─→ GetUserIDObjectIDActivitySum(userID, objectID)       // ✅ 实时汇总该用户该对象的净声望
            └─→ SQL: SUM(rank) WHERE user_id=? AND object_id=? AND cancelled=0
                 └─→ req.Rank = sum
                      ├─ 去重命中 → UpdateNotificationContent(更新 Content 中的 Rank)
                      └─ 未命中   → AddNotification(写入新通知，Content 含当前 Rank)
```

---

## 三、关注者推送白名单里问题更新动作全代码无设置点的死动作

### 3.1 白名单定义

[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L333-L337)

```go
if msg.NotificationAction != constant.NotificationUpdateQuestion &&  // 🚩 死动作
    msg.NotificationAction != constant.NotificationAnswerTheQuestion &&
    msg.NotificationAction != constant.NotificationUpdateAnswer &&
    msg.NotificationAction != constant.NotificationAcceptAnswer {
    return
}
```

白名单中 4 个动作：

| 动作常量 | 全代码是否有设置 `msg.NotificationAction = 该值` | 实际使用 |
|---------|:------------------------------------------:|---------|
| `NotificationUpdateQuestion` | ❌ **无设置点** | 死动作 |
| `NotificationAnswerTheQuestion` | ✅ answer_service.go L777 | 新回答时通知问题作者 |
| `NotificationUpdateAnswer` | ✅ answer_service.go L759 | 回答修订审核通过时 |
| `NotificationAcceptAnswer` | ✅ answer_repo.go L325 | 采纳回答时 |

### 3.2 死动作验证

搜索全代码库 `"NotificationAction.*=.*NotificationUpdateQuestion"` 和 `".*=.*constant\.NotificationUpdateQuestion"` 均无匹配。

对比另外三个动作的设置点：

```go
// ✅ NotificationAnswerTheQuestion 有设置
msg.NotificationAction = constant.NotificationAnswerTheQuestion
// [answer_service.go notificationAnswerTheQuestion]

// ✅ NotificationUpdateAnswer 有设置
msg.NotificationAction = constant.NotificationUpdateAnswer
// [answer_service.go notificationUpdateAnswer]

// ✅ NotificationAcceptAnswer 有设置
msg.NotificationAction = constant.NotificationAcceptAnswer
// [answer_repo.go sendAcceptAnswerNotification]

// ❌ NotificationUpdateQuestion — 无任何地方设置
```

### 3.3 死动作的可能来源

`NotificationUpdateQuestion` 的常量定义（[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/base/constant/notification.go#L24)）：

```go
NotificationUpdateQuestion = "notification.action.update_question"
```

该常量在 `NotificationMsgTypeMapping` 中也被映射为 MsgType=1（帖子类），且在插件 `NotificationType` 枚举中也有定义。但在业务层（question_service / revision_service / review_service）均未被实际赋值给任何通知。

**推测**：这是预留动作，原计划用于"问题被编辑/修订后通知关注者"，但实际功能未实现或被废弃（问题编辑走的是 revision 审核路径，但审核通过时 revision_service 只发送 `NotificationUpdateAnswer`，不发送问题更新的通知）。

### 3.4 实际影响

因为白名单检查是"非 A 且非 B 且非 C 且非 D → return"，缺少 D 的设置点只是白名单中多了一个永远不会命中的条件，不影响其他三个动作的正常运行。属于**代码垃圾**而非 bug。

---

## 四、动作映射表里评论被删除归入帖子类却全代码无设置点的死映射

### 4.1 映射表中的定义

[notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/base/constant/notification.go#L100)

```go
var NotificationMsgTypeMapping = map[string]int{
    // ...
    NotificationYourQuestionWasDeleted: 1,  // ✅ 有设置点：question_service.go L1605
    NotificationYourAnswerWasDeleted:   1,  // ✅ 有设置点：answer_service.go L643
    NotificationYourCommentWasDeleted:  1,  // 🚩 归入帖子类 MsgType=1，但无设置点
}
```

### 4.2 死映射验证

搜索全代码库 `NotificationAction.*=.*constant\.NotificationYourCommentWasDeleted` 无匹配。

对比另外两个"被删除"动作的设置点：

```go
// ✅ NotificationYourQuestionWasDeleted 有设置
if setStatus == entity.QuestionStatusDeleted && questionInfo.Status != entity.QuestionStatusDeleted {
    msg.NotificationAction = constant.NotificationYourQuestionWasDeleted
}
// [question_service.go L1605]

// ✅ NotificationYourAnswerWasDeleted 有设置
msg.NotificationAction = constant.NotificationYourAnswerWasDeleted
// [answer_service.go L643]

// ❌ NotificationYourCommentWasDeleted — 无任何地方设置
```

### 4.3 死映射的可能来源

评论删除服务 [comment_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/comment/comment_service.go) 中，管理员/用户删除评论时只处理声望回滚和 activity 表标记，没有构建 `NotificationMsg` 发给评论作者"你的评论被删除了"的通知。

**推测**：这是预留动作，原计划对齐问题/回答删除的通知模式，让评论作者也收到删除通知，但实际未实现。

### 4.4 实际影响

`NotificationYourCommentWasDeleted` 常量也未出现在关注者推送白名单、插件同步、邮件模板等任何下游逻辑中。属于**完全孤立的死代码常量 + 死映射条目**，不影响任何运行时行为。

### 4.5 四个删除类动作的现状汇总

| 动作常量 | MsgType 映射 | 设置点 | 通知接收者 | 状态 |
|---------|:-----------:|:------:|---------|:----:|
| `NotificationYourQuestionWasDeleted` | 1（帖子） | ✅ question_service.go | 问题作者 | 生效 |
| `NotificationYourAnswerWasDeleted` | 1（帖子） | ✅ answer_service.go | 回答作者 | 生效 |
| `NotificationYourCommentWasDeleted` | 1（帖子） | ❌ 无 | — | **死映射** |
| `NotificationYourQuestionIsClosed` | 1（帖子） | ✅ （question_service.go setStatus=Closed） | 问题作者 | 生效 |

---

## 五、徽章弹窗读取函数内部反序列化缓存 JSON、挑出最近一个徽章、点击后清除这条完整生命周期

### 5.1 缓存结构回顾

缓存键：`answer:red-dot:badge:{userID}`

缓存值：[RedDotBadgeAwardCache](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/schema/notification_schema.go#L120-L169)

```json
{
  "badge_award_list": {
    "notif_abc123": {
      "notification_id": "notif_abc123",
      "badge_id": "badge_gold",
      "name": "",      // 读取时填充
      "icon": "",      // 读取时填充
      "level": 0       // 读取时填充
    },
    "notif_def456": {
      "notification_id": "notif_def456",
      "badge_id": "badge_silver",
      ...
    }
  }
}
```

写入时只存 `notification_id` 和 `badge_id`，`name/icon/level` 留空，读取时实时查 badge 表填充。

### 5.2 读取：反序列化 + 排序挑选最早一个

[getBadgeAward](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/notification_service.go#L99-L128)

```go
func (ns *NotificationService) getBadgeAward(ctx context.Context, userID string) (badgeAward *schema.RedDotBadgeAward) {
    // 1. 取 Redis 值
    key := fmt.Sprintf(constant.RedDotCacheKey, constant.NotificationTypeBadgeAchievement, userID)
    cacheData, exist, err := ns.data.Cache.GetString(ctx, key)
    if !exist {
        return nil
    }

    // 2. ✅ JSON 反序列化
    c := schema.NewRedDotBadgeAwardCache()
    c.FromJSON(cacheData)  // 内部：json.Unmarshal([]byte(data), r)

    // 3. ✅ 挑出最早（NotificationID 字典序最小）的一个
    award := c.GetBadgeAward()
    // GetBadgeAward() 实现 [notification_schema.go L132-L142]:
    //   - 如果 BadgeAwardList 为空 → return nil
    //   - 收集所有 NotificationID → sort.Strings(ids)
    //   - return BadgeAwardList[ids[0]]  // ✅ 字典序最小 = 最早授予的

    if award == nil {
        return nil
    }

    // 4. 查 badge 表补充展示字段
    badgeInfo, exists, err := ns.badgeRepo.GetByID(ctx, award.BadgeID)
    if !exists {
        return nil
    }
    award.Name = translator.Tr(handler.GetLangByCtx(ctx), badgeInfo.Name)  // i18n 翻译
    award.Icon = badgeInfo.Icon
    award.Level = badgeInfo.Level

    return award
}
```

**挑选策略**：NotificationID 是数据库自增 ID 的短编码，字典序最小 = 创建时间最早。前端每次 GetRedDot 只展示最早一个未读徽章，读完点"已读"后清除该条目，下次展示下一个。

### 5.3 点击清除：单条标记已读时移除徽章弹窗缓存

[ClearIDUnRead](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification/notification_service.go#L184-L206)

```go
func (ns *NotificationService) ClearIDUnRead(ctx context.Context, userID string, id string) error {
    // 1. 校验通知归属（只能读自己的通知）
    notificationInfo, exist, err := ns.notificationRepo.GetById(ctx, id)
    if !exist || notificationInfo.UserID != userID {
        return nil
    }

    // 2. DB 标记已读
    if notificationInfo.IsRead == schema.NotificationNotRead {
        err := ns.notificationRepo.ClearIDUnRead(ctx, userID, id)
        // ...
    }

    // 3. ✅ 移除徽章弹窗中该 notificationID 对应的条目
    err = ns.notificationCommon.RemoveBadgeAwardAlertCache(ctx, userID, id)
    // ...

    // 4. 减少对应类型的红点计数
    _ = ns.notificationCommon.DecreaseRedDot(ctx, userID, notificationInfo.Type)
    return nil
}
```

### 5.4 RemoveBadgeAwardAlertCache 内部实现

[RemoveBadgeAwardAlertCache](file:///d:/fz/0601-1/solo-dogfeeding/code/65-answer/internal/service/notification_common/notification.go#L308-L325)

```go
func (ns *NotificationCommon) RemoveBadgeAwardAlertCache(ctx context.Context,
    userID, notificationID string) (err error) {

    key := fmt.Sprintf(constant.RedDotCacheKey, constant.NotificationTypeBadgeAchievement, userID)
    cacheData, exist, err := ns.data.Cache.GetString(ctx, key)
    if !exist {
        return nil
    }

    // 1. 反序列化
    c := schema.NewRedDotBadgeAwardCache()
    c.FromJSON(cacheData)

    // 2. 删除指定 notificationID 条目
    c.RemoveBadgeAward(notificationID)
    // RemoveBadgeAward() 实现 [notification_schema.go L164-L169]:
    //   delete(r.BadgeAwardList, notificationID)

    // 3. 列表为空 → 删除整个 Redis 键
    if len(c.BadgeAwardList) == 0 {
        return ns.data.Cache.Del(ctx, key)
    }

    // 4. 列表非空 → 重新序列化写回 Redis
    return ns.data.Cache.SetString(ctx, key, c.ToJSON(), constant.RedDotCacheTime)
}
```

### 5.5 徽章弹窗完整生命周期

```
授予徽章
  │
  ├─→ badge_award_service.Award()
  │    ├─→ DB 写入 badge_award 表
  │    └─→ notificationQueueService.Send({
  │           Type: Achievement(2),
  │           ObjectType: BadgeAwardObjectType,
  │           NotificationAction: NotificationEarnedBadge,
  │           ExtraInfo: {"badge_id": "..."}
  │        })
  │
  └─→ AddNotification() 消费
       ├─→ RankAgentEnabled? → return（⚠️ 启用则跳过）
       ├─→ ObjectType == BadgeAwardObjectType?
       │    └─→ 是: 直接塞 ObjectMap{"badge_id", ...}，不走 objectInfoService
       ├─→ 成就去重（按 userID+objectID+type=2）
       │    └─→ 徽章授予 objectID = badge_award.ID，每次授予唯一，几乎不可能命中去重
       ├─→ AddNotification() 写库 → msg_type = 0（NotificationEarnedBadge 不在映射表）
       ├─→ addRedDot → achievement 红点 +1
       ├─→ ✅ AddBadgeAwardAlertCache(userID, notificationID, badgeID)
       │     └─→ answer:red-dot:badge:{userID} 写入 JSON {"badge_award_list":{...}}
       └─→ (成就类型不触发关注者推送和插件同步)

前端 GetRedDot 轮询
  │
  └─→ getBadgeAward(userID)
       ├─→ Redis Get answer:red-dot:badge:{userID}
       ├─→ JSON 反序列化 → RedDotBadgeAwardCache
       ├─→ BadgeAwardList 收集所有 notificationID → 字典序排序
       ├─→ 取 ids[0] 对应的 RedDotBadgeAward
       ├─→ badgeRepo.GetByID(badgeID) 查徽章详情
       └─→ 填充 Name(i18n) / Icon / Level → 返回给前端弹窗展示

用户点击"已读" / 点击通知
  │
  └─→ PUT /notification/read/state { id: notificationID }
       └─→ ClearIDUnRead(userID, id)
            ├─→ DB UPDATE notification SET is_read = 1 WHERE id = ? AND user_id = ?
            ├─→ ✅ RemoveBadgeAwardAlertCache(userID, notificationID)
            │     ├─→ Redis Get → JSON 反序列化
            │     ├─→ delete(BadgeAwardList, notificationID)
            │     ├─→ len(BadgeAwardList) == 0 ? Redis DEL key : Redis SET 回写剩余 JSON
            │     └─→ 下次 GetRedDot 展示下一个最早的徽章（如果还有）
            └─→ DecreaseRedDot(userID, Type=Achievement) → achievement 红点 -1
```

### 5.6 生命周期关键节点

| 阶段 | 操作 | Redis 状态 | DB 状态 |
|------|------|-----------|---------|
| 授予 | AddBadgeAwardAlertCache | `{list: {id1: {badge_id}}}` | badge_award 写入 + notification 写入（未读） |
| 再次授予 | AddBadgeAwardAlertCache | `{list: {id1: {...}, id2: {...}}}` | 第二条 notification（未读） |
| 读取展示 | getBadgeAward | 不变（读操作） | 不变 |
| 点击已读 id1 | RemoveBadgeAwardAlertCache | `{list: {id2: {...}}}` | notification(id1) → is_read=1 |
| 点击已读 id2 | RemoveBadgeAwardAlertCache | **DEL 整个键** | notification(id2) → is_read=1 |
