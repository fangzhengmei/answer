# 标签积分配置、通知扇出与 Cancelled 字段语义分析

## 一、`edit.accepted` 与 `tag.edit_accepted` 的关系

### 1.1 两个 config 在初始化数据中的定义

**位置：** [init_data.go#L230-L237](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/migrations/init_data.go#L230-L237)

| ID | Key | Value | 说明 |
|----|-----|-------|------|
| 4 | `tag.edit_accepted` | `2` | **标签**编辑被采纳的积分 |
| 11 | `edit.accepted` | `2` | **通用**编辑被采纳的积分 |

两者默认值相同，都是 `2` 积分。

### 1.2 `edit.accepted` 的使用方

#### 使用方 1：ActivityType 常量

**位置：** [activity_type/activity_type.go#L34](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/activity_type/activity_type.go#L34)

```go
const (
    // ...
    EditAccepted = "edit.accepted"
)
```

并且在 `ActivityTypeList` 中被列出，说明这是系统级活动类型。

#### 使用方 2：ReviewActivityRepo 常量

**位置：** [review_repo.go#L50](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity/review_repo.go#L50)

```go
const (
    EditAccepted = "edit.accepted"
)
```

#### 使用方 3：Review() —— 积分发放的核心

**位置：** [review_repo.go#L69-L125](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity/review_repo.go#L69-L125)

```go
func (ar *ReviewActivityRepo) Review(ctx context.Context, act *schema.PassReviewActivity) error {
    // 通过 key 读取 config 表
    cfg, err := ar.configService.GetConfigByKey(ctx, EditAccepted)  // "edit.accepted"
    
    addActivity := &entity.Activity{
        UserID:           act.UserID,
        TriggerUserID:    converter.StringToInt64(act.TriggerUserID),
        ObjectID:         act.ObjectID,
        OriginalObjectID: act.OriginalObjectID,
        ActivityType:     cfg.ID,           // config.ID = 11
        Rank:             cfg.GetIntValue(), // config.Value = 2
        HasRank:          1,                 // 标记该 activity 影响积分
        RevisionID:       converter.StringToInt64(act.RevisionID),
    }
    
    ar.data.DB.Transaction(func(session *xorm.Session) {
        // 1. 锁定用户行 FOR UPDATE
        session.ID(addActivity.UserID).ForUpdate().Get(user)
        
        // 2. 防重：检查是否已存在同 (UserID, ActivityType, RevisionID) 的 activity
        session.Where("user_id = ? AND activity_type = ? AND revision_id = ?", ...).Get(existsActivity)
        if exist {
            return nil, nil  // 幂等：已发放过则跳过
        }
        
        // 3. 增量更新用户积分 Rank
        ar.userRankRepo.ChangeUserRank(ctx, session, addActivity.UserID, user.Rank, addActivity.Rank)
        
        // 4. 插入 activity 记录
        session.Insert(addActivity)
    })
}
```

#### 使用方 4：RevisionAudit —— 审核通过后发放积分

**位置：** [revision_service.go#L157-L163](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/revision_service.go#L157-L163)

```go
// 所有修订审核通过后（无论对象类型是 Question / Answer / Tag），统一调用
err = rs.reviewActivity.Review(ctx, &schema.PassReviewActivity{
    UserID:           revisioninfo.UserID,    // 编辑者（要给积分的人）
    TriggerUserID:    req.UserID,             // 审核者
    ObjectID:         revisioninfo.ObjectID,  // 被编辑的对象 ID（问题/答案/标签）
    OriginalObjectID: "0",
    RevisionID:       revisioninfo.ID,        // revision ID（用于幂等去重）
})
```

**关键：** `reviewActivity.Review()` 被调用的位置**不区分对象类型**，Question、Answer、Tag 的修订审核通过都走同一条路径，使用同一个 config key `edit.accepted`。

### 1.3 `tag.edit_accepted` 的使用方

**结论：`tag.edit_accepted` 是孤儿 config，代码中没有任何使用方。**

使用 grep 在整个代码库中搜索 `tag\.edit_accepted`：
- `internal/migrations/init_data.go`：仅在默认配置初始化时出现
- **没有任何业务代码**读取这个 config key

### 1.4 证据链总结

| 维度 | `edit.accepted` (ID=11) | `tag.edit_accepted` (ID=4) |
|------|------------------------|---------------------------|
| 默认值 | 2 | 2 |
| ActivityTypeList 注册 | ✅ 在 activity_type 中 | ❌ 不在 |
| 常量定义 | ✅ 3 处定义（activity_type / review_repo） | ❌ 0 处 |
| GetConfigByKey 读取 | ✅ ReviewActivityRepo.Review() | ❌ 0 处 |
| 发放积分 | ✅ 所有修订审核通过 | ❌ 不发放 |
| Question 编辑通过 | ✅ 走此路径 | ❌ 不走 |
| Answer 编辑通过 | ✅ 走此路径 | ❌ 不走 |
| Tag 编辑通过 | ✅ 走此路径（混用通用路径） | ❌ 不走 |

### 1.5 历史推测

`tag.edit_accepted` 极有可能是早期版本的遗留配置：
- 最初设计时计划为 Question/Answer/Tag 分别提供独立的 edit_accepted 积分配置
- 后来重构时统一为 `edit.accepted`，但 `tag.edit_accepted` 没有从初始化数据中删除
- 目前它静默存在于 config 表中，不产生任何效果

### 1.6 风险点

| 风险 | 说明 |
|------|------|
| **管理员配置误导** | 后台若暴露了 `tag.edit_accepted` 配置项，管理员修改它不会有任何效果 |
| **积分差异化缺失** | 标签编辑和问题/答案编辑默认都是 2 积分，但标签编辑通常贡献度低于问题编辑，若需差异化无法通过现有配置实现 |
| **孤儿配置清理风险** | 未来若代码不小心引用了 `tag.edit_accepted`，会出现两套积分配置 |

---

## 二、提问后向标签关注者扇出通知的链路

### 2.1 扇出链路全景图

```
用户提交问题
    │
    ▼
QuestionService.AddQuestion() / Review 审核通过
    │
    ├─ 条件: question.Status == QuestionStatusAvailable
    │
    ├─ 构造通知消息: CreateNewQuestionNotificationMsg()
    │     │
    │     ├─ QuestionID / QuestionTitle / QuestionAuthorUserID
    │     ├─ Tags: [tag.SlugName, ...]
    │     └─ TagIDs: [tag.ID, ...]
    │
    ▼
externalNotificationQueueService.Send(ctx, msg)
    │
    ▼
external_notification Queue (内存 channel, buffer=128)
    │
    ▼ (单 goroutine 消费)
    │
ExternalNotificationService.Handler(ctx, msg)
    │
    ├─ msg.NewQuestionTemplateRawData != nil → handleNewQuestionNotification()
    │
    ▼
handleNewQuestionNotification(msg)
    │
    ├─ getNewQuestionSubscribers(msg)
    │     │
    │     ├─ 第 1 部分: 标签关注者（核心扇出）
    │     │     │
    │     │     ├─ for tagID ∈ msg.TagIDs:
    │     │     │     └─ followRepo.GetFollowUserIDs(tagID)
    │     │     │           │
    │     │     │           └─ SELECT user_id FROM activity
    │     │     │                WHERE object_id = tagID
    │     │     │                  AND activity_type = (follow-type)
    │     │     │                  AND cancelled = 0
    │     │     │
    │     │     ├─ followerMapping[userID] = true  ← 去重（一个用户关注多个标签只发一次）
    │     │     │
    │     │     └─ GetByUsersAndSource(AllNewQuestionForFollowingTagsSource)
    │     │           │
    │     │           └─ 查 user_notification_config，筛选出启用了"关注标签新问题通知"的用户
    │     │
    │     ├─ 第 2 部分: 全站新问题订阅者
    │     │     │
    │     │     └─ GetBySource(AllNewQuestionSource)
    │     │           │
    │     │           └─ 查通知配置中开启了"所有新问题通知"的用户
    │     │
    │     └─ 第 3 步: 移除提问者本人（自己不通知自己）
    │
    ├─ for subscriber ∈ subscribers:
    │     for channel ∈ subscriber.Channels:
    │         channel.Enable && channel.Key == EmailChannel → sendNewQuestionNotificationEmail()
    │               │
    │               ├─ checkUserStatusBeforeNotification() → 被禁用户跳过
    │               ├─ 获取用户语言 → 设置 i18n ctx
    │               ├─ emailService.NewQuestionTemplate() → 渲染邮件模板
    │               ├─ 生成 UnsubscribeCode
    │               └─ emailService.SendAndSaveCodeWithTime() → 发送 + 写入验证码表
    │
    └─ syncNewQuestionNotificationToPlugin(msg)
          │
          ├─ 同样收集标签关注者 + 全站订阅者 + 移除本人
          └─ plugin.CallNotification() → 每个通知插件 fn.Notify()
```

### 2.2 扇出的两个触发入口

#### 入口 1：提问时立即扇出（状态=Available）

**位置：** [question_service.go#L450-L460](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/question_service.go#L450-L460)

```go
if question.Status == entity.QuestionStatusAvailable {
    newTags, newTagsErr := qs.tagCommon.GetTagListByNames(ctx, tagNameList)
    if newTagsErr != nil {
        qs.externalNotificationQueueService.Send(ctx,
            schema.CreateNewQuestionNotificationMsg(question.ID, question.Title, question.UserID, tags))
    } else {
        qs.externalNotificationQueueService.Send(ctx,
            schema.CreateNewQuestionNotificationMsg(question.ID, question.Title, question.UserID, newTags))
    }
}
```

**触发条件：** 提问后立即判定为 Available（无需审核）。

#### 入口 2：审核通过后扇出（Pending → Available）

**位置：** [review_service.go#L291-L296](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/review/review_service.go#L291-L296)

```go
tags, err := cs.tagCommon.GetObjectEntityTag(ctx, questionInfo.ID)
cs.externalNotificationQueueService.Send(ctx,
    schema.CreateNewQuestionNotificationMsg(
        questionInfo.ID, questionInfo.Title, questionInfo.UserID, tags))
```

**触发条件：** 问题原本是 Pending 状态，管理员审核通过后变为 Available。

### 2.3 通知消息构造

**位置：** [new_question_queue_schema.go#L38-L52](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/schema/new_question_queue_schema.go#L38-L52)

```go
func CreateNewQuestionNotificationMsg(
    questionID, questionTitle, questionAuthorUserID string, tags []*entity.Tag,
) *ExternalNotificationMsg {
    msg := &ExternalNotificationMsg{
        NewQuestionTemplateRawData: &NewQuestionTemplateRawData{
            QuestionAuthorUserID: questionAuthorUserID,
            QuestionID:           questionID,
            QuestionTitle:        questionTitle,
        },
    }
    for _, tag := range tags {
        msg.NewQuestionTemplateRawData.Tags = append(msg.NewQuestionTemplateRawData.Tags, tag.SlugName)
        msg.NewQuestionTemplateRawData.TagIDs = append(msg.NewQuestionTemplateRawData.TagIDs, tag.ID)
    }
    return msg
}
```

**重要：** `TagIDs` 中存的是标签 ID（**不是主标签 ID**），与 tag_rel 表中存的一致。

### 2.4 核心扇出：`getNewQuestionSubscribers()`

**位置：** [new_question_notification.go#L74-L138](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/notification/new_question_notification.go#L74-L138)

算法分 **3 步**，每一步结果**去重合并**：

```
Step 1: 标签关注者（按 TagIDs 逐个查询后合并去重）
  ├─ for tagID ∈ msg.TagIDs:
  │     followRepo.GetFollowUserIDs(tagID)
  │        → SELECT user_id FROM activity
  │             WHERE object_id = tagID
  │               AND cancelled = 0  ← 只取有效关注
  └─ 按 userID 去重，查询其通知配置中 AllNewQuestionForFollowingTagsSource 是否开启

Step 2: 全站新问题订阅者
  └─ userNotificationConfigRepo.GetBySource(AllNewQuestionSource)
       → 通知配置表中 notification_source = "all_new_question" 的用户

Step 3: 移除提问者本人
  └─ delete(subscribersMapping, QuestionAuthorUserID)
```

### 2.5 通知配置与过滤

用户可以在个人设置中配置是否接收以下两类通知（位于 `user_notification_config` 表）：

| NotificationSource | 说明 | 对应扇出步骤 |
|-------------------|------|-------------|
| `all_new_question_for_following_tags` | 我关注的标签有新问题 | Step 1 |
| `all_new_question` | 所有新问题 | Step 2 |

**每个用户**还可以单独为每类通知配置启用的 channel（站内信 / 邮件 / 插件通知）。

### 2.6 标签扇出的特殊边界场景

#### 场景 1：合并标签后扇出

若标签 A 被合并到标签 B：
- 旧问题的 tag_rel 已指向 B.ID
- 用户之前关注的是 A.ID，关注记录 activity 中 `object_id = A.ID`
- 新问题的 TagIDs 是 `[B.ID, ...]`
- **GetFollowUserIDs(B.ID) 查不到关注 A 的用户**
- **结果：合并标签后，原本关注源标签 A 的用户**不会收到新问题通知

这是合并标签后**搜索同步之外的另一个一致性缺口**。

#### 场景 2：用户关注了问题的多个标签

用户同时关注 tagA 和 tagB，问题同时打了 tagA 和 tagB。
- `followerMapping[userID] = true` 去重机制保证只收到 **1 封**邮件，不会重复。

#### 场景 3：邮件发送频率限制

**位置：** [new_question_notification.go#L140-L160](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/notification/new_question_notification.go#L140-L160)

```go
key = constant.NewQuestionNotificationLimitCacheKeyPrefix + userID
cache: GetInt64(key) / SetInt64(key, 1, 24h) / Increase(key, 1)
if old >= NewQuestionNotificationLimitMax → 今日已达上限 → 跳过
```

**全站新问题通知**的用户每天最多收到 **N 封**邮件（`NewQuestionNotificationLimitMax`），避免轰炸。

**标签关注者没有频率限制**，但 Step 2 进入的通知会先经过此限制。

### 2.7 插件同步扇出

除了邮件外，还会通过 `syncNewQuestionNotificationToPlugin()` 向所有启用的 Notification 插件广播：
- 通知类型区分 `NotificationNewQuestionFollowedTag`（关注标签） vs `NotificationNewQuestion`（全站）
- 支持每个插件配置独立的 ExternalID（用于 SSO 推送）

---

## 三、`Cancelled` 字段在各场景下的真实语义

### 3.1 `Cancelled` 字段的原始定义

**Entity 定义（entity.Activity）：**

| 值 | 常量 | 名称 |
|----|------|------|
| 0 | `ActivityAvailable` | 有效活动 |
| 1 | `ActivityCancelled` | 已取消 |

### 3.2 各场景下 Cancelled 的使用方式

#### 场景 1：关注 / 取消关注

**关注 Follow():** [follow_repo.go#L60-L125](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity/follow_repo.go#L60-L125)

```go
// 恢复：Cancelled → 0
if has && existsActivity.Cancelled == entity.ActivityCancelled {
    session.Where("id = ?").Cols("cancelled").
        Update(&entity.Activity{Cancelled: entity.ActivityAvailable})
}
// 新建：默认 Cancelled = 0
Insert(&entity.Activity{
    ...
    Cancelled: entity.ActivityAvailable,  // 0
})
// 同时更新 tag.follow_count +1
updateFollows(ctx, session, objectID, 1)
```

**取消关注 FollowCancel():** [follow_repo.go#L127-L169](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity/follow_repo.go#L127-L169)

```go
// 软删除：Cancelled → 1
session.Where("id = ?").Cols("cancelled", "cancelled_at").
    Update(&entity.Activity{
        Cancelled:   entity.ActivityCancelled,  // 1
        CancelledAt: time.Now(),
    })
// 同时更新 tag.follow_count -1
updateFollows(ctx, session, objectID, -1)
```

**语义：** Cancelled=0 表示用户正在关注；Cancelled=1 表示用户曾经关注但已取消。这是**典型的软删除**模式。

#### 场景 2：合并标签时的关注迁移

**MigrateFollowers():** [follow.go#L189-L265](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity_common/follow.go#L189-L265)

```go
ar.data.DB.Transaction(func(session) {
    // Step 1: 物理 DELETE 源标签所有 follow activity（不走 Cancelled）
    session.Table("activity").
        Where("object_id = ? AND activity_type = ?", sourceObjectID, activityType).
        Delete(&entity.Activity{})
    
    // Step 2: 对目标标签 + 源关注者的交集 → 恢复 Cancelled=0（之前可能取消过）
    session.Table("activity").
        Where("object_id = ? AND user_id IN ?", targetObjectID, sourceFollowers).
        Cols("cancelled").
        Update(&entity.Activity{Cancelled: entity.ActivityAvailable})
    
    // Step 3: 查询目标标签的 Cancelled=0 关注者
    SELECT user_id FROM activity WHERE object_id=target AND cancelled=0
    
    // Step 4: 不重复的源关注者 → 插入新记录 Cancelled=0
    Insert(&entity.Activity{
        UserID:           uid,
        ObjectID:         targetObjectID,
        Cancelled:        entity.ActivityAvailable,
    })
})
```

**语义：** 合并时源标签的 follow 记录被**物理删除**（不经过 Cancelled=1）。目标标签的旧记录如果之前是 Cancelled=1 就恢复为 Cancelled=0。

#### 场景 3：标签软删除（status=10）

**RemoveTag():** [tag_service.go#L75-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L75-L107)

```go
// 只改 tag.status = 10
tagRepo.RemoveTag(ctx, req.TagID)

// 不清理 activity 表中 Cancelled=0 的 follow 记录
// 不更新 tag.follow_count
```

**语义：** 标签软删除时，activity 表中的 follow 记录**完全不做处理**，Cancelled 字段仍然是 0。

**后续影响：**
- `GetFollowUserIDs(deletedTagID)` → `WHERE cancelled = 0` → **仍然能查到用户**
- 但被删除的标签没有新问题创建 → 不会有通知扇出（因为新问题不会关联这个 tag ID）
- `GetFollowIDs()` 查用户关注列表 → 返回被删除的 tag ID → `GetTagListByIDs()` 会过滤 status=10 的标签 → **前端看不到**
- **activity 表有脏数据但无功能影响**

#### 场景 4：标签恢复（status=1 → 10）

**RecoverTag():** [tag_service.go#L114-L138](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L114-L138)

```go
tagRepo.RecoverTag(ctx, req.TagID)  // status = 1

// 同样不处理 activity 表
// Cancelled 仍然是 0
// tag.follow_count 仍然是删除前的值
```

**语义：** 恢复后之前的关注关系依然有效，因为 Cancelled=0 还在。用户不需要重新关注。

#### 场景 5：投票 / 采纳等其他类型 activity

Vote / Accept 等操作同样使用 Cancelled 字段，语义与关注类似：
- Cancelled=0 表示投票有效
- Cancelled=1 表示取消投票（如取消采纳、撤销点赞）
- 每次切换时同步更新计数（如 answer.accept_count, question.vote_count）

### 3.3 Cancelled 语义总结表

| 场景 | Cancelled=0 | Cancelled=1 | 是否物理删除？ |
|------|------------|-------------|----------------|
| 用户关注标签 | ✅ 正在关注 | 已取消关注 | 否 |
| 用户取消关注 | - | ✅ 标记取消 | 否（软删除） |
| 合并标签源标签 follow | - | - | ✅ **物理 DELETE** |
| 合并标签目标标签旧 follow | ✅ 恢复关注（之前可能=1） | - | 否 |
| 标签软删除（status=10） | **保持不变**（仍是0） | - | 否（不处理activity） |
| 标签恢复（status=1） | **保持不变**（仍是0） | - | 否（不处理activity） |
| 投票 / 采纳 | ✅ 有效操作 | 已撤销操作 | 否 |
| 重复关注恢复 | ✅ 从1改为0 | - | 否 |

### 3.4 Cancelled 相关的查询约束

**所有读取 follow 关系的查询都强制带 `cancelled = 0` 条件：**

| 函数 | 查询条件 |
|------|---------|
| `GetFollowUserIDs()` [follow.go#L110](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity_common/follow.go#L110) | `WHERE object_id = ? AND activity_type = ? AND cancelled = 0` |
| `GetFollowIDs()` [follow.go#L128](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity_common/follow.go#L128) | `WHERE user_id = ? AND activity_type = ? AND cancelled = 0` |
| `IsFollowed()` [follow.go#L150-L159](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity_common/follow.go#L150-L159) | `Get() + at.Cancelled == ActivityCancelled → false` |
| `GetFollowAmount()` [follow.go#L59-L91](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity_common/follow.go#L59-L91) | **不经过 activity 表**，直接读 `tag.follow_count` |

### 3.5 风险与不一致

| 风险 | 位置 | 说明 |
|------|------|------|
| **合并源标签计数不更新** | 合并流程 | 物理删除了 activity 但没 `updateFollows(sourceObjectID, -N)` |
| **合并目标标签计数不更新** | 合并流程 | 新增和恢复了关注但没 `updateFollows(targetObjectID, +N)` |
| **重复关注的竞态** | Follow() | `Get() → if Cancelled=1 → Update()` 两步非原子，并发下可能漏更新 |
| **follow_count 与实际数不一致** | 长期运行 | follow_count 只在 Follow/FollowCancel 时用 `Incr/Decr` 更新，与 activity 表 `COUNT WHERE cancelled=0` 可能漂移 |
| **软删除标签 activity 残留** | RemoveTag | 被删除标签的 Cancelled=0 activity 永远不会被清理 |
| **合并标签通知扇出缺口** | 见 2.6 节 | 用户关注源标签，合并后新问题 TagIDs=目标标签ID，不会向该用户扇出通知 |

---

## 四、总结

### 4.1 三个问题的答案汇总

| 问题 | 结论 |
|------|------|
| `edit.accepted` vs `tag.edit_accepted` | `edit.accepted` (ID=11) 是实际使用的通用积分配置，所有修订审核通过都走它；`tag.edit_accepted` (ID=4) 是孤儿配置，无任何使用方 |
| 提问 → 标签关注者通知扇出 | 经过 external_notification 队列 → getNewQuestionSubscribers() 按 TagIDs 逐个查 `GetFollowUserIDs(cancelled=0)` → 去重 → 过滤通知配置 → 移除本人 → 按 channel 发送邮件/插件通知 |
| Cancelled 字段语义 | 0=有效关注/活动；1=取消；合并时**物理 DELETE** 不走 Cancelled；标签软删除不清理 activity（功能无影响但有脏数据）；所有 follow 查询都强制 `cancelled = 0` |

### 4.2 改进建议

1. **删除 `tag.edit_accepted` 孤儿配置**，或在 `ReviewActivityRepo.Review()` 中按 ObjectType 分发读取不同的 config key 以支持差异化积分
2. **合并标签后同步关注映射**：将 `OriginalObjectID` 改为源标签 ID，或在 `GetFollowUserIDs()` 中用同义词表做 JOIN 查询覆盖被合并的源标签
3. **合并标签同步 follow_count**：在 `MigrateFollowers` 中补充 `updateFollows` 调用
4. **标签软删除清理 follow activity**：删除标签时将相关 activity 设为 Cancelled=1 或同时更新 follow_count=0
5. **Follow() 防并发**：将 `Get() → Update/Insert` 改为单条 UPSERT SQL 或加行锁
