# 标签体系与关注标签 代码理解分析

## 一、整体架构概览

标签体系采用**前后端分离 + 分层架构**设计，后端基于 Go 语言 Gin + XORM，前端基于 React。整体分层如下：

```
前端层:
  ├── TagSelector 组件          (标签搜索/选择/创建入口)
  ├── FollowingTags 组件        (用户关注标签管理面板)
  ├── MergeTagModal 组件        (标签合并弹窗)
  └── services/client/tag.ts    (前端 API 封装)

Controller 层:
  └── tag_controller.go         (HTTP 路由入口，权限校验)

Service 层:
  ├── tag_service.go            (标签业务主服务: 删除/恢复/合并/同义词/关注列表)
  └── tag_common.go             (标签通用服务: 归一化/创建/更新/关联维护/计数)

Repo 层:
  ├── tag_repo.go               (标签主表 CRUD，同义词维护)
  ├── tag_common_repo.go        (标签通用查询，软删除恢复复用)
  ├── tag_rel_repo.go           (标签-对象关联表维护，迁移)
  └── follow.go                 (关注关系存储，跟随者迁移)

Entity 层:
  ├── tag_entity.go             (tag 表实体)
  ├── tag_rel_entity.go         (tag_rel 关联表实体)
  └── activity_entity.go        (activity 关注活动表实体)
```

---

## 二、标签实体与数据模型

### 2.1 Tag 主表 [tag_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/entity/tag_entity.go)

| 字段 | 类型 | 说明 |
|------|------|------|
| `ID` | BIGINT(20) PK | 标签唯一 ID |
| `SlugName` | VARCHAR(35) UNIQUE | **标签 URL 标识名（唯一索引，归一化存储）** |
| `DisplayName` | VARCHAR(35) | 展示名称 |
| `MainTagID` | BIGINT(20) | **主标签ID，>0 表示该标签是同义词** |
| `MainTagSlugName` | VARCHAR(35) | **主标签的 SlugName（冗余存储，便于查询）** |
| `OriginalText` / `ParsedText` | MEDIUMTEXT | 描述原文(Markdown)与渲染HTML |
| `FollowCount` | INT | 关注人数（缓存计数） |
| `QuestionCount` | INT | 关联问题数（缓存计数） |
| `Status` | INT | 状态: 1=可用, 10=已删除 |
| `Recommend` | BOOL | 是否推荐标签（站点级强制标签） |
| `Reserved` | BOOL | 是否保留标签（不可移除） |
| `UserID` | BIGINT | 创建者 |

**关键设计要点：**
- `MainTagID` 字段是同义词机制的核心。**所有标签分两类：主标签（MainTagID=0）和同义词标签（MainTagID>0）**。
- `SlugName` 有唯一索引，全局不允许重复。

### 2.2 TagRel 关联表 [tag_rel_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/entity/tag_rel_entity.go)

| 字段 | 类型 | 说明 |
|------|------|------|
| `ID` | BIGINT PK autoincr | 关联 ID |
| `ObjectID` | BIGINT INDEX UNIQUE(s) | 关联对象 ID（目前是 Question ID） |
| `TagID` | BIGINT INDEX UNIQUE(s) | 标签 ID |
| `Status` | INT | 状态: 1=可用, 2=隐藏, 10=已删除 |

**关键约束：** `(ObjectID, TagID)` 构成逻辑唯一键，通过数据库 `UNIQUE(s)` 联合索引保证**同一问题不能重复打同一标签**。

### 2.3 Activity 关注表 [activity_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/entity/activity_entity.go)

关注关系**不使用独立表**，而是通过 `activity` 表的 `activity_type=follow` 存储：

| 字段 | 说明 |
|------|------|
| `UserID` | 关注者用户 ID |
| `ObjectID` | 被关注对象 ID（标签 ID/问题 ID/用户 ID） |
| `ActivityType` | 活动类型（区分 follow/vote 等） |
| `Cancelled` | 0=有效关注, 1=已取消 |
| `OriginalObjectID` | 原始对象 ID（用于合并溯源） |

