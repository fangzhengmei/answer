# 投票与声望计分系统代码理解

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

## 二、投票请求流程（从 HTTP 到 DB）

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

系统初始化时的典型配置（[init_data.go:L248-L249](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/migrations/init_data.go#L248-L249)）：
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
// 在 ChangeUserRank [user_rank_repo.go:L91-L94]
if deltaRank < 0 && userCurrentScore + deltaRank < 1 {
    deltaRank = 1 - userCurrentScore  // 只扣到 1 为止
}
```

同样逻辑也存在于 [setActivityRankToZeroIfUserReachLimit](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L257-L263) 的负声望分支。

**效果**：用户声望永远 ≥ 1，新用户默认为 1（或初始注册声望）。

#### 保护 3：声望 Agent 开关

```go
// [user_rank_repo.go:L87-L89]
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

`vote_count` 更新在**事务外**执行（[vote_service.go:L127](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go#L127)），且采用**重新 COUNT 聚合**而非增量加减。

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

**场景：同一用户对同一问题先投 Down 再迅速切换到 Up**

```
VoteService.VoteUp(req.IsCancel=false):
  ① CancelVote(down)
       事务1: soft-delete Down 相关 Activities; 回滚 rank
  ② Vote(up)
       事务2: votePreCheck(up未投过) → 插入 Up Activities; +rank
```

两个操作是**串行的两个独立事务**（不在一个事务里）。若中途失败（如事务1成功但事务2失败），用户会处于「没 Down 也没 Up」的中立状态，不会出现双重状态。

---

## 八、声望每日上限的竞态

这是系统中一个**值得注意的语义**：

每日上限检查在事务内执行，但检查是 `SUM(rank)` 然后 `IF >= limit THEN Rank=0`。

若同一作者在**毫秒级**收到三笔并发 UP 投票（每笔 +10，limit=200，当前已用 195）：
- 事务都读到 SUM=195 → 都判断未超限 → 都执行 +10
- 最终作者声望 = 195 + 30 = 225，突破上限 25 点

**原因**：ForUpdate 只锁 user 表行，不锁 activity 表聚合。`CheckReachLimit` 读 activity 表不受 user 表行锁保护（MySQL InnoDB 的 MVCC）。

**修正思路（代码中未实现，仅供理解）**：
1. 将每日已获声望**冗余存储到 user 表字段**（如 `daily_rank_earned`），这样可被 ForUpdate 一起锁定
2. 或使用 `SELECT ... FOR UPDATE` 锁定 activity 表的相关行（范围锁性能较差）
3. 或接受这种短暂超限，在日终异步任务中修正

当前实现选择「接受轻微超限」，是正确性与性能的平衡。

---

## 九、总结：状态变化全景图

```
用户点击"UP 投票"按钮
        │
        ▼
┌─────────────────────────────────────────────────┐
│ Controller: rankService.CheckVotePermission     │ 权限门槛
│           actionService.VerifyCaptcha           │ 验证码
└───────────────────────┬─────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│ VoteService: objectService.GetInfo              │ 1. 不能投自己 / 已删对象
│            createVoteOperationInfo              │ 2. 生成 2 条 Activity(操作+被操作)
│            CancelVote(down)                     │ 3. 撤销反向票 (如果有)
│            Vote(up)                             │ 4. 执行正向票
│            GetAndSaveVoteResult                 │ 5. COUNT 回写 vote_count
│            sendEvent                            │ 6. 发通知/搜索事件
└───────────────────────┬─────────────────────────┘
                        │ Vote(up) 内部：
                        ▼
          ┌───────────────────────────┐
          │  事务开启                  │
          │    ▼                      │
          │  acquireUserInfo(ForUpdate)│  锁用户行
          │    ▼                      │
          │  votePreCheck              │  快速去重(非事务内)
          │    ▼                      │
          │  setActivityRankToZero     │  每日上限保护 + 下界保护
          │    ▼                      │
          │  saveActivitiesAvailable   │  INSERT/UPDATE activity(幂等去重)
          │    ▼                      │
          │  changeUserRank(INCR)      │  原子 UPDATE user.rank
          │    ▼                      │
          │  事务提交                  │
          └─────────────┬─────────────┘
                        │
            ┌───────────┴───────────┐
            ▼                       ▼
    Activity 表（事实源）         User 表（声望）
    cancelled=0                  rank += 10
    rank=配置值                  （每日上限≈200）
            │                       │
            │ COUNT(*) WHERE        │ 用于权限判断
            │   cancelled=0         │ CheckVotePermission
            ▼                       ▼
    Question/Answer/Comment 表的 vote_count
    （冗余缓存，每次重算）
```

### 关键代码文件索引

| 关注点 | 文件 | 关键行 |
|--------|------|--------|
| HTTP 入口 | [vote_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/vote_controller.go) | L68-L110, L122-L163 |
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
