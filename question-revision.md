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

> Review 用于**新发布内容**的审核（新问题、新回答），**问题修改走的是 Revision 机制**，两者分离。Review 表只处理 `1=Pending` / `2=Approved` / `3=Rejected` 状态。

### 0.3 活动（Activity）实体

定义：`internal/entity/activity_entity.go` L30-L43

| 字段 | 说明 |
|------|------|
| `ActivityType` | 活动类型（int，映射自 `config` 表中 `question.edited` 等 key 的 ID） |
| `ObjectID` / `OriginalObjectID` | 关联对象 ID |
| `RevisionID` | **关联到 Revision.ID（int64），Activity 与 Revision 的关联字段** |
| `UserID` / `TriggerUserID` | 对象作者 / 触发人（例如编辑人） |
| `Cancelled` | 是否已被撤销（0=有效，1=已撤销） |

Revision 与 Activity 是 **1:N 关系：一个 Revision 可能关联 0~2 条 Activity（见第 4.2 节）。

---

## 1. 提交流程（用户提交修改）

```
前端编辑页 (ui/src/pages/Questions/Ask/index.tsx
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
    permission.QuestionEditWithoutReview, // canList[2]  ★ 免审核
    permission.TagUseReservedTag,             // canList[3]
    permission.TagAdd,                      // canList[4]
    permission.LinkUrlLimit,              // canList[5]
})

// L657：判断是否是对象所有者（问题作者）
objectOwner := qc.rankService.CheckOperationObjectOwner(ctx, req.UserID, req.ID)

// L658-L660：拼装到 req
req.CanEdit = canList[0] || objectOwner
req.NoNeedReview = canList[2] || objectOwner   // ★ 核心分流点
req.CanUseReservedTag = canList[3]
```

**其他前置校验**：

- `UpdateQuestionCheckTags()`：校验标签变更（保留标签不能被删、新增保留标签需权限。
- 检查是否有**未完成审核中的修订**（在 Service 层，见 1.3）。
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

根据 `req.NoNeedReview` 决定是**直接写库**还是**仅创建待审核修订**。

#### 步骤 4：创建 Revision 记录

构造 DTO：

```go
// L1026-L1031
revisionDTO := &schema.AddRevisionDTO{
    UserID:   question.UserID,     // 若需审核会被覆盖为 req.UserID
    ObjectID: question.ID,
    Title:    question.Title,
    Log:      req.EditSummary,  // 修改摘要
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

`autoUpdateRevisionID=true`：表示**立即**把新 revision ID 写到问题表的 `revision_id` 字段（免审核路径直接生效；待审核路径也会更新，但仅当审核通过后正式把内容覆盖。Repo 层实现在 `internal/repo/revision/revision_repo.go` L57-L85 `AddRevision` —— 在同一个事务里先 `Insert(revision)` 再 `UpdateObjectRevisionId`。

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

即满足**任一**条件即免审核：

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
| question 主表 | 立即更新标题/正文 | **不变** |
| 标签关系 | 立即变更 | **不变** |
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

审核操作在 Service 内还会再做一次**对象类型匹配校验`internal/service/content/revision_service.go` L132-L148：

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

- 通过 `req.GetCanReviewObjectTypes()` 按审核权限**过滤对象类型，只能看到自己有权审核的那类修订。
- 取数据库中 `status=1(Unreviewed)` 的 revision 记录（pageSize=1，**一次只取一条，前端"跳过"按钮就是翻页）。
- 同时组装两块信息：
  - `info` → 当前**线上版本**（从 question/answer/tag 主表读取，作为 oldData）。
  - `unreviewed_info` → **待审核版本**（从 revision.content 反序列化，作为 newData）。
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

问题主表**不做任何改动，修订记录保留供追溯。**Reject ** 不会产生任何活动记录。

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

// ② 记录"审核通过"类型的活动（edit.accepted"，见 4.2 节）
rs.reviewActivity.Review(ctx, &schema.PassReviewActivity{...})

// ③ 通知作者：您的修改已被采纳（成就类站内信）
rs.notificationQueueService.Send(ctx, &schema.NotificationMsg{
    Type: schema.NotificationTypeAchievement, ...
})
```

---

## 4. 变更展示

变更展示分两块：**审核时的 Diff 对比** 和 **历史版本时间线**。本节按代码详细说明。

