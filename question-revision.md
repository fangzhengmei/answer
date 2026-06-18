# 已发布问题修改代码实现全流程梳理

本文档按 **提交 → 权限分流 → 版本审核 → 变更展示** 的代码顺序梳理已发布问题修改在代码中的完整实现链路。引用路径均为仓库相对路径（仓库根 = `code/36-answer/`）。

---

## 0. 核心数据模型

### 0.1 修订（Revision）实体

定义：`internal/entity/revision_entity.go`

| 字段 | 说明 |
|------|------|
| `ObjectType` | 对象类型：Question(1) / Answer(2) / Tag(3)，映射见 `internal/base/constant/object.go` 中的 `ObjectTypeStrMapping` |
| `ObjectID` | 被修改对象的 ID |
| `Title` / `Content` | 修改后的标题 / 内容（Content 存 JSON 序列化后的完整对象快照） |
| `Log` | 修改摘要（edit_summary） |
| `Status` | 修订状态：`0=Normal` / `1=Unreviewed` / `2=ReviewPass` / `3=ReviewReject` |
| `UserID` | 提交修改的用户 ID |
| `ReviewUserID` | 审核人 ID |

4 种状态常量定义在 `internal/entity/revision_entity.go` L28-L34。

### 0.2 审核（Review）实体

定义：`internal/entity/review_entity.go`

> Review 用于新发布内容的审核（新问题、新回答），问题修改走的是 Revision 机制，两者分离。Review 表只处理 `1=Pending` / `2=Approved` / `3=Rejected` 状态。

### 0.3 活动（Activity）实体

定义：`internal/entity/activity_entity.go` L30-L43

| 字段 | 说明 |
|------|------|
| `ActivityType` | 活动类型（int，映射自 `config` 表中 `question.edited` 等 key 的 ID） |
| `ObjectID` / `OriginalObjectID` | 关联对象 ID |
| `RevisionID` | 关联到 Revision.ID（int64），Activity 与 Revision 的关联字段 |
| `UserID` / `TriggerUserID` | 对象作者 / 触发人（例如编辑人） |
| `Cancelled` | 是否已被撤销（0=有效，1=已撤销） |

Revision 与 Activity 是 1:N 关系：一个 Revision 可能关联 0 至 2 条 Activity（见第 4.2 节）。

---

## 1. 提交流程（用户提交修改）

```
前端编辑页 ui/src/pages/Questions/Ask/index.tsx
    ↓ modifyQuestion()
Controller: UpdateQuestion [PUT /answer/api/v1/question]
    ↓ rankService.CheckOperationPermissionsForRanks()
    ↓ 权限分流（第 2 节）
Service: QuestionService.UpdateQuestion()
    ↓ 构造 AddRevisionDTO
    ↓ revisionService.AddRevision()
    ↓ （若免审核）直接写库 + 更新标签 + 发送 ActQuestionEdited 活动
```

### 1.1 前端：编辑提交

**入口页面**：`ui/src/pages/Questions/Ask/index.tsx`

- 通过 URL 参数 `qid` 判断编辑模式（`isEdit = qid !== undefined`）。
- 编辑时额外渲染 `edit_summary` 字段（修改摘要）。
- 提交函数 `submitModifyQuestion()` → `modifyQuestion(ep)`，其中 `ep` 含：`id`, `title`, `content`, `tags`, `edit_summary`, `captcha_code`, `captcha_id`。
- 响应中 `wait_for_review=true` 时跳转后携带 `state.isReview` 提示等待审核。

API 层：服务方法 `modifyQuestion` → `PUT /answer/api/v1/question`（`ui/src/services/client/question.ts`）。

### 1.2 Controller 层：权限注入 + 前置校验

**入口**：`internal/controller/question_controller.go` L623-L705 `UpdateQuestion`。

关键步骤：

```go
// L631-L638：一次性检查 6 项权限，返回对应布尔数组 + 所需声望值
canList, requireRanks, err := qc.rankService.CheckOperationPermissionsForRanks(ctx, req.UserID, []string{
    permission.QuestionEdit,                // canList[0]
    permission.QuestionDelete,              // canList[1]
    permission.QuestionEditWithoutReview,   // canList[2]  免审核
    permission.TagUseReservedTag,           // canList[3]
    permission.TagAdd,                      // canList[4]
    permission.LinkUrlLimit,                // canList[5]
})

// L657：判断是否是对象所有者（问题作者）
objectOwner := qc.rankService.CheckOperationObjectOwner(ctx, req.UserID, req.ID)

// L658-L660：拼装到 req
req.CanEdit = canList[0] || objectOwner
req.NoNeedReview = canList[2] || objectOwner   // 核心分流点
req.CanUseReservedTag = canList[3]
```

