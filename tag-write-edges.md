# 标签写入路径边缘场景分析

## 一、同义词在写入路径上如何收敛到主标签

### 1.1 结论先行

**`ObjectChangeTag` 本身不做任何同义词到主标签的 tag_id 改写。** 同义词的收敛完全依赖前端 `TagSelector` 组件调用的搜索接口 `SearchTagLike` —— 该接口在返回结果时已将同义词标签替换为主标签。

换句话说：**写入侧是"盲"的，它相信前端传进来的 SlugName 对应的 tag_id 就是最终要关联的 tag_id**。前端选到的是什么，后端就存什么。

### 1.2 收敛链路全景

```
用户在 TagSelector 输入 "golang"（假设 "go" 是 "golang" 的同义词，"编程语言" 是主标签）
    │
    ▼
fetchTags("golang")
    │
    ▼
GET /answer/api/v1/question/tags?tag=golang
    │
    ▼
SearchTagLike() [tag_common.go#L114-L165]
    │
    ├─ 按关键词模糊查询 tag 表，匹配到同义词 golang
    ├─ 遍历结果，对每个 tag.MainTagID != 0 的结果：
    │     │
    │     ├─ 查主标签 "编程语言"
    │     └─ 替换 tag.ID = 主标签.ID, tag.SlugName = 主标签.SlugName, ...
    │
    └─ 按主标签 SlugName 去重（repetitiveTag map）
    │
    ▼
前端收到候选列表，显示的都是主标签信息
    │
    ▼
用户选中 → onChange([{SlugName: "编程语言", ID: "12345", ...}])
    │
    ▼
提交表单时发送的 tags: [{SlugName: "编程语言", ...}]
    │
    ▼
ObjectChangeTag({ObjectID: questionID, Tags: [{SlugName: "编程语言"}], ...})
    │
    ├─ GetTagListByNames(["编程语言"]) → 查到主标签
    └─ CreateOrUpdateTagRelList(questionID, [主标签ID])
    │
    ▼
tag_rel 表存储的是主标签 ID，完成收敛
```

### 1.3 CheckTag() 的防御性校验

