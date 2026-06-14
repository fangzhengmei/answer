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

---

## 十、事件总线与下游链路全景（SendEvent 之后）

### 10.1 四条完全独立的异步队列

系统中有**四条独立的内存队列**，各自承载不同的语义。它们共享同一个底层 `queue.Queue[T]` 泛型实现，但相互之间**没有任何关联**，消费者和发送者也不同：

| 队列名 | 消息类型 | 容量 | 发送者 | 唯一消费者 | 核心用途 |
|--------|----------|------|--------|------------|----------|
| **eventqueue** | `*schema.EventMsg` | 128 | question/answer/comment/report/meta/user 服务 + VoteService | BadgeEventService | **徽章（Badge）解锁判定** |
| **noticequeue(内部)** | `*schema.NotificationMsg` | 128 | NotificationCommon / BadgeAwardService / VoteRepo | NotificationCommon.AddNotification | **站内通知（Inbox + Achievement）** |
| **noticequeue(外部)** | `*schema.ExternalNotificationMsg` | 128 | user_notification_config 等 | ExternalNotificationService.Handler | **邮件/第三方插件通知** |
| **activityqueue** | `*schema.ActivityMsg` | 128 | question/answer/comment/revision/tag 服务 | ActivityCommon.HandleActivity | **非投票类活动时间线** |
| **vector_sync** | `*Task` | 128 | question/answer/comment/review 服务 | vector_sync.handle（内置） | **向量搜索索引同步** |

> **关键：投票不走 activityqueue**。投票类 Activity（`question.vote_up` 等）在 vote_repo.go 的事务中直接 INSERT，不经过异步队列。activityqueue 承载的是创建/编辑/关闭/删除等非投票行为。

### 10.2 队列底层实现（单 Handler + 单 Worker）

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

## 十一、投票 SendEvent 的精确下游

### 11.1 SendEvent 发出了什么