**其他前置校验**：

- `UpdateQuestionCheckTags()`：校验标签变更（保留标签不能被删、新增保留标签需权限）。
- 检查是否有未完成审核中的修订（在 Service 层，见 1.3）。
- 新增标签权限校验。

**响应**：`UpdateQuestionResp{ UrlTitle, WaitForReview: !req.NoNeedReview }`。

### 1.3 Service 层：QuestionService.UpdateQuestion

**核心文件**：`internal/service/content/question_service.go` L906-L1085。

#### 步骤 1：校验是否已有待审核修订

```go
// L910-L917：如果该问题已有待审核修订，则禁止再次提交
_, existUnreviewed, err := qs.revisionService.ExistUnreviewedByObjectID(ctx, req.ID)
if existUnreviewed {
    return errors.BadRequest(reason.QuestionCannotUpdate)
}
```

对应 Repo 层查询：`internal/repo/revision/revision_repo.go` L143-L151 `ExistUnreviewedByObjectID` —— SQL 条件为 `status = 1(Unreviewed)`。

#### 步骤 2：校验内容是否真的变化

```go
// L971-L976：标题、正文、标签三者都没变 → 直接忽略
isChange := qs.tagCommon.CheckTagsIsChange(ctx, tagNameList, oldtagNameList)
if dbinfo.Title == req.Title && dbinfo.OriginalText == req.Content && !isChange {
    return
}
```

#### 步骤 3：权限分流（见下节第 2 部分）

根据 `req.NoNeedReview` 决定是直接写库还是仅创建待审核修订。

#### 步骤 4：创建 Revision 记录

构造 DTO：

```go
// L1026-L1031
revisionDTO := &schema.AddRevisionDTO{
    UserID:   question.UserID,     // 若需审核会被覆盖为 req.UserID
    ObjectID: question.ID,
    Title:    question.Title,
    Log:      req.EditSummary,     // 修改摘要
}
```

将问题 + 标签序列化为 JSON 存入 `Content`：

```go
// L1063-L1066
questionWithTagsRevision := qs.changeQuestionToRevision(ctx, question, Tags)
infoJSON, _ := json.Marshal(questionWithTagsRevision)
revisionDTO.Content = string(infoJSON)
revisionID, err := qs.revisionService.AddRevision(ctx, revisionDTO, true)
```

`autoUpdateRevisionID=true`：表示立即把新 revision ID 写到问题表的 `revision_id` 字段（免审核路径直接生效；待审核路径也会更新，但仅当审核通过后正式把内容覆盖）。Repo 层实现在 `internal/repo/revision/revision_repo.go` L57-L85 `AddRevision` —— 在同一个事务里先 `Insert(revision)` 再 `UpdateObjectRevisionId`。

#### 步骤 5：免审核路径的额外操作

- 更新问题标题、正文、解析文、`last_edit_user_id`、`post_update_time`。
- 同步标签变更（`tagCommon.ObjectChangeTag`）。
- 发送活动消息 `ActQuestionEdited` → 后续计入声望（见第 4.2 节）。
- 触发事件总线 `EventQuestionUpdate` 和向量同步。

---

## 2. 权限分流逻辑

权限分流的核心是 `req.NoNeedReview` 这个布尔值。

### 2.1 Controller 中的判定

**文件**：`internal/controller/question_controller.go` L657-L660

```go
objectOwner := qc.rankService.CheckOperationObjectOwner(ctx, req.UserID, req.ID)
req.NoNeedReview = canList[2] || objectOwner
// canList[2] 对应 permission.QuestionEditWithoutReview
```

即满足任一条件即免审核：

1. 用户拥有 `rank.question.edit_without_review` 权限（声望达标或角色授权）。
2. 用户是问题作者（`ObjectCreatorUserID == UserID`）。

### 2.2 RankService 的权限判定链路

**文件**：`internal/service/rank/rank_service.go` L87-L121 `CheckOperationPermission`

```
CheckOperationPermission(userID, action, objectID)
    │
    ├─ ① getUserPowerMapping() → 查角色-权限表（管理员/版主直接放行）
    │      （admin/moderator 在 role_power_rel 中通常配好了所有 action）
    │
    ├─ ② 若传了 objectID → GetInfo() 拿到 ObjectCreatorUserID
    │      → ObjectCreatorUserID == userID ? return true（作者豁免）
    │
    └─ ③ checkUserRank() → 从 config 表读 "rank."+action 对应所需声望
           → 用户当前 Rank >= 所需声望 ? return true
```

权限常量见：

