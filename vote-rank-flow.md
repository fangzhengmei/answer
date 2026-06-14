# 投票与声望计分系统完整代码理解

## 第一部分 投票与声望核心流程（同步执行）

---

## 一、系统整体架构

投票与声望系统是一个分层架构，分为五层：

| 层级 | 职责 | 核心文件 |
|------|------|----------|
| Controller 层 | HTTP 入口，参数绑定、权限校验、验证码、请求分发 | [vote_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/vote_controller.go) |
| Service 层（content） | 投票业务编排：自检、互斥取消、操作委托、结果聚合、事件发送 | [vote_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go) |
| Service 层（rank） | 声望权限检查：判断用户是否达最低声望门槛 | [rank_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/rank/rank_service.go) |
| Repo 层（activity/vote） | 数据持久化：去重、事务、活动记录、声望变更、计数更新 | [vote_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go) |
| Repo 层（rank） | 用户声望数据库操作：每日上限、原子增减、最小保护 | [user_rank_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/rank/user_rank_repo.go) |

核心数据实体：
- [Activity](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/entity/activity_entity.go#L30-L44)：活动/投票记录表（一切状态的**事实源**）
- [User](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/entity/user_entity.go#L43-L73)：用户表，包含 `rank`（声望）字段
- Question/Answer/Comment：各自包含 `vote_count` 计数字段

---

## 二、投票请求主流程（从 HTTP 到 DB）

### 2.1 入口：VoteController

投票分为两个端点：
- `POST /answer/api/v1/vote/up` → [VoteUp](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/vote_controller.go#L68-L110)
- `POST /answer/api/v1/vote/down` → [VoteDown](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/vote_controller.go#L122-L163)

**Controller 层的前置检查**（按顺序）：

1. **参数绑定**：`VoteReq{ObjectID, IsCancel, CaptchaID, CaptchaCode}`，ObjectID 需从短 ID 解码
2. **声望权限检查**：调用 `rankService.CheckVotePermission()`，比较用户当前 `rank` 与配置中的最低声望要求（如 `rank.question.vote_up`）
3. **非管理员验证码**：`actionService.ActionRecordVerifyCaptcha`，通过后记录本次操作
4. **委托调用**：调用 `VoteService.VoteUp / VoteDown`

### 2.2 业务编排：VoteService.VoteUp

以 [VoteUp](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go#L89-L137) 为例，核心步骤：

```
步骤1: GetInfo(objectID) 获取对象信息
       ↓ 检查对象是否已删除 → 错误 NewObjectAlreadyDeleted
       ↓ 检查是否给自己投票 → 错误 DisallowVoteYourSelf
       
步骤2: 构造 VoteOperationInfo（含 Activity 列表）

步骤3: 分支判断 req.IsCancel:
  ├─ 是 Cancel → 调用 voteRepo.CancelVote(up)
  └─ 非 Cancel →
       ① 先调用 voteRepo.CancelVote(down) // 互斥：取消反向票
       ② 再调用 voteRepo.Vote(up)          // 执行正向票

步骤4: GetAndSaveVoteResult() 重新统计并回写 vote_count
步骤5: 非 Cancel 情况下 SendEvent → 事件队列（搜索索引、通知）
```

**关键设计**：投票切换（up↔down）时**先取消反向票再投目标票**，保证同一用户对同一对象只有一种投票状态。

### 2.3 VoteOperationInfo 与 Activity 生成

[createVoteOperationInfo](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go#L243-L256) 与 [getActivities](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go#L258-L299) 是**声望影响配置的入口**。

每次投票会生成 **2 条 Activity**（评论只有 1 条）：

| 对象类型 | 投票方向 | Activity 1（操作人） | Activity 2（被操作人） |
|----------|----------|----------------------|------------------------|
| Question | UP | `question.vote_up` 触发者=自己 | `question.voted_up` 触发者=投我票的人 |
| Question | DOWN | `question.vote_down` | `question.voted_down` |
| Answer | UP | `answer.vote_up` | `answer.voted_up` |
| Answer | DOWN | `answer.vote_down` | `answer.voted_down` |
| Comment | UP | `comment.vote_up` | （无，评论不影响作者声望） |

**被操作人 vs 操作人**的区分逻辑（[vote_service.go:L289-L295](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go#L289-L295)）：
- action 名包含 `voted`（被动态）：`ActivityUserID = ObjectCreatorUserID`，`TriggerUserID = OperatingUserID`
- action 名不含 `voted`（主动态）：`ActivityUserID = OperatingUserID`，`TriggerUserID = "0"`

每条 Activity 的 `Rank` 值从 config 表读取（如 `question.voted_up` 默认 +10，`question.vote_down` 可能是 0 或负数）。

---

## 三、去重机制（防止重复投票）

### 3.1 两层去重设计

系统采用**「预检查 + 保存时检查」**双重去重：

#### 第一层：votePreCheck（预检查）

[votePreCheck](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L210-L222) 在事务开始**之前**执行：

```go
// 查询条件四元组唯一确定一条投票记录
(user_id, trigger_user_id, activity_type, object_id)
```

若所有 Activities 都已存在且 `Cancelled == 0`（ActivityAvailable），则直接返回 `noNeedToVote=true`，跳过整个事务。

#### 第二层：saveActivitiesAvailable（保存时兜底）

[saveActivitiesAvailable](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L312-L358) 在**事务内**执行，防止并发场景下预检查与保存之间的 TOCTOU（Time-Of-Check-Time-Of-Use）竞态：

```
查询 (object_id, user_id, trigger_user_id, activity_type)
  ↓
  ├─ 存在 && Cancelled==0（已投过正向票）
  │     → 设置该 Activity.Rank = 0（本次不再增加声望），跳过
  │
  ├─ 存在 && Cancelled==1（之前取消了，软删除状态）
  │     → UPDATE: Cancelled=0, Rank=新值, HasRank=新值
  │
  └─ 不存在（第一次投）
        → INSERT 一条新的 Activity 记录
```

### 3.2 去重的唯一性键

**逻辑唯一键**（非数据库约束，是查询条件）：
```
object_id + user_id + trigger_user_id + activity_type
```

- `object_id`：问题/回答/评论 ID
- `user_id`：该 Activity 的**声望归属人**（操作人 or 被操作人）
- `trigger_user_id`：触发此行为的用户 ID（主动投票者），"0" 表示无触发者
- `activity_type`：投票类型的配置 ID（int，来自 config 表）

**重要**：同一用户对同一问题投 UP，会产生**两条** Activity 记录：
1. `(user_id=投票者, trigger_user_id=0, activity_type=QuestionVoteUpID)` —— 操作人自己的记录
2. `(user_id=问题作者, trigger_user_id=投票者ID, activity_type=QuestionVotedUpID)` —— 被操作人的记录

两者的去重键不同，因此是两条独立记录。

---

## 四、取消投票机制

### 4.1 取消的触发场景

取消有两种触发路径：

**场景 A：显式取消（用户点了已投票按钮）**
```
req.IsCancel = true
→ VoteService.VoteUp/VoteDown 直接调 voteRepo.CancelVote(目标方向)
```

**场景 B：互斥取消（用户切换投票方向）**
```
用户有 Down 票 → 想投 Up
→ VoteService 先调 voteRepo.CancelVote(down)  // 先撤反向
→ 再调 voteRepo.Vote(up)                      // 再投正向
```

### 4.2 CancelVote 的执行流程

[CancelVote](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L131-L181)：

```
步骤1: getExistActivity() → 查出当前方向下所有已有的 Activity
步骤2: 过滤 Cancelled==1 的记录；若全部已取消，直接 return（幂等）
步骤3: 开启数据库事务：
       ① acquireUserInfo(... ForUpdate ...)   // 行锁相关用户
       ② cancelActivities():
            - 二次检查：若该条已 Cancelled==1 → 将其 Rank=0（不再回滚声望）
            - UPDATE Cancelled=1, CancelledAt=now()
       ③ rollbackUserRank():
            - 对每条 Activity 执行 ChangeUserRank(user, old_rank, -rank)
步骤4: 事务外发送 Achievement 通知（仅 Rank!=0 的）
```

### 4.3 软删除 vs 硬删除

系统使用**软删除**：Activity 表的 `Cancelled` 字段：
- `0`（ActivityAvailable）= 有效，计入统计和声望
- `1`（ActivityCancelled）= 已取消，不计入统计

**设计意图**：
1. 保留完整的操作轨迹，可做审计、反作弊分析
2. 重新投票时可复用记录（UPDATE 而非 INSERT），减少碎片
3. `cancelActivities` 对已取消记录的二次检查实现了**取消的幂等性**

---

## 五、计数更新机制（vote_count）

### 5.1 计数的两级存储

| 存储位置 | 字段 | 用途 | 更新时机 |
|----------|------|------|----------|
| activity 表 | COUNT(*) WHERE cancelled=0 | **事实源**（Source of Truth） | 每条 Activity 的 INSERT/UPDATE |
| question/answer/comment 表 | vote_count | **展示用的冗余缓存** | 每次投票后重算 |

### 5.2 GetAndSaveVoteResult 的执行

[GetAndSaveVoteResult](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L183-L189) 在 Vote/CancelVote 成功后调用：

```go
func (vr *VoteRepo) GetAndSaveVoteResult(ctx, objectID, objectType) (up, down int64, err) {
    up   = vr.countVoteUp(ctx, objectID, objectType)    // COUNT WHERE activity_type=UP_TYPE AND cancelled=0
    down = vr.countVoteDown(ctx, objectID, objectType)  // COUNT WHERE activity_type=DOWN_TYPE AND cancelled=0
    err  = vr.updateVotes(ctx, objectID, objectType, int(up-down))
    return
}
```

[updateVotes](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L440-L454) 根据 objectType 分别更新对应表的 `vote_count = up - down`。

**关键性质**：`vote_count` 字段**从不增量更新**，总是从 activity 表**重新 COUNT 聚合**。这意味着即使并发出现不一致，下一次投票自然会修正。

### 5.3 计数的统计口径

[countVote](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L427-L438) 查询条件：
```sql
WHERE object_id = ? 
  AND activity_type = ? 
  AND cancelled = 0   -- 只统计有效票
```

因此：
- 取消的票（cancelled=1）不会被统计
- 切换投票方向时，因为先 Cancel 反向再 Vote 正向，COUNT 结果天然正确

---

## 六、用户声望影响机制（Rank）

### 6.1 声望的来源：配置驱动

每条 Activity 的 `Rank` 值在 [getActivities](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go#L280-L297) 中从 `config` 表读取：

```go
cfg, _ := configService.GetConfigByKey(ctx, action)  // 如 "question.voted_up"
activity.Rank = cfg.GetIntValue()                    // 配置的值，例如 +10
```

系统初始化时的典型配置：
- `daily_rank_limit = 200`：每日声望增长上限
- `daily_rank_limit.exclude = ["answer.accepted"]`：采纳答案的声望不计入每日上限

### 6.2 声望变更的执行路径

**正向（投票）**：[changeUserRank](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L268-L286)
```
遍历 op.Activities:
  跳过 Rank==0 的
  调用 userRankRepo.ChangeUserRank(user_id, current_rank, +rank)
    → UPDATE user SET rank = rank + ? WHERE id = ?
```

**反向（取消）**：[rollbackUserRank](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L288-L306)
```
遍历 activities:
  跳过 Rank==0 的
  调用 userRankRepo.ChangeUserRank(user_id, current_rank, -rank)
```

### 6.3 三个声望保护机制

#### 保护 1：每日声望上限

[setActivityRankToZeroIfUserReachLimit](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L239-L266) 在事务内执行：

```go
if activity.Rank > 0 {
    reach, _ := userRankRepo.CheckReachLimit(user, maxDailyRank)
    if reach {
        activity.Rank = 0   // 超过上限，本次声望归零
    }
}
```

[CheckReachLimit](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/rank/user_rank_repo.go#L61-L81) 的统计口径：
```sql
SELECT SUM(`rank`) FROM activity 
WHERE user_id = ? AND cancelled = 0 
  AND updated_at BETWEEN 今日0点 AND 今日24点
```

若 `SUM(rank) >= daily_rank_limit`（默认 200），则该 Activity 的声望值被置 0，但**投票本身仍然有效**。

#### 保护 2：声望下界（最低为 1）

当声望增减会让用户声望低于 1 时：

```go
// 在 ChangeUserRank
if deltaRank < 0 && userCurrentScore + deltaRank < 1 {
    deltaRank = 1 - userCurrentScore  // 只扣到 1 为止
}
```

同样逻辑也存在于 [setActivityRankToZeroIfUserReachLimit](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L257-L263) 的负声望分支。

**效果**：用户声望永远 ≥ 1，新用户默认为 1（或初始注册声望）。

#### 保护 3：声望 Agent 开关

```go
if plugin.RankAgentEnabled() || deltaRank == 0 {
    return nil   // 插件接管声望系统时，完全跳过
}
```

### 6.4 声望与权限的关系

投票前在 Controller 层调用 [CheckVotePermission](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/rank/rank_service.go#L178-L223)：

```
1. 获取对象类型 (question/answer/comment)
2. 拼接权限名: permission.QuestionVoteUp → "question.vote_up"
3. 查询该用户的 role 是否有此 power（管理员/版主直接放行）
4. 若无 role 权限 → 比较 user.rank >= config["rank.question.vote_up"]
5. rank 不够 → 返回 Forbidden + 需要达到的声望值
```

**形成闭环**：声望 → 解锁投票权限 → 投票行为 → 改变声望。

---

## 七、并发一致性保障

### 7.1 数据库事务边界

[Vote](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L91-L114) 和 [CancelVote](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L151-L169) 的核心操作全部包裹在一个数据库事务中：

```
事务内操作顺序：
  1. acquireUserInfo(..., ForUpdate)     -- A. 锁定用户行
  2. setActivityRankToZeroIfUserReachLimit / -- B. 预处理声望值
     （取消时跳过此步）
  3. saveActivitiesAvailable             -- C. 插入/更新活动记录
     cancelActivities                    -- C'. 或取消活动记录
  4. changeUserRank                      -- D. 用户声望原子增减
     rollbackUserRank                    -- D'. 或原子回滚
```

**事务保证**：C 和 D 要么全成功要么全失败，不会出现「投了票但声望没加」或「加了声望但没投票记录」。

### 7.2 ForUpdate 行锁

[acquireUserInfo](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L224-L237) 使用 `SELECT ... FOR UPDATE`：

```go
session.In("id", userIDs).ForUpdate().Find(&us)
```

**解决的问题**：
- 防止并发写同一用户 `rank` 字段导致的**丢失更新**（Lost Update）
- 保证「读用户当前 rank → 计算 delta → UPDATE rank」三步之间用户 rank 不被其他事务修改

**死锁规避**：`acquireUserInfo` 一次把所有涉及用户（操作人 + 被操作人）用 `IN (...)` 查询**同时锁住**，避免逐行锁定导致的 AB-BA 死锁。

### 7.3 声望变更的原子性

[ChangeUserRank](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/rank/user_rank_repo.go#L96-L99) 使用 SQL 的 `INCR` 原子操作：

```go
session.ID(userID).Incr("`rank`", deltaRank).Update(&entity.User{})
```

生成 SQL：`UPDATE user SET rank = rank + ? WHERE id = ?`

**即使不用 ForUpdate 也不会有丢失更新**，因为单个 UPDATE 语句是原子的。但 ForUpdate 保证了「检查每日上限」和「执行 UPDATE」之间不会被插入其他声望增长。

### 7.4 计数的最终一致性

`vote_count` 更新在**事务外**执行，且采用**重新 COUNT 聚合**而非增量加减。

这是有意的设计权衡：
- **优点**：天然幂等、天然容忍并发重复、不会漂移；即使失败下次也会修正
- **缺点**：COUNT 在大表上性能略差；极端瞬时并发下，用户可能看到旧值后被修正（最终一致而非强一致）

### 7.5 并发场景的完整走查

**场景：两个用户同时给同一个答案的作者投 UP 票**

```
用户A事务:                                用户B事务:
┌────────────────────────────┐          ┌────────────────────────────┐
│ BEGIN                      │          │ BEGIN                      │
│ SELECT * FROM user         │──────┐   │ SELECT * FROM user         │─── 等待（同一作者行被锁）
│   WHERE id IN (A, Author)  │      │   │   WHERE id IN (B, Author)  │   │
│   FOR UPDATE               │      │   │   FOR UPDATE               │   │
├────────────────────────────┤      │   └────────────────────────────┘   │
│ 检查 Author 每日上限(通过)  │      │                                    │
│ INSERT 2 条 Activity       │      │                                    │
│ UPDATE user SET rank+=10   │      │                                    │
│   (A加0, Author加10)       │      │                                    │
│ COMMIT ◄───────────────────┘      │                                    │
│                                   │   （获得锁）                        │
│                                   ├────────────────────────────────────┤
│                                   │ 检查 Author 每日上限(10<200 通过)  │
│                                   │ INSERT 2 条 Activity               │
│                                   │ UPDATE user SET rank+=10           │
│                                   │   (B加0, Author加10 → 20)          │
│                                   │ COMMIT                             │
└───────────────────────────────────┴────────────────────────────────────┘

最终：作者声望 +20，每日已用 20，Activity 表 4 条记录
答案表 vote_count: 两个 up → COUNT 后为 2
完全正确。
```

### 7.6 声望每日上限的竞态

这是系统中一个**值得注意的语义**：

每日上限检查在事务内执行，但检查是 `SUM(rank)` 然后 `IF >= limit THEN Rank=0`。

若同一作者在**毫秒级**收到三笔并发 UP 投票（每笔 +10，limit=200，当前已用 195）：
- 事务都读到 SUM=195 → 都判断未超限 → 都执行 +10
- 最终作者声望 = 195 + 30 = 225，突破上限 25 点

**原因**：ForUpdate 只锁 user 表行，不锁 activity 表聚合。`CheckReachLimit` 读 activity 表不受 user 表行锁保护（MySQL InnoDB 的 MVCC）。

当前实现选择「接受轻微超限」，是正确性与性能的平衡。

---

## 第二部分 异步事件下游链路

---

## 八、事件总线与下游链路全景（SendEvent 之后）

### 8.1 四条完全独立的异步队列

系统中有**四条独立的内存队列**，各自承载不同的语义。它们共享同一个底层 `queue.Queue[T]` 泛型实现，但相互之间**没有任何关联**，消费者和发送者也不同：

| 队列名 | 消息类型 | 容量 | 发送者 | 唯一消费者 | 核心用途 |
|--------|----------|------|--------|------------|----------|
| **eventqueue** | `*schema.EventMsg` | 128 | question/answer/comment/report/meta/user 服务 + VoteService | BadgeEventService | **徽章（Badge）解锁判定** |
| **noticequeue(内部)** | `*schema.NotificationMsg` | 128 | NotificationCommon / BadgeAwardService / VoteRepo | NotificationCommon.AddNotification | **站内通知（Inbox + Achievement）** |
| **noticequeue(外部)** | `*schema.ExternalNotificationMsg` | 128 | user_notification_config 等 | ExternalNotificationService.Handler | **邮件/第三方插件通知** |
| **activityqueue** | `*schema.ActivityMsg` | 128 | question/answer/comment/revision/tag 服务 | ActivityCommon.HandleActivity | **非投票类活动时间线** |
| **vector_sync** | `*Task` | 128 | question/answer/comment/review 服务 | vector_sync.handle（内置） | **向量搜索索引同步** |

> **关键：投票不走 activityqueue**。投票类 Activity（`question.vote_up` 等）在 vote_repo.go 的事务中直接 INSERT，不经过异步队列。activityqueue 承载的是创建/编辑/关闭/删除等非投票行为。

### 8.2 队列底层实现（单 Handler + 单 Worker）

[queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/base/queue/queue.go) 的核心设计：

**架构**：
```
                    ┌──────────────────────┐
Send() → ──────→   │  chan T (buffer=128) │  ──────→ 单一 for-range goroutine
                    └──────────────────────┘               │
                                                           ▼
                                              q.handler(msg)  单一函数
```

**关键性质**：

1. **单 Handler 覆盖模型**：`RegisterHandler` 只是 `q.handler = handler` 的**赋值**，不是追加列表。后注册者会**覆盖**先注册者。因此每条队列**只能有一个消费者**。

2. **单 Worker 顺序消费**：`startWorker()` 只起一个 goroutine，`for msg := range q.queue` 逐条取出处理。消息严格按入队顺序 FIFO 消费。

3. **发送时阻塞**：`Send()` 使用 `select q.queue <- msg`，若缓冲满则阻塞等待（除非 ctx 取消）。背压机制 = 阻塞调用者线程。

4. **进程内队列**：纯内存 channel，**无持久化**。进程崩溃或重启后，未消费消息**永久丢失**。

5. **无自动重试**：`processMessage` 中 handler 返回 error 只会 `log.Errorf`，不会重入队列。

---

## 九、eventqueue 队列与徽章解锁

### 9.1 注册与订阅关系

**唯一订阅者**：[BadgeEventService](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/badge/badge_event_handler.go)
```go
// NewBadgeEventService 构造时:
eventQueueService.RegisterHandler(n.Handler)  // 唯一一个 handler
```

### 9.2 SendEvent 发出了什么

[VoteService.sendEvent](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go#L301-L323) 只在**非 Cancel 的正向投票**后调用（取消投票不发 EventMsg）：

```go
func (vs *VoteService) sendEvent(ctx, op, userID, authorUserID string) {
    event := schema.NewEvent(constant.EventQuestionVote, userID)  // 默认 EventQuestionVote
    if op == "answer" {
        event.EventType = constant.EventAnswerVote
    } else if op == "comment" {
        event.EventType = constant.EventCommentVote
    }
    event.AddExtra("vote_up_amount", fmt.Sprintf("%d", upVotes))
    vs.eventQueueService.Send(ctx, event)   // 只发到 eventqueue！
}
```

**投票只发 eventqueue**，不直接发 noticequeue。通知的产生走另外两条路：
- **站内成就通知**：在 vote_repo.go 的事务外通过 `achievementCommon.SendNotification`（声望变更通知）
- **徽章解锁通知**：徽章解锁成功后 BadgeAwardService 自己发 noticequeue

### 9.3 Event → 徽章规则映射

`EventRuleMapping` 定义：

```go
constant.EventQuestionVote:   {b.FirstVotedPost, b.ReachQuestionVote},
constant.EventAnswerVote:     {b.FirstVotedPost, b.ReachAnswerVote},
constant.EventCommentVote:    {b.FirstVotedPost},
```

投票事件触发三类徽章判定：

| 规则处理器 | 判定逻辑 | 授予对象 | AwardKey（防复用键） |
|-----------|----------|----------|---------------------|
| **FirstVotedPost** | 读 DB `WHERE handler="FirstVotedPost"` 的所有徽章，无条件返回（Award 层再去重） | 投票人(event.UserID) | objectID（问题/回答/评论ID） |
| **ReachQuestionVote** | 读 `vote_up_amount` Extra 值，若 ≥ `badge.param.amount` 则命中 | **作者**(event.QuestionUserID) | 问题ID |
| **ReachAnswerVote** | 同上，针对回答 | **作者**(event.AnswerUserID) | 回答ID |

### 9.4 Handler 执行流程

[BadgeEventService.Handler](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/badge/badge_event_handler.go#L64-L77)：
```
步骤1: eventRuleRepo.HandleEventWithRule(msg)
       → 遍历该 EventType 的所有规则处理器
       → 每个处理器返回 []*BadgeAward{UserID, BadgeID, AwardKey}
       → 聚合所有待授予

步骤2: 遍历 awards:
       badgeAwardService.Award(award.BadgeID, award.UserID, award.AwardKey)
         ├─ badgeRepo.GetByID → 校验徽章存在且 Active
         ├─ badgeAwardRepo.CheckIsAward(badgeID, userID, awardKey, badge.Single)
         │    Single=1(一次性): 同用户同徽章只需 awardKey 任意
         │    Single=0(多次性): 同用户同徽章需要 awardKey 唯一
         │    → 已授予: return nil（幂等跳过）
         ├─ INSERT badge_award 表
         └─ noticequeue.Send(NotificationMsg{
              Type=Achievement,
              ObjectType=BadgeAwardObjectType,
              NotificationAction=NotificationEarnedBadge
            })  // 发成就通知

步骤3: 任意处理器 error 只 log.Errorf，不中断其他处理器
```

### 9.5 消费失败语义

- **handler 返回 error**：只 `log.Errorf("[%s] handler error: %v")`，消息**被丢弃**不重放
- **Award() 返回 error**：每个 award 独立 `log.Debugf`，不影响其他徽章授予
- **无死信队列**：没有 DLQ，失败后只能等待下一次同类事件触发再尝试

---

## 十、noticequeue 队列与站内通知

### 10.1 两条 noticequeue 的区分

注意：`noticequeue` 包下有**两种不同的 Service 类型**，对应两条独立的 channel 队列：

```go
// 队列A: 站内通知(内部) —— buffer 128
type Service queue.Service[*schema.NotificationMsg]
func NewService() Service { return queue.New[*schema.NotificationMsg]("notification", 128) }

// 队列B: 外部通知(邮件/插件) —— buffer 128
type ExternalService queue.Service[*schema.ExternalNotificationMsg]
func NewExternalService() ExternalService { return queue.New[*schema.ExternalNotificationMsg]("external_notification", 128) }
```

| 队列 | 注册者 | Handler |
|------|--------|---------|
| A(内部) | NotificationCommon | `AddNotification` |
| B(外部) | ExternalNotificationService | `.Handler`（邮件分发） |

因为类型参数不同，两者的 RegisterHandler 互不干扰，不存在覆盖问题。

### 10.2 AddNotification 内部执行流程

输入 `NotificationMsg{Type, TriggerUserID, ReceiverUserID, ObjectID, ObjectType, Title, NotificationAction, ...}`：

```
步骤1: 快速拦截
  ├─ Type==Achievement && plugin.RankAgentEnabled() → return nil
  └─ 获取 ObjectInfo（对象标题/问题ID/回答ID/评论ID的映射表）

步骤2: 分类型处理
  ┌─ Type==Achievement(声望类)
  │    ① GetByUserIdObjectIdTypeId(user, object, type=2) 去重
  │    ② GetUserIDObjectIDActivitySum(user, object) 重算 rank 累计
  │    ③ 已存在: UPDATE notification.content.rank（同对象只留一条成就通知，更新累计值）
  │    └─ 不存在: 继续步骤3
  │
  └─ Type==Inbox(消息类)
       └─ 直接继续步骤3（不去重，每条都是独立消息）

步骤3: 构造 Notification 实体并 INSERT
  - UserID=ReceiverUserID, MsgType=NotificationMsgTypeMapping[Action]
  - Content=JSON{UserInfo, ObjectInfo, Rank, NotificationAction, ...}
  - IsRead=NotRead, Status=Normal

步骤4: 红点系统
  - Redis INCR key=`reddot:inbox:{userID}` 或 `reddot:achievement:{userID}`
  - TTL=RedDotCacheTime（默认 180 天）
  - 若 BadgeAward 类型：额外写 AddBadgeAwardAlertCache（Redis JSON 列表）

步骤5: 异步扩展（不阻塞 handler）
  - go SendNotificationToAllFollower():
      仅限 Event=UpdateQuestion/AnswerQuestion/UpdateAnswer/AcceptAnswer
      查询 follow 了该问题的所有用户 → 给每人 copy 一份 msg 重入 noticequeue
      (NoNeedPushAllFollow=true 防止递归)
  - syncNotificationToPlugin():
      Type==Inbox 时，调用 plugin.CallNotification → 所有已注册插件的 fn.Notify()
```

### 10.3 成就通知 vs 站内收件箱的区别

| 维度 | Inbox (Type=1) | Achievement (Type=2) |
|------|----------------|----------------------|
| 触发场景 | 有人回答/评论/编辑/采纳你的内容 | 你获得声望 / 你解锁徽章 |
| 去重策略 | 不去重，每条独立 INSERT | `(user, object, type=2)` 唯一，存在就 UPDATE rank |
| 发送者 | 所有内容变更场景 | vote_repo 声望通知 + BadgeAwardService 徽章通知 |

---

## 十一、搜索与向量索引更新链路（重要：不走 eventqueue！）

### 11.1 文本搜索索引（keyword search）

**触发方式：repo 层直接同步调用 plugin，非异步队列**

| 操作 | 触发位置 | 调用 |
|------|----------|------|
| 问题 CUD/状态变更/接受/置顶 | [question_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/question/question_repo.go) | `qr.UpdateSearch(ctx, questionID)` |
| 回答 CUD/状态变更/接受 | [answer_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/answer/answer_repo.go) | `ar.updateSearch(ctx, answerID)` |
| 问题通过审核 | [question_service.go:L415](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/question_service.go#L415) | `qs.questionRepo.UpdateSearch` |

**投票不触发搜索索引重建！** `VoteService` 中没有任何 `UpdateSearch` 调用。问题/回答的 `vote_count`（搜索文档的 `Score` 字段）在投票后不会立即同步到搜索索引。

**UpdateSearch 的内部实现**：
```
① plugin.CallSearch → 检查是否注册了 Search 插件，未注册直接 return
② DB 重新读取问题/回答全文 + tags + 最新 VoteCount / AnswerCount / ViewCount ...
③ s.UpdateContent(ctx, &plugin.SearchContent{Score=question.VoteCount, ...})
   → 由具体插件实现（meilisearch/typesense/algolia 等）的 upsert 逻辑
```

### 11.2 向量搜索索引（embedding / semantic search）

**触发方式：独立 vector_sync 异步队列**

[vector_sync.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/vector_sync/vector_sync.go) 队列：
```go
q := queue.New[*Task]("vector_sync", 128)
q.RegisterHandler(func(ctx, msg) error {
    // 3次重试（vector_sync 是唯一实现重试逻辑的队列！）
    for attempt := 1; attempt <= 3; attempt++ {
        handleOnce(upsert/delete, question/answer, objectID)
            → BuildQuestion/AnswerContentByID → 重新读DB生成向量
            → vectorSearch.UpdateContent / DeleteContent
    }
    return lastErr  // 3次都失败才记录 error log
})
```

**同样，投票不触发向量索引同步**。只有内容 CUD/审核流转才 Send 到 vector_sync。

### 11.3 投票导致的 Score 滞后

投票后 DB 中 `question.vote_count` 已是新值，但：
- **文本搜索 Score** 落后 → 下次创建/编辑/审核触发时才刷新
- **向量搜索 Score** 同样落后 → 下次内容变更才重建

这是有意的设计权衡（性能优先），若需强一致需要在 VoteService 末尾手动追加 `UpdateSearch` 和 `vectorSyncService.Send`。

---

## 十二、消费顺序与失败回退机制

### 12.1 投票成功后的实际执行顺序

以一次「对回答投 UP，之前是 Down 票」为例，从 VoteService 层看：

```
同步阶段（HTTP 请求线程内，强顺序保证）:
 T1: voteRepo.CancelVote(down)
       ├─ 事务: BEGIN → ForUpdate 锁用户 → soft-delete Activities
       │            → 回滚 User.rank → COMMIT
       └─（事务外）SendNotification → 入队 noticequeue

 T2: voteRepo.Vote(up)
       ├─ 事务: BEGIN → ForUpdate 锁用户 → saveActivities
       │            → 加 User.rank → COMMIT
       └─（事务外）SendNotification → 入队 noticequeue

 T3: voteRepo.GetAndSaveVoteResult → UPDATE answer SET vote_count=?
 T4: vs.sendEvent → 入队 eventqueue

─────────────────────────────────────────────────────

异步阶段（各队列 goroutine 内，**顺序不可控**）:
  ▲ goroutine A(eventqueue):  处理 EventAnswerVote
  │                            → FirstVotedPost / ReachAnswerVote
  │                            → 若命中徽章 → Award → 入队 noticequeue
  │
  ▲ goroutine B(noticequeue): 处理 T1 的通知（撤销 Down 的成就通知）
  │                            → INSERT / UPDATE notification 表 + 红点
  ▲ goroutine B(noticequeue): 处理 T2 的通知（投 UP 的声望通知）
  ▲ goroutine B(noticequeue): 若已有徽章解锁通知 → 处理徽章成就通知

  注意：因为 eventqueue 和 noticequeue 是**两个独立 goroutine**，
       徽章入队 noticequeue 的时机相对于 T1/T2 的通知是**不可预测**的，
       可能插入到任意位置。
```

### 12.2 失败回退机制总结

| 层级 | 失败场景 | 回退/重试策略 |
|------|----------|---------------|
| **vote_repo 事务** | 任一步骤 error | 事务 ROLLBACK，活动记录 + 声望都不生效，HTTP 返回错误 |
| **GetAndSaveVoteResult** | DB 更新 vote_count 失败 | VoteService 直接返回 error 给前端（前面事务已提交，**不会回滚活动和声望**。vote_count 可被下次投票修复） |
| **SendEvent / SendNotification** | channel 满阻塞 | 阻塞 HTTP 线程（背压）；ctx 取消则丢弃消息 |
| **eventqueue handler** | 徽章判定/授予 error | 仅 log，消息丢弃，不重试。下次同类型事件可再次触发同样规则 |
| **noticequeue AddNotification** | INSERT notification / Redis 红点失败 | handler 返回 error → log，消息丢弃。红点可能少加 1（用户实际会看到有新通知但红点为0，下次进入列表时修正） |
| **vector_sync handler** | 向量更新失败 | **重试 3 次**，仍失败则 log，消息丢弃 |
| **SendNotificationToAllFollower** | goroutine 内的 DB/入队失败 | log.Error，该部分粉丝收不到通知，无补偿 |

### 12.3 关键幂等性保证

| 操作 | 幂等机制 |
|------|----------|
| 活动记录去重 | `(object_id, user_id, trigger_user_id, activity_type)` + Cancelled 状态机 |
| 声望增减 | Activity 存了 Rank 值，去重时复用；ChangeUserRank 仅对 Rank≠0 的生效 |
| 徽章授予 | `CheckIsAward(badgeID, userID, awardKey, Single)`：一次性徽章 user 唯一，多次性 awardKey 唯一 |
| 成就通知去重 | `GetByUserIdObjectIdTypeId(user, object, type=2)`：同用户同对象只有一条 Achievement 通知，已有则 UPDATE rank 累加值 |
| 投票计数 | 每次全量 COUNT(*)，天然幂等 |

---

## 第三部分 防御与运营

---

## 十三、反作弊限流三层校验

系统有**三层防刷/反作弊机制**，独立运行，共同作用：

### 13.1 第一层：内容提交去重（重复请求拦截）

[RateLimitMiddleware.DuplicateRequestRejection](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/base/middleware/rate_limit.go#L46-L65)：

**原理**：基于 Redis 的**幂等键**去重。

```
key = MD5(userID + requestPath + requestBody)
```

执行流程：
```
步骤1: 计算唯一 key（同用户+同路径+同请求内容 → 同 key）
步骤2: 检查 Redis 中 key 是否存在
       → 存在：返回 DuplicateRequestError，拦截请求
       → 不存在：SET key=timestamp, TTL=3秒（RateLimitCacheTime）
步骤3: 请求处理成功后 → DuplicateRequestClear(key) 删除 key
       请求处理失败 → 不清除，3秒内无法重试完全相同的请求
```

**作用场景**：
- 防止用户因网络卡顿重复点击提交按钮造成重复发帖
- 防止攻击者重放相同请求
- 适用于所有 POST/PUT 修改操作（提问、回答、评论、投票等）

**配置**：
- TTL: `RateLimitCacheTime = 3 * time.Second`
- 存储：Redis Cache 层

### 13.2 第二层：操作级验证码校验

在 `VoteController` 中，非管理员投票需通过验证码：

[vote_controller.go:L88-L94](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/vote_controller.go#L88-L94)：

```go
// 非管理员每次投票都要验证码
if !userInfo.IsAdmin {
    actionService.ActionRecordVerifyCaptcha(ctx, constant.CaptchaVote, 
        userInfo.UserID, req.CaptchaID, req.CaptchaCode)
}
```

验证码由 `ActionCommon` 管理，每次投票后需要重新获取新的验证码，防止脚本高频刷票。

### 13.3 第三层：声望门槛

投票前在 Controller 层调用 [CheckVotePermission](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/rank/rank_service.go#L178-L223)：

```go
// 比较 user.rank >= config["rank.question.vote_up"]
can, needRank, _ := rankService.CheckVotePermission(ctx, objectID, userID, "vote_up")
```

低声望新用户无法立即投票，需先通过提问/回答积累声望，提高刷票成本。

### 13.4 三层防御的叠加效果

| 层级 | 针对的攻击 | 实现方式 | 粒度 |
|------|-----------|----------|------|
| 1. 去重中间件 | 重放/重复提交 | Redis 幂等键（3秒 TTL） | 单请求 |
| 2. 验证码 | 脚本/高频刷票 | 图片验证码 + Action 计数 | 单次操作 |
| 3. 声望门槛 | 小号/批量刷票 | `rank >= 阈值` | 账号级 |

**投票的完整校验链**：
```
HTTP /vote/up
  │
  ▼
RateLimitMiddleware.DuplicateRequestRejection  → 拦截 3 秒内完全相同的请求
  │
  ▼
vote_controller.VoteUp
  ├─ rankService.CheckVotePermission          → 检查声望门槛
  ├─ actionService.ActionRecordVerifyCaptcha  → 验证验证码
  ├─ actionService.ActionRecordAdd            → 记录本次操作
  └─ voteService.VoteUp
       ├─ voteRepo.Vote(事务内去重)           → 数据库层面 (object_id, user_id, trigger_user_id, activity_type) 唯一键去重
       ├─ voteRepo.GetAndSaveVoteResult
       └─ sendEvent
```

---

## 十四、管理员撤销违规投票的代码路径

### 14.1 入口：管理员后台处理举报

管理员在后台处理举报时，通过 `ReportHandle.UpdateReportedObject` 入口对违规内容进行处置：

[ReportHandle.UpdateReportedObject](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/report_handle/report_handle.go#L53-L68)

对不同对象类型，管理员可执行的操作不同：

| 对象 | 操作类型 | 调用 |
|------|----------|------|
| Question | UnlistPost / DeletePost / ClosePost / EditPost | `OperationQuestion` / `RemoveQuestion` / `CloseQuestion` / `UpdateQuestion` |
| Answer | DeletePost / EditPost | `RemoveAnswer` / `UpdateAnswer` |
| Comment | DeletePost / EditPost | `RemoveComment` / `UpdateComment` |

违规投票的典型处置路径是 **DELETE（删除内容）**：
- 问题：[report_handle.go:L76-L78](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/report_handle/report_handle.go#L76-L78) → `rh.questionService.RemoveQuestion(IsAdmin=true)`
- 回答：[report_handle.go:L102-L104](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/report_handle/report_handle.go#L102-L104) → `rh.answerService.RemoveAnswer`

### 14.2 RemoveQuestion 的完整流程

[QuestionService.RemoveQuestion](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/question_service.go#L555-L664)：

```
步骤1: 权限校验
  - 管理员：无限制
  - 普通用户：只能删自己的问题，且无已采纳回答、无多个回答、回答 VoteCount≤0

步骤2: UPDATE question SET status=DELETED

步骤3: 更新用户统计
  - 重新 COUNT 用户可用问题数 → user.question_count

步骤4: tag 关联清理
  - 移除问题-tag 关联 → 更新 tag 统计

步骤5: 清理问题关联链接

步骤6: 异步事件（非事务内）
  - activityqueue.Send ActQuestionDeleted
  - eventqueue.Send EventQuestionDelete
  - vector_sync.Delete（移除向量索引）
  - plugin.Search.DeleteContent（移除搜索索引）
```

### 14.3 关键发现：声望回滚**已被注释**！

**这是最核心的结论**：在 `RemoveQuestion` 和 `RemoveAnswer` 中，声望补偿（rollback）的代码**已被注释掉**，注释编号是 #2372：

[question_service.go:L639-L644](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/question_service.go#L639-L644)：
```go
// #2372 In order to simplify the process and complexity, as well as to consider if it is in-house,
// facing the problem of recovery.
// err = qs.answerActivityService.DeleteQuestion(ctx, questionInfo.ID, questionInfo.CreatedAt, questionInfo.VoteCount)
// if err != nil {
// 	 log.Errorf("user DeleteQuestion rank rollback error %s", err.Error())
// }
```

[answer_service.go:L184-L189](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/answer_service.go#L184-L189)：
```go
// #2372 In order to simplify the process and complexity, as well as to consider if it is in-house,
// facing the problem of recovery.
// err = as.answerActivityService.DeleteAnswer(ctx, answerInfo.ID, answerInfo.CreatedAt, answerInfo.VoteCount)
// if err != nil {
// 	log.Errorf("delete answer activity change failed: %s", err.Error())
// }
```

### 14.4 注释后的实际影响

当管理员删除违规内容时：

| 项目 | 处理方式 |
|------|----------|
| **问题/回答本身** | `status=Deleted` 软删除，不再展示 |
| **活动记录（Activity 表）** | **保留**，既不 Cancelled=1 也不 DELETE |
| **投票计数（vote_count）** | 删除时**不更新**，但因为状态是 Deleted，下次 GetInfo 已读不到 |
| **用户声望（user.rank）** | **不回滚！** 之前所有投票带来的声望增减全部保留 |
| **每日上限统计** | 因为 Activity 保留，仍会计入每日上限 |
| **搜索/向量索引** | 已删除（vector_sync + plugin.Search） |
| **用户贡献度展示** | 因为 Activity 保留，用户主页「声望历史」和「投票历史」仍能看到 |

### 14.5 设计意图（从注释推测）

注释说明：
> "为了简化流程和复杂度，同时考虑如果是内部环境面临的恢复问题。"

保留 Activity 不回滚声望的理由可能是：
1. 回滚所有相关投票的声望需要找出所有给该内容投票的用户，遍历并 UPDATE，复杂度高
2. 删除内容后可能需要恢复，保留 Activity 便于恢复时直接恢复状态
3. 违规内容删除后，违规投票者的不当得利（或损失）暂时保留，由管理员后续人工处理
4. 内部环境下这种简化是可接受的

---

## 第四部分 反向查询与展示

---

## 十五、Activity 表到用户主页声望和贡献度的映射

Activity 表是**一切用户贡献度数据的事实源**。用户主页有三个核心展示组件，全部从 Activity 表聚合查询：

| 展示组件 | 接口 | 查询 SQL 模式 |
|----------|------|--------------|
| 声望历史曲线 | `/personal/rank/page` | 按 `created_at` 倒序，`has_rank=1 AND cancelled=0 AND rank>0` |
| 投票历史列表 | `/personal/vote/page` | 按 `updated_at` 倒序，`activity_type IN (vote_up/down 四类)` |
| 用户排行榜 | `/user/ranking` | 最近 7 天 `SUM(rank)` 排名 / `COUNT(vote)` 排名 |

### 15.1 声望历史（贡献度曲线）

**接口**：`GET /answer/api/v1/personal/rank/page`

**调用链**：
[RankController.GetRankPersonalWithPage](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/rank_controller.go#L51-L61)
→ [RankService.GetRankPersonalPage](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/rank/rank_service.go#L262-L289)
→ [UserRankRepo.UserRankPage](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/rank/user_rank_repo.go#L201-L215)

**查询 SQL**：
```sql
SELECT * FROM activity
WHERE user_id = ?
  AND has_rank = 1          -- 只有带声望的活动（排除纯操作记录）
  AND cancelled = 0         -- 排除已取消的
  AND `rank` > 0            -- 只显示正声望（扣声望不展示在历史中）
ORDER BY created_at DESC
LIMIT ?, ?
```

**字段映射**（[RankService.decorateRankPersonalPageResp](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/rank/rank_service.go#L291-L320)）：
| Activity 字段 | 展示字段 | 处理 |
|--------------|----------|------|
| `created_at` | `CreatedAt` | Unix 时间戳，前端绘制曲线 X 轴 |
| `rank` | `Reputation` | 本次获得的声望值，Y 轴 |
| `activity_type` | `RankType` | 通过 config 表映射为字符串（如 "question.voted_up" → 翻译为"回答被赞"） |
| `object_id` | `ObjectID/Title` | 关联对象信息（问题标题/链接） |

### 15.2 投票历史列表

**接口**：`GET /answer/api/v1/personal/vote/page`

**调用链**：
[VoteController.UserVotes](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/vote_controller.go#L176-L186)
→ [VoteService.ListUserVotes](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go#L188-L229)
→ [VoteRepo.ListUserVotes](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L191-L208)

**查询 SQL**：
```sql
SELECT * FROM activity
WHERE user_id = ?
  AND cancelled = 0
  AND activity_type IN (QuestionVoteUpID, QuestionVoteDownID,
                        AnswerVoteUpID,   AnswerVoteDownID)
ORDER BY updated_at DESC
LIMIT ?, ?
```

**注意**：这里的 `user_id` 是**操作人**（投票者），不是被操作人。只返回 `activity_type` 为四类投票的记录。

### 15.3 用户排行榜

**接口**：`GET /answer/api/v1/user/ranking`

**调用链**：
[UserController.UserRanking](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/user_controller.go)
→ [UserService.UserRanking](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/user_service.go#L790-L823)

**包含三组统计**，时间范围均为**最近 7 天**（`endTime=now, startTime=now-7d`）：

#### 第一组：声望增长排名
[ActivityRepo.GetUsersWhoHasGainedTheMostReputation](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity_common/activity_repo.go#L147-L163)：
```sql
SELECT user_id, SUM(`rank`) AS rank_amount
FROM activity
WHERE has_rank = 1
  AND cancelled = 0
  AND created_at BETWEEN startTime AND endTime
GROUP BY user_id
ORDER BY rank_amount DESC
LIMIT 20
```

#### 第二组：投票活跃度排名
[ActivityRepo.GetUsersWhoHasVoteMost](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity_common/activity_repo.go#L165-L191)：
```sql
SELECT user_id, COUNT(*) AS vote_count
FROM activity
WHERE cancelled = 0
  AND activity_type IN (QuestionVoteUp, QuestionVoteDown, ...)
  AND created_at BETWEEN startTime AND endTime
GROUP BY user_id
ORDER BY vote_count DESC
LIMIT 20
```

#### 第三组：工作人员列表
独立查询，管理员和版主直接显示在排行榜末尾。

### 15.4 完整的 Activity → 展示映射图

```
Activity 表 (事实源)
│
├─ 展示1: /personal/rank/page（声望历史）
│   WHERE: has_rank=1 AND cancelled=0 AND rank>0
│   ORDER: created_at DESC
│   → 每条: {时间, 声望值, 行为类型(翻译后), 关联对象标题}
│
├─ 展示2: /personal/vote/page（投票历史）
│   WHERE: activity_type IN (4种投票类型) AND cancelled=0
│   ORDER: updated_at DESC
│   → 每条: {投票时间, 投票方向(up/down), 关联对象标题}
│
├─ 展示3: /user/ranking（排行榜）
│   ├─ 声望榜: SUM(rank) GROUP BY user_id LIMIT 20
│   ├─ 投票榜: COUNT(*) GROUP BY user_id LIMIT 20
│   └─ 工作人员: 管理员+版主
│
└─ 数据来源:
    ├─ user.rank 字段（当前声望快照，由 activity 记录累加而来）
    ├─ activity.rank 字段（单次声望变更量）
    ├─ activity.has_rank 字段（标记是否是声望变更类活动）
    └─ activity.activity_type → config 表 → 翻译为用户可见的行为描述
```

---

## 十六、声望的三层存储模型

| 层级 | 位置 | 用途 | 更新时机 | 一致性 |
|------|------|------|----------|--------|
| L1 事实源 | `activity` 表的 `rank` 字段 | 审计、历史查询、统计 | 每次投票/活动时 INSERT | 绝对准确 |
| L2 快照 | `user` 表的 `rank` 字段 | 权限判断、展示 | 每次投票时原子 INCR | 强一致（事务内） |
| L3 展示 | 用户主页排行榜/历史 | 前端展示 | 页面查询时实时聚合 | 最终一致 |

```
投票事务:
  INSERT activity(rank=10)        → L1 写入
  UPDATE user SET rank=rank+10    → L2 原子更新
  ──────────────────────────────
页面查询 /personal/rank/page:
  SELECT * FROM activity WHERE user_id=? ORDER BY created_at DESC  → L1 读
页面查询 /user/ranking:
  SELECT SUM(rank) FROM activity GROUP BY user_id                  → L1 聚合
```

### 16.1 每日上限的统计口径

[CheckReachLimit](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/rank/user_rank_repo.go#L61-L81) 统计：
```sql
SELECT SUM(`rank`) FROM activity
WHERE user_id = ?
  AND cancelled = 0
  AND updated_at BETWEEN today_start AND today_end
```

使用 `updated_at` 而非 `created_at`，意味着：
- 取消投票（UPDATE `cancelled=1` + `updated_at=now`）会更新这条记录的时间戳到取消当天
- 但因为 `cancelled=1`，不会被计入统计
- 恢复投票（UPDATE `cancelled=0`）也会更新 `updated_at`，可能把这条声望的"归属日"改为恢复日

---

## 十七、完整链路时序图（一次 UP 投票切换）

```
  Client                VoteService           VoteRepo            eventqueue         noticequeue
    │  POST /vote/up       │                     │                    │                   │
    │─────────────────────>│                     │                    │                   │
    │                      │  CancelVote(down)   │                    │                   │
    │                      │────────────────────>│  事务(回滚声望)     │                   │
    │                      │                     │──────┐             │                   │
    │                      │                     │ COMMIT│             │                   │
    │                      │                     │<─────┘             │                   │
    │                      │                     │  SendNotif(ach)    │                   │
    │                      │                     │───────────────────────────────────────>│
    │                      │  Vote(up)           │                    │                   │
    │                      │────────────────────>│  事务(写活动+加声望)│                   │
    │                      │                     │──────┐             │                   │
    │                      │                     │ COMMIT│             │                   │
    │                      │                     │<─────┘             │                   │
    │                      │                     │  SendNotif(ach)    │                   │
    │                      │                     │───────────────────────────────────────>│
    │                      │                     │                    │                   │
    │                      │ GetAndSaveVoteResult│                    │                   │
    │                      │────────────────────>│  COUNT + UPDATE    │                   │
    │                      │<────────────────────│                    │                   │
    │                      │  SendEvent          │                    │                   │
    │                      │────────────────────────────────────────>│                   │
    │<─────────────────────│                     │                    │                   │
    │  200 OK {vote_count} │                     │                    │                   │
    │                      │                     │                    │                   │
    │ ══════════════════════════════════════════════════════════════════════════════════════
    │ 以下全是异步消费 goroutine，客户端不可见                                           │
    │                      │                     │  BadgeHandler       │  AddNotification  │
    │                      │                     │  徽章规则匹配        │  成就通知入库     │
    │                      │                     │  → Award徽章        │  Redis红点+1      │
    │                      │                     │  → 发成就通知─────────────────────────>│
    │                      │                     │                    │  徽章成就入库     │
```

---

## 十八、完整代码索引

| 关注点 | 文件 | 关键行 |
|--------|------|--------|
| **核心流程** | | |
| HTTP 入口(投票) | [vote_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/vote_controller.go) | L68-L110, L122-L163 |
| 投票编排 | [vote_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go) | L89-L137, L140-L186 |
| Activity 生成 | [vote_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go) | L243-L299 |
| 投票持久化 | [vote_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go) | L72-L129, L210-L358 |
| 取消投票 | [vote_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go) | L131-L181, L363-L389 |
| 声望增减 | [user_rank_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/rank/user_rank_repo.go) | L84-L101 |
| 每日上限 | [vote_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go) | L239-L266 |
| 权限检查 | [rank_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/rank/rank_service.go) | L178-L223 |
| 数据实体 | [activity_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/entity/activity_entity.go) | L24-L63 |
| 活动类型配置 | [activity_type.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/activity_type/activity_type.go) | L22-L76 |
| 投票状态常量 | [acticity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/base/constant/acticity.go) | L24-L40 |
| **事件下游** | | |
| 队列底层实现 | [queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/base/queue/queue.go) | L29-L130 |
| eventqueue 定义 | [event_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/eventqueue/event_queue.go) | L27-L31 |
| 徽章事件处理 | [badge_event_handler.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/badge/badge_event_handler.go) | L32-L77 |
| Event→徽章规则映射 | [badge_event_rule.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/badge/badge_event_rule.go) | L37-L85, L196-L236 |
| 徽章授予(含去重) | [badge_award_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/badge/badge_award_service.go) | L148-L191 |
| 投票 SendEvent | [vote_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go) | L301-L323 |
| 站内通知队列定义 | [notice_queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/noticequeue/notice_queue.go) | L27-L37 |
| AddNotification 逻辑 | [notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/notification_common/notification.go) | L109-L220, L222-L244 |
| 外部通知(邮件) | [external_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/notification/external_notification.go) | L39-L97 |
| 文本搜索同步(问题) | [question_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/question/question_repo.go) | L584-L634 |
| 文本搜索同步(回答) | [answer_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/answer/answer_repo.go) | L463-L529 |
| 向量搜索同步(含重试) | [vector_sync.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/vector_sync/vector_sync.go) | L51-L115 |
| Activity 队列消费 | [activity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/activity_common/activity.go) | L50-L91 |
| EventMsg 结构 | [event_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/schema/event_schema.go) | L27-L114 |
| NotificationMsg 结构 | [notification_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/schema/notification_schema.go) | L75-L95 |
| 事件类型常量 | [event.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/base/constant/event.go) | L24-L56 |
| 通知动作常量 | [notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/base/constant/notification.go) | L24-L46 |
| **防御与运营** | | |
| 举报处理入口 | [report_handle.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/report_handle/report_handle.go) | L53-L138 |
| 删除问题 | [question_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/question_service.go) | L555-L664 |
| 删除回答 | [answer_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/answer_service.go) | L119-L202 |
| 声望回滚注释 | [question_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/question_service.go) | L639-L644 |
| 反作弊去重中间件 | [rate_limit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/base/middleware/rate_limit.go) | L46-L73 |
| 去重 Redis 实现 | [limit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/limit/limit.go) | L45-L65 |
| 声望阈值校验 | [rank_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/rank/rank_service.go) | L178-L223 |
| 投票验证码 | [vote_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/vote_controller.go) | L88-L94 |
| **展示与查询** | | |
| 声望历史查询 | [user_rank_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/rank/user_rank_repo.go) | L201-L215 |
| 声望历史装饰 | [rank_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/rank/rank_service.go) | L291-L320 |
| 投票历史查询 | [vote_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go) | L191-L208 |
| 投票历史装饰 | [vote_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go) | L188-L229 |
| 声望增长榜 | [activity_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity_common/activity_repo.go) | L147-L163 |
| 投票活跃度榜 | [activity_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity_common/activity_repo.go) | L165-L191 |
| 排行榜编排 | [user_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/user_service.go) | L790-L823 |
| **运行时维度** | | |
| 配置缓存(repo) | [config_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/config/config_repo.go) | L48-L143 |
| 配置服务层 | [config_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/config/config_service.go) | L50-L118 |
| 外部通知处理 | [external_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/notification/external_notification.go) | L39-L110 |
| 新回答邮件通知 | [new_answer_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/notification/new_answer_notification.go) | L32-L82 |
| 邀请回答邮件通知 | [invite_answer_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/notification/invite_answer_notification.go) | L32-L82 |
| 新评论邮件通知 | [new_comment_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/notification/new_comment_notification.go) | L32-L82 |
| 新问题邮件通知 | [new_question_notification.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/notification/new_question_notification.go) | L44-L115 |
| 邮件发送+模板 | [email_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/export/email_service.go) | L123-L157, L168-L360 |
| 邮件模板数据 | [email_template.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/schema/email_template.go) | L28-L145 |
| 用户通知偏好 | [user_notification_config_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/user_notification_config/user_notification_config_service.go) | L42-L115 |
| 用户通知偏好实体 | [user_notification_config_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/entity/user_notification_config_entity.go) | L25-L33 |
| 队列单测 | [queue_test.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/base/queue/queue_test.go) | L36-L253 |

---

## 第五部分 运行时维度

---

## 十九、声望奖励配置修改后的生效机制与缓存刷新

### 19.1 配置读取的两级缓存

声望奖励配置（如 `rank.question.vote_up`、`rank.question.voted_up`、`daily_rank_limit`）存储在 `config` 表中，通过 [ConfigRepo](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/config/config_repo.go) 读取，采用**先查 Redis 缓存、未命中再查 DB 并回填**的模式。

#### GetConfigByKey 的读取路径（[config_repo.go:L75-L100](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/config/config_repo.go#L75-L100)）

```
步骤1: 尝试从 Redis 读取
  cacheKey = "config:kc:" + key
  → 命中: 反序列化 JSON 返回
  → 未命中: 继续步骤2

步骤2: 查询数据库
  SELECT * FROM config WHERE key = ?
  → 不存在: 返回 error
  → 存在: 继续步骤3

步骤3: 回填 Redis 缓存
  SET cacheKey = config.JsonString(), TTL = ConfigCacheTime
```

**缓存 TTL**：[ConfigCacheTime = 1 * time.Hour](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/base/constant/cache_key.go#L42)

**缓存 Key 格式**：
| 查询方式 | Redis Key | 格式 |
|----------|-----------|------|
| 按 Key 查 | `config:kc:{key}` | 如 `config:kc:rank.question.vote_up` |
| 按 ID 查 | `config:ic:{id}` | 如 `config:ic:42` |

### 19.2 配置修改的缓存刷新

管理员在后台修改配置时，走 [UpdateConfig](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/config/config_repo.go#L114-L143)：

```go
func (cr configRepo) UpdateConfig(ctx, key, value string) error {
    // 1. 查旧记录获取 ID
    oldConfig := &entity.Config{Key: key}
    exist, _ := cr.data.DB.Get(oldConfig)

    // 2. UPDATE config SET value = ? WHERE id = ?
    cr.data.DB.ID(oldConfig.ID).Update(&entity.Config{Value: value})

    // 3. 同时刷新两个缓存 Key
    oldConfig.Value = value
    cacheVal := oldConfig.JsonString()
    cr.data.Cache.SetString(ctx, "config:kc:"+key, cacheVal, ConfigCacheTime)
    cr.data.Cache.SetString(ctx, "config:ic:"+oldConfig.ID, cacheVal, ConfigCacheTime)
}
```

**关键性质**：
- 更新 DB 后**立即刷新 Redis 缓存**，不存在延迟窗口
- 两个缓存 Key（按 key 和按 id）**同时刷新**，不会出现不一致
- 缓存刷新失败仅 `log.Error(err)`，不影响主流程返回

### 19.3 声望配置修改的实际生效时序

以管理员将 `rank.question.voted_up` 从 `10` 改为 `15` 为例：

```
管理员提交修改
  │
  ▼
configRepo.UpdateConfig("rank.question.voted_up", "15")
  ├─ UPDATE config SET value='15' WHERE key='rank.question.voted_up'
  ├─ SET Redis "config:kc:rank.question.voted_up" = {id:42, key:..., value:"15"}
  └─ SET Redis "config:ic:42" = {id:42, key:..., value:"15"}
  │
  ▼ (立即生效)
下一次投票: getActivities()
  → configService.GetConfigByKey("rank.question.voted_up")
  → Redis 命中 → 返回 value=15
  → Activity.Rank = 15   ← 新值立即生效
```

**注意**：已存在的 Activity 记录的 `Rank` 字段**不会自动更新**。修改配置只影响**未来新创建的 Activity**，已入库的 Rank 值保持不变。这是有意的设计——历史声望变更应反映当时的规则。

### 19.4 配置读取失败的降级

[getActivities](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go#L280-L297) 中配置读取失败的处理：

```go
cfg, err := configService.GetConfigByKey(ctx, action)
if err != nil {
    log.Warnf("get config by key error: %v", err)
    // 不返回 error，Activity.Rank 保持为 0（默认值）
}
activity.Rank = cfg.GetIntValue()
```

**降级行为**：配置读取失败 → `Rank = 0` → 投票仍然成功，但本次不增加声望。用户不会收到错误提示，但声望丢失。

### 19.5 GetConfigByKeyFromDB（绕过缓存）

[GetStringValueFromDB](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/config/config_service.go#L69-L75) 提供了**绕过缓存直接查 DB** 的能力，但投票链路**不使用**此方法，始终走缓存路径。

---

## 二十、ExternalNotificationService 外部通知邮件链路

### 20.1 投票不直接触发外部邮件

**核心结论**：投票行为本身**不会**触发 `ExternalNotificationService` 的邮件发送。

`ExternalNotificationService.Handler`（[external_notification.go:L74-L97](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/notification/external_notification.go#L74-L97)）只处理以下四种 `ExternalNotificationMsg`：

| 触发条件 | 处理方法 | 邮件模板 |
|----------|----------|----------|
| `NewQuestionTemplateRawData != nil` | `handleNewQuestionNotification` | 新问题通知 |
| `NewCommentTemplateRawData != nil` | `handleNewCommentNotification` | 新评论通知 |
| `NewAnswerTemplateRawData != nil` | `handleNewAnswerNotification` | 新回答通知 |
| `NewInviteAnswerTemplateRawData != nil` | `handleInviteAnswerNotification` | 邀请回答通知 |

投票产生的 `ExternalNotificationMsg` 不携带以上任何 RawData 字段，因此 Handler 会走到 `log.Errorf("unknown notification message: %+v", msg)` 分支直接丢弃。

### 20.2 外部通知的完整发送链路（以新回答为例）

当有人回答问题时，外部通知的完整路径：

```
AnswerService.CreateAnswer
  │
  ▼
notificationCommon.SendExternalNotification(
  ReceiverUserID, ReceiverEmail, ReceiverLang,
  NewAnswerTemplateRawData{...})
  │
  ▼
noticequeue.ExternalService.Send(ExternalNotificationMsg)
  │
  ▼ (异步 goroutine)
ExternalNotificationService.Handler
  │
  ▼
handleNewAnswerNotification(ctx, msg)
  │
  ├─① 查询用户通知偏好
  │   userNotificationConfigRepo.GetByUserIDAndSource(userID, InboxSource)
  │   → 无配置: return nil（不发送）
  │
  ├─② 解析 channels JSON
  │   channels = NewNotificationChannelsFormJson(config.Channels)
  │   → 遍历每个 channel:
  │     if !channel.Enable → continue
  │     if channel.Key == "email" → 调 sendEmail
  │
  └─③ sendNewAnswerNotificationEmail
       ├─ checkUserStatusBeforeNotification(userID)
       │   → 用户不存在或已封禁: return（静默丢弃）
       ├─ 设置接收者语言: ctx = context.WithValue(AcceptLanguage)
       ├─ 渲染邮件模板: emailService.NewAnswerTemplate(ctx, rawData)
       │   → title, body = translator.TrWithData(lang, templateKey, data)
       └─ 发送邮件: emailService.SendAndSaveCodeWithTime(...)
            ├─ emailRepo.SetCode(userID, code, content, 24h) // 保存退订码
            └─ emailService.Send(toEmail, subject, body)
                 ├─ GetEmailConfig() → 从 config 表读取 SMTP 配置
                 ├─ SMTPHost 为空 → log.Warn("skip send email") → return
                 ├─ gomail.NewMessage() 构造邮件
                 ├─ gomail.NewDialer(host, port, user, pass)
                 └─ d.DialAndSend(m)
                      → 失败: log.Errorf("send email failed: %s", err)
                      → 成功: log.Infof("send email success")
```

### 20.3 用户通知偏好筛选机制

[UserNotificationConfig](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/entity/user_notification_config_entity.go#L25-L33) 实体：

```sql
CREATE TABLE user_notification_config (
  id BIGINT PK AUTO_INCREMENT,
  user_id BIGINT,        -- 用户 ID
  source VARCHAR(64),    -- 通知来源: "inbox" / "all_new_question" / "all_new_question_for_following_tags"
  channels TEXT,         -- JSON: [{"key":"email","enable":true}]
  enabled BOOL,          -- 快速判断是否有任何启用渠道
  UNIQUE KEY (user_id, source)
)
```

**筛选流程**：
1. 按 `(user_id, source=inbox)` 查询用户配置
2. 无记录 → 不发送（默认关闭）
3. 有记录 → 解析 `channels` JSON，遍历每个 channel
4. `channel.Enable == false` → 跳过
5. `channel.Key == "email"` → 触发邮件发送

**注册默认配置**（[user_notification_config_service.go:L93-L97](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/user_notification_config/user_notification_config_service.go#L93-L97)）：
```go
func (us *UserNotificationConfigService) SetDefaultUserNotificationConfig(ctx, userIDs) {
    us.userNotificationConfigRepo.Add(ctx, userIDs,
        string(constant.InboxSource), `[{"key":"email","enable":true}]`)
}
```
新注册用户默认开启邮件通知。

### 20.4 邮件模板渲染

[EmailService](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/export/email_service.go) 的模板渲染流程（以 NewAnswerTemplate 为例，[email_service.go:L233-L263](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/export/email_service.go#L233-L263)）：

```
步骤1: 获取站点信息（siteInfo.Name, siteInfo.SiteUrl, seoInfo.Permalink）
步骤2: 构造模板数据结构
  NewAnswerTemplateData{
    SiteName:       siteInfo.Name,
    DisplayName:    raw.AnswerUserDisplayName,
    QuestionTitle:  raw.QuestionTitle,
    AnswerUrl:      display.AnswerURL(permalink, siteUrl, questionID, title, answerID),
    AnswerSummary:  raw.AnswerSummary,
    UnsubscribeUrl: siteUrl + "/users/unsubscribe?code=" + unsubscribeCode,
  }
步骤3: 翻译渲染（按用户语言）
  title = translator.TrWithData(lang, "email.new_answer.title", templateData)
  body  = translator.TrWithData(lang, "email.new_answer.body", safeTemplateData)
  // safeTemplateData 中对 HTML 展示字段做了 escapeEmailHTMLText
步骤4: 返回 (title, body)
```

**XSS 防护**：`body` 模板中的 `SiteName`、`DisplayName`、`QuestionTitle`、`AnswerSummary` 等字段在渲染前通过 `html.EscapeString()` 转义，防止存储型 XSS 通过邮件传播。URL 字段（`AnswerUrl`、`UnsubscribeUrl`）不转义。

### 20.5 SMTP 发送与失败重试

[EmailService.Send](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/export/email_service.go#L123-L157)：

```go
func (es *EmailService) Send(ctx, toEmailAddr, subject, body string) {
    ec, _ := es.GetEmailConfig(ctx)   // 每次发送都重新读取 SMTP 配置
    if len(ec.SMTPHost) == 0 {
        log.Warnf("smtp host is empty, skip send email")
        return                        // 未配置 SMTP → 静默跳过
    }

    m := gomail.NewMessage()
    m.SetHeader("From", fromName + " <" + fromEmail + ">")
    m.SetHeader("To", toEmailAddr)
    m.SetHeader("Subject", subject)
    m.SetBody("text/html", body)

    d := gomail.NewDialer(ec.SMTPHost, ec.SMTPPort, ec.SMTPUsername, ec.SMTPPassword)
    if ec.IsSSL() { d.SSL = true }
    if ec.IsTLS() { d.SSL = false }
    if os.Getenv("SKIP_SMTP_TLS_VERIFY") != "" {
        d.TLSConfig = &tls.Config{InsecureSkipVerify: true}
    }

    if err := d.DialAndSend(m); err != nil {
        log.Errorf("send email to %s failed: %s", toEmailAddr, err)
    } else {
        log.Infof("send email to %s success", toEmailAddr)
    }
}
```

**失败重试机制**：
- **无重试**。`DialAndSend` 失败后只 `log.Errorf`，邮件**永久丢失**
- **无死信队列**：没有重入 noticequeue 或存 DB 待重发的逻辑
- **无退避策略**：SMTP 连接失败后不会 exponential backoff
- **环境变量绕过 TLS 验证**：`SKIP_SMTP_TLS_VERIFY=1` 可跳过 TLS 证书校验（仅限开发环境）

### 20.6 投票与外部邮件的间接关联

虽然投票不直接触发外部邮件，但投票引发的**二级事件**可能间接触发：

```
用户 A 对问题投 UP
  → sendEvent(EventQuestionVote)
    → eventqueue → BadgeEventService
      → 若命中 ReachQuestionVote 徽章 → Award → noticequeue(Achievement)
        → noticequeue(内部): AddNotification (不触发邮件)
        → 不发 ExternalNotificationMsg（徽章没有外部通知模板）
```

**结论**：投票链路在当前代码中**完全不触发外部邮件通知**。外部邮件仅在新问题/新回答/新评论/邀请回答四种场景触发。

---

## 二十一、日志埋点与队列监控的单测覆盖盲点

### 21.1 投票链路的日志埋点现状

| 层级 | 文件 | 日志级别 | 触发场景 | 日志内容 |
|------|------|----------|----------|----------|
| VoteRepo | [vote_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go) | `log.Error` | acquireUserInfo 查询失败 | 原始 error |
| VoteRepo | vote_repo.go | `log.Error` | setActivityRankToZero 读取用户信息失败 | 原始 error |
| VoteRepo | vote_repo.go | `log.Error` | saveActivitiesAvailable 更新失败 | 原始 error |
| VoteRepo | vote_repo.go | `log.Error` | cancelActivities 已取消 Activity 更新失败 | activity.ID + "not exist" |
| VoteRepo | vote_repo.go | `log.Error` | changeUserRank/rollbackUserRank 失败 | 原始 error |
| VoteRepo | vote_repo.go | `log.Errorf` | countVoteUp/Down 查询失败 | "get vote up/down count error" |
| VoteRepo | vote_repo.go | `log.Error` | updateVotes 更新 vote_count 失败 | 原始 error |
| VoteService | [vote_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go) | `log.Error` | sendEvent 失败 | 原始 error |
| VoteService | vote_service.go | `log.Error` | GetAndSaveVoteResult 失败 | 原始 error |
| VoteService | vote_service.go | `log.Warnf` | getActivities 配置读取失败 | "get config by key error" |

**日志缺失的场景**：
| 缺失场景 | 影响 |
|----------|------|
| 投票成功无 Info 日志 | 无法通过日志统计投票 QPS/延迟 |
| 去重命中（votePreCheck 返回 noNeedToVote）无日志 | 无法判断去重拦截频率 |
| 每日上限触发（Rank=0）无日志 | 无法监控上限命中的用户和时间 |
| vote_count 重算值与旧值差异大无日志 | 无法发现计數漂移 |
| 事务耗时无日志 | 无法定位慢投票事务 |
| 队列 Send 阻塞无日志 | 无法发现队列背压 |

### 21.2 队列深度 Metric 的完全缺失

系统中**没有任何 metric 埋点**。搜索整个 `internal` 目录，未找到 `prometheus`、`histogram`、`counter`、`metric` 相关代码。

**缺失的队列 metric**：

| Metric | 作用 | 当前状态 |
|--------|------|----------|
| `queue_depth{queue="eventqueue"}` | 监控队列积压深度 | ❌ 无 |
| `queue_latency{queue="eventqueue"}` | 消息从入队到消费的延迟 | ❌ 无 |
| `queue_handler_duration{queue="noticequeue"}` | Handler 执行耗时 | ❌ 无 |
| `queue_handler_error_total{queue="eventqueue"}` | Handler 失败计数 | ❌ 无 |
| `queue_send_blocked_total{queue="*"}` | Send 阻塞次数 | ❌ 无 |
| `vote_transaction_duration` | 投票事务耗时 | ❌ 无 |
| `rank_change_total{direction="up/down"}` | 声望增减计数 | ❌ 无 |

**影响**：
- 队列积压时无法告警（只能靠用户投诉"没收到徽章"才发现）
- 投票事务变慢时无法定位（无 P99 延迟监控）
- 声望异常增长时无法检测（无增量监控）

### 21.3 单测覆盖盲点

项目中与投票/队列相关的单测文件：

| 测试文件 | 覆盖范围 | 缺失场景 |
|----------|----------|----------|
| [queue_test.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/base/queue/queue_test.go) | 队列基础操作：发送/接收/多消息/并发/Close 竞态 | 见下表 |

**queue_test.go 覆盖的场景**：
- ✅ 单条消息发送接收
- ✅ 100 条消息顺序消费
- ✅ 无 Handler 时消息丢弃
- ✅ 先 Send 后 RegisterHandler
- ✅ Close 后所有消息处理完毕
- ✅ 10 goroutine 并发 Send
- ✅ 并发 RegisterHandler 竞态
- ✅ Send/Close 并发竞态（100 次迭代）

**queue_test.go 缺失的场景**：
| 缺失测试 | 风险 |
|----------|------|
| Handler 返回 error 的路径 | 验证 error 仅 log 不重入队列 |
| channel 满时 Send 阻塞行为 | 验证背压和 ctx 取消 |
| ctx 取消后 Send 返回行为 | 验证不阻塞已取消的请求 |
| Close 后 Send 行为 | 验证不 panic（当前测试只验证了不 panic 但没断言返回值） |
| 消息处理顺序保证 | 验证 FIFO 严格性（当前只验证了计数） |

**完全缺失的投票层单测**：

| 缺失的测试模块 | 应覆盖场景 |
|----------------|-----------|
| vote_repo_test | 去重两层（votePreCheck + saveActivitiesAvailable）、并发投票竞争、每日上限触发、声望下界保护 |
| vote_service_test | 互斥取消（Cancel down + Vote up）编排、sendEvent 仅正向投票触发 |
| cancel_vote_test | CancelVote 幂等性、已取消记录二次取消 Rank=0、声望回滚正确性 |
| rank_change_test | ChangeUserRank 原子 INCR、负数扣减到 1 截断、RankAgent 开关跳过 |
| badge_event_test | EventRuleMapping 规则匹配、CheckIsAward 去重、Award 失败不中断其他 |
| notification_test | Achievement 去重更新 rank、Inbox 不去重、红点计数 |

### 21.4 盲点的风险总结

| 维度 | 当前状态 | 最坏风险 | 建议优先级 |
|------|----------|----------|------------|
| 日志埋点 | 仅 Error 级别，无业务指标 | 投票失败只能靠用户投诉发现 | P1 - 补充 Info/Warn 日志 |
| 队列 Metric | 完全缺失 | 队列积压无告警，徽章延迟无感知 | P1 - 至少补 queue_depth |
| 投票单测 | 完全缺失 | 去重/取消/声望保护逻辑无回归保障 | P1 - 补 vote_repo_test |
| 队列单测 | 基础覆盖 | Handler error 路径未验证 | P2 - 补 error 路径测试 |
| 邮件重试 | 无重试 | SMTP 临时故障导致邮件永久丢失 | P2 - 至少加 1 次重试 |