虽然写入侧不主动改写，但有一道**被动防御校验**：`CheckTag()` [tag_common.go#L547-L597](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L547-L597)

```go
func (ts *TagCommonService) CheckTag(ctx context.Context, tags []string, userID string) error {
    tagListInDb, err := ts.GetTagListByNames(ctx, tags)
    
    checktags := make([]string, 0)
    for _, tag := range tagListInDb {
        if tag.MainTagID != 0 {
            // 发现入参中包含同义词标签 → 直接报错
            checktags = append(checktags, fmt.Sprintf("\"%s\"", tag.SlugName))
        }
    }
    if len(checktags) > 0 {
        // "Should not contain synonym tags \"golang\", \"go\""
        return errors.BadRequest(reason.TagNotContainSynonym)
    }
    return nil
}
```

**调用位置：**
- 更新同义词 `UpdateTagSynonymReq.Check()` [tag_schema.go#L266-L271]
- 合并标签 `MergeTagReq.Check()`
- 提问/编辑问题前置校验

这个函数的作用是：**如果前端/API 调用方绕过了搜索接口，直接传了同义词的 SlugName，后端会拒绝请求**，而不是静默改写。

这就保证了——即使前端 SearchTagLike 出 bug，后端也不会把同义词 tag_id 写入 tag_rel。

### 1.4 ObjectChangeTag 是否改写过 tag_id？

**结论：完全没有。** 逐行阅读 `ObjectChangeTag()` [tag_common.go#L660-L742](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L660-L742)：

```go
func (ts *TagCommonService) ObjectChangeTag(...) error {
    // Step 1: 收集所有 SlugName → ToLower
    thisObjTagNameList := []string{}   // 存 SlugName
    thisObjTagIDList := []string{}     // 存 tag_id

    // Step 2: 批量查询已存在标签
    tagListInDb, _ := GetTagListByNames(ctx, thisObjTagNameList)
    
    tagInDbMapping := map[string]*entity.Tag{}
    for _, tag := range tagListInDb {
        tagInDbMapping[strings.ToLower(tag.SlugName)] = tag
        thisObjTagIDList = append(thisObjTagIDList, tag.ID)  // ⚠️ 直接用查到的 tag.ID
    }

    // Step 3: 不存在的标签 → 自动创建
    for _, tag := range objectTagData.Tags {
        if _, ok := tagInDbMapping[...]; !ok {
            item := &entity.Tag{
                SlugName: strings.ReplaceAll(tag.SlugName, " ", "-"),  // 归一化
                MainTagID: 0,  // ⚠️ 新标签一律是主标签
                ...
            }
            addTagList = append(addTagList, item)
        }
    }

    // Step 4: 插入新标签 → 把新标签 ID 也加入 thisObjTagIDList
    AddTagList(ctx, addTagList)
    for _, tag := range addTagList {
        thisObjTagIDList = append(thisObjTagIDList, tag.ID)
    }

    // Step 5: 关联
    CreateOrUpdateTagRelList(ctx, objectTagData.ObjectID, thisObjTagIDList)
}
```

**关键点：**
1. **第 2 步查出来什么 tag_id，就用什么**，没有任何 `if tag.MainTagID != 0 { tag.ID = tag.MainTagID }` 的改写。
2. **第 3 步新建的标签 `MainTagID=0`**，永远是主标签，不会是同义词。
3. **第 4 步到第 5 步直接传递**，没有任何中间处理。

**因此，同义词收敛完全依赖写入前的 SearchTagLike 替换 + CheckTag 校验两道防线。**

### 1.5 潜在风险：前端没走搜索接口直接传 SlugName

如果有人用 API 直接调用提问接口，传了同义词 SlugName（如 `tags: [{SlugName: "golang"}]`）：
- 首先被 `CheckTag` 拦住（**正常提问路径**）
- 但 `ObjectChangeTag` 本身并没有这个校验，**如果有人绕过 QuestionService，直接调用 TagCommonService.ObjectChangeTag，同义词 tag_id 会被写进 tag_rel 表**。

建议：在 `ObjectChangeTag` 内部第 2 步遍历 `tagListInDb` 时也加一道 `MainTagID != 0` 的校验，作为防御性兜底。

---

## 二、合并标签后问题搜索索引的同步缺口

### 2.1 MergeTag 完整流程回顾

**入口：** [tag_service.go#L439-L495](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L439-L495)

```
MergeTag(sourceTagID → targetTagID)

Step 1: 查源标签 + 源标签所有同义词 → addSynonymTagList
Step 2: 查目标标签
Step 3: UpdateTagSynonym() 把源标签及其同义词 MainTagID 全部改为目标标签ID
Step 4: MigrateFollowers() 迁移关注者
Step 5: MigrateTagQuestions() 迁移问题关联
        └─ tagRelRepo.MigrateTagObjects(sourceTagID, targetTagID)
Step 6: RefreshTagQuestionCount([源ID, 目标ID])
```

### 2.2 MigrateTagObjects 具体做了什么

**实现：** [tag_rel_repo.go#L209-L261](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/tag/tag_rel_repo.go#L209-L261)

```go
func (tr *tagRelRepo) MigrateTagObjects(ctx context.Context, sourceTagId, targetTagId string) error {
    tr.data.DB.Transaction(func(session *xorm.Session) {
        // 1. 查源标签所有 TagRel
        sourceObjects := SELECT * FROM tag_rel WHERE tag_id = sourceTagId
        
        // 2. 查目标标签已有 TagRel，构造 existingMap[ObjectID] = true
        existingTargets := SELECT * FROM tag_rel WHERE tag_id = targetTagId
        
        // 3. 对源标签每个 ObjectID，若不在目标标签已有中 → 插入新的目标 TagRel
        //    （继承原 Status）
        newRelations := []*TagRel{{TagID: targetTagId, ObjectID: source.ObjectID, Status: source.Status}}
        INSERT newRelations
        
        // 4. 物理 DELETE 源标签所有 TagRel
        DELETE FROM tag_rel WHERE tag_id = sourceTagId
    })
}
```

**合并后 tag_rel 表状态：**
- 所有涉及的问题，其 `tag_rel.tag_id` 已从 sourceTagId 改为 targetTagId
- 源标签 tag_rel 记录已**物理删除**

### 2.3 问题搜索索引更新机制

系统有**两套**搜索索引，更新触发点不同：

#### 索引 1：Search 插件（全文搜索）

**更新函数：** `questionRepo.UpdateSearch()` [question_repo.go#L583-L635](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/question/question_repo.go#L583-L635)

```go
func (qr *questionRepo) UpdateSearch(ctx context.Context, questionID string) error {
    // 1. 查问题详情
    question, exist, _ := qr.GetQuestion(ctx, questionID)
    
    // 2. 查问题所有标签（TagRelStatusAvailable）
    tagListList := SELECT * FROM tag_rel WHERE object_id=questionID AND status=Available
    for _, tag := range tagListList {
        tags = append(tags, tag.TagID)  // ⚠️ 存的是 tag_id，不是 SlugName
    }
    
    // 3. 组装 SearchContent
    content := &plugin.SearchContent{
        ObjectID: questionID,
        Tags:     tags,  // 标签列表（tag_id 数组）
        ...
    }
    s.UpdateContent(ctx, content)  // 调用插件更新索引
}
```

**触发时机（question_repo.go 中的 12 处调用）：**
- 新增问题：`AddQuestion()` 后
- 更新问题：`UpdateQuestion()`/`UpdateQuestionStatus()`/`LinkQuestion()` 后
- 删除/恢复问题：`RemoveQuestion()`/`RecoverQuestion()` 后
- 删除用户所有问题：`RemoveAllUserQuestion()` 后

#### 索引 2：Vector 向量搜索（语义搜索）

**更新入口：** `vectorSyncService.Send()` → `vector_sync.handle()` [vector_sync.go#L59-L115](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/vector_sync/vector_sync.go#L59-L115)

```go
func handleOnce(ctx, data, vectorSearch, action, objectType, objectID) {
    if action == ActionUpsert {
        switch objectType {
        case ObjectTypeQuestion:
            content, _ = BuildQuestionContentByID(ctx, data, objectID)
        }
        vectorSearch.UpdateContent(ctx, content)
    }
}
```

**注意：Vector 索引构建的是 `Title + OriginalText + 所有 Answers + 所有 Comments`，不包含 Tags。** 标签对向量搜索没有直接影响。

**触发时机（review_service.go 和 question_service.go）：**
- 审核通过/拒绝问题、答案、评论
- 问题增删改
- 答案增删改
- 评论增删改

### 2.4 MergeTag 的搜索索引同步缺口

**核心问题：MergeTag 完成后，没有任何地方调用 `UpdateSearch()` 或 `vectorSyncService.Send()`。**

| 步骤 | 操作 | 触发 Search Update? | 触发 Vector Update? |
|------|------|---------------------|----------------------|
| Step 3 | UPDATE tag SET MainTagID=targetID | ❌ 否 | ❌ 否 |
| Step 4 | DELETE/INSERT activity (follow) | ❌ 否 | ❌ 否 |
| Step 5 | DELETE/INSERT tag_rel | ❌ 否 | ❌ 否 |
| Step 6 | UPDATE tag SET question_count=? | ❌ 否 | ❌ 否 |

**缺口导致的具体影响：**

#### 影响 1：Search 插件中的 Tags 字段过时

Search 插件的 `SearchContent.Tags` 存的是 `tag_id` 数组。合并前：
```json
{
  "ObjectID": "q1",
  "Tags": ["source_tag_id_1", "other_tag_id"]
}
```

合并后，tag_rel 表中 `source_tag_id_1` 已被替换为 `target_tag_id_1`，但 Search 索引里仍然是旧值：
- 用户在 Search 插件中用 source_tag_id_1 搜索 → **还能搜到问题，但该标签实际已被合并/删除**（取决于是否仍存在）
- 用户用 target_tag_id_1 搜索 → **搜不到这个问题**，但这个问题实际上已经打上了目标标签

#### 影响 2：按标签筛选搜索结果不一致

如果 Search 插件支持按 tag_id 过滤（常见功能），合并后的一段时间内：
- 目标标签的搜索结果**不包含**刚迁移过来的问题
- 源标签的搜索结果**还包含**这些问题（但点进去看标签，显示的是目标标签）

#### 影响 3：Display Name 不变的情况下 Search 插件可能不受影响

如果 Search 插件在搜索时用 tag_id 关联查当前 tag 表的 DisplayName，而不是存索引时的快照，那么标签改名/合并后的 DisplayName 可以正确显示。但 tag_id 过滤仍然会错。

### 2.5 缺口修复建议

在 `MergeTag` Step 5 之后、Step 6 之前增加：

```go
// Step 5.5: 更新所有被迁移问题的搜索索引
// 查出所有源标签涉及的 ObjectID（在 Step 5 内事务中已记录）
sourceTagObjects := SELECT DISTINCT object_id FROM tag_rel WHERE tag_id = sourceTagId
// （实际上需要在 MigrateTagObjects 内收集并返回这些 ID）

for _, questionID := range sourceTagObjects {
    _ = questionRepo.UpdateSearch(ctx, questionID)
    vectorSyncService.Send(ctx, &vector_sync.Task{
        Action:     vector_sync.ActionUpsert,
        ObjectType: vector_sync.ObjectTypeQuestion,
        ObjectID:   questionID,
    })
}
```

---

## 三、标签编辑修订在待审、通过、拒绝三条路径上对 tag 实体与搜索的影响

### 3.1 标签编辑修订的入口与权限

**API：** `PUT /answer/api/v1/tag` → [tag_controller.go#L166-L187](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/controller/tag_controller.go#L166-L187)

权限注入逻辑：
```go
canList, _ := tc.rankService.CheckOperationPermissions(ctx, req.UserID, []string{
    permission.TagEdit,               // canList[0] 能否编辑
    permission.TagEditWithoutReview,  // canList[1] 能否免审核编辑
})
req.NoNeedReview = canList[1]  // true = 直接落库，false = 进待审队列
```

**权限真值表：**

| 用户角色 | TagEdit | TagEditWithoutReview | NoNeedReview |
|----------|---------|----------------------|--------------|
| 普通用户 | ✅ | ❌ | false（待审） |
| 版主/管理员 | ✅ | ✅ | true（直接通过） |

### 3.2 三条路径的完整流程图

```
用户提交编辑标签请求（SlugName/DisplayName/OriginalText 变更）
    │
    ▼
TagCommonService.UpdateTag(req) [tag_common.go#L866-L948]
    │
    ├─ 检查是否有 Unreviewed 的修订 → 有则拒绝
    │
    ├─ 归一化 SlugName
    │
    ├─ 对比新旧内容，完全相同 → 直接 return
    │
    ├─ 根据 req.NoNeedReview 分支：
    │
    │   ┌─── NoNeedReview = true（免审核）───┐
    │   │                                      │
    │   │  Step A: tagRepo.UpdateTag(tagInfo) │
    │   │    → tag 表立即变更                   │
    │   │                                      │
    │   │  Step B: 若主标签 SlugName 变了      │
    │   │    → 同步所有同义词 MainTagSlugName   │
    │   │                                      │
    │   │  Step C: revision.Status = Pass      │
    │   │                                      │
    │   │  Step D: 发送 ActTagEdited activity  │
    │   │                                      │
    │   │  对 tag 实体: ✅ 立即生效             │
    │   │  对 Search:  ❌ 不触发 UpdateSearch   │
    │   │  对 Vector:  ❌ 不触发                │
    │   │                                      │
    │   └──────────────────────────────────────┘
    │
    │   ┌─── NoNeedReview = false（需审核）───┐
    │   │                                      │
    │   │  Step A: revision.Status = Unreviewed│
    │   │    → tag 表**不变**                   │
    │   │                                      │
    │   │  Step B: AddRevision() 落库          │
    │   │                                      │
    │   │  Step C: 不发 activity               │
    │   │                                      │
    │   │  对 tag 实体: ❌ 不变                 │
    │   │  对 Search:  ❌ 不变                  │
    │   │  对 Vector:  ❌ 不变                  │
    │   │                                      │
    │   └──────────────────────────────────────┘
    │
    ▼
AddRevision() 落库 revision 表
```

### 3.3 路径 1：待审（Unreviewed）

**触发条件：** 普通用户编辑标签（NoNeedReview=false）

**实现位置：** [tag_common.go#L930-L936](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L930-L936)

```go
} else {
    revisionDTO.Status = entity.RevisionUnreviewedStatus
}

tagInfoJson, _ := json.Marshal(tagInfo)
revisionDTO.Content = string(tagInfoJson)
revisionID, err := ts.revisionService.AddRevision(ctx, revisionDTO, true)
```

**效果：**
| 对象 | 是否变化 | 说明 |
|------|----------|------|
| `tag` 表 | ❌ 不变 | `UPDATE tag` 未执行 |
| `revision` 表 | ✅ 新增一条 | `status=Unreviewed`，`Content` 是新 tag 实体 JSON |
| `activity` 表 | ❌ 不变 | 不发 ActTagEdited |
| Search 插件索引 | ❌ 不变 | 无触发点 |
| Vector 索引 | ❌ 不变 | 无触发点 |
| 前端展示详情 | ❌ 不变 | 读的是 tag 表 |
| 标签搜索结果 | ❌ 不变 | 读的是 tag 表 |

**重要特性：编辑锁**
待审期间，任何人不能再编辑该标签：
```go
_, existUnreviewed, _ := ts.revisionService.ExistUnreviewedByObjectID(ctx, req.TagID)
if existUnreviewed {
    return errors.BadRequest(reason.AnswerCannotUpdate)
}
```

### 3.4 路径 2：审核通过（Pass）

**触发入口：** 管理员在待审队列中审核通过 → `RevisionService.RevisionAudit()` [revision_service.go#L106-L179](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/revision_service.go#L106-L179)

```go
func (rs *RevisionService) RevisionAudit(ctx context.Context, req *schema.RevisionAuditReq) error {
    // 查 revision
    revisioninfo, exist, _ := rs.revisionRepo.GetRevisionByID(ctx, req.ID)
    
    // 通过 or 拒绝
    if req.IsApprove() {
        // 更新 revision.Status = Pass
        rs.revisionRepo.UpdateReviewStatus(..., entity.RevisionReviewPassStatus)
        
        // 根据 ObjectType 分发
        switch revisioninfo.ObjectType {
        case constant.TagObjectType:
            rs.revisionAuditTag(ctx, revisionitem)
        case constant.QuestionObjectType:
            rs.revisionAuditQuestion(ctx, revisionitem)
        // ...
        }
    } else {
        // 更新 revision.Status = Rejected
        rs.revisionRepo.UpdateReviewStatus(..., entity.RevisionReviewRejectStatus)
    }
}
```

**标签通过审核的具体实现：** `revisionAuditTag()` [revision_service.go#L291-L335](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/revision_service.go#L291-L335)

```go
func (rs *RevisionService) revisionAuditTag(ctx context.Context, revisionitem *schema.GetRevisionResp) error {
    taginfo, ok := revisionitem.ContentParsed.(*schema.GetTagResp)
    if ok {
        // Step 1: 更新 tag 实体
        tag := &entity.Tag{
            ID:           taginfo.TagID,
            OriginalText: taginfo.OriginalText,
            ParsedText:   taginfo.ParsedText,
            // ⚠️ 注意：这里只传了 OriginalText 和 ParsedText
            // SlugName 和 DisplayName 没在这里更新？
            // 需要看 tagRepo.UpdateTag 的具体实现
        }
        rs.tagRepo.UpdateTag(ctx, tag)  // xorm: WHERE id=? UPDATE ...
        
        // Step 2: 检查主标签 SlugName 是否变化 → 同步同义词
        tagInfo, exist, _ := rs.tagCommon.GetTagByID(ctx, taginfo.TagID)
        if tagInfo.MainTagID == 0 && len(tagInfo.SlugName) > 0 {
            tagList, _ := rs.tagRepo.GetTagList(ctx, &entity.Tag{MainTagID: converter.StringToInt64(tagInfo.ID)})
            updateTagSlugNames := []string{}
            for _, tag := range tagList {
                updateTagSlugNames = append(updateTagSlugNames, tag.SlugName)
            }
            rs.tagRepo.UpdateTagSynonym(ctx, updateTagSlugNames, 
                converter.StringToInt64(tagInfo.ID), tagInfo.MainTagSlugName)
        }
        
        // Step 3: 发送 ActTagEdited activity
        rs.activityQueueService.Send(ctx, &schema.ActivityMsg{
            ActivityTypeKey: constant.ActTagEdited,
            RevisionID:      revisionitem.ID,
        })
    }
}
```

#### tagRepo.UpdateTag 的隐藏细节

**实现：** [tag_repo.go#L62-L69](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/tag/tag_repo.go#L62-L69)

```go
func (tr *tagRepo) UpdateTag(ctx context.Context, tag *entity.Tag) error {
    // xorm Update(bean) 的默认行为：
    // 只更新 bean 中非零值字段，零值字段不更新
    _, err = tr.data.DB.Context(ctx).Where(builder.Eq{"id": tag.ID}).Update(tag)
}
```

**这意味着 `revisionAuditTag` 中：**
```go
tag := &entity.Tag{
    ID:           taginfo.TagID,
    OriginalText: taginfo.OriginalText,
    ParsedText:   taginfo.ParsedText,
    // DisplayName, SlugName 字段没填，默认零值 "", xorm 不会更新这两列
}
```

**问题：审核通过后，DisplayName 和 SlugName 的变更不会落库！**

对比免审核路径的实现 [tag_common.go#L910-L913](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L910-L913)：
```go
// 免审核路径：完整 tagInfo 对象（已在前面填好了 SlugName/DisplayName）
canUpdate = true
err = ts.tagRepo.UpdateTag(ctx, tagInfo)
// tagInfo 包含: {ID, SlugName, DisplayName, OriginalText, ParsedText}
```

**这是一个 Bug。** 普通用户修改了标签 DisplayName 或 SlugName，提交审核后，管理员点通过，实际只有 OriginalText/ParsedText 生效，DisplayName/SlugName 没变。

**审核通过效果：**

| 对象 | 是否变化 | 说明 |
|------|----------|------|
| `tag.OriginalText` | ✅ 变化 | 正确更新 |
| `tag.ParsedText` | ✅ 变化 | 正确更新 |
| `tag.DisplayName` | ❌ **不变** | Bug：xorm 零值不更新 |
| `tag.SlugName` | ❌ **不变** | Bug：xorm 零值不更新 |
| `tag.MainTagSlugName`（同义词） | ⚠️ 可能错 | 基于未更新的 tagInfo.SlugName 同步 |
| `revision` 表 | ✅ status → Pass | |
| `activity` 表 | ✅ ActTagEdited | |
| Search 插件索引 | ❌ 不变 | 无触发点 |
| Vector 索引 | ❌ 不变 | 无触发点 |

**Search 索引不变的影响：**
- Search 插件中的 Tags 存的是 tag_id，标签 SlugName/DisplayName 变化不影响 tag_id，所以全文搜索中的按 tag_id 过滤不受影响。
- 但如果 Search 插件对 DisplayName 做了快照索引（比如按标签名称搜索问题），则快照是旧值。

### 3.5 路径 3：审核拒绝（Rejected）

**触发入口：** 管理员在待审队列中审核拒绝 → `RevisionService.RevisionAudit(req.IsApprove()=false)`

**实现：** [revision_service.go#L106-L179](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/revision_service.go#L106-L179)

```go
if req.IsApprove() {
    // ... 通过逻辑
} else {
    err = rs.revisionRepo.UpdateReviewStatus(
        ctx, req.ID, req.UserID, entity.RevisionReviewRejectStatus)
}
```

**拒绝的效果最简单：**
| 对象 | 是否变化 | 说明 |
|------|----------|------|
| `tag` 表 | ❌ 完全不变 | 没有任何 UPDATE |
| `revision` 表 | ✅ status → Rejected | |
| `activity` 表 | ❌ 不变 | 不发 ActTagEdited |
| Search 插件索引 | ❌ 不变 | |
| Vector 索引 | ❌ 不变 | |
| 编辑锁 | ✅ 释放 | 下次可以再提交编辑 |

### 3.6 三条路径对比总表

| 维度 | 待审（Unreviewed） | 审核通过（Pass） | 审核拒绝（Rejected） |
|------|-------------------|------------------|---------------------|
| tag.OriginalText | ❌ 不变 | ✅ 更新 | ❌ 不变 |
| tag.ParsedText | ❌ 不变 | ✅ 更新 | ❌ 不变 |
| tag.DisplayName | ❌ 不变 | ❌ **Bug：不变** | ❌ 不变 |
| tag.SlugName | ❌ 不变 | ❌ **Bug：不变** | ❌ 不变 |
| 同义词 MainTagSlugName | ❌ 不变 | ⚠️ 基于旧 SlugName 同步 | ❌ 不变 |
| revision 表 | ✅ 新增 Unreviewed | ✅ status → Pass | ✅ status → Rejected |
| activity 表 | ❌ 不发 | ✅ ActTagEdited | ❌ 不发 |
| Search 索引 | ❌ 不变 | ❌ 不变 | ❌ 不变 |
| Vector 索引 | ❌ 不变 | ❌ 不变 | ❌ 不变 |
| 编辑锁 | ✅ 锁定，不可再编辑 | ✅ 释放 | ✅ 释放 |

### 3.7 修订 Bug 修复建议

`revisionAuditTag()` 中构造 tag 对象时应该从 `revisionitem.ContentParsed` 中取完整字段：

```go
func (rs *RevisionService) revisionAuditTag(ctx context.Context, revisionitem *schema.GetRevisionResp) error {
    taginfo, ok := revisionitem.ContentParsed.(*schema.GetTagResp)
    if ok {
        tag := &entity.Tag{
            ID:           taginfo.TagID,
            SlugName:     taginfo.SlugName,       // ✅ 补全
            DisplayName:  taginfo.DisplayName,    // ✅ 补全
            OriginalText: taginfo.OriginalText,
            ParsedText:   taginfo.ParsedText,
        }
        // 用 MustCols 强制更新所有字段
        rs.data.DB.Context(ctx).ID(tag.ID).
            MustCols("slug_name", "display_name", "original_text", "parsed_text").
            Update(tag)
    }
}
```

或者直接复用 `tag_common.UpdateTag` 中已有的完整对象构造逻辑，避免两份代码不一致。

---

## 四、总结：标签写入路径的 5 个关键缺口

| 编号 | 问题 | 影响范围 | 严重度 |
|------|------|----------|--------|
| 1 | **ObjectChangeTag 无同义词校验**：绕过 QuestionService 直接调用的话，同义词 tag_id 会被写入 tag_rel | API 安全 | 中 |
| 2 | **MergeTag 不触发 Search 索引更新**：合并后所有受影响问题的 SearchContent.Tags 还是旧 tag_id | 搜索一致性 | 高 |
| 3 | **MergeTag 不触发 Vector 索引更新**：影响较小（Vector 不索引 Tags）但完整性有缺口 | 搜索一致性 | 低 |
| 4 | **revisionAuditTag DisplayName/SlugName 不更新**：普通用户修改名称后审核通过，名称实际没变 | 数据正确性 | **高** |
| 5 | **标签编辑审核通过不触发 Search 更新**：若 Search 插件索引了 DisplayName 快照，则显示旧值 | 搜索一致性 | 中 |