- `internal/service/permission/permission_name.go` L25-L26：`QuestionEdit`、`QuestionEditWithoutReview`；L60：`QuestionAudit`。
- `internal/base/constant/privilege.go`：`rank.question.edit_without_review` 等 key。

### 2.3 Service 层的分流

**文件**：`internal/service/content/question_service.go` L1033-L1081。

```go
if req.NoNeedReview {
    canUpdate = true
}

if !canUpdate {
    // ===== 需审核路径 =====
    revisionDTO.Status = entity.RevisionUnreviewedStatus   // 1
    revisionDTO.UserID = req.UserID                        // 记录实际修改人
    // 不写 question 主表，不更新标签，不发送活动
} else {
    // ===== 免审核路径 =====
    revisionDTO.Status = entity.RevisionReviewPassStatus   // 2
    // 立即写 question 主表（title/original_text/parsed_text 等）
    qs.questionRepo.UpdateQuestion(ctx, question, [...]string{...})
    // 立即同步标签
    qs.tagCommon.ObjectChangeTag(ctx, &objectTagData, minimumTags)
    // 立即发送 ActQuestionEdited 活动（L1071-L1077）
}
```

**关键区别总结**：

| 环节 | 免审核（NoNeedReview=true） | 需审核（NoNeedReview=false） |
|------|---------------------------|----------------------------|
| revision.Status | `2(ReviewPass)` | `1(Unreviewed)` |
| question 主表 | 立即更新标题/正文 | 不变 |
| 标签关系 | 立即变更 | 不变 |
| revision_id | 更新 | 也更新（问题被修改） |
| 活动事件 | 立刻发送 ActQuestionEdited | 审核通过后发送 |
| 用户可见效果 | 修改立即生效 | 仍显示旧版本 |

### 2.4 审核者权限

审核修订的权限在 `internal/controller/revision_controller.go` 中每次接口调用时校验：

```go
// 获取待审核列表 / 执行审核操作时，都会检查以下三个权限
canList, _ := rc.rankService.CheckOperationPermissions(ctx, req.UserID, []string{
    permission.QuestionAudit,   // 审核问题
    permission.AnswerAudit,     // 审核回答
    permission.TagAudit,        // 审核标签
})
```

审核操作在 Service 内还会再做一次对象类型匹配校验（`internal/service/content/revision_service.go` L132-L148）：

- 问题修订 → 必须有 `CanReviewQuestion`。
- 回答修订 → 必须有 `CanReviewAnswer`。
- 标签修订 → 必须有 `CanReviewTag`。

---

## 3. 版本审核流程

```
审核列表 GET /answer/api/v1/revisions/unreviewed
    ↓
RevisionController.GetUnreviewedRevisionList
    ↓
RevisionService.GetUnreviewedRevisionPage()
    ↓ 组装 DiffContent 所需 newData/oldData
    ↓ 前端 SuggestContent 展示差异（第 4 节）

审核操作 PUT /answer/api/v1/revisions/audit
    ↓
RevisionController.RevisionAudit
    ↓
RevisionService.RevisionAudit()
    ├─ reject → 仅把 revision.Status 改为 3
    └─ approve → revisionAuditQuestion() → 写 question 主表 + 同步标签
                    → revision.Status 改为 2
                    → 发 edit.accepted 活动（声望）+ ActQuestionEdited 活动 + 通知
```

### 3.1 获取待审核修订列表

**Controller**：`internal/controller/revision_controller.go` L95-L117 `GetUnreviewedRevisionList`。

**Service**：`internal/service/content/revision_service.go` L338-L381 `GetUnreviewedRevisionPage`。

- 通过 `req.GetCanReviewObjectTypes()` 按审核权限过滤对象类型，只能看到自己有权审核的那类修订。
- 取数据库中 `status=1(Unreviewed)` 的 revision 记录（pageSize=1，一次只取一条，前端"跳过"按钮就是翻页）。
- 同时组装两块信息：
  - `info` → 当前线上版本（从 question/answer/tag 主表读取，作为 oldData）。
  - `unreviewed_info` → 待审核版本（从 revision.content 反序列化，作为 newData）。
- 返回结构 `GetUnreviewedRevisionResp{ Type, Info, UnreviewedInfo }`。

### 3.2 审核操作（Approve/Reject）

**入口**：`internal/controller/revision_controller.go` L128-L149 `RevisionAudit`。

**Service**：`internal/service/content/revision_service.go` L106-L180 `RevisionAudit`。

#### 拒绝（Reject）

```go
if req.Operation == schema.RevisionAuditReject {
    // L117-L119：仅把 revision.Status 置为 3（Reject），登记审核人
    err = rs.revisionRepo.UpdateStatus(ctx, req.ID, entity.RevisionReviewRejectStatus, req.UserID)
    return
}
```

