# 已发布问题修改代码实现全流程梳理

本文档按 **提交 → 权限分流 → 版本审核 → 变更展示** 的顺序梳理已发布问题（Question）修改在代码中的完整实现链路。

---

## 0. 核心数据模型

### 0.1 修订（Revision）实体

定义文件：[revision_entity.go](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/entity/revision_entity.go)

| 字段 | 说明 |
|------|------|
| `ObjectType` | 对象类型：Question(1) / Answer(2) / Tag(3)，映射见 `constant.ObjectTypeStrMapping` |
| `ObjectID` | 被修改对象的 ID |
| `Title` / `Content` | 修改后的标题 / 内容（Content 存 JSON 序列化后的完整对象快照） |
| `Log` | 修改摘要（edit_summary） |
| `Status` | 修订状态：`0=Normal` / `1=Unreviewed` / `2=ReviewPass` / `3=ReviewReject` |
| `UserID` | 提交修改的用户 ID |
| `ReviewUserID` | 审核人 ID |

### 0.2 审核（Review）实体

定义文件：[review_entity.go](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/entity/review_entity.go)

> Review 用于**新发布内容**的审核（新问题、新回答），**问题修改走的是 Revision 机制**，两者分离。

| 字段 | 说明 |
|------|------|
| `ObjectType` / `ObjectID` | 同 Revision |
| `Status` | `1=Pending` / `2=Approved` / `3=Rejected` |
| `Submitter` | 触发审核的插件 Slug |
| `Reason` | 插件给出的审核理由 |

---

## 1. 提交流程（用户提交修改）

```
前端编辑页(Ask/index.tsx)
    ↓ modifyQuestion()
Controller: UpdateQuestion [PUT /answer/api/v1/question]
    ↓ rankService.CheckOperationPermissionsForRanks()
    ↓ 权限分流（第2节）
Service: QuestionService.UpdateQuestion()
    ↓ 构造 AddRevisionDTO
    ↓ revisionService.AddRevision()
    ↓ （若免审核）直接写库 + 更新标签
```

### 1.1 前端：编辑提交

