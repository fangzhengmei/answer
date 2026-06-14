# 标签系统写入侧内部机制深入分析

## 一、关注计数增减的函数实现

标签的关注计数（`follow_count`）通过两条独立路径维护：**单用户关注/取消** 和 **标签合并时批量迁移**。

### 1.1 单用户关注：Follow()

**实现位置：** [follow_repo.go#L60-L125](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity/follow_repo.go#L60-L125)

```go
func (ar *FollowRepo) Follow(ctx context.Context, objectID, userID string) error {
    // 1. 解析对象类型，获取 activity_type
    activityType, _ := ar.activityRepo.GetActivityTypeByObjectType(ctx, objectTypeStr, "follow")
    
    _, err = ar.data.DB.Transaction(func(session *xorm.Session) (result any, err error) {
        // 2. 查询该用户是否已有该对象的 follow activity 记录
        has, err = session.Where(...)
            .And("user_id = ?", userID)
            .And("object_id = ?", objectID)
            .Get(&existsActivity)

        // 3. 若已存在且是活动状态（Cancelled=0）→ 直接返回，不重复关注
        if has && existsActivity.Cancelled == entity.ActivityAvailable {
            return
        }

        // 4. 若已存在但已取消 → 恢复为活动状态
        if has {
            _, err = session.Where("id = ?", existsActivity.ID).
                Cols(`cancelled`).
                Update(&entity.Activity{
                    Cancelled: entity.ActivityAvailable,
                })
        } else {
            // 5. 不存在 → 插入新的 activity 记录
            _, err = session.Insert(&entity.Activity{
                UserID:           userID,
                ObjectID:         objectID,
                OriginalObjectID: objectID,
                ActivityType:     activityType,
                Cancelled:        entity.ActivityAvailable,
            })
        }

        // 6. 原子递增 follow_count
        err = ar.updateFollows(ctx, session, objectID, 1)
        return
    })
}
```

### 1.2 单用户取消关注：FollowCancel()

**实现位置：** [follow_repo.go#L127-L169](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity/follow_repo.go#L127-L169)

```go
func (ar *FollowRepo) FollowCancel(ctx context.Context, objectID, userID string) error {
    _, err = ar.data.DB.Transaction(func(session *xorm.Session) (result any, err error) {
        // 1. 查询 activity 记录
        has, err = session.Where(...).Get(&existsActivity)
        if !has || existsActivity.Cancelled == entity.ActivityCancelled {
            return  // 已取消，直接返回
        }

        // 2. 标记为已取消（软删除，非物理删除）
        _, err = session.Where("id = ?", existsActivity.ID).
            Cols("cancelled").
            Update(&entity.Activity{
                Cancelled:   entity.ActivityCancelled,
                CancelledAt: time.Now(),
            })

        // 3. 原子递减 follow_count
        err = ar.updateFollows(ctx, session, objectID, -1)
        return
    })
}
```

### 1.3 原子更新计数：updateFollows()

**实现位置：** [follow_repo.go#L171-L187](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity/follow_repo.go#L171-L187)

```go
func (ar *FollowRepo) updateFollows(_ context.Context, session *xorm.Session, 
    objectID string, follows int) error {
    switch objectType {
    case "tag":
        _, err = session.Where("id = ?", objectID).
            Incr("follow_count", follows).  // 数据库原子自增/自减
            Update(&entity.Tag{})
    }
}
```

### 1.4 关注计数读取：GetFollowAmount()

**实现位置：** [activity_common/follow.go#L59-L91](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity_common/follow.go#L59-L91)

```go
func (ar *FollowRepo) GetFollowAmount(ctx context.Context, objectID string) (follows int, err error) {
    // 直接从 tag 表的 follow_count 字段读取，而非实时 COUNT activity 表
    model := &entity.Tag{}
    _, err = ar.data.DB.Context(ctx).Where("id = ?", objectID).
        Cols("`follow_count`").Get(model)
    follows = model.FollowCount
}
```

### 1.5 关键设计要点

| 特性 | 说明 |
|------|------|
| **原子性** | `Follow()` 和 `FollowCancel()` 都包裹在数据库事务中，`updateFollows` 使用 `Incr()` 原子自增 |
| **幂等性** | 重复关注/取消不会产生副作用，通过先查询状态再操作保证 |
| **计数存储** | 计数缓存到 `tag.follow_count` 字段，查询时直接读标签表而非 COUNT activity |
| **软删除** | 取消关注是 `cancelled=1`，不物理删除 activity 记录 |
| **幂等性风险** | `updateFollows` 没有先校验当前计数，并发场景下若重复调用可能导致计数漂移 |

---

## 二、保留标签在问题编辑时的判定逻辑

保留标签（`Reserved=true`）是一种特殊权限标签，只有版主/管理员可以使用。判定逻辑分布在**提问**和**编辑问题**两个阶段。

### 2.1 提问阶段校验：AddQuestionCheckTags()

**实现位置：** [question_service.go#L223-L234](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/question_service.go#L223-L234)

```go
func (qs *QuestionService) AddQuestionCheckTags(ctx context.Context, tags []*entity.Tag) ([]string, error) {
    list := make([]string, 0)
    for _, tag := range tags {
        if tag.Reserved {
            list = append(list, tag.DisplayName)
        }
    }
    if len(list) > 0 {
        return list, errors.BadRequest(reason.RequestFormatError)
    }
    return []string{}, nil
}
```

在 `AddQuestion()` 和 `CheckAddQuestion()` 中调用：
```go
if !req.CanUseReservedTag {
    taglist, err := qs.AddQuestionCheckTags(ctx, Tags)
    // 错误信息: "xxx, yyy can only be used by moderators."
}
```

### 2.2 编辑阶段校验：CheckChangeReservedTag()

**实现位置：** [tag_common.go#L623-L658](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L623-L658)

这是保留标签最核心的判定函数，返回 4 个值：
```go
func (ts *TagCommonService) CheckChangeReservedTag(
    ctx context.Context, 
    oldobjectTagData, objectTagData []*entity.Tag,
) (CheckOldTag bool, CheckNewTag bool, CheckOldTaglist []string, CheckNewTaglist []string)
```

#### 算法逻辑：

```
1. 构建新标签中保留标签的 map: reservedTagsMap[slugName] = true

2. 遍历旧标签中的保留标签:
   a. 若旧保留标签不在新标签中 → 加入 needTagsMap (说明用户试图移除保留标签)
   b. 若旧保留标签在新标签中 → 标记 reservedTagsMap[slugName] = false (说明是旧标签中已有的)

3. 遍历 reservedTagsMap:
   a. 若值仍为 true → 加入 notNeedTagsMap (说明用户试图新增保留标签)

4. 返回判定:
   - 若 needTagsMap 非空 → CheckOldTag=false (旧保留标签被移除了)
   - 若 notNeedTagsMap 非空 → CheckNewTag=false (新增了保留标签)
   - 否则 → 两者都为 true (保留标签未被动过)
```

**调用位置：**
- 编辑问题 `UpdateQuestionCheckTags()` [question_service.go#L666-L730](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/question_service.go#L666-L730)
- 保存问题 `UpdateQuestion()` [question_service.go#L983-L1008](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/question_service.go#L983-L1008)

```go
if !req.CanUseReservedTag {
    CheckOldTag, CheckNewTag, CheckOldTaglist, CheckNewTaglist := 
        qs.CheckChangeReservedTag(ctx, oldTags, Tags)
    
    if !CheckOldTag {
        // 错误: "The reserved tag xxx must be present."
    }
    if !CheckNewTag {
        // 错误: "xxx can only be used by moderators."
    }
}
```

### 2.3 判定真值表

| 场景 | CheckOldTag | CheckNewTag | 说明 |
|------|-------------|-------------|------|
| 旧有保留标签，新标签中保留 | true | true | ✅ 合法 |
| 旧有保留标签 A，新标签移除 A | **false** | true | ❌ 不能移除保留标签 |
| 旧无保留标签，新标签加 A | true | **false** | ❌ 不能新增保留标签 |
| 旧有 A，新标签有 A+B（B 是保留） | true | **false** | ❌ 不能新增保留标签 |
| 旧有 A，新标签有 B（移除 A 加 B） | **false** | **false** | ❌ 两者都违规 |
| CanUseReservedTag=true | - | - | ✅ 跳过校验 |

### 2.4 保留标签校验的调用链路

```
AddQuestion()
    ├─ 第一次校验: CheckAddQuestion() → AddQuestionCheckTags()  ← 预校验
    ├─ 创建 Question
    └─ 第二次校验: 内部再调用 AddQuestionCheckTags()          ← 保存前再次校验

UpdateQuestion()
    ├─ UpdateQuestionCheckTags()
    │     └─ CheckChangeReservedTag(oldTags, newTags)
    └─ 保存前第二次校验 CheckChangeReservedTag()

QuestionController.UpdateQuestion()
    └─ req.CanUseReservedTag 由权限系统注入
       (permission.TagEditReserved)
```

**注意：** 校验被调用了两次（第一次在 `UpdateQuestionCheckTags`，第二次在 `UpdateQuestion` 主逻辑中），防止中间状态被篡改。

---

## 三、合并过程中标签关注计数的同步问题

### 3.1 MergeTag 整体流程

**实现位置：** [tag_service.go#L439-L495](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L439-L495)

```
MergeTag(sourceTagID → targetTagID)

1. 查询源标签 + 源标签的所有同义词（MainTagID=源标签ID）
   → addSynonymTagList = [源标签.SlugName, 同义词1.SlugName, ...]

2. 查询目标标签

3. 更新源标签和其所有同义词的 MainTagID = 目标标签ID
   → UpdateTagSynonym(addSynonymTagList, targetTagID, targetTagSlugName)

4. 迁移关注者: MigrateFollowers(源ID → 目标ID, "follow")

5. 迁移问题关联: MigrateTagQuestions(源ID → 目标ID)

6. 刷新源+目标标签的 question_count
```

### 3.2 MigrateFollowers 内部实现

**实现位置：** [activity_common/follow.go#L163-L265](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity_common/follow.go#L163-L265)

这是一个包含 4 个步骤的事务操作：

```go
func (ar *FollowRepo) MigrateFollowers(ctx context.Context, 
    sourceObjectID, targetObjectID, action string) error {
    
    _, err = ar.data.DB.Transaction(func(session *xorm.Session) (result any, err error) {
        // ========== 步骤 1: 物理删除源标签的所有 follow activity ==========
        _, err = session.Table("activity").
            Where("object_id = ? AND activity_type = ?", 
                sourceObjectID, activityType).
            Delete(&entity.Activity{})
        
        // ========== 步骤 2: 恢复目标标签中源标签关注者的取消状态 ==========
        // 如果源标签的关注者之前关注过目标标签但取消了，现在恢复为关注
        _, err = session.Table("activity").
            Where("object_id = ? AND activity_type = ?", targetObjectID, activityType).
            And("user_id IN ?", sourceFollowers).
            Cols("cancelled").
            Update(&entity.Activity{Cancelled: entity.ActivityAvailable})
        
        // ========== 步骤 3: 查询目标标签当前的有效关注者 ==========
        targetFollowers := SELECT user_id FROM activity 
            WHERE object_id=targetObjectID AND cancelled=0
        
        // ========== 步骤 4: 为不重复的源关注者创建新的 follow activity ==========
        for _, uid := range sourceFollowers {
            if !existingFollowers[uid] {
                session.Insert(&entity.Activity{
                    UserID:           uid,
                    ObjectID:         targetObjectID,
                    OriginalObjectID: targetObjectID,  // 注意: 这里用的是目标ID，不是源ID
                    ActivityType:     activityType,
                    CreatedAt:        time.Now(),      // 注意: 丢失原始关注时间
                    Cancelled:        entity.ActivityAvailable,
                })
            }
        }
        return nil, nil
    })
}
```

### 3.3 关注计数同步的关键缺陷

#### 缺陷 1：源标签 follow_count 未更新

`MigrateFollowers` 步骤 1 **物理删除**了源标签的所有 activity 记录，但**没有更新源标签的 `follow_count` 字段**。

```go
// MigrateFollowers 中缺失:
err = ar.updateFollows(ctx, session, sourceObjectID, -len(userIDs))  // 源标签计数清零
```

**影响：** 合并后源标签变成同义词，一般不会再展示，但如果有人通过 API 直接查询源标签，`follow_count` 仍是旧值，与 activity 表实际数量不一致。

#### 缺陷 2：目标标签 follow_count 未正确更新

`MigrateFollowers` 步骤 4 为新增的关注者插入了新的 activity 记录，但**没有调用 `updateFollows` 递增目标标签的 `follow_count`**。

**计数维护的正确流程应该是：**
```
目标标签新关注数 = 
    原目标标签关注者中不包含的源关注者数量  // 步骤4 新增的
  - 步骤2 中恢复关注的人数                  // 从 cancelled→available，这部分原计数未包含
```

但当前代码中，步骤 2 和步骤 4 都**没有**调用 `Incr("follow_count")`。

**那目标标签的 follow_count 是怎么更新的？** —— **它没有被更新**！

合并完成后，`GetFollowAmount(目标标签ID)` 读取的仍是合并前的 `follow_count` 值，少了迁移过来的关注者数量。

**只有当后续有人关注/取消关注目标标签时，`Incr/Decr` 才会基于错误的基数继续增减，导致长期漂移。**

#### 缺陷 3：CreatedAt 重置

步骤 4 中新插入的 activity 记录 `CreatedAt = time.Now()`，丢失了用户原始关注源标签的时间。这会影响：
- 关注时间排序
- 统计最早关注时间
- 用户行为分析

#### 缺陷 4：OriginalObjectID 覆盖

新插入的 activity 记录 `OriginalObjectID = targetObjectID`，而不是 `sourceObjectID`。这意味着：
- 无法追溯这批关注者是从哪个标签合并过来的
- 失去了溯源能力（`OriginalObjectID` 字段本就是为了保留原始对象 ID）

### 3.4 问题关联迁移中的计数同步

`MigrateTagQuestions` [tag_rel_repo.go#L209-L261](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/tag/tag_rel_repo.go#L209-L261) 在迁移完成后，**没有**更新 `question_count`，而是由 `MergeTag` 最后一步统一刷新：

```go
err = ts.tagCommonService.RefreshTagQuestionCount(
    ctx, []string{targetTagInfo.ID, sourceTag.ID})
```

`RefreshTagQuestionCount` 会对每个标签执行：
```go
count, _ := ts.tagRelRepo.CountTagRelByTagID(ctx, tagID)  // 实时 COUNT
ts.tagCommonRepo.UpdateTagQuestionCount(ctx, tagID, int(count))  // 写回
```

**这是正确的做法**，用实时 COUNT 覆盖，避免了增量计算的错误累积。

但 `follow_count` **没有采用同样的刷新机制**，这是不一致的设计。

---

## 四、同义词主标签归一化逻辑

同义词机制是标签系统的核心特性，通过 `MainTagID` 和 `MainTagSlugName` 两个字段实现。归一化逻辑分布在**写入侧**（设置同义词/修改主标签）和**读取侧**（查询/搜索时自动跳转）。

### 4.1 写入侧：UpdateTagSynonym()

**实现位置：** [tag_service.go#L298-L400](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L298-L400)

流程：
```
1. req.Format(): 所有同义词 SlugName → ToLower
2. 校验: 主标签 SlugName 不能出现在同义词列表
3. 批量查询已存在的同义词标签
4. 不存在的同义词 → AddTagList() 作为新标签插入
5. 查询旧同义词列表，计算需要移除的同义词
   → UpdateTagSynonym(removeList, MainTagID=0, "")
6. 对所有（新增+已存在）同义词
   → UpdateTagSynonym(addList, MainTagID=主标签ID, MainTagSlugName=主标签Slug)
```

**底层数据库操作：** [tag_repo.go#L102-L112](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/tag/tag_repo.go#L102-L112)

```go
func (tr *tagRepo) UpdateTagSynonym(ctx context.Context, 
    tagSlugNameList []string, mainTagID int64, mainTagSlugName string) error {
    bean := &entity.Tag{
        MainTagID:        mainTagID,
        MainTagSlugName:  mainTagSlugName,
    }
    session := tr.data.DB.Context(ctx).
        In("slug_name", tagSlugNameList).
        MustCols("main_tag_id", "main_tag_slug_name")  // 强制更新这两列
    _, err = session.Update(bean)
}
```

### 4.2 写入侧：主标签 SlugName 变更同步

当主标签的 `SlugName` 修改时，需要同步更新其所有同义词的 `MainTagSlugName`。

**实现位置：** [tag_common.go#L910-L930](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L910-L930)

```go
if tagInfo.MainTagID == 0 && len(tagInfo.SlugName) > 0 {
    // 查询所有以该标签为主标签的同义词
    tagList, err := rs.tagRepo.GetTagList(ctx, 
        &entity.Tag{MainTagID: converter.StringToInt64(tagInfo.ID)})
    
    updateTagSlugNames := make([]string, 0)
    for _, tag := range tagList {
        updateTagSlugNames = append(updateTagSlugNames, tag.SlugName)
    }
    // 批量更新同义词的 MainTagSlugName
    err = rs.tagRepo.UpdateTagSynonym(
        ctx, updateTagSlugNames, 
        converter.StringToInt64(tagInfo.ID), 
        tagInfo.MainTagSlugName)
}
```

**调用位置：**
- 标签编辑审核通过 `revisionAuditTag()` [revision_service.go#L310-L324](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/revision_service.go#L310-L324)
- 普通更新标签 `UpdateTag()` [tag_common.go#L910-L930](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L910-L930)

### 4.3 读取侧：搜索归一化 SearchTagLike()

**实现位置：** [tag_common.go#L114-L165](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L114-L165)

```go
for _, tag := range tagList {
    if tag.MainTagID != 0 {
        mainTag, exist, err := ts.GetTagByID(ctx, strconv.FormatInt(tag.MainTagID, 10))
        if exist {
            // 替换为同义词的主标签信息
            tag.ID = mainTag.ID
            tag.SlugName = mainTag.SlugName
            tag.DisplayName = mainTag.DisplayName
            tag.MainTagID = mainTag.MainTagID
            tag.MainTagSlugName = mainTag.MainTagSlugName
        }
    }
    repetitiveTag[tag.SlugName] = tag  // 按主标签 SlugName 去重
}
```

### 4.4 读取侧：标签详情跳转 GetTagInfo()

**实现位置：** [tag_service.go#L164-L174](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L164-L174)

```go
if tagInfo.MainTagID > 0 {
    // 查询主标签，返回主标签信息
    resp.TagID = strconv.FormatInt(tagInfo.MainTagID, 10)
    resp.SlugName = tagInfo.MainTagSlugName
    mainTag, exist, err := ts.tagCommonService.GetTagByID(ctx, resp.TagID)
    if exist {
        resp.DisplayName = mainTag.DisplayName
        resp.OriginalText = mainTag.OriginalText
        resp.ParsedText = mainTag.ParsedText
    }
}
```

### 4.5 同义词归一化的缺陷

#### 缺陷 1：多级同义词链

```
场景：
1. 创建标签 A（MainTagID=0）
2. 设置 B 是 A 的同义词（B.MainTagID=A.ID）
3. 再把 A 设置为 C 的同义词（A.MainTagID=C.ID）

结果：
- B.MainTagID 仍然指向 A.ID（A 已经是同义词了）
- 查询 B 时，SearchTagLike() 跳转到 A，再跳转 C → 两级跳转正确
- 但修改 C 的 SlugName 时，同步逻辑 [tag_common.go#L910-L930] 
  只会查 MainTagID=C.ID 的标签（即 A），B 的 MainTagSlugName 不会更新
- B.MainTagSlugName 仍然是 A 旧的 SlugName
```

**根本原因：** `MainTagID` 只支持一级，没有级联更新机制。

#### 缺陷 2：同义词标签可以被设为其他标签的主标签

`UpdateTagSynonym()` 没有校验 `MainTagID=0`（必须是主标签才能作为目标），理论上可以把一个同义词标签设为另一个标签的主标签。

#### 缺陷 3：MainTagSlugName 与实际 SlugName 可能不一致

若主标签的 SlugName 在 `UpdateTag()` 中被修改，但同义词的 `MainTagSlugName` 未同步（例如事务失败），会出现数据不一致。

此时通过同义词查询详情时：
```go
resp.SlugName = tagInfo.MainTagSlugName  // 可能是旧的
```
用户看到的 SlugName 与实际 URL 不一致。

---

## 五、创建、合并、删除操作对修订记录的影响

修订记录（revision）是内容变更的审计轨迹。标签相关操作对修订记录的处理差异很大。

### 5.1 修订记录通用接口 AddRevision()

**实现位置：** [revision_common/revision_service.go#L60-L74](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/revision_common/revision_service.go#L60-L74)

```go
func (rs *RevisionService) AddRevision(ctx context.Context, 
    req *schema.AddRevisionDTO, autoUpdateRevisionID bool) (revisionID string, err error) {
    req.ObjectID = uid.DeShortID(req.ObjectID)
    rev := &entity.Revision{}
    _ = copier.Copy(rev, req)
    err = rs.revisionRepo.AddRevision(ctx, rev, autoUpdateRevisionID)
    return rev.ID, nil
}
```

### 5.2 创建标签：AddTag()

**实现位置：** [tag_common.go#L342-L384](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L342-L384)

```go
func (ts *TagCommonService) AddTag(ctx context.Context, req *schema.AddTagReq) (*schema.AddTagResp, error) {
    // ... 插入 tag 表 ...
    
    // 添加修订记录
    revisionDTO := &schema.AddRevisionDTO{
        UserID:   req.UserID,
        ObjectID: tag.ID,
        Title:    tag.SlugName,
    }
    tagInfoJson, _ := json.Marshal(tag)
    revisionDTO.Content = string(tagInfoJson)  // 完整序列化整个 tag 实体
    revisionID, err := ts.revisionService.AddRevision(ctx, revisionDTO, true)
    
    // 发送 activity 消息
    ts.activityQueueService.Send(ctx, &schema.ActivityMsg{
        ActivityTypeKey: constant.ActTagCreated,
        RevisionID:       revisionID,  // 关联修订ID
    })
}
```

### 5.3 创建同义词标签：UpdateTagSynonym()

**实现位置：** [tag_service.go#L344-L371](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L344-L371)

当同义词列表中包含数据库中不存在的标签时，会先创建新标签，然后为每个新标签添加修订记录：

```go
if len(needAddTagList) > 0 {
    err = ts.tagCommonService.AddTagList(ctx, needAddTagList)
    for _, tag := range needAddTagList {
        revisionDTO := &schema.AddRevisionDTO{
            UserID:   req.UserID,
            ObjectID: tag.ID,
            Title:    tag.SlugName,
        }
        tagInfoJson, _ := json.Marshal(tag)
        revisionDTO.Content = string(tagInfoJson)
        revisionID, err := ts.revisionService.AddRevision(ctx, revisionDTO, true)
        
        ts.activityQueueService.Send(ctx, &schema.ActivityMsg{
            ActivityTypeKey: constant.ActTagCreated,
            RevisionID:       revisionID,
        })
    }
}
```

**注意：** 只有**新建**的同义词标签会有修订记录。已存在的标签只是 `MainTagID` 被更新，**没有修订记录**。

### 5.4 合并标签：MergeTag()

**实现位置：** [tag_service.go#L439-L495](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L439-L495)

**MergeTag 全程没有添加任何修订记录！**

5 个步骤：
1. 查询源标签及其同义词 → ✅ 无 revision
2. 查询目标标签 → ✅ 无 revision
3. `UpdateTagSynonym()` 更新 `MainTagID` → ✅ 无 revision（直接 UPDATE tag 表）
4. `MigrateFollowers()` 迁移关注者 → ✅ 无 revision（直接操作 activity 表）
5. `MigrateTagQuestions()` 迁移问题关联 → ✅ 无 revision（直接 DELETE + INSERT tag_rel 表）

**这意味着标签合并操作是"黑盒"，没有任何审计轨迹。** 你无法从修订记录中看出：
- 谁在什么时候合并了哪两个标签
- 合并前两个标签的状态是什么
- 合并后的数据是否正确

### 5.5 删除标签：RemoveTag()

**实现位置：** [tag_service.go#L75-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L75-L107)

```go
func (ts *TagService) RemoveTag(ctx context.Context, req *schema.RemoveTagReq) error {
    // 前置校验: 无关联问题 + 无同义词
    tagCount, _ := ts.tagCommonService.CountTagRelByTagID(ctx, req.TagID)
    tagSynonymCount, _ := ts.tagRepo.GetTagSynonymCount(ctx, req.TagID)
    
    // 软删除: status = 10
    err = ts.tagRepo.RemoveTag(ctx, req.TagID)
    
    // 发送 activity 消息（无 RevisionID）
    ts.activityQueueService.Send(ctx, &schema.ActivityMsg{
        ActivityTypeKey: constant.ActTagDeleted,
        // 没有 RevisionID
    })
}
```

**删除操作同样没有添加修订记录。** 只发送了 `ActTagDeleted` activity 消息，没有 revision 记录。

### 5.6 编辑标签：UpdateTag() + 审核通过

**编辑标签**有两条路径，都正确添加了修订记录：

**路径 1：无需审核的编辑（本人/管理员）**
- `UpdateTag()` [tag_common.go#L870-L932] 调用 `AddRevision()`
- 发送 `ActTagEdited` activity，关联 `RevisionID`

**路径 2：需要审核的编辑（其他用户）**
- 先提交 revision，状态为 `Unreviewed`
- 管理员审核通过 `revisionAuditTag()` [revision_service.go#L291-L335]
  - 更新 tag 表
  - 发送 `ActTagEdited` activity，关联 `RevisionID`

### 5.7 恢复标签：RecoverTag()

**实现位置：** [tag_service.go#L114-L138](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L114-L138)

```go
func (ts *TagService) RecoverTag(ctx context.Context, req *schema.RecoverTagReq) error {
    // status = 1 (Available)
    err = ts.tagRepo.RecoverTag(ctx, req.TagID)
    
    // 发送 activity 消息（无 RevisionID）
    ts.activityQueueService.Send(ctx, &schema.ActivityMsg{
        ActivityTypeKey: constant.ActTagUndeleted,
    })
}
```

**恢复操作也没有修订记录。**

### 5.8 修订记录处理对比表

| 操作 | 添加 Revision | 发送 Activity | 关联 RevisionID | 说明 |
|------|--------------|---------------|-----------------|------|
| 创建标签 | ✅ 有 | ✅ ActTagCreated | ✅ | 完整记录 |
| 创建同义词（新标签） | ✅ 有 | ✅ ActTagCreated | ✅ | 新标签有记录 |
| 创建同义词（已有标签变同义词） | ❌ 无 | ❌ 无 | - | 仅 UPDATE MainTagID，无轨迹 |
| 编辑标签（无需审核） | ✅ 有 | ✅ ActTagEdited | ✅ | 完整记录 |
| 编辑标签（需审核） | ✅ 有（待审核） | ✅ ActTagEdited（审核通过后） | ✅ | 完整记录 |
| 合并标签（5 步全流程） | ❌ 无 | ❌ 无 | - | **完全无审计轨迹** |
| 删除标签 | ❌ 无 | ✅ ActTagDeleted | ❌ 无 | 只有 activity，无 revision |
| 恢复标签 | ❌ 无 | ✅ ActTagUndeleted | ❌ 无 | 只有 activity，无 revision |
| 主标签 SlugName 变更同步同义词 | ❌ 无 | ❌ 无 | - | 批量 UPDATE，无轨迹 |

### 5.9 修订记录内容格式

修订记录的 `Content` 字段是完整 JSON 序列化的 `entity.Tag`：

```json
{
  "id": "12345",
  "slug_name": "golang",
  "display_name": "Go",
  "main_tag_id": 0,
  "main_tag_slug_name": "",
  "original_text": "## Go 语言\n\nGo 是一门...",
  "parsed_text": "<h2>Go 语言</h2>...",
  "follow_count": 100,
  "question_count": 50,
  "recommend": true,
  "reserved": false,
  "status": 1,
  "user_id": "67890",
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-01T00:00:00Z"
}
```

在时间线对比时，通过 `parseItem()` [revision_service.go#L429-L492](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/revision_service.go#L429-L492) 反序列化并展示。

### 5.10 修订记录缺失的风险

1. **合并操作无法回滚**：没有合并前的修订记录快照，若合并出错无法精确回滚到合并前状态。
2. **审计合规问题**：标签合并、删除、恢复都是高风险操作，缺少修订记录意味着无法审计谁在什么时候做了什么。
3. **同义词变更无轨迹**：将已有标签设为同义词/取消同义词关系，`MainTagID` 字段变更没有记录。
4. **恢复标签数据丢失**：删除标签时没有保留快照，恢复时只能靠软删除的记录本身，如果中间被其他操作修改了部分字段，恢复时会带上修改。

---

## 六、改进建议汇总

### 6.1 关注计数同步改进

1. **MigrateFollowers 中补充 follow_count 更新**：
   ```go
   // 步骤 1 后：源标签计数清零
   err = ar.updateFollows(ctx, session, sourceObjectID, -len(userIDs))
   
   // 步骤 4 后：目标标签计数增加新增关注者数量
   newFollowerCount := len(newFollowers) + restoredFollowerCount
   err = ar.updateFollows(ctx, session, targetObjectID, newFollowerCount)
   ```

2. **或采用 RefreshTagQuestionCount 模式**，在 `MergeTag` 最后增加：
   ```go
   // 实时 COUNT activity 表后写回
   RefreshTagFollowCount(ctx, []string{targetTagID, sourceTagID})
   ```

3. **保留原始关注时间**：`OriginalObjectID = sourceObjectID`，`CreatedAt` 取源 activity 的 CreatedAt。

### 6.2 同义词多级链改进

1. `UpdateTagSynonym()` 增加校验：主标签必须满足 `MainTagID == 0`。
2. 主标签被设为其他标签的同义词时，级联更新所有旧同义词的 `MainTagID` 到新的主标签。
3. `SearchTagLike()` 改为循环跳转直到 `MainTagID == 0`，防止多级链显示错误。

### 6.3 修订记录完善

1. **MergeTag 添加修订记录**：为源标签和目标标签各添加一条修订记录，记录合并操作。
2. **RemoveTag/RecoverTag 添加修订记录**：删除时保留快照，恢复时可以对比。
3. **UpdateTagSynonym 已有标签变同义词时添加修订记录**：记录 `MainTagID` 变更。
4. **主标签 SlugName 同步同义词时添加修订记录**：批量更新也应留痕。

### 6.4 保留标签校验优化

1. 将两次 `CheckChangeReservedTag` 调用合并为一次，结果复用。
2. 在 `RemoveTag` 中增加 `Reserved` 校验，防止误删保留标签。