问题主表不做任何改动，修订记录保留供追溯。Reject 不会产生任何活动记录。

#### 通过（Approve）— 以问题为例 `revisionAuditQuestion`

`internal/service/content/revision_service.go` L182-L233。

```go
func (rs *RevisionService) revisionAuditQuestion(ctx context.Context, revisionitem *schema.GetRevisionResp) {
    questioninfo := revisionitem.ContentParsed.(*schema.QuestionInfoResp)  // 反序列化的新版本

    // 1. 写 question 主表：title / original_text / parsed_text / updated_at / post_update_time / last_edit_user_id
    question := &entity.Question{
        ID: questioninfo.ID, Title: ..., OriginalText: ..., ParsedText: ...,
        LastEditUserID: revisionitem.UserID, ...
    }
    rs.questionRepo.UpdateQuestion(ctx, question, []string{...})

    // 2. 同步标签变更（会自动对比增删）
    objectTagData.Tags = objectTagTags  // 从修订的 tags 字段中提取
    rs.tagCommon.ObjectChangeTag(ctx, &objectTagData, minimumTags)

    // 3. 发送 "问题被编辑" 活动 → 后续加声望
    rs.activityQueueService.Send(ctx, &schema.ActivityMsg{
        ActivityTypeKey: constant.ActQuestionEdited,
        RevisionID: revisionitem.ID, ...
    })
}
```

通过审核的通用收尾（`RevisionAudit` L153-L176）：

```go
// ① 更新 revision 状态为 2（ReviewPass）
rs.revisionRepo.UpdateStatus(ctx, req.ID, entity.RevisionReviewPassStatus, req.UserID)

// ② 记录"审核通过"类型的活动（edit.accepted，见 4.2 节）
rs.reviewActivity.Review(ctx, &schema.PassReviewActivity{...})

// ③ 通知作者：您的修改已被采纳（成就类站内信）
rs.notificationQueueService.Send(ctx, &schema.NotificationMsg{
    Type: schema.NotificationTypeAchievement, ...
})
```

---

## 4. 变更展示

变更展示分三块：审核时的 Diff 对比、历史版本时间线、拒绝修订不外显。本节按代码精确对齐。

### 4.1 审核页面的 Diff 对比

**前端组件**：`ui/src/pages/Review/components/SuggestContent/index.tsx`。

**核心 Diff 组件**：`ui/src/components/DiffContent/index.tsx`。

#### 数据组装（SuggestContent L142-L180）

```ts
// oldData  = 当前线上版本（info 里的内容）
// newData  = 待审核修订（unreviewed_info.content 反序列化的内容）

if (type === 'question') {
  newData = { title: reviewInfo.title, original_text: reviewInfo.content, tags: reviewInfo.tags };
  oldData = { title: info.title,        original_text: info.content,        tags: info.tags };
}
```

#### Diff 渲染（DiffContent 组件，见 `ui/src/components/DiffContent/index.tsx`）

- **标题 Diff**：调用 `diffText(newTitle, oldTitle)`，把 `<` 转义后用 `dangerouslySetInnerHTML` 渲染。
- **标签 Diff**：分别计算 `addTags` 和 `deleteTags`；新增标签加 `state='add'`，删除加 `state='delete'`，并按原位置插入；CSS 类 `.review-text-add / .review-text-delete` 上色。
- **正文 Diff**：`diffText(newData.original_text, oldData?.original_text)` → 用 `pre-line font-monospace small` 逐字对比。

`diffText` 是一个基于经典 diff 算法的工具函数，输出带 `<ins>/<del>` 标签的 HTML。

#### 操作按钮

- **Approve** → `revisionAudit(id, 'approve')`，成功后自动加载下一条待审核。
- **Reject** → `revisionAudit(id, 'reject')`，同样加载下一条。
- **Skip** → `page+1` 跳过（不改变 revision 状态）。

### 4.2 主时间线：活动记录查询生成时间线

前端入口：`ui/src/pages/Timeline/index.tsx`。

前端调用链：`getTimelineData({ object_id, show_vote })`（`ui/src/services/client/timeline.ts`）→ `GET /answer/api/v1/activity/timeline`。

后端入口：`internal/controller/activity_controller.go` L53-L68 `GetObjectTimeline`。

后端 Service：`internal/service/activity/activity.go` L93-L165 `GetObjectTimeline`。

**核心结论：主时间线完全由 `activity` 表生成，列表阶段不查询 revision 表内容，revision.ID 仅作为引用字段保存在时间线项中。**

#### 步骤 1：查 activity 表

