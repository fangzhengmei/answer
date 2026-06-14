# 管理员撤票、反作弊限流与用户贡献度曲线代码理解

## 一、管理员撤销违规投票的代码路径

### 1.1 入口：管理员后台处理举报

管理员在后台处理举报时，通过 `ReportHandle.UpdateReportedObject` 入口对违规内容进行处置：

[ReportHandle.UpdateReportedObject](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/report_handle/report_handle.go#L53-L68)：

```go
func (rh *ReportHandle) UpdateReportedObject(ctx, report *entity.Report, req *schema.ReviewReportReq) {
    // 根据 ObjectID 解析对象类型
    switch objectType {
    case "question":
        rh.updateReportedQuestionReport(...)
    case "answer":
        rh.updateReportedAnswerReport(...)
    case "comment":
        rh.updateReportedCommentReport(...)
    }
}
```

对不同对象类型，管理员可执行的操作不同：

| 对象 | 操作类型 | 调用 |
|------|----------|------|
| Question | UnlistPost / DeletePost / ClosePost / EditPost | `OperationQuestion` / `RemoveQuestion` / `CloseQuestion` / `UpdateQuestion` |
| Answer | DeletePost / EditPost | `RemoveAnswer` / `UpdateAnswer` |
| Comment | DeletePost / EditPost | `RemoveComment` / `UpdateComment` |

违规投票的典型处置路径是 **DELETE（删除内容）**：
- 问题：[report_handle.go:L76-L78](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/report_handle/report_handle.go#L76-L78) → `rh.questionService.RemoveQuestion(IsAdmin=true)`
- 回答：[report_handle.go:L102-L104](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/report_handle/report_handle.go#L102-L104) → `rh.answerService.RemoveAnswer`

### 1.2 RemoveQuestion 的完整流程

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

### 1.3 关键发现：声望回滚**已被注释**！

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

### 1.4 注释后的实际影响

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

### 1.5 设计意图（从注释推测）

注释说明：
> "为了简化流程和复杂度，同时考虑如果是内部环境面临的恢复问题。"

保留 Activity 不回滚声望的理由可能是：
1. 回滚所有相关投票的声望需要找出所有给该内容投票的用户，遍历并 UPDATE，复杂度高
2. 删除内容后可能需要恢复，保留 Activity 便于恢复时直接恢复状态
3. 违规内容删除后，违规投票者的不当得利（或损失）暂时保留，由管理员后续人工处理
4. 内部环境下这种简化是可接受的

---

## 二、反作弊限流校验

系统有**三层防刷/反作弊机制**，独立运行，共同作用：

### 2.1 第一层：内容提交去重（重复请求拦截）

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
- TTL: `RateLimitCacheTime = 3 * time.Second` ([cache_key.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/base/constant/cache_key.go))
- 存储：Redis Cache 层

### 2.2 第二层：操作级验证码校验

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

### 2.3 第三层：声望门槛

投票前在 Controller 层调用 [CheckVotePermission](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/rank/rank_service.go#L178-L223)：

```go
// 比较 user.rank >= config["rank.question.vote_up"]
can, needRank, _ := rankService.CheckVotePermission(ctx, objectID, userID, "vote_up")
```

低声望新用户无法立即投票，需先通过提问/回答积累声望，提高刷票成本。

### 2.4 三层防御的叠加效果

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

## 三、Activity 表到用户主页声望和贡献度的映射

Activity 表是**一切用户贡献度数据的事实源**。用户主页有三个核心展示组件，全部从 Activity 表聚合查询：

| 展示组件 | 接口 | 查询 SQL 模式 |
|----------|------|--------------|
| 声望历史曲线 | `/personal/rank/page` | 按 `created_at` 倒序，`has_rank=1 AND cancelled=0 AND rank>0` |
| 投票历史列表 | `/personal/vote/page` | 按 `updated_at` 倒序，`activity_type IN (vote_up/down 四类)` |
| 用户排行榜 | `/user/ranking` | 最近 7 天 `SUM(rank)` 排名 / `COUNT(vote)` 排名 |

### 3.1 声望历史（贡献度曲线）

**接口**：`GET /answer/api/v1/personal/rank/page`

**调用链**：
[RankController.GetRankPersonalWithPage](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/rank_controller.go#L51-L61)
→ [RankService.GetRankPersonalPage](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/rank/rank_service.go#L262-L289)
→ [UserRankRepo.UserRankPage](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/rank/user_rank_repo.go#L201-L215)

**查询 SQL**（[user_rank_repo.go:L206-L207](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/rank/user_rank_repo.go#L206-L207)）：
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

### 3.2 投票历史列表

**接口**：`GET /answer/api/v1/personal/vote/page`

**调用链**：
[VoteController.UserVotes](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/vote_controller.go#L176-L186)
→ [VoteService.ListUserVotes](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go#L188-L229)
→ [VoteRepo.ListUserVotes](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L191-L208)

**查询 SQL**（[vote_repo.go:L194-L201](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go#L194-L201)）：
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

### 3.3 用户排行榜

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
  AND activity_type IN (QuestionVoteUp, QuestionVoteDown, ...)  -- 所有投票类 activity_type
  AND created_at BETWEEN startTime AND endTime
GROUP BY user_id
ORDER BY vote_count DESC
LIMIT 20
```

#### 第三组：工作人员列表
独立查询，管理员和版主直接显示在排行榜末尾。

### 3.4 完整的 Activity → 展示映射图

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

### 3.5 与删除内容的交互

由于删除内容时 Activity **不清理也不回滚**（见 1.3 节注释），导致：
- 用户主页仍能看到给已删除内容投票的记录和声望
- 排行榜仍包含已删除内容带来的声望
- 这是有意的设计选择（#2372 简化流程），但可能在违规处置场景下需要人工补偿

---

## 四、声望的三层存储模型

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

### 4.1 每日上限的统计口径

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

## 五、代码索引

| 关注点 | 文件 | 关键行 |
|--------|------|--------|
| 举报处理入口 | [report_handle.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/report_handle/report_handle.go) | L53-L138 |
| 删除问题 | [question_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/question_service.go) | L555-L664 |
| 删除回答 | [answer_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/answer_service.go) | L119-L202 |
| 声望回滚注释 | [question_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/question_service.go) | L639-L644 |
| 反作弊去重中间件 | [rate_limit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/base/middleware/rate_limit.go) | L46-L73 |
| 去重 Redis 实现 | [limit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/limit/limit.go) | L45-L65 |
| 声望历史查询 | [user_rank_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/rank/user_rank_repo.go) | L201-L215 |
| 声望历史装饰 | [rank_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/rank/rank_service.go) | L291-L320 |
| 投票历史查询 | [vote_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity/vote_repo.go) | L191-L208 |
| 投票历史装饰 | [vote_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/vote_service.go) | L188-L229 |
| 声望增长榜 | [activity_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity_common/activity_repo.go) | L147-L163 |
| 投票活跃度榜 | [activity_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/repo/activity_common/activity_repo.go) | L165-L191 |
| 排行榜编排 | [user_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/content/user_service.go) | L790-L823 |
| 声望阈值校验 | [rank_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/service/rank/rank_service.go) | L178-L223 |
| 投票验证码 | [vote_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/63-answer/internal/controller/vote_controller.go) | L88-L94 |
