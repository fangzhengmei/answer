# 历史版本保留、标签图片与删除遗留——Revision 对文件清理的反向影响与最终归宿

---

## 一、旧 Revision 持有图片 URL 导致的反向漏洞：孤儿文件永远不会被清理

### 1.1 问题本质

`CleanOrphanUploadFiles` 的设计初衷是清理**不再被引用**的文件。但由于 `revision` 表保存了每一次编辑的历史版本，且 `GetLastRevisionByFileURL` 搜索**所有** revision 记录，导致**曾经出现过的图片 URL 永远能被匹配到**，文件永远不会被判定为"孤儿"。

这是与"评论图片漏删"方向相反的漏洞：
- **评论图片问题**：漏匹配（false negative），应该保留的被删了
- **旧 revision 问题**：误匹配（false positive），应该删的永远保留

### 1.2 代码层面的证据

**GetLastRevisionByFileURL 无状态过滤** [revision_repo.go#L166-L173](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/repo/revision/revision_repo.go#L166-L173)

```go
func (rr *revisionRepo) GetLastRevisionByFileURL(ctx context.Context, 
    fileURL string) (revision *entity.Revision, exist bool, err error) {
    
    revision = &entity.Revision{}
    exist, err = rr.data.DB.Context(ctx).
        Where("content LIKE ?", "%"+fileURL+"%").  // 搜索所有 revision
        Desc("created_at").                         // 取最新的
        Get(revision)
    return
}
```

**关键点**：
- `WHERE content LIKE '%url%'` — 没有 `status = ?` 过滤，也没有 `object_type = ?` 限制
- 搜索**整张 revision 表**的所有历史版本
- 只要有**任何一条** revision 的 content 中包含该 URL，`exist = true`
- 即使是三年前的旧版本、被拒绝的审核版本、已删除对象的版本，都会命中

### 1.3 完整场景复现

```
第 1 天: 用户发帖，内容含图片 A.jpg
  → question.original_text = "![img](A.jpg)"
  → revision v1.content = { ... original_text: "![img](A.jpg)" ... }
  → FileRecord { ObjectID: "0", FileURL: "A.jpg" }

第 2 天: 48h 后第一次清理扫描
  → GetLastRevisionByFileURL("A.jpg")
  → 命中 revision v1
  → 回填 ObjectID = question_id
  → 文件保留 ✓

第 3 天: 用户编辑帖子，把图片 A.jpg 换成 B.jpg
  → question.original_text = "![img](B.jpg)"
  → revision v2.content = { ... original_text: "![img](B.jpg)" ... }
  → FileRecord { ObjectID: "question_id", FileURL: "B.jpg" }

第 5 天: 第二次清理扫描
  → 检查 A.jpg 的 FileRecord
  → ObjectID = "question_id" (非零)
  → GetLastRevisionByObjectID("question_id")
  → 命中 revision v2 (最新版本)
  → revision v2 的 content 中包含 A.jpg 吗？
  → ❌ 不包含 → 判定为孤儿？ → 删除？
  → 等等，不对...让我重新看代码
```

等一下，让我重新梳理。有两种搜索路径：

**路径 A：ObjectID 已回填 → GetLastRevisionByObjectID**

[file_record_service.go#L128-L137](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/file_record/file_record_service.go#L128-L137)

```go
if checker.IsNotZeroString(fileRecord.ObjectID) {
    _, exist, err := fs.revisionRepo.GetLastRevisionByObjectID(ctx, fileRecord.ObjectID)
    if exist {
        continue  // 仍被引用
    }
    // 未找到 → 文件可清理
}
```

`GetLastRevisionByObjectID` 是按 `object_id = ?` 查最新 revision，只要有 revision 就返回 `exist=true`。**它不检查 content 中是否包含该 URL**。

**路径 B：ObjectID 为 "0" → GetLastRevisionByFileURL**

```go
} else {
    lastRevision, exist, err := fs.revisionRepo.GetLastRevisionByFileURL(ctx, fileRecord.FileURL)
    if exist {
        fileRecord.ObjectID = lastRevision.ObjectID  // 回填
        continue
    }
}
```

`GetLastRevisionByFileURL` 搜索所有 revision 的 content，找包含该 URL 的。

### 1.4 反向漏洞的真实触发条件

**关键洞察**：两种路径的行为不同——

| 路径 | 判断依据 | 检查 URL 内容? |
|------|---------|--------------|
| A. ObjectID 已回填 | 该 ObjectID 是否有 revision 存在 | ❌ 只看有没有 revision，不看 content |
| B. ObjectID 为 "0" | 所有 revision 中是否有包含该 URL 的 | ✅ 搜索 content LIKE '%url%' |

**路径 A 的特性**：
- 一旦 FileRecord 的 ObjectID 被回填，后续检查只看该 ObjectID 是否有 revision 存在
- **不关心 revision 的 content 中是否还包含该 URL**
- 只要对象（Q/A/Tag）还存在且有 revision（几乎总是成立），文件就永远保留
- 即使图片已经从正文中移除了，也不会被清理

**路径 B 的特性**（ObjectID 为 "0" 的文件）：
- 搜索所有 revision 的 content，只要有任何一个 revision 包含该 URL 就命中
- 旧版本、被拒绝的审核版本、已删除对象的版本都算
- 所以**曾经被用过的图片永远能被搜到**，除非对应的 revision 记录被物理删除

### 1.5 最终结论：双重保险导致文件永不清理

**路径 A 的问题**：ObjectID 回填后，不再检查 content，只看对象是否有 revision。只要对象不被硬删除，文件永远保留。

**路径 B 的问题**：即使 ObjectID 为 "0"，只要该 URL 曾出现在任何 revision 中（包括旧版本、已拒绝版本），就会命中。

**结果**：**一旦图片被上传并插入过任何帖子，它就永远不会被清理**，即使：
1. 图片已从最新版本的正文中移除
2. 包含该图片的帖子已被软删除
3. 包含该图片的修订版已被审核拒绝

### 1.6 设计意图 vs 实际效果

| 设计意图 | 实际效果 |
|---------|---------|
| 通过 revision 引用追踪文件使用情况 | revision 历史版本导致文件"永久被引用" |
| 48 小时宽限期内不清理 | 48 小时后也清理不掉（永远命中） |
| 惰性回填 ObjectID 优化后续查询 | 回填后不再检查 content，彻底失效 |
| 保留编辑历史可回滚 | 历史版本"锁死"了所有曾经引用过的文件 |

---

## 二、Tag 描述图片在 tag.original_text 与 revision.content 双重持有时的清理覆盖

### 2.1 Tag 的双存储模型

Tag 描述与 Question/Answer 一样采用双字段存储：

[tag_entity.go#L43-L44](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/entity/tag_entity.go#L43-L44)

```go
OriginalText    string    `xorm:"not null MEDIUMTEXT original_text"`  // Markdown
ParsedText      string    `xorm:"not null MEDIUMTEXT parsed_text"`    // 预渲染 HTML
```

预渲染在 Schema Check() 中完成 [tag_schema.go#L211-L213](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/schema/tag_schema.go#L211-L213)

```go
func (r *UpdateTagReq) Check() (errFields []*validator.FormErrorField, err error) {
    r.ParsedText = converter.Markdown2HTML(r.OriginalText)
    return nil, nil
}
```

### 2.2 Tag Revision 的写入

Tag 也写 revision [tag_service.go#L349-L362](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/tag/tag_service.go#L349-L362)

```go
// update tag revision
for _, tag := range needAddTagList {
    revisionDTO := &schema.AddRevisionDTO{
        UserID:   req.UserID,
        ObjectID: tag.ID,
        Title:    tag.SlugName,
    }
    tagInfoJson, _ := json.Marshal(tag)   // 整个 Tag 实体 JSON 序列化
    revisionDTO.Content = string(tagInfoJson)
    revisionID, err := ts.revisionService.AddRevision(ctx, revisionDTO, true)
}
```

`allowRecord()` 确认 Tag 记录 revision [revision_repo.go#L194-L195](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/repo/revision/revision_repo.go#L194-L195)

```go
case constant.ObjectTypeStrMapping["tag"]:
    return true
```

### 2.3 清理覆盖分析

Tag 描述图片的 URL 出现在两处：

```
tag 表                        revision 表
┌──────────────────┐          ┌──────────────────┐
│ original_text    │          │ content (JSON)   │
│  └─ ![img](url)  │          │  └─ original_    │
│                  │          │     text         │
│ parsed_text      │          │     └─ ![img]()  │
│  └─ <img src=>   │          └──────────────────┘
└──────────────────┘
```

**CleanOrphanUploadFiles 的搜索路径覆盖**：

| 搜索方式 | 是否覆盖 Tag | 原因 |
|---------|------------|------|
| `GetLastRevisionByObjectID` | ✅ 部分 | 若 FileRecord 的 ObjectID = tag_id，可找到 tag 的 revision |
| `GetLastRevisionByFileURL` | ✅ 完全 | 在所有 revision 中 LIKE 搜索，tag revision 也会被命中 |

**但有一个前提**：Tag 描述图片上传时使用的 Source 是什么？

如果 Tag 描述的图片上传使用 `user_post` source（和帖子一样走 `post` 子目录），那么它会走普通的 revision 搜索路径，能够被覆盖。

### 2.4 与 Question/Answer 相同的反向漏洞

Tag 描述图片和 Q/A 图片一样，受困于**旧 revision 永久保留**的问题：

1. 用户编辑 Tag 描述，从含图片 A.jpg 改成含图片 B.jpg
2. revision v1 含 A.jpg，revision v2 含 B.jpg
3. 清理时：
   - B.jpg：ObjectID = tag_id，路径 A → 有 revision → 保留 ✓
   - A.jpg：若 ObjectID 已回填，路径 A → 有 revision → 保留（即使已移除）⚠️
   - A.jpg：若 ObjectID 为 "0"，路径 B → revision v1 含 A.jpg → 命中 → 回填 → 保留 ⚠️

**结论**：Tag 描述图片同样受"旧 revision 反向漏洞"影响，曾经使用过的图片永远不会被清理。

### 2.5 Tag 删除后的情况

Tag 有软删除状态 `TagStatusDeleted = 10` [tag_entity.go#L26](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/entity/tag_entity.go#L26)。

Tag 被软删除后：
- `tag` 表记录仍在，`status = 10`
- `revision` 表记录仍在（不会因 tag 删除而删除）
- 路径 A（ObjectID 已回填）：`GetLastRevisionByObjectID(tag_id)` 仍能找到 revision → 文件保留
- 路径 B（ObjectID 为 "0"）：`GetLastRevisionByFileURL` 仍能命中 tag revision → 文件保留

**结论**：即使 Tag 被软删除，它引用过的图片文件仍然不会被清理。

---

## 三、Question/Answer 永久删除后残留 Revision 与所引用文件的最终归宿

### 3.1 删除方式：软删除为主

**Question 删除** [question_service.go#L595-L596](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/content/question_service.go#L595-L596)

```go
questionInfo.Status = entity.QuestionStatusDeleted
err = qs.questionRepo.UpdateQuestionStatusWithOutUpdateTime(ctx, questionInfo)
```

**Answer 删除** [answer_repo.go#L88-L98](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/repo/answer/answer_repo.go#L88-L98)

```go
func (ar *answerRepo) RemoveAnswer(ctx context.Context, answerID string) (err error) {
    answerID = uid.DeShortID(answerID)
    _, err = ar.data.DB.Context(ctx).ID(answerID).Cols("status").Update(&entity.Answer{
        Status: entity.AnswerStatusDeleted,
    })
    ...
}
```

**结论**：用户/管理员触发的删除都是**软删除**，只更新 `status = 10 (Deleted)`，不物理删除记录。

> 注意：`questionRepo.RemoveQuestion` 虽然叫 "Remove"，但实际是 `DELETE FROM question` 的硬删除。但这个方法**只在测试代码中调用**，业务代码走的是 `UpdateQuestionStatusWithOutUpdateTime` 软删除路径。

### 3.2 软删除后 Revision 的命运

**Revision 不会被级联删除**。软删除 Q/A 时：
- `question` / `answer` 表：`status` 更新为 `Deleted`
- `revision` 表：**无任何变化**，所有历史版本完整保留

证据：`RemoveQuestion` / `RemoveAnswer` 的代码中**没有任何删除 revision 的逻辑**。

### 3.3 软删除后文件的命运

| 搜索路径 | 软删除后是否命中 | 原因 |
|---------|----------------|------|
| 路径 A：ObjectID 已回填 → `GetLastRevisionByObjectID` | ✅ 命中 | revision 记录仍存在，按 object_id 查询能找到 |
| 路径 B：ObjectID 为 "0" → `GetLastRevisionByFileURL` | ✅ 命中 | 所有 revision 中 LIKE 搜索，仍能找到 |

**结论**：Question/Answer 被软删除后，**它引用的所有图片文件都不会被清理**。

### 3.4 设计意图：支持"恢复"功能

软删除 + 保留 revision + 保留文件 = 支持**完整恢复**（Undeletion）。

- [migrations/v17.go#L35](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/migrations/v17.go#L35) 有 `recover question` 权限
- [question_common.go#L276](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/question_common/question.go#L276) 有 `UnDeleteQuestion` 方法
- Answer 也有 `RecoverAnswer` 方法 [answer_repo.go#L100-L111](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/repo/answer/answer_repo.go#L100-L111)

**恢复流程**：
```
软删除 (status = Deleted)
  → 记录仍在 DB 中
  → revision 仍在
  → 文件仍在磁盘上
  → 管理员可以 "恢复" (status = Available)
  → 所有内容（含图片）完整恢复
```

### 3.5 硬删除的可能性

如果未来引入硬删除（物理删除 question/answer 记录），需要考虑：

**问题 1：revision 是否级联删除？**
- 如果级联删除 → 文件清理的路径 A 失效（ObjectID 查不到），但路径 B 可能仍能命中其他 revision
- 如果不级联 → 和软删除一样，文件永远保留

**问题 2：文件是否同步删除？**
- 如果同步删除 → 与 Q/A 记录一起删除所有引用的文件（需要解析 content 提取 URL）
- 如果不同步删除 → 依赖 `CleanOrphanUploadFiles`，但因 revision 仍在而清不掉

### 3.6 最终归宿总结

```
用户软删除 Question/Answer
  │
  ├─ question/answer 表: status = Deleted (软删除)
  ├─ revision 表: 全部保留 (不动)
  ├─ FileRecord: 全部保留 (不动)
  ├─ 磁盘文件: 全部保留 (不动)
  │
  └─ 清理扫描 CleanOrphanUploadFiles
       │
       ├─ 路径 A (ObjectID != 0): GetLastRevisionByObjectID
       │    → 找到 revision → exist = true → 保留文件
       │
       └─ 路径 B (ObjectID = 0): GetLastRevisionByFileURL
            → 搜到 revision → exist = true → 回填 ObjectID → 保留文件

结果: 软删除的帖子引用的所有图片文件永远保留在磁盘上，
      直到 revision 记录被物理删除（当前代码中不会发生）
```

### 3.7 对比 Branding 的同步清理模式

Branding 文件有**修改时同步清理旧文件**的机制 [siteinfo_service.go#L657-L699](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/siteinfo/siteinfo_service.go#L657-L699)，而 Q/A/Tag 没有。

| 对象 | 删除/替换时同步清理 | 定时清理兜底 |
|------|-------------------|-------------|
| Branding | ✅ `CleanUpRemovedBrandingFiles` | ✅ `IsBrandingFileUsed` |
| Avatar | ❌ 无 | ✅ `IsAvatarFileUsed` |
| Question | ❌ 无 | ❌ 永远命中（revision 保留） |
| Answer | ❌ 无 | ❌ 永远命中（revision 保留） |
| Tag | ❌ 无 | ❌ 永远命中（revision 保留） |
| Comment | ❌ 无 | ❌ 搜不到（无 revision，误删） |

---

## 四、全局视野：文件清理的三类状态

综合所有分析，文件清理系统可以分为三种典型状态：

```
┌─────────────────────────────────────────────────────────────────┐
│                    文件引用追踪全景图                             │
├──────────────┬──────────────────┬──────────┬───────────────────┤
│     对象     │   引用位置       │ 能否命中 |    清理结果       │
├──────────────┼──────────────────┼──────────┼───────────────────┤
│ Question     │ revision.content │  ✅ 命中  │ ❌ 永远不清理     │
│  (含旧版本)  │  (所有历史版本)  │          │  (反向漏洞)       │
├──────────────┼──────────────────┼──────────┼───────────────────┤
│ Answer       │ revision.content │  ✅ 命中  │ ❌ 永远不清理     │
│  (含旧版本)  │  (所有历史版本)  │          │  (反向漏洞)       │
├──────────────┼──────────────────┼──────────┼───────────────────┤
│ Tag          │ revision.content │  ✅ 命中  │ ❌ 永远不清理     │
│  (含旧版本)  │  (所有历史版本)  │          │  (反向漏洞)       │
├──────────────┼──────────────────┼──────────┼───────────────────┤
│ Comment      │ comment.original │  ❌ 未命中 │ ⚠️ 48h 后误删    │
│              │  _text           │          │  (盲区漏洞)       │
├──────────────┼──────────────────┼──────────┼───────────────────┤
│ User Bio     │ user.bio         │  ❌ 未命中 │ ⚠️ 48h 后误删    │
│              │                  │          │  (盲区漏洞)       │
├──────────────┼──────────────────┼──────────┼───────────────────┤
│ Branding     │ site_info.content│  ✅ 命中  │ ✅ 同步+定时正常  │
│              │  (专用检查)      │          │                   │
├──────────────┼──────────────────┼──────────┼───────────────────┤
│ Avatar       │ user.avatar      │  ✅ 命中  │ ✅ 定时正常       │
│              │  (专用检查)      │          │                   │
└──────────────┴──────────────────┴──────────┴───────────────────┘
```

### 核心设计矛盾

`CleanOrphanUploadFiles` 的设计基于一个假设：**revision 表是所有正文引用的单一真相来源**。

这个假设导致了两个相反方向的问题：

1. **盲区（漏匹配）**：Comment 和 User Bio 的图片不在 revision 中 → 被误删
2. **反向（过匹配）**：Q/A/Tag 的旧 revision 永久保留 → 曾经出现的图片永远不删

### 修复方向

**修复盲区（漏匹配）**：
- 为 Comment 增加专用搜索路径（`commentRepo.IsFileURLUsed()`）
- 为 User Bio 增加专用搜索路径（`userRepo.IsBioFileUsed()`）

**修复反向（过匹配）**：
- `GetLastRevisionByObjectID` 路径需要额外检查最新 revision 的 content 是否包含该 URL
- `GetLastRevisionByFileURL` 路径应只搜索**最新版本**（每个 object_id 只取最新一条）的 content，而非所有历史版本
- 或者：清理时只检查对象当前的 `original_text` 字段，不查 revision