`GetObjectTimeline` L109 调用 `activityRepo.GetObjectAllActivity(ctx, req.ObjectID, req.ShowVote)`，Repo 层在 `internal/repo/activity/activity_repo.go` L52-L66：

```go
session := ar.data.DB.Context(ctx).Desc("id")
if !showVote {
    session.NotIn("activity_type", activityTypeNotShown)   // 排除 voted_up、voted_down 等投票类
}
err = session.Find(&activityList, &entity.Activity{OriginalObjectID: objectID})
```

- 查询条件：`OriginalObjectID = objectID`（是 `OriginalObjectID` 字段，不是 `ObjectID`）。
- 排序：`id DESC`（活动自增 ID 倒序，最新活动在前）。
- `show_vote=false` 时排除 `VoteActivityTypeList` 中的活动类型。

#### 步骤 2：每条 activity 组装为时间线项

`GetObjectTimeline` L113-L162 遍历 activityList，逐条构造 `ActObjectTimeline`（结构定义在 `internal/schema/activity.go` L51-L62）：

| 字段 | 来源 | 说明 |
|------|------|------|
| `RevisionID` | `converter.IntToString(act.RevisionID)` | 直接从 activity 表取，**不查 revision 表** |
| `ActivityType` | `strings.Cut(cfg.Key, ".")` 取后半段 → `formatActivity()` | 配置表 `config.id → cfg.Key`（如 42 → `question.edited` → `edited`） |
| `Comment` | `getTimelineActivityComment()` 按类型填充 | edited 类型时才会真正去查 revision 表（见下） |
| `UserInfo.ID` | `act.TriggerUserID`（优先）/ `act.UserID` | down vote 非管理员时 Username="N/A" |
| `Cancelled` | `act.Cancelled == entity.ActivityCancelled` | 被撤销的活动仍展示，但带取消标记 |

**隐藏过滤**：`formatActivity()`（`internal/service/activity/activity.go` L433-L449）对 `voted_up` / `voted_down` / `follow` 返回 `isHidden=true`，直接 `continue` 跳过，不出现在返回数组中。`vote_up` → `upvote`，`vote_down` → `downvote`，`accepted` → `accept`。

#### 步骤 3：Comment 字段按需读取 revision.Log（edit_summary）

`GetObjectTimeline` L160 调用 `getTimelineActivityComment()`（`internal/service/activity/activity.go` L199-L233），按类型分支：

- **`objectType == comment`** → 取 `commentCommonService.GetComment(objectID).ParsedText`。
- **`activityType == edited`** → 真正查 revision 表：`revisionService.GetRevision(ctx, revisionID)`，返回 `converter.Markdown2HTML(revision.Log)`（即用户填写的修改摘要，经 Markdown 转 HTML）。
- **`activityType == closed`** → 取 meta 表的关闭原因。
- 其他类型 → 空字符串。

**仅在 edited 活动上才会额外访问一次 revision 表**，其他活动类型在列表阶段不查 revision。

#### Activity 与 Revision 的关联规则

关联字段写入规则在 `internal/service/activity_common/activity.go` L84-L86 `HandleActivity`：

```go
if len(msg.RevisionID) > 0 {
    act.RevisionID = converter.StringToInt64(msg.RevisionID)
}
```

`ActivityMsg.RevisionID` 非空时才写入 `activity.RevisionID`。审核通过后实际上会产生两条 Activity，两条 Activity 各有用途、互不重复，**不会导致问题时间线出现两条修订展示**：

| 场景 | Activity Key | 写入方式 | OriginalObjectID | ObjectID | HasRank | RevisionID | 进入问题时间线？ | 作用 |
|------|--------------|----------|-------------------|----------|---------|------------|-----------------|------|
| 问题免审核编辑 | `question.edited` | `activityQueueService.Send` → `HandleActivity` | 问题 ID | 问题 ID | 0（HandleActivity 未设置，默认为 0） | 新 revision.ID | ✅ 是 | 时间线展示"edited"修订行，展开后按 revision_id 对比新旧版本 |
| 问题审核通过编辑 | `question.edited` | `activityQueueService.Send` → `HandleActivity` | 问题 ID | 问题 ID | 0 | 被审核的 revision.ID | ✅ 是 | 同上（和免审核编辑产生的是同一条时间线项，`activity_type = edited`，一条修订只出现一次） |
| 问题审核通过（声望） | `edit.accepted` | `reviewActivity.Review()` 直接写 DB（不走 HandleActivity） | **`"0"`** | 问题 ID | **`1`**，`Rank=config["edit.accepted"].Value=2` | 被审核的 revision.ID | ❌ 否 | 只用于给修改者加声望；因 `OriginalObjectID="0"`，时间线查询 `OriginalObjectID=问题ID` 时不会命中；即使命中也会被前端 `activity_type="accept"` 渲染为采纳链接而非修订展开按钮 |
| 提问 | `question.asked` | `activityQueueService.Send` → `HandleActivity` | 问题 ID | 问题 ID | 0 | 首个 revision.ID | ✅ 是 | 时间线展示"asked"行 |
| 投票/评论/关闭等 | 对应类型 | `activityQueueService.Send` → `HandleActivity` | 问题 ID / 回答 ID | 对象 ID | 视配置而定 | 无 | 视类型而定 | 与内容版本无关 |