### 4.1 审核页面的 Diff 展示

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

### 4.2 历史版本时间线

#### 时间线数据来源

时间线页面：`ui/src/pages/Timeline/index.tsx`。

前端通过 `getTimelineData()` → `GET /answer/api/v1/activity/timeline`（`ui/src/services/client/timeline.ts`）。

后端入口：`internal/controller/activity_controller.go` L53-L68 `GetObjectTimeline`。

**核心 Service**：`internal/service/activity/activity.go` L93-L165 `GetObjectTimeline`。

时间线数据有两个来源，按 ID 倒序合并展示：

**来源 1：Activity 表（活动记录）**

- 通过 `activityRepo.GetObjectAllActivity(ctx, req.ObjectID, req.ShowVote)`（`internal/repo/activity/activity_repo.go` L52-L66）。SQL 按 `id DESC` 查 `activity` 表。`OriginalObjectID = objectID` 的全部活动。

- 活动类型映射（`internal/base/constant/acticity.go`）：
  - `question.asked`（提问）
  - `question.edited`（编辑）
  - `question.closed` / `question.reopened`
  - `question.answered`（有新回答）
  - `question.accept`（采纳回答）
  - `question.upvote` / `question.downvote`（投票）
  - `question.rollback`（回滚）
  - `question.deleted` / `question.undeleted`
  - `question.pin` / `question.unpin`
  - 以及回答、标签的对应活动。

格式化为前端可读字符串（`internal/service/activity/activity.go` L433-L449 `formatActivity`）：
- `vote_up` → `upvote`
- `vote_down` → `downvote`
- `accepted` → `accept`
- `voted_up` / `voted_down` / `follow` → **隐藏（isHidden=true，L434-L438）。

**来源 2：Revision 表（修订详情）**

时间线本身**不是直接查 Revision 表，而是通过 Activity 的 `RevisionID` 字段关联到 Revision。

`ActObjectTimeline` 结构（`internal/schema/activity.go` L51-L62）：

```go
type ActObjectTimeline struct {
    ActivityID   string         `json:"activity_id"`
    RevisionID   string         `json:"revision_id"`  // ★ 关联到 revision.id
    ActivityType string         `json:"activity_type"`
    Comment      string         `json:"comment"`
    ...
}
```

Activity 和 Revision 的关系（**不是所有 Activity 都关联 Revision。关联规则（`internal/service/activity_common/activity.go` L69-L90 `HandleActivity`）：

```go
// 当 ActivityMsg.RevisionID 非空时才写入 activity.RevisionID
if len(msg.RevisionID) > 0 {
    act.RevisionID = converter.StringToInt64(msg.RevisionID)
}
```

所以：

| 场景 | Activity 类型 | 是否带 RevisionID | 说明 |
|------|-----------|-------------------|------|
| 问题免审核编辑 | `question.edited` | ✅ 是（`question_service.go` L1071-L1077） | 立即发送，`RevisionID=新创建的 revision.ID |
| 问题审核通过编辑 | `question.edited` | ✅ 是（`revision_service.go` L224-L230） | 审核通过后发送，RevisionID=被审核的 revision.ID |
| 问题审核通过（声望） | `edit.accepted` | ✅ 是（`repo/activity/review_repo.go` L68-L125） | 单独一条 Activity，给修改者加声望 |
| 提问 `question.asked` | ✅ 是（创建问题时也会产生首个 revision） |
| 投票/评论/关闭等 | ❌ 否 | 与内容版本无关，不关联 revision |

一个"审核通过"的编辑会产生**两条 Activity**（`question.edited`（用于时间线展示内容变化，另一条 `edit.accepted`（用于给修改者加声望，HasRank=1）。两条 Activity 都带同一个 RevisionID。

#### 时间线评论区 Comment 字段来源：`internal/service/activity/activity.go` L199-L233 `getTimelineActivityComment`：

```go
if activityType == constant.ActEdited {   // "edited"
    revision, err := as.revisionService.GetRevision(ctx, revisionID)
    // 返回 revision.Log（用户填写的修改摘要）Markdown2HTML(revision.Log)
}
```

即时间线中"edited"那一行的评论文字就是 revision.Log（edit_summary）。

#### 时间线详情接口（点击时间线某一行展开）：`GET /answer/api/v1/activity/timeline/detail`。

**入口**：`internal/controller/activity_controller.go` L78-L89 `GetObjectTimelineDetail`。

**Service**：`internal/service/activity/activity.go` L261-L274 `GetObjectTimelineDetail`。

参数 `new_revision_id` 和 `old_revision_id` 分别反序列化出两个 Revision.Content 字段，返回 `{NewRevision, OldRevision}`。

前端 `ui/src/pages/Timeline/components/Item/index.tsx` L44-L63 `handleItemClick`：

```ts
// revisionList = timeline.filter(item => item.revision_id > 0)
// 找点击项在 revisionList 中的位置 idIndex
// oldId = revisionList[idIndex + 1].revision_id（即前一个版本）
// 第一个版本 oldId = 0
getTimelineDetail({ new_revision_id, old_revision_id: oldId })
```

然后用 `DiffContent` 组件渲染两个版本差异（同审核页面同一个 DiffContent`）。