[VoteService.sendEvent](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go#L301-L323) 只在**非 Cancel 的正向投票**后调用（取消投票不发 EventMsg）：

```go
func (vs *VoteService) sendEvent(ctx, op, userID, authorUserID string) {
    event := schema.NewEvent(constant.EventQuestionVote, userID)  // 默认 EventQuestionVote
    if op == "answer" {
        event.EventType = constant.EventAnswerVote
    } else if op == "comment" {
        event.EventType = constant.EventCommentVote
    }
    // ...填充 QuestionID/AnswerID/CommentID 和各自的 UserID
    event.AddExtra("vote_up_amount", fmt.Sprintf("%d", upVotes))
    vs.eventQueueService.Send(ctx, event)   // 只发到 eventqueue！
}
```

**投票只发 eventqueue**，不直接发 noticequeue。通知的产生走另外两条路：
- **站内成就通知**：在 vote_repo.go 的事务外通过 `achievementCommon.SendNotification`（声望变更通知）
- **徽章解锁通知**：徽章解锁成功后 BadgeAwardService 自己发 noticequeue

### 11.2 投票后的完整链路图

```
用户提交 UP 投票
    │
    ▼
[Controller] 权限/验证码
    │
    ▼
[VoteService.VoteUp]
    ├─① voteRepo.CancelVote(down)        // 事务：撤反向票+回滚声望
    ├─② voteRepo.Vote(up)                // 事务：写Activity+加声望
    │    │
    │    └─（事务外）SendNotification     // → noticequeue(内部): 声望成就通知
    │
    ├─③ voteRepo.GetAndSaveVoteResult    // vote_count 重算
    └─④ sendEvent                         // → eventqueue: 投票事件（徽章判定）
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    异步消费（各队列独立）                        │
│                                                                 │
│  [eventqueue:BadgeEventService.Handler]                         │
│    ├─ EventRuleMapping[EventQuestionVote] = [FirstVotedPost,    │
│    │                                             ReachQuestionVote]│
│    ├─ 若命中徽章条件:                                            │
│    │   BadgeAwardService.Award()                                │
│    │     ├─ CheckIsAward（防重复）                               │
│    │     ├─ INSERT badge_award                                  │
│    │     └─ Send NotificationMsg → noticequeue: 徽章成就通知    │
│    └─ return nil  // 失败只打日志，不重试                        │
│                                                                 │
│  [noticequeue:NotificationCommon.AddNotification]               │
│    ├─ 区分 Inbox(1) vs Achievement(2)                          │
│    ├─ Achievement 去重：(user,object,type) 已存在则更新 rank    │
│    ├─ INSERT notification 表                                    │
│    ├─ Redis 红点计数 +1                                         │
│    ├─ 徽章类型: AddBadgeAwardAlertCache                         │
│    ├─ go SendNotificationToAllFollower()  // 广播给粉丝(新问答) │
│    └─ syncNotificationToPlugin()  // 调 plugin.Notification     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 十二、eventqueue 队列与徽章解锁

### 12.1 注册与订阅关系

**唯一订阅者**：[BadgeEventService](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/badge/badge_event_handler.go)
```go
// NewBadgeEventService 构造时:
eventQueueService.RegisterHandler(n.Handler)  // 唯一一个 handler
```

### 12.2 Event → 徽章规则映射

[badge_event_rule.go:L47-L68](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/badge/badge_event_rule.go#L47-L68) 的 `EventRuleMapping` 定义：

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

### 12.3 Handler 执行流程

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

### 12.4 消费失败语义

- **handler 返回 error**：只 `log.Errorf("[%s] handler error: %v")`，消息**被丢弃**不重放
- **Award() 返回 error**：每个 award 独立 `log.Debugf`，不影响其他徽章授予
- **无死信队列**：没有 DLQ，失败后只能等待下一次同类事件触发再尝试

---

## 十三、noticequeue 队列与站内通知

### 13.1 两条 noticequeue 的区分

注意：`noticequeue` 包下有**两种不同的 Service 类型**，对应两条独立的 channel 队列：

```go
// 队列A: 站内通知(内部) —— buffer 128
type Service queue.Service[*schema.NotificationMsg]
func NewService() Service { return queue.New[*schema.NotificationMsg]("notification", 128) }

// 队列B: 外部通知(邮件/插件) —— buffer 128
type ExternalService queue.Service[*schema.ExternalNotificationMsg]
func NewExternalService() ExternalService { return queue.New[*schema.ExternalNotificationMsg]("external_notification", 128) }
```

| 队列 | 注册者 | 注册位置 | Handler |
|------|--------|----------|---------|
| A(内部) | [NotificationCommon](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/notification_common/notification.go#L96) | `NewNotificationCommon` | `AddNotification` |
| B(外部) | [ExternalNotificationService](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/notification/external_notification.go#L70) | `NewExternalNotificationService` | `.Handler`（邮件分发） |

因为类型参数不同，两者的 RegisterHandler 互不干扰，不存在覆盖问题。

### 13.2 AddNotification 内部执行流程

[notification.go:L109-L220](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/notification_common/notification.go#L109-L220)，输入 `NotificationMsg{Type, TriggerUserID, ReceiverUserID, ObjectID, ObjectType, Title, NotificationAction, ...}`：

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

### 13.3 成就通知 vs 站内收件箱的区别

| 维度 | Inbox (Type=1) | Achievement (Type=2) |
|------|----------------|----------------------|
| 触发场景 | 有人回答/评论/编辑/采纳你的内容 | 你获得声望 / 你解锁徽章 |
| 去重策略 | 不去重，每条独立 INSERT | `(user, object, type=2)` 唯一，存在就 UPDATE rank |
| 发送者 | 所有内容变更场景 | vote_repo 声望通知 + BadgeAwardService 徽章通知 |

---

## 十四、搜索索引更新链路（重要：不走 eventqueue！）

### 14.1 文本搜索索引（keyword search）

**触发方式：repo 层直接同步调用 plugin，非异步队列**

| 操作 | 触发位置 | 调用 |
|------|----------|------|
| 问题 CUD/状态变更/接受/置顶 | [question_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/question/question_repo.go) | `qr.UpdateSearch(ctx, questionID)` |
| 回答 CUD/状态变更/接受 | [answer_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/answer/answer_repo.go) | `ar.updateSearch(ctx, answerID)` |
| 问题通过审核 | [question_service.go:L415](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/question_service.go#L415) | `qs.questionRepo.UpdateSearch` |

**投票不触发搜索索引重建！** `VoteService` 中没有任何 `UpdateSearch` 调用。问题/回答的 `vote_count`（搜索文档的 `Score` 字段）在投票后不会立即同步到搜索索引。

**UpdateSearch 的内部实现**（[question_repo.go:L584-L634](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/question/question_repo.go#L584-L634)）：
```
① plugin.CallSearch → 检查是否注册了 Search 插件，未注册直接 return
② DB 重新读取问题/回答全文 + tags + 最新 VoteCount / AnswerCount / ViewCount ...
③ s.UpdateContent(ctx, &plugin.SearchContent{Score=question.VoteCount, ...})
   → 由具体插件实现（meilisearch/typesense/algolia 等）的 upsert 逻辑
```

### 14.2 向量搜索索引（embedding / semantic search）

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

### 14.3 投票导致的 Score 滞后

投票后 DB 中 `question.vote_count` 已是新值，但：
- **文本搜索 Score** 落后 → 下次创建/编辑/审核触发时才刷新
- **向量搜索 Score** 同样落后 → 下次内容变更才重建

这是有意的设计权衡（性能优先），若需强一致需要在 VoteService 末尾手动追加 `UpdateSearch` 和 `vectorSyncService.Send`。

---

## 十五、消费顺序与失败回退机制

### 15.1 投票成功后的实际执行顺序

以一次「对回答投 UP，之前是 Down 票」为例，从 VoteService 层看：

```
同步阶段（HTTP 请求线程内，强顺序保证）:
 T1: voteRepo.CancelVote(down)
       ├─ 事务: BEGIN → ForUpdate 锁用户 → soft-delete Activities
       │            → 回滚 User.rank → COMMIT
       └─（事务外）achievementCommon.SendNotification → 入队 noticequeue

 T2: voteRepo.Vote(up)
       ├─ 事务: BEGIN → ForUpdate 锁用户 → saveActivities
       │            → 加 User.rank → COMMIT
       └─（事务外）achievementCommon.SendNotification → 入队 noticequeue

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

### 15.2 失败回退机制总结

| 层级 | 失败场景 | 回退/重试策略 |
|------|----------|---------------|
| **vote_repo 事务** | 任一步骤 error | 事务 ROLLBACK，活动记录 + 声望都不生效，HTTP 返回错误 |
| **GetAndSaveVoteResult** | DB 更新 vote_count 失败 | VoteService 直接返回 error 给前端（前面事务已提交，**不会回滚活动和声望**。vote_count 可被下次投票修复） |
| **SendEvent / SendNotification** | channel 满阻塞 | 阻塞 HTTP 线程（背压）；ctx 取消则丢弃消息 |
| **eventqueue handler** | 徽章判定/授予 error | 仅 log，消息丢弃，不重试。下次同类型事件可再次触发同样规则 |
| **noticequeue AddNotification** | INSERT notification / Redis 红点失败 | handler 返回 error → log，消息丢弃。红点可能少加 1（用户实际会看到有新通知但红点为0，下次进入列表时修正） |
| **vector_sync handler** | 向量更新失败 | **重试 3 次**，仍失败则 log，消息丢弃 |
| **SendNotificationToAllFollower** | goroutine 内的 DB/入队失败 | log.Error，该部分粉丝收不到通知，无补偿 |

### 15.3 关键幂等性保证

| 操作 | 幂等机制 |
|------|----------|
| 活动记录去重 | `(object_id, user_id, trigger_user_id, activity_type)` + Cancelled 状态机 |
| 声望增减 | Activity 存了 Rank 值，去重时复用；ChangeUserRank 仅对 Rank≠0 的生效 |
| 徽章授予 | `CheckIsAward(badgeID, userID, awardKey, Single)`：一次性徽章 user 唯一，多次性 awardKey 唯一 |
| 成就通知去重 | `GetByUserIdObjectIdTypeId(user, object, type=2)`：同用户同对象只有一条 Achievement 通知，已有则 UPDATE rank 累加值 |
| 投票计数 | 每次全量 COUNT(*)，天然幂等 |

---

## 十六、完整链路时序图（一次 UP 投票切换）

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

## 十七、下游链路代码索引

| 关注点 | 文件 | 关键行 |
|--------|------|--------|
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