**为什么审核通过后不会出现两条修订行：**

1. **`question.edited` 才是修订展示来源**：它的 `OriginalObjectID=问题ID`，会被 `GetObjectAllActivity` 命中；经 `strings.Cut("question.edited", ".")` 取后半段得到 `"edited"`，再经 `formatActivity` 原样返回（isHidden=false）；前端 `Item/index.tsx` L74-L90 中 `activity_type === 'edited'` 匹配展开按钮渲染条件，点击时按 `revision_id` 拉取 Diff 内容。
2. **`edit.accepted` 不进入时间线**：它的 `OriginalObjectID = "0"`（写死在 `revision_service.go` L161），而时间线查询条件是 `OriginalObjectID = objectID`（问题 ID），在 SQL 层就被过滤掉，根本不会出现在时间线返回数组里。
3. **`edit.accepted` 即使被查到也不会渲染成修订行**：假设绕过查询条件，其 `cfg.Key="edit.accepted"` 经 `strings.Cut` 后得到 `"accepted"`，`formatActivity("accepted")` 返回 `(false, "accept")`，前端 `Item/index.tsx` L91-L96 对 `activity_type === 'accept'` 会渲染为采纳链接（跳转到回答页），不在"有展开按钮的修订类型"列表（edited / asked / rollback / created / answered）中。

**两条 Activity 各自写入的字段对照**（`question.edited` vs `edit.accepted`）：

| 字段 | `question.edited`（HandleActivity） | `edit.accepted`（ReviewActivityRepo.Review） |
|------|-------------------------------------|---------------------------------------------|
| UserID | `msg.UserID`（修改人） | `act.UserID`（修改人） |
| TriggerUserID | `msg.TriggerUserID` | `act.TriggerUserID`（审核人） |
| ObjectID | `uid.DeShortID(msg.ObjectID)`（问题ID） | `act.ObjectID`（问题ID） |
| OriginalObjectID | `uid.DeShortID(msg.OriginalObjectID)`（**问题ID**） | `act.OriginalObjectID`（**"0"**，硬编码，写 review_service.go L161） |
| ActivityType | `config["question.edited"].ID` | `config["edit.accepted"].ID` |
| Rank | 未设置（0） | `config["edit.accepted"].Value = 2` |
| HasRank | 未设置（0） | **`1`** |
| RevisionID | `converter.StringToInt64(msg.RevisionID)` | `converter.StringToInt64(act.RevisionID)` |
| Cancelled | `ActivityAvailable`（0） | 未设置（0） |

**写入路径的区别**：

- `question.edited` 通过 **活动队列** `activityQueueService.Send(ctx, msg)` → 异步消费者 `ActivityCommon.HandleActivity` → `activityRepo.AddActivity` 入库（`internal/service/activity_common/activity.go` L68-L91）。
- `edit.accepted` **不走活动队列**，直接在 `RevisionAudit` 中同步调用 `reviewActivity.Review()`（`internal/repo/activity/review_repo.go` L69-L125），在同一个 DB 事务里完成用户 Rank 变更 + Activity 插入，并做了去重判断（`user_id + activity_type + revision_id` 三元组唯一）。

### 4.3 展开行：根据修订编号读新旧版本内容

主时间线返回的 `ActObjectTimeline.RevisionID` 是字符串，前端在点击展开时根据该编号查询详情。

#### 前端：计算新旧版本编号

`ui/src/pages/Timeline/index.tsx` L98-L99 先从返回的时间线里筛出带修订编号的子集：

```ts
const revisionList =
    timelineData?.timeline?.filter((item) => item.revision_id > 0) || [];
```

`ui/src/pages/Timeline/components/Item/index.tsx` L44-L63 `handleItemClick` 根据点击项在 revisionList 中的索引找相邻版本：

```ts
const revisionItem = revisionList?.find((v) => v.revision_id === id);
let oldId;
if (revisionList?.length > 0 && revisionItem) {
  const idIndex = revisionList.indexOf(revisionItem) || 0;
  if (idIndex === revisionList.length - 1) {
    oldId = 0;                       // 最后一项 = 最早版本，无旧版本可对比
  } else {
    oldId = revisionList[idIndex + 1].revision_id;   // 列表按时间倒序，下一项 = 更早的版本
  }
}
const res = await getTimelineDetail({ new_revision_id: id, old_revision_id: oldId });
```