### 4.3 拒绝修订（Status=3）是否展示？

**结论：拒绝修订在对外不展示。**

核查代码确认：

1. **时间线接口（public revision 列表接口（public revision list endpoint`internal/service/content/revision_service.go` L383-L427 `GetRevisionList` → 调用 `revisionRepo.GetRevisionList`。

2. **Repo 层 `internal/repo/revision/revision_repo.go` L175-L185：

```go
func (rr *revisionRepo) GetRevisionList(ctx context.Context, revision *entity.Revision) (revisionList []entity.Revision, err error) {
    revisionList = []entity.Revision{}
    err = rr.data.DB.Context(ctx).Where(builder.Eq{
        "object_id": revision.ObjectID,
    }).OrderBy("created_at DESC").Find(&revisionList)
}
```

Repo 层**没有过滤 Status 过滤**，全量返回。但上层 Service `parseItem()` 反序列化 Content 时**不区分状态，Status 字段被序列化为 JSON（`internal/schema/revision_schema.go` L93-L108 `GetRevisionResp.Status`）。

3. **时间线接口走 Activity，而拒绝修订不会出现在时间线上，因为：

   - 拒绝修订**不会写入 Activity**（`revision_service.go` L117-L119 Reject 分支只 `UpdateStatus`，不发送 Activity。
   - 没有 Activity → 时间线不会出现这行。

4. **审核队列**（`/revisions/unreviewed）只查 `status=1`（Unreviewed）（`internal/repo/revision/revision_repo.go` L201-L218 `GetUnreviewedRevisionPage`），拒绝的修订（status=3）不会出现在审核队列。

5. **前端**：`ui/src/pages/Timeline/index.tsx` L98-L99 `revisionList = timeline.filter(item => item.revision_id > 0) —— 只过滤出有 revision_id 的时间线项（即只有 Activity 关联到的 Revision。

**总结：拒绝修订**不会在公开历史中不可见，但数据库里保留记录（status=3），不会出现在任何公开页面。

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
                       │   用户点击"编辑问题"     │
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
               │          tags, edit_summary, captcha) │
               └───────────────┬───────────────────────┘
                               │
                               ▼
      ┌────────────────────────────────────────────────────┐
      │ Controller.UpdateQuestion                         │
      │  ① rank 检查 6 项权限                              │
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
  ③ revision.Status = 2 (Pass)     ③ 前端提示"等待审核"
  ④ 发送 ActQuestionEdited
  ⑤ 问题立即展示新版本              ┌─────────────────────┐
              │                    │ 审核人浏览审核队列  │
              │                    │ GET /revisions/     │
              │                    │     unreviewed      │
              │                    └──────────┬──────────┘
              │                               │ DiffContent
              │                               ▼
              │                    ┌──────────────────────┐
              │                    │ Approve  │  Reject    │
              │                    └────┬─────┴─────┬──────┘
              │                         │           │
              │                         ▼           ▼
              │            revisionAuditQuestion()  Status=3(Reject)
              │              ① 写 question 主表
              │              ② 同步标签
              │              ③ revision.Status = 2
              │              ④ 活动（edit.accepted + 声望)
              │              ⑤ ActQuestionEdited 活动
              │              ⑥ 通知作者
              └────────────────────────►│
                                        ▼
                          ┌─────────────────────────┐
                          │ 问题详情展示最新版本     │
                          │ Timeline 列出历史修订（经 Activity → Revision│
                          └─────────────────────────┘
```