**入口页面**：[ui/src/pages/Questions/Ask/index.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/ui/src/pages/Questions/Ask/index.tsx#L304-L332)

- 通过 URL 参数 `qid` 判断是编辑模式（`isEdit = qid !== undefined`）。
- 编辑时额外渲染 `edit_summary` 字段（修改摘要）。
- 提交函数 `submitModifyQuestion()` 调用 `modifyQuestion(ep)`，其中 `ep` 包含：
  - `id`, `title`, `content`, `tags`, `edit_summary`, `captcha_code`, `captcha_id`
- 响应中 `wait_for_review=true` 时，跳转后携带 `state.isReview` 给用户提示等待审核。

**API 层**：服务方法 `modifyQuestion` → `PUT /answer/api/v1/question`

### 1.2 Controller 层：权限注入 + 前置校验

**入口**：[question_controller.go UpdateQuestion](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/controller/question_controller.go#L623-L705)

关键步骤：

```go
// L631-L638：一次性检查6项权限，返回对应布尔数组 + 所需声望值
canList, requireRanks, err := qc.rankService.CheckOperationPermissionsForRanks(ctx, req.UserID, []string{
    permission.QuestionEdit,                // canList[0]
    permission.QuestionDelete,              // canList[1]
    permission.QuestionEditWithoutReview,   // canList[2]  ★ 免审核
    permission.TagUseReservedTag,           // canList[3]
    permission.TagAdd,                      // canList[4]
    permission.LinkUrlLimit,                // canList[5]
})

// L657：判断是否是对象所有者（问题作者）
objectOwner := qc.rankService.CheckOperationObjectOwner(ctx, req.UserID, req.ID)

// L658-L660：拼装到 req
req.CanEdit = canList[0] || objectOwner
req.NoNeedReview = canList[2] || objectOwner   // ★ 核心分流点
req.CanUseReservedTag = canList[3]
```

**其他前置校验**：
- `UpdateQuestionCheckTags()`：校验标签变更（保留标签不能被删、新增保留标签需权限）。
- 检查是否有**未完成审核中的修订**（在 Service 层，见 1.3）。
- 新增标签权限校验。

**响应**：`UpdateQuestionResp{ UrlTitle, WaitForReview: !req.NoNeedReview }`

### 1.3 Service 层：QuestionService.UpdateQuestion

**核心文件**：[question_service.go UpdateQuestion](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/service/content/question_service.go#L906-L1085)

#### 步骤 1：校验是否已有待审核修订

```go
// L910-L917：如果该问题已有待审核修订，则禁止再次提交
_, existUnreviewed, err := qs.revisionService.ExistUnreviewedByObjectID(ctx, req.ID)
if existUnreviewed {
    return errors.BadRequest(reason.QuestionCannotUpdate)
}
```

#### 步骤 2：校验内容是否真的变化

```go
// L971-L976：标题、正文、标签三者都没变 → 直接忽略
isChange := qs.tagCommon.CheckTagsIsChange(ctx, tagNameList, oldtagNameList)
if dbinfo.Title == req.Title && dbinfo.OriginalText == req.Content && !isChange {
    return
}
```

#### 步骤 3：权限分流（见下节第2部分）

根据 `req.NoNeedReview` 决定是**直接写库**还是**仅创建待审核修订**。

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

`autoUpdateRevisionID=true`：表示**立即**把新 revision ID 写到问题表的 `revision_id` 字段（免审核路径直接生效；待审核路径也会更新，仅当审核通过后正式把内容覆盖）。

#### 步骤 5：免审核路径的额外操作

- 更新问题标题、正文、解析文、`last_edit_user_id`、`post_update_time`。
- 同步标签变更（`tagCommon.ObjectChangeTag`）。
- 发送活动消息 `ActQuestionEdited` → 后续计入声望。
- 触发事件总线 `EventQuestionUpdate` 和向量同步。

---

## 2. 权限分流逻辑

权限分流的核心是 `req.NoNeedReview` 这个布尔值。

### 2.1 Controller 中的判定

**文件**：[question_controller.go L657-L660](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/controller/question_controller.go#L657-L660)

```go
objectOwner := qc.rankService.CheckOperationObjectOwner(ctx, req.UserID, req.ID)
req.NoNeedReview = canList[2] || objectOwner
// canList[2] 对应 permission.QuestionEditWithoutReview
```

即满足**任一**条件即免审核：
1. 用户拥有 `rank.question.edit_without_review` 权限（声望达标或角色授权）。
2. 用户是问题作者（`ObjectCreatorUserID == UserID`）。

### 2.2 RankService 的权限判定链路

**文件**：[rank_service.go CheckOperationPermission](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/service/rank/rank_service.go#L87-L121)

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

所以 `QuestionEditWithoutReview` 的通过路径：
- 角色权限（Admin/Moderator 角色被授予该 power）。
- 或者 **声望值 ≥ 系统配置的阈值**（默认值在 config 表中配置）。

### 2.3 Service 层的分流

**文件**：[question_service.go L1033-L1061](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/service/content/question_service.go#L1033-L1061)

```go
if req.NoNeedReview {
    canUpdate = true
}

if !canUpdate {
    // ===== 需审核路径 =====
    revisionDTO.Status = entity.RevisionUnreviewedStatus   // 1
    revisionDTO.UserID = req.UserID                        // 记录实际修改人
    // 不写 question 主表，不更新标签
} else {
    // ===== 免审核路径 =====
    revisionDTO.Status = entity.RevisionReviewPassStatus   // 2
    // 立即写 question 主表（title/original_text/parsed_text 等）
    qs.questionRepo.UpdateQuestion(ctx, question, [...]string{...})
    // 立即同步标签
    qs.tagCommon.ObjectChangeTag(ctx, &objectTagData, minimumTags)
}
```

**关键区别总结**：

| 环节 | 免审核（NoNeedReview=true） | 需审核（NoNeedReview=false） |
|------|---------------------------|----------------------------|
| revision.Status | `2(ReviewPass)` | `1(Unreviewed)` |
| question 主表 | 立即更新标题/正文 | **不变** |
| 标签关系 | 立即变更 | **不变** |
| revision_id | 更新 | 也更新 |
| 活动事件 | 立刻发送 | 审核通过后发送 |
| 用户可见效果 | 修改立即生效 | 仍显示旧版本 |

### 2.4 审核者权限

审核修订的权限在 [revision_controller.go](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/controller/revision_controller.go) 中每次接口调用时校验：

```go
// 获取待审核列表 / 执行审核操作时，都会检查以下三个权限
canList, _ := rc.rankService.CheckOperationPermissions(ctx, req.UserID, []string{
    permission.QuestionAudit,   // 审核问题
    permission.AnswerAudit,     // 审核回答
    permission.TagAudit,        // 审核标签
})
```

审核操作在 Service 内还会再做一次**对象类型匹配校验**：
- 问题修订 → 必须有 `CanReviewQuestion`（见 `revisionAudit` L132）。
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
    ↓ 前端 SuggestContent 展示差异（第4节）

审核操作 PUT /answer/api/v1/revisions/audit
    ↓
RevisionController.RevisionAudit
    ↓
RevisionService.RevisionAudit()
    ├─ reject → 仅把 revision.Status 改为 3
    └─ approve → revisionAuditQuestion() → 写 question 主表 + 同步标签
                    → revision.Status 改为 2
                    → 发活动、通知
```

### 3.1 获取待审核修订列表

**Controller**：[revision_controller.go GetUnreviewedRevisionList](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/controller/revision_controller.go#L95-L117)

**Service**：[revision_service.go GetUnreviewedRevisionPage](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/service/content/revision_service.go#L338-L381)

- 通过 `req.GetCanReviewObjectTypes()` 按审核权限**过滤对象类型**，只能看到自己有权审核的那类修订。
- 取数据库中 `status=1(Unreviewed)` 的 revision 记录（pageSize=1，**一次只取一条**，前端"跳过"按钮就是翻页）。
- 同时组装两块信息：
  - `info` → 当前**线上版本**（从 question/answer/tag 主表读取，作为 oldData）。
  - `unreviewed_info` → **待审核版本**（从 revision.content 反序列化，作为 newData）。
- 返回结构 `GetUnreviewedRevisionResp{ Type, Info, UnreviewedInfo }`。

### 3.2 审核操作（Approve/Reject）

**入口**：[revision_controller.go RevisionAudit](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/controller/revision_controller.go#L128-L149)

**Service**：[revision_service.go RevisionAudit](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/service/content/revision_service.go#L106-L180)

#### 拒绝（Reject）

```go
if req.Operation == schema.RevisionAuditReject {
    // L117-L119：仅把 revision.Status 置为 3（Reject），登记审核人
    err = rs.revisionRepo.UpdateStatus(ctx, req.ID, entity.RevisionReviewRejectStatus, req.UserID)
    return
}
```

问题主表**不做任何改动**，修订记录保留供追溯。

#### 通过（Approve）— 以问题为例 `revisionAuditQuestion`

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

// ② 记录"审核通过"类型的活动（供作者端时间线展示）
rs.reviewActivity.Review(ctx, &schema.PassReviewActivity{...})

// ③ 通知作者：您的修改已被采纳（成就类站内信）
rs.notificationQueueService.Send(ctx, &schema.NotificationMsg{
    Type: schema.NotificationTypeAchievement, ...
})
```

---

## 4. 变更展示

变更展示分两块：**审核时的 Diff 对比** 和 **历史版本时间线**。

### 4.1 审核页面的 Diff 展示

**前端组件**：[ui/src/pages/Review/components/SuggestContent/index.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/ui/src/pages/Review/components/SuggestContent/index.tsx)

**核心 Diff 组件**：[ui/src/components/DiffContent/index.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/ui/src/components/DiffContent/index.tsx)

#### 数据组装（SuggestContent L142-L180）

```ts
// oldData  = 当前线上版本（info 里的内容）
// newData  = 待审核修订（unreviewed_info.content 反序列化的内容）

if (type === 'question') {
  newData = { title: reviewInfo.title, original_text: reviewInfo.content, tags: reviewInfo.tags };
  oldData = { title: info.title,        original_text: info.content,        tags: info.tags };
}
```

#### Diff 渲染（DiffContent 组件）

- **标题 Diff**：调用 `diffText(newTitle, oldTitle)`，把 `<` 转义后用 `dangerouslySetInnerHTML` 渲染。
- **标签 Diff**：分别计算 `addTags` 和 `deleteTags`；新增标签加 `state='add'`，删除加 `state='delete'`，并按原位置插入；CSS 类 `.review-text-add / .review-text-delete` 上色。
- **正文 Diff**：`diffText(newData.original_text, oldData?.original_text)` → 用 `pre-line font-monospace small` 逐字对比。

`diffText` 是一个基于经典 diff 算法的工具函数，输出带 `<ins>/<del>` 标签的 HTML。

#### 操作按钮

- **Approve** → `revisionAudit(id, 'approve')`，成功后自动加载下一条待审核。
- **Reject** → `revisionAudit(id, 'reject')`，同样加载下一条。
- **Skip** → `page+1` 跳过（不改变 revision 状态）。

### 4.2 历史版本时间线

**页面**：[ui/src/pages/Timeline/index.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/ui/src/pages/Timeline/index.tsx)

时间线数据来自 `getTimelineData()`，后端会把：
- **revision 列表**（仅 `status=0 或 2`，即正常版本或审核通过版本）
- **活动记录**（提问、编辑、投票、关闭等）

合并成一条时间线按时间倒序展示。

**后端获取历史修订列表接口**：

- [revision_controller.go GetRevisionList](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/controller/revision_controller.go#L63-L84)
- 调用 `RevisionService.GetRevisionList()` → `parseItem()` 反序列化每个 revision.content 还原对应版本快照。
- 前端过滤只展示 `status in {Normal, ReviewPass}`，拒绝的修订不对外可见。

**问题详情页的提示**：
- 提交修改后若 `WaitForReview=true`，通过路由 state `isReview` 告知用户"修改正在审核中"。
- 若问题已有待审核修订，`CheckCanUpdateRevision` 接口会返回 toast 提示，阻止再次进入编辑页。

---

## 5. 关键常量与状态速查

| 常量 / 状态 | 值 | 出处 |
|-------------|----|------|
| `RevisionNormalStatus` | 0 | [revision_entity.go L28](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/entity/revision_entity.go#L28) |
| `RevisionUnreviewedStatus` | 1 | 同上 L30 |
| `RevisionReviewPassStatus` | 2 | 同上 L32 |
| `RevisionReviewRejectStatus` | 3 | 同上 L34 |
| `permission.QuestionEdit` | `"question.edit"` | [permission_name.go L25](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/service/permission/permission_name.go#L25) |
| `permission.QuestionEditWithoutReview` | `"question.edit_without_review"` | 同上 L26 |
| `permission.QuestionAudit` | `"question.audit"` | 同上 L60 |
| `constant.SuggestedPostEdit` | `"suggested_post_edit"` | [revision.go L29](file:///d:/fz/0601-2/solo-dogfeeding/code/36-answer/internal/base/constant/revision.go#L29) |

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
              │              ④ 活动 + 通知作者
              └────────────────────────►│
                                        ▼
                          ┌─────────────────────────┐
                          │ 问题详情展示最新版本     │
                          │ Timeline 列出历史修订   │
                          └─────────────────────────┘
```