- `new_revision_id` = 被点击行的 revision_id。
- `old_revision_id` = revisionList 中下一项的 revision_id（更早的版本）。如果是最早一项则为 `"0"`。

#### 后端：按修订编号直接读取 revision 内容

请求 `GET /answer/api/v1/activity/timeline/detail`（`ui/src/services/client/timeline.ts` `getTimelineDetail`）。

Controller：`internal/controller/activity_controller.go` L78-L89 `GetObjectTimelineDetail`。

Service：`internal/service/activity/activity.go` L261-L274 `GetObjectTimelineDetail`：

```go
// 校验新旧 revision 所属对象对当前用户的可见性（校验对象，不校验 revision.Status）
ensureTimelineRevisionVisible(ctx, req.NewRevisionID, req.UserID, req.IsAdminModerator)
ensureTimelineRevisionVisible(ctx, req.OldRevisionID, req.UserID, req.IsAdminModerator)
resp.OldRevision, _ = as.getOneObjectDetail(ctx, req.OldRevisionID)
resp.NewRevision, _ = as.getOneObjectDetail(ctx, req.NewRevisionID)
```

`ensureTimelineRevisionVisible`（L276-L286）逻辑：`revisionID == "0"` 直接通过；否则取 revision → 取 objectInfo → `validateTimelineObjectVisibility` 检查对象是否对该用户可见（已删除/未审核的问题，非作者非管理员不可见）。**该函数不校验 revision.Status。**

`getOneObjectDetail`（L372-L431）按 ID 读 revision 并反序列化快照：

```go
if revisionID == "0" {
    return nil, nil                              // "0" 表示无旧版本，返回 nil
}
revision, err := as.revisionService.GetRevision(ctx, revisionID)   // 按 ID 直接读，不过滤 status
objInfo, _ := as.objectInfoService.GetInfo(ctx, revision.ObjectID)
switch objInfo.ObjectType {
case constant.QuestionObjectType:
    data := &entity.QuestionWithTagsRevision{}
    json.Unmarshal([]byte(revision.Content), data)     // 反序列化 revision.Content 快照
    resp.Title = data.Title
    resp.OriginalText = data.OriginalText
    // ... 循环组装 Tags
case constant.AnswerObjectType:
    data := &entity.Answer{}
    json.Unmarshal([]byte(revision.Content), data)
    resp.OriginalText = data.OriginalText
case constant.TagObjectType:
    data := &entity.Tag{}
    json.Unmarshal([]byte(revision.Content), data)
    resp.Title = data.DisplayName
}
```

- `revisionID == "0"` → 返回 nil。
- `revisionService.GetRevision` 按 ID 直接读取，**不校验 revision.Status**。
- 按对象类型 `json.Unmarshal(revision.Content)` 反序列化当时的完整快照，提取 Title / OriginalText / Tags。
- 返回 `{NewRevision, OldRevision}`，前端用 `DiffContent` 组件渲染两个版本差异（与审核页面同一个组件）。

### 4.4 拒绝修订（Status=3）不外显

**结论：拒绝修订对外展示不可见。** 代码中有三条机制分别在不同入口保证这一点，数据库中拒绝修订记录（status=3）仍保留供追溯。

#### 机制 1：主时间线不出现（无 Activity）

主时间线来源于 `activity` 表。Reject 分支（`internal/service/content/revision_service.go` L117-L119）只调用 `UpdateStatus` 改状态为 3，**不发送任何 Activity**：

```go
if req.Operation == schema.RevisionAuditReject {
    err = rs.revisionRepo.UpdateStatus(ctx, req.ID, entity.RevisionReviewRejectStatus, req.UserID)
    return
}
```

没有对应 Activity → 时间线列表不会有该行 → 前端 `revisionList = timeline.filter(item => item.revision_id > 0)` 也不会包含该 revision_id → 点击展开时不会请求该修订的详情。

#### 机制 2：修订列表接口在控制器层过滤 status

`GET /answer/api/v1/revisions` 的 Controller（`internal/controller/revision_controller.go` L63-L84）在拿到 Service 全量结果后显式过滤：

```go
resp, err := rc.revisionListService.GetRevisionList(ctx, req)
list := make([]schema.GetRevisionResp, 0)
for _, item := range resp {
    if item.Status == entity.RevisionNormalStatus || item.Status == entity.RevisionReviewPassStatus {
        list = append(list, item)      // 只保留 status ∈ {0, 2}
    }
}
```