---

## 三、标签归一化机制

标签归一化是防止标签冗余、重复的核心机制。**归一化分为 4 层**，分别位于：前端输入层、Schema 校验层、Service 业务层、Repo 数据库层。

### 3.1 归一化规则

归一化的对象是 `SlugName`，规则统一为：
```
1. 全部转为小写: strings.ToLower()
2. 空格替换为连字符: strings.ReplaceAll(s, " ", "-")
3. 去重校验（基于唯一索引）
```

### 3.2 各层归一化的位置

**第一层：Schema 校验层（请求入参）**

- [tag_schema.go#L66-L69](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/schema/tag_schema.go#L66-L69): `GetTagInfoReq.Check()` 将 `Name` 转小写
- [tag_schema.go#L181-L185](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/schema/tag_schema.go#L181-L185): `AddTagReq.Check()` 将 `SlugName` 转小写 + Markdown 渲染
- [tag_schema.go#L282-L286](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/schema/tag_schema.go#L282-L286): `UpdateTagSynonymReq.Format()` 将所有同义词 `SlugName` 转小写

**第二层：Service 业务层（标签创建/变更）**

- [tag_common.go#L342-L384](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L342-L384): `AddTag()` 中双重归一化：
  ```go
  slugName := strings.ReplaceAll(req.SlugName, " ", "-")
  slugName = strings.ToLower(slugName)
  ```
- [tag_common.go#L660-L742](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L660-L742): `ObjectChangeTag()` 中：
  - 对入参每个 `Tag.SlugName` 先 `ToLower` (L677)
  - 对不存在的新标签创建时，再做 `ReplaceAll(空格, "-")` (L700)
- [tag_common.go#L886-L887](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L886-L887): `UpdateTag()` 更新时，同样进行两次归一化

**第三层：前端组件层（用户输入）**

- [TagSelector/index.tsx#L113-L120](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/ui/src/components/TagSelector/index.tsx#L113-L120): `filterTags()` 去重比较时转小写
- [TagSelector/index.tsx#L182-L184](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/ui/src/components/TagSelector/index.tsx#L182-L184): `handleClick()` 选择时转小写比较
- [TagSelector/index.tsx#L208-L210](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/ui/src/components/TagSelector/index.tsx#L208-L210): `handleRemove()` 移除时转小写比较

**第四层：Repo 查询层（兼容兜底）**

- [tag_common_repo.go#L68-L77](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/tag_common/tag_common_repo.go#L68-L77): `GetTagBySlugName()` 查询时使用 `LOWER(slug_name) = ?` 进行数据库级不区分大小写匹配

### 3.3 归一化的风险点

虽然有多道防线，但仍有**潜在不一致**：
1. Schema 层的 `AddTagReq.Check()` 只做了 `ToLower`，**没有做空格替换**，空格替换在 Service 层才做，若调用方绕过 Service 直接用 Repo 会有不一致。
2. 前端 `useTagModal` 传入的标签名若包含特殊字符，只有空格和大小写被处理，其他字符（如下划线、点号）未被处理。
3. 同义词的 SlugName 归一化在 `UpdateTagSynonym` 中只做了 `ToLower`（L284），**没有做空格替换**，与创建标签时的规则不完全一致。

---

## 四、标签创建流程

### 4.1 独立创建标签（AddTag API）

**入口：** `POST /answer/api/v1/tag` → [tag_controller.go#L128-L149](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/controller/tag_controller.go#L128-L149)

流程：
```
1. Controller: 校验 rank 权限 permission.TagAdd
2. Schema.Check(): SlugName → ToLower, OriginalText → ParsedText
3. TagCommonService.AddTag():
   a. 按 SlugName 查询是否已存在 → 存在返回 TagAlreadyExist
   b. 归一化: 空格→"-" + ToLower
   c. 构造 entity.Tag 对象
   d. 调用 tagCommonRepo.AddTagList() 插入
      ├─ 先尝试 updateDeletedTag: 若同 SlugName 的已删除标签存在 → 直接恢复
      └─ 否则生成唯一 ID 后 Insert
   e. 添加修订记录 revisionService.AddRevision()
   f. 发送活动队列 ActTagCreated
4. 返回 AddTagResp{SlugName}
```

**关键特性：软删除标签复用** — [tag_common_repo.go#L251-L267](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/tag_common/tag_common_repo.go#L251-L267) 中的 `updateDeletedTag()` 逻辑：当新增一个已被软删除的同名标签时，**不是新建记录，而是将旧记录的 Status 从 10 恢复为 1**，保留原 ID。

### 4.2 提问/编辑时动态创建标签（ObjectChangeTag）

用户在提问时若输入了系统中不存在的标签，会被**自动创建**。

**入口：** `question_service.go` → 调用 [tag_common.go#L660-L742](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L660-L742) `ObjectChangeTag()`

流程：
```
1. 检查最少标签数 < minimumTags → 返回错误 TagMinCount
2. 对所有入参 SlugName → ToLower
3. GetTagListByNames 批量查询已存在的标签
4. 未找到的标签 → 构造新 entity.Tag（空格→"-"）
5. 批量 AddTagList 插入新标签（同样触发软删除恢复）
6. 为每个新标签添加 revision 和 Activity(ActTagCreated)
7. 调用 CreateOrUpdateTagRelList 更新关联
```

---

## 五、标签与问题的关联维护

### 5.1 关联更新核心函数

`CreateOrUpdateTagRelList` 是关联维护的核心原子操作，位于 [tag_common.go#L798-L864](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L798-L864)。

输入：`objectId`（问题ID）+ `tagIDs`（新标签ID列表）

内部逻辑分为 4 步：
```
第 1 步：计算差集
  ├── 获取该 Object 的旧关联列表 oldTagRelList
  ├── 旧关联中不在新 tagIDs 内的 → 标记为 deleteTagRel（待软删除）
  └── 收集所有需要刷新计数的 tagID（新+旧）

第 2 步：获取关联默认状态
  └── GetTagRelDefaultStatusByObjectID() 根据问题的 Show/Status
      决定关联 Status 是 Available(1) 还是 Hide(2)

第 3 步：三类更新（按需执行）
  ├── a. deleteTagRel 非空 → RemoveTagRelListByIDs() 软删除关联
  ├── b. 新 tagIDs 中无历史关联的 → AddTagRelList() 插入
  └── c. 有历史关联但被删除过的 → EnableTagRelByIDs() 恢复

第 4 步：刷新计数
  └── RefreshTagQuestionCount() 对所有涉及的标签重新计算 question_count
```

### 5.2 关联维护的状态流转

`TagRel` 的状态有三种：
```
Available(1) ←→ Hide(2)     // 随问题的显隐联动
     ↓                  ↑
  Deleted(10) → 恢复 → Available/Hide
```

**联动机制：**
- 问题被隐藏/删除时，问题关联的所有标签关联也被 Hide（[tag_rel_repo.go#L88-L95](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/tag/tag_rel_repo.go#L88-L95)）
- 问题被恢复时，关联也被 Show（[tag_rel_repo.go#L97-L104](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/tag/tag_rel_repo.go#L97-L104)）
- 关联默认状态查询 [tag_rel_repo.go#L196-L207](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/tag/tag_rel_repo.go#L196-L207) 中**硬编码了 question 表的字段**（show/status），说明 TagRel 目前只与 Question 绑定。

### 5.3 question_count 计数维护

**计数刷新入口：** `RefreshTagQuestionCount` [tag_common.go#L748-L762](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L748-L762)

```go
for _, tagID := range tagIDs {
    count, err := ts.tagRelRepo.CountTagRelByTagID(ctx, tagID)  // 实时 COUNT SQL
    ts.tagCommonRepo.UpdateTagQuestionCount(ctx, tagID, int(count))  // 写回 tag 表
}
```

**刷新触发时机：**
- 每次 `CreateOrUpdateTagRelList` 执行后（所有涉及标签全部刷新）
- 每次问题标签变化后（`RefreshTagCountByQuestionID`）
- 标签合并完成后（`MergeTag` 最后一步）

### 5.4 同义词关联的查询重定向

用户搜索同义词标签时，在 `SearchTagLike()` [tag_common.go#L114-L165](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L114-L165) 中会做**查询结果归一化**：
- 如果搜索到的标签有 `MainTagID != 0`（是同义词），则把返回结果中的 `ID/SlugName/DisplayName` 全部**替换为主标签的信息**
- 最后用 `repetitiveTag` map 按 SlugName 去重

这样用户选到同义词标签时，实际关联的是主标签 ID。

---

## 六、标签选择器（前端 TagSelector）

### 6.1 组件定位

[TagSelector/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/ui/src/components/TagSelector/index.tsx) 是全站标签选择的唯一入口组件，被以下场景复用：
- 提问页：选择/创建问题标签
- 编辑问题：修改标签
- 关注标签面板：选择要关注的标签（hiddenCreateBtn=true）
- 管理后台：标签设置

### 6.2 核心交互流程

```
用户输入 → handleSearch()
  ├─ 400ms debounce → fetchTags(searchValue)
  │     └─ queryTags(str) 调用 GET /answer/api/v1/question/tags
  │         ↓
  │     SearchTagLike 后端返回（同义词→主标签归一化）
  │         ↓
  │     filterTags() 过滤已选（不区分大小写比较 slug_name）
  │         ↓
  │     渲染 Dropdown 候选项
  │
  ├─ 回车 Enter:
  │   ├─ 有候选结果 → handleClick(tags[currentIndex]) 选中
  │   └─ 无结果 + 有权限 → tagModal.onShow(searchValue) 弹出创建新标签
  │
  └─ 候选点击 → handleClick(val)
        ├─ 检查未重复 → onChange([...value, val])
        └─ 已重复 → 2s 闪烁高亮重复的标签

点击标签 × 按钮 → handleRemove(val)
  └─ onChange(value.filter(...))
```

### 6.3 权限控制

```tsx
const { data: userPermission } = useUserPermission('tag.add');
```
创建按钮是否显示取决于 `tag.add` 权限。无权限时只选不创。

---

## 七、关注标签实现

### 7.1 关注/取消关注

关注是通用机制，通过 Activity 表实现，标签、用户、问题共用。

**关注操作存储：** `follow.go` 中通过写入 `activity` 表 `activity_type=follow` 的记录实现。取消关注不是删除记录，而是 `Cancelled=1`。

### 7.2 获取用户关注标签列表

**API：** `GET /answer/api/v1/tags/following` → [tag_controller.go#L297-L301](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/controller/tag_controller.go#L297-L301)

实现位于 [tag_service.go#L214-L249](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L214-L249) `GetFollowingTags()`：
```
1. followCommon.GetFollowIDs(userID, "tag")
   └─ SELECT object_id FROM activity
      WHERE user_id=? AND activity_type=? AND cancelled=0
2. tagCommonService.GetTagListByIDs(objIDs) 批量查询标签实体
3. 对每个标签：
   ├─ 若 MainTagID > 0（同义词）→ 查主标签并返回主标签的 SlugName
   └─ 组装 GetFollowingTagsResp
```

### 7.3 更新关注标签（批量）

**API：** `PUT /answer/api/v1/follow/tags` → `UpdateFollowTagsReq{SlugNameList[]}`

前端组件 [FollowingTags/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/ui/src/components/FollowingTags/index.tsx)：
- 进入编辑态 → 渲染 TagSelector（hiddenCreateBtn=true，禁用创建）
- 保存时 `followTags({ slug_name_list: newTags })` 提交

### 7.4 单个标签的关注状态判断

`checkTagIsFollow` [tag_service.go#L497-L507](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L497-L507)：
- 查询 `activity` 表当前用户是否有 `cancelled=0` 的 follow 记录
- 在 `GetTagInfo`、`GetTagWithPage` 等返回列表中，为每个标签填充 `IsFollower` 字段

### 7.5 follow_count 计数维护

与 question_count 不同，**follow_count 的维护不在标签通用服务层**，而是在活动操作时通过 `GetFollowAmount` [follow.go#L58-L91](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity_common/follow.go#L58-L91) 直接读 tag 表的 `follow_count` 字段返回。实际更新由活动处理流程触发。

---

## 八、标签合并与删除风险

### 8.1 标签合并（MergeTag）

**API：** `POST /answer/api/v1/tag/merge`，**仅管理员/版主可用**。

前端入口：[MergeTagModal/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/ui/src/pages/Tags/Info/components/MergeTagModal/index.tsx)

后端实现：[tag_service.go#L439-L495](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L439-L495) `MergeTag()`

**合并 5 步流程：**
```
1. 查询源标签 + 源标签的所有同义词（MainTagID=源标签ID）
   → 将源标签及其同义词组成 addSynonymTagList

2. 查询目标标签（必须存在）

3. 更新源标签和其所有同义词的 MainTagID = 目标标签ID
   → UpdateTagSynonym() 把它们全部变为目标标签的同义词

4. 迁移关注者: followCommon.MigrateFollowers(源ID → 目标ID, "follow")
   [follow.go#L163-L265]
   ├─ 事务操作
   ├─ (a) 删除源标签的所有 follow activity 记录
   ├─ (b) 对源标签关注者中已取消关注目标标签的 → 恢复为关注
   ├─ (c) 过滤出目标标签尚未有关注的用户
   └─ (d) 为这些用户创建目标标签的 follow activity

5. 迁移问题关联: MigrateTagQuestions(源ID → 目标ID)
   [tag_rel_repo.go#L209-L261]
   ├─ 事务操作
   ├─ (a) 查源标签的所有 TagRel 记录
   ├─ (b) 查目标标签已有的 TagRel，构造 existingMap 去重
   ├─ (c) 对不重复的对象 → 插入新的目标 TagRel（继承原 Status）
   ├─ (d) DELETE 源标签的所有 TagRel（物理删除，非软删除）
   └─ 刷新源+目标标签的 question_count
```

### 8.2 合并的风险点

| 风险 | 位置 | 说明 |
|------|------|------|
| **重复关注计数** | [follow.go#L192-L197](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity_common/follow.go#L192-L197) | 步骤 4a 中直接 DELETE 源标签的 follow 记录，**没有同步更新源标签的 follow_count 字段**。合并后源标签 `follow_count` 仍是旧值（但因为变成同义词不再展示，影响有限）。 |
| **物理删除关联记录** | [tag_rel_repo.go#L251-L255](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/tag/tag_rel_repo.go#L251-L255) | 步骤 5d 用的是 `DELETE FROM tag_rel`（物理删除），而全站标准做法是软删除（Status=10）。若合并操作需要回滚，无法通过 Status 恢复。 |
| **无事务包裹 5 步整体** | [tag_service.go#L439-L495](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L439-L495) | 5 个步骤中，步骤 4（关注者迁移）和步骤 5（问题关联迁移）各自内部有事务，但**5 步之间没有外层事务**。若步骤 3 完成（同义词 MainTagID 已改），步骤 4 或 5 失败，会出现数据不一致。 |
| **无法合并到同义词** | 隐含 | 合并时目标标签是否也可以是同义词？代码中 `GetTagByID(targetTagID)` 不校验 `MainTagID=0`。若目标标签本身是同义词，会出现"同义词的同义词"的多级结构，而查询层只做一级跳转。 |
| **自合并校验缺失** | 隐含 | 未校验 `SourceTagID != TargetTagID`，若管理员选了同一标签做源和目标，步骤 5 会先删除自己的关联再插入，可能出问题。 |
| **关注者的 CreateAt 变化** | [follow.go#L247-L260](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/activity_common/follow.go#L247-L260) | 迁移时 `CreatedAt=time.Now()`，丢失原始关注时间。 |

### 8.3 标签删除（RemoveTag）

**API：** `DELETE /answer/api/v1/tag`

实现：[tag_service.go#L75-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L75-L107)

**删除前置校验：**
```go
// 1. 有关联问题 → 不允许删除
tagCount := CountTagRelByTagID(tagID)
if tagCount > 0 → return TagIsUsedCannotDelete

// 2. 有同义词标签 → 不允许删除
tagSynonymCount := GetTagSynonymCount(tagID)
if tagSynonymCount > 0 → return TagIsUsedCannotDelete
```

**删除方式：软删除** — 只更新 `tag.status = TagStatusDeleted(10)`，保留记录。

### 8.4 删除的风险点

| 风险 | 说明 |
|------|------|
| **删除后软删除记录的 SlugName 仍占唯一索引** | 但通过 `updateDeletedTag()` 机制，新建同名标签时会自动恢复这条记录（见 4.1），整体安全。 |
| **删除后关注者残留** | 删除标签只改了 tag.status=10，**没有清理 activity 表的 follow 记录**。调用 `GetFollowIDs()` 拿到 ID 后 `GetTagListByIDs()` 会因为 status=10 过滤掉，所以功能上不影响列表展示，但 DB 中有脏数据。 |
| **删除时 revision 未处理** | 删除操作没有在 revision 表中做"已删除版本"记录，若恢复时 revision 断档。 |
| **保留标签不可删除缺失校验** | `RemoveTag` 没有检查 `Reserved=true`，理论上可以删除保留标签。 |

---

## 九、同义词机制详解

### 9.1 设置同义词（UpdateTagSynonym）

**API：** `PUT /answer/api/v1/tag/synonym`

实现：[tag_service.go#L298-L400](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L298-L400)

流程：
```
1. req.Format(): 所有同义词 SlugName → ToLower
2. 校验：主标签 SlugName 不能出现在同义词列表
3. 批量查询已存在的同义词标签
4. 对数据库中不存在的同义词 → 作为新标签插入（AddTagList）
5. 查询旧同义词列表，计算需要移除的同义词
   └─ UpdateTagSynonym(removeList, MainTagID=0, "")  → 取消同义词关系
6. 对所有（新增+已存在）同义词
   └─ UpdateTagSynonym(addList, MainTagID=主标签ID, MainTagSlugName=主标签Slug)
```

### 9.2 同义词的查询透明化

用户搜索同义词 → 自动转为搜索主标签（见 `SearchTagLike`）。
用户访问同义词 slug 详情页 → `GetTagInfo` [tag_service.go#L164-L174](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L164-L174) 中 `if MainTagID>0` 会重定向到主标签信息。

### 9.3 同义词风险

| 风险 | 说明 |
|------|------|
| **主标签变成同义词的链式风险** | 若设置了 A 的同义词 B，后又把 A 设置成 C 的同义词 → 则 B 的 MainTagID 仍指向 A（已变同义词），出现多级链。查询时只跳一级，会导致 B 展示到已变同义词的 A 的信息而非最终 C。 |
| **MainTagSlugName 同步** | 主标签 SlugName 修改时 `UpdateTag()` [tag_common.go#L914-L928](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L914-L928) 只更新了其同义词的 MainTagSlugName。但若**同义词的同义词**（多级）不会被级联更新。 |

---

## 十、API 速查

| 方法 | 路径 | 说明 | 鉴权 |
|------|------|------|------|
| GET | `/answer/api/v1/question/tags` | 搜索标签（TagSelector 用） | 否 |
| GET | `/answer/api/v1/tags` | 按 slug_names 批量查标签 | 否 |
| GET | `/answer/api/v1/tags/page` | 分页浏览标签（按 popular/name/newest） | 否 |
| GET | `/answer/api/v1/tag` | 获取单个标签详情 | 否 |
| POST | `/answer/api/v1/tag` | 创建标签 | TagAdd 权限 |
| PUT | `/answer/api/v1/tag` | 修改标签 | TagEdit 权限 |
| DELETE | `/answer/api/v1/tag` | 删除标签 | TagDelete 权限 |
| POST | `/answer/api/v1/tag/recover` | 恢复删除标签 | TagUnDelete 权限 |
| GET | `/answer/api/v1/tag/synonyms` | 查询标签同义词 | 否 |
| PUT | `/answer/api/v1/tag/synonym` | 设置同义词列表 | TagSynonym 权限 |
| POST | `/answer/api/v1/tag/merge` | 合并两个标签 | 管理员/版主 |
| GET | `/answer/api/v1/tags/following` | 获取我的关注标签 | 登录 |
| PUT | `/answer/api/v1/follow/tags` | 批量更新关注标签 | 登录 |

---

## 十一、总结：核心协作关系图

```
用户操作 (前端)
    │
    ├─ [提问/编辑问题] ──────────────────────────┐
    │   TagSelector 选标签/创建标签               │
    │         │                                   │
    │         ▼                                   ▼
    │   POST /question           ObjectChangeTag() ── 校验最少标签数
    │         │                   │                    ├─ 不存在的标签自动创建
    │         │                   │                    └─ CreateOrUpdateTagRelList()
    │         │                   │                         ├─ 新增/删除/恢复关联
    │         │                   │                         └─ RefreshTagQuestionCount()
    │         │                   │
    ├─ [标签详情页]                │
    │   创建/编辑/删除/合并        │
    │         │                   │
    │         ▼                   │
    │   TagController             │
    │         │                   │
    │         ▼                   ▼
    │   TagService ────────► TagCommonService
    │         │                   │
    │         │                   ├─ AddTag/UpdateTag
    │         │                   ├─ CreateOrUpdateTagRelList
    │         │                   └─ RefreshTagQuestionCount
    │         │
    │         ├─ RemoveTag()      → 前置校验: 无关联 + 无同义词
    │         ├─ MergeTag()       → 5 步: 改同义词/迁关注/迁关联/刷计数
    │         ├─ GetFollowingTags → Activity follow 查询
    │         └─ UpdateTagSynonym → 增/删同义词关系
    │
    └─ [侧边栏关注标签面板]
        FollowingTags 组件
              │
              ▼
        TagSelector (hiddenCreateBtn)
              │
              ▼
        PUT /follow/tags → Activity follow 批量增删
```

---

## 十二、改进建议

1. **合并操作外层事务**：`MergeTag()` 中 5 个步骤应包裹在**单个数据库事务**中，防止部分成功部分失败。
2. **自合并与目标同义词校验**：在 `MergeTag` 入口增加 `SourceTagID != TargetTagID` 校验，以及 `TargetTag.MainTagID == 0`（目标必须是主标签）校验。
3. **TagRel 软删除**：`MigrateTagObjects` 步骤 5d 从物理删除改为 `Status=10` 软删除，保留审计轨迹。
4. **多级同义词防环**：设置同义词时递归检查，不允许将已有主标签的标签再设为主标签，防止出现多级链式。
5. **保留标签删除保护**：`RemoveTag` 增加 `Reserved=true` 拦截。
6. **删除标签清理关注**：删除标签时，清理 activity 表中该标签的 follow 记录（或标记 cancelled=1），并同步所有关注者的关系。
7. **归一化规则统一**：在 tag_schema.go 和 service 层抽出统一的 `NormalizeSlugName()` 函数，所有入口调用同一方法。