Repo 层 `GetRevisionList`（`internal/repo/revision/revision_repo.go` L175-L185）确实不过滤状态、全量返回，但 Controller 层只放行 `Normal(0)` 和 `ReviewPass(2)`，`Reject(3)` 和 `Unreviewed(1)` 都被剔除。

#### 机制 3：审核队列只查待审核状态

审核队列 `GET /answer/api/v1/revisions/unreviewed` 的 Repo 层 `GetUnreviewedRevisionPage`（`internal/repo/revision/revision_repo.go` L201-L218）只查 `status=1`（Unreviewed），拒绝（3）也不会再出现在审核队列。

#### 补充：详情端点本身不校验状态

需注意 `getOneObjectDetail`（见 4.3）按 ID 直接读 revision 内容，不校验 revision.Status。理论上知道拒绝修订的 ID 且对象可见时能读到内容，但因机制 1 保证拒绝修订的 ID 不会进入时间线列表，正常 UI 流程无法触达该端点读取拒绝修订内容。

---

## 5. 关键常量与状态速查

| 常量 / 状态 | 值 | 出处 |
|-------------|----|------|
| `RevisionNormalStatus` | 0 | `internal/entity/revision_entity.go` L28 |
| `RevisionUnreviewedStatus` | 1 | 同上 L30 |
| `RevisionReviewPassStatus` | 2 | 同上 L32 |
| `RevisionReviewRejectStatus` | 3 | 同上 L34 |
| `permission.QuestionEdit` | `"question.edit"` | `internal/service/permission/permission_name.go` L25 |
| `permission.QuestionEditWithoutReview` | `"question.edit_without_review"` | 同上 L26 |
| `permission.QuestionAudit` | `"question.audit"` | 同上 L60 |
| `constant.SuggestedPostEdit` | `"suggested_post_edit"` | `internal/base/constant/revision.go` L29 |
| `constant.ActQuestionEdited` | `"question.edited"` | `internal/base/constant/acticity.go` L51 |
| `constant.ActEdited` | `"edited"` | `internal/base/constant/acticity.go` L25 |
| `EditAccepted` | `"edit.accepted"` | `internal/repo/activity/review_repo.go` L50 |

---

## 6. 全流程一张图

```
                       ┌──────────────────────────┐
                       │   用户点击"编辑问题"      │
                       └────────────┬─────────────┘
                                    │
                                    ▼
                  ┌───────────────────────────────────┐
                  │ editCheck() → 检查是否有待审核修订│
                  └────────────┬──────────────────────┘
                               │通过
                               ▼
               ┌───────────────────────────────────────┐
               │ 用户提交 modifyQuestion(title,content,│
               │          tags, edit_summary, captcha)  │
               └───────────────┬───────────────────────┘
                               │
                               ▼
      ┌────────────────────────────────────────────────────┐
      │ Controller.UpdateQuestion                          │
      │  ① rank 检查 6 项权限                               │
      │  ② objectOwner = 是否作者                          │
      │  ③ NoNeedReview = EditWithoutReview ∥ objectOwner │
      └───────────────────────┬────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
  NoNeedReview=true (免审核)         NoNeedReview=false (需审核)
              │                               │
              ▼                               ▼
  ① 写 question 主表(title/text)    ① 不写主表，仅创建 revision
  ② 同步标签变更                    ② revision.Status = 1 (Unreviewed)
  ③ revision.Status = 2 (Pass)      ③ 前端提示"等待审核"
  ④ 发送 ActQuestionEdited
  ⑤ 问题立即展示新版本              ┌─────────────────────┐
              │                    │ 审核人浏览审核队列  │
              │                    │ GET /revisions/     │
              │                    │     unreviewed      │
              │                    └──────────┬──────────┘
              │                               │ DiffContent
              │                               ▼
              │                    ┌──────────────────────┐
              │                    │ Approve  │  Reject   │
              │                    └────┬─────┴─────┬─────┘
              │                         │           │
              │                         ▼           ▼
              │            revisionAuditQuestion()  Status=3(Reject)
              │              ① 写 question 主表     (不产生 Activity)
              │              ② 同步标签             → 时间线不出现
              │              ③ revision.Status = 2
              │              ④ 活动 (edit.accepted + 声望)
              │              ⑤ ActQuestionEdited 活动
              │              ⑥ 通知作者
              └────────────────────────►│
                                        ▼
                          ┌─────────────────────────┐
                          │ 问题详情展示最新版本    │
                          │ Timeline 活动列表       │
                          │ (Activity 表生成)       │
                          │ 展开→按 revision_id 读 │
                          └─────────────────────────┘
```
