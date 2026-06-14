# 推荐标签强制写入校验与标签 Activity 消费链路分析

## 一、ExistRecommend 在提问场景下的拦截位置

### 1.1 ExistRecommend 函数实现

**实现位置：** [tag_common.go#L272-L295](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L272-L295)

```go
func (ts *TagCommonService) ExistRecommend(ctx context.Context, tags []*schema.TagItem) (bool, error) {
    // Step 1: 读取站点标签配置
    taginfo, err := ts.siteInfoService.GetSiteTag(ctx)
    if err != nil {
        return false, err
    }
    
    // Step 2: 短路判断 —— 如果未开启 RequiredTag 或没有配置推荐标签 → 直接放行
    if !taginfo.RequiredTag || len(taginfo.RecommendTags) == 0 {
        return true, nil
    }
    
    // Step 3: 将入参 SlugName 归一化（空格→"-"）
    tagNames := make([]string, 0)
    for _, item := range tags {
        item.SlugName = strings.ReplaceAll(item.SlugName, " ", "-")
        tagNames = append(tagNames, item.SlugName)
    }
    
    // Step 4: 批量查询标签实体
    list, err := ts.GetTagListByNames(ctx, tagNames)
    if err != nil {
        return false, err
    }
    
    // Step 5: 遍历结果，只要有一个 Recommend=true → 返回 true
    for _, item := range list {
        if item.Recommend {
            return true, nil
        }
    }
    return false, nil
}
```

**返回值语义：**
- `true` = 检查通过（要么不需要检查，要么选了推荐标签）
- `false` = 检查不通过（必须选推荐标签但没选）

### 1.2 提问场景下的三个拦截位置

`ExistRecommend` 在提问流程中被调用了 **3 次**，覆盖了提问的预校验、保存和编辑三个入口：

#### 拦截点 1：提问预校验 CheckAddQuestion()

**位置：** [question_service.go#L262-L274](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/question_service.go#L262-L274)

```go
func (qs *QuestionService) CheckAddQuestion(ctx context.Context, req *schema.QuestionAdd) (...) {
    // ... 内容长度校验 ...
    
    recommendExist, err := qs.tagCommon.ExistRecommend(ctx, req.Tags)
    if !recommendExist {
        // 错误: "Must enter the recommended tag"
        err = errors.BadRequest(reason.RecommendTagEnter)
        return errorlist, err
    }
    
    // ... 保留标签校验 ...
}
```

**调用场景：** 前端点击提问按钮时，先调 CheckAddQuestion 做预校验，返回错误提示给用户。

#### 拦截点 2：提问保存 AddQuestion()

**位置：** [question_service.go#L334-L346](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/question_service.go#L334-L346)

```go
func (qs *QuestionService) AddQuestion(ctx context.Context, req *schema.QuestionAdd) (...) {
    // ... 最少标签数校验 ...
    // ... 最少内容长度校验 ...
    
    recommendExist, err := qs.tagCommon.ExistRecommend(ctx, req.Tags)
    if !recommendExist {
        // 错误: "Must enter the recommended tag"
        err = errors.BadRequest(reason.RecommendTagEnter)
        return errorlist, err
    }
    
    // ... 保留标签校验 ...
    // ... 创建问题 ...
}
```

**调用场景：** 实际提交问题时再次校验，防止绕过前端预校验直接调 API。

#### 拦截点 3：编辑问题 UpdateQuestion()

**位置：** [question_service.go#L1010-L1022](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/question_service.go#L1010-L1022)

```go
func (qs *QuestionService) UpdateQuestion(ctx context.Context, req *schema.QuestionUpdate) (...) {
    // ... 保留标签校验 ...
    
    // Check whether mandatory labels are selected
    recommendExist, err := qs.tagCommon.ExistRecommend(ctx, req.Tags)
    if !recommendExist {
        // 错误: "Must enter the recommended tag"
        err = errors.BadRequest(reason.RecommendTagEnter)
        return errorlist, err
    }
    
    // ... 修订记录 ...
}
```

**调用场景：** 编辑已有问题时，若用户移除了所有推荐标签，也会被拦截。

### 1.3 校验顺序在提问流程中的位置

```
提问/编辑问题 → 校验链：
    ① 最少标签数（minimumTags）          ← GetMinimumTags()
    ② 最少内容长度                        ← siteInfo 配置
    ③ 推荐标签必选（ExistRecommend）      ← 本函数
    ④ 保留标签不可新增/移除               ← AddQuestionCheckTags / CheckChangeReservedTag
```

**注意：** 推荐标签校验排在保留标签校验**之前**。这意味着如果用户同时没选推荐标签且使用了保留标签，会先收到推荐标签的错误提示。

### 1.4 ExistRecommend 的关键细节

| 细节 | 说明 |
|------|------|
| **只查 `tag.Recommend` 字段** | 不查 `tag.Reserved`，保留标签不算推荐标签 |
| **同义词标签的 Recommend** | 取的是同义词标签自身的 `Recommend` 字段，不是主标签的。但由于 `TagsFormatRecommendAndReserved` 在查询时已将同义词的 Recommend 替换为主标签的值（见 2.3 节），实际效果等价 |
| **归一化方式** | 只做了 `strings.ReplaceAll(SlugName, " ", "-")`，**没有 `ToLower`**。而 `GetTagListByNames` 内部做了 `ToLower`，所以不影响匹配 |
| **只要求至少一个** | `for _, item := range list { if item.Recommend { return true } }` —— 只要选了至少一个推荐标签即通过 |

---

## 二、RequiredTag 配置如何生效

### 2.1 配置存储

RequiredTag 存储在 `site_info` 表的 `type="tags"` 配置中：

**Schema 定义：** [siteinfo_schema.go#L126-L132](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/schema/siteinfo_schema.go#L126-L132)

```go
type SiteTagsReq struct {
    ReservedTags  []*SiteWriteTag `json:"reserved_tags"`
    RecommendTags []*SiteWriteTag `json:"recommend_tags"`
    RequiredTag   bool            `json:"required_tag"`    // 核心：是否强制要求推荐标签
    UserID        string          `json:"-"`
}
```

**初始值：** [init.go#L340-L341](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/migrations/init.go#L340-L341)

```go
"required_tag":   false,          // 默认关闭
"recommend_tags": []string{},     // 默认无推荐标签
```

### 2.2 读取路径

```
GetSiteTag()  [siteinfo_common/siteinfo_service.go#L210]
    │
    └─ GetSiteInfoByType(ctx, constant.SiteTypeTags, &SiteTagsResp{})
         │
         └─ 从 site_info 表读取 type="tags" 的 JSON → 反序列化为 SiteTagsResp
              │
              ├─ RequiredTag: bool
              ├─ RecommendTags: []*SiteWriteTag
              └─ ReservedTags: []*SiteWriteTag
```

**SiteTagsResp 与 SiteTagsReq 共享结构**，RecommendTags 中每个元素包含 `SlugName` 和 `DisplayName`。

### 2.3 RequiredTag 的三重生效机制

RequiredTag 不仅在提问校验时生效，还控制着**标签列表中 Recommend 字段的展示**。

#### 生效点 1：ExistRecommend 短路判断

```go
if !taginfo.RequiredTag || len(taginfo.RecommendTags) == 0 {
    return true, nil   // 不强制要求，直接放行
}
```

当 `RequiredTag=false` 时，**所有提问都不需要选推荐标签**。

#### 生效点 2：TagsFormatRecommendAndReserved —— 控制 Recommend 字段展示

**实现位置：** [tag_common.go#L468-L482](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L468-L482)

```go
func (ts *TagCommonService) TagsFormatRecommendAndReserved(ctx context.Context, tagList []*entity.Tag) {
    tagConfig, err := ts.siteInfoService.GetSiteTag(ctx)
    if err != nil {
        log.Error(err)
        return
    }
    if !tagConfig.RequiredTag {
        for _, tag := range tagList {
            tag.Recommend = false   // ⚠️ RequiredTag=false 时，所有标签的 Recommend 都被强制置为 false
        }
    }
}
```

**影响范围：** 该函数在以下场景被调用：
- `SearchTagLike()` — TagSelector 搜索时
- `GetObjectEntityTag()` — 获取问题标签时
- `GetTagPage()` — 分页浏览标签时
- `GetTagInfo()` — 查看标签详情时
- `BatchGetObjectTag()` — 批量获取对象标签时
- `object_info.GetInfo()` — 通用对象信息查询

**效果：** 当 `RequiredTag=false` 时，**整个站点的所有标签在前端都看不到 Recommend 标识**，即使用户调用 API 也看不到。这是一种"软开关"——推荐标签的配置还在数据库中，但开关关闭后全局隐藏。

#### 生效点 3：tagFormatRecommendAndReserved —— 单标签版本

**实现位置：** [tag_common.go#L484-L496](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L484-L496)

```go
func (ts *TagCommonService) tagFormatRecommendAndReserved(ctx context.Context, tag *entity.Tag) {
    tagConfig, err := ts.siteInfoService.GetSiteTag(ctx)
    if !tagConfig.RequiredTag {
        tag.Recommend = false
    }
}
```

在 `AddTag()`、`UpdateTag()` 后返回标签信息时调用。

### 2.4 RequiredTag 生效机制流程图

```
管理员在后台设置:
  RequiredTag = true
  RecommendTags = [{SlugName: "bug", ...}, {SlugName: "feature", ...}]
    │
    ▼
SaveSiteTags() [siteinfo_service.go#L236]
    │
    ├─ CheckTag(recommendTags) 校验不能含同义词
    ├─ CheckTag(reservedTags)  校验不能含同义词
    ├─ recommend tags can't contain reserved tag（互斥去重）
    │
    ├─ SetTagsAttribute(ctx, recommendTags, "recommend")
    │     ├─ oldTags = GetRecommendTagList()  → 旧推荐标签
    │     ├─ UpdateTagsAttribute(oldTags, "recommend", false)  → 取消旧推荐
    │     └─ UpdateTagsAttribute(newTags, "recommend", true)   → 设为新推荐
    │
    ├─ SetTagsAttribute(ctx, reservedTags, "reserved")
    │     └─ （同理）
    │
    └─ 保存 site_info 表 JSON
         └─ required_tag: true, recommend_tags: [...], reserved_tags: [...]
```

### 2.5 UpdateTagsAttribute 的数据库操作

**实现位置：** [tag_common_repo.go#L279-L289](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/repo/tag_common/tag_common_repo.go#L279-L289)

```go
func (tr *tagCommonRepo) UpdateTagsAttribute(ctx context.Context, tags []string, attribute string, value bool) error {
    bean := &entity.Tag{}
    switch attribute {
    case "recommend":
        bean.Recommend = value
    case "reserved":
        bean.Reserved = value
    }
    session := tr.data.DB.Context(ctx).In("slug_name", tags).Cols(attribute).UseBool(attribute)
    _, err = session.Update(bean)
}
```

**关键特性：**
- 用 `In("slug_name", tags)` 按 SlugName 列表批量更新
- `UseBool(attribute)` 强制更新布尔字段（xorm 默认跳过 false 值）
- 先将旧的推荐/保留标签全部设为 false，再将新的设为 true —— **全量替换逻辑**

### 2.6 RequiredTag 的影响总结

| 场景 | RequiredTag=true | RequiredTag=false |
|------|-----------------|-------------------|
| 提问时是否必须选推荐标签 | ✅ 是（否则报错） | ❌ 否 |
| 编辑问题时是否必须保留推荐标签 | ✅ 是 | ❌ 否 |
| 标签列表是否显示 Recommend 徽标 | ✅ 显示 | ❌ 隐藏（数据库字段仍为 true） |
| 推荐标签是否参与搜索排序 | ✅ `ORDER BY recommend DESC` | ❌ 所有 Recommend=false |

---

## 三、标签 Activity 消费链路

### 3.1 Activity 队列框架

系统使用**内存队列**（非消息中间件）异步处理 Activity 消息。

**队列实现：** [queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/base/queue/queue.go)

```
Queue[ActivityMsg] ("activity", bufferSize=128)
    │
    ├─ Send(ctx, msg)     → 入队（channel 缓冲区 128）
    │     └─ 队列满时阻塞
    │     └─ 队列已关闭时丢弃
    │
    ├─ RegisterHandler()  → 注册处理函数
    │     └─ ActivityCommon.HandleActivity
    │
    └─ startWorker()      → 单 goroutine 消费
          └─ for msg := range q.queue { processMessage(msg) }
                └─ handler(context.TODO(), msg)  // ⚠️ 使用 Background Context
```

**关键特性：**
- **单消费者**：只有一个 goroutine 消费，保证顺序性但不保证高吞吐
- **无重试**：handler 失败只打日志，不重新入队
- **无持久化**：进程重启时队列中未处理的消息丢失
- **阻塞发送**：队列满时 `Send()` 会阻塞调用方

### 3.2 标签 Activity 的发送方

| ActivityTypeKey | 发送位置 | 触发条件 | 是否带 RevisionID |
|-----------------|----------|----------|-------------------|
| `tag.created` | [tag_common.go#L376-L382](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L376-L382) AddTag() | 独立创建标签 | ✅ 是 |
| `tag.created` | [tag_common.go#L727-L733](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L727-L733) ObjectChangeTag() | 提问时自动创建新标签 | ✅ 是 |
| `tag.created` | [tag_service.go#L344-L371](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L344-L371) UpdateTagSynonym() | 同义词列表中的新标签 | ✅ 是 |
| `tag.edited` | [tag_common.go#L941-L947](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag_common/tag_common.go#L941-L947) UpdateTag() | 免审核编辑标签 | ✅ 是 |
| `tag.edited` | [revision_service.go#L310-L324](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/content/revision_service.go#L310-L324) revisionAuditTag() | 审核通过标签编辑 | ✅ 是 |
| `tag.deleted` | [tag_service.go#L100-L104](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L100-L104) RemoveTag() | 删除标签 | ❌ 否 |
| `tag.undeleted` | [tag_service.go#L136-L140](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/tag/tag_service.go#L136-L140) RecoverTag() | 恢复标签 | ❌ 否 |
| `tag.rollback` | （未找到发送点） | 标签回滚 | - |

**注意：** `tag.rollback` 虽然在常量定义和 config 表中存在，但代码中**没有找到任何发送点**，可能是预留但未实现。

### 3.3 ActivityMsg 结构

```go
type ActivityMsg struct {
    UserID           string
    TriggerUserID    int64
    ObjectID         string          // 标签 ID（DeShortID 前）
    OriginalObjectID string          // 标签 ID（DeShortID 前）
    ActivityTypeKey  ActivityTypeKey // 如 "tag.created"
    RevisionID       string          // 修订记录 ID（可选）
}
```

### 3.4 HandleActivity 消费处理

**实现位置：** [activity_common/activity.go#L68-L91](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/activity_common/activity.go#L68-L91)

```go
func (ac *ActivityCommon) HandleActivity(ctx context.Context, msg *schema.ActivityMsg) error {
    // Step 1: 将 ActivityTypeKey 字符串转为数字 ID
    //   "tag.created" → 查 config 表 key="tag.created" → 得到 config.ID (如 102)
    activityType, err := ac.activityRepo.GetActivityTypeByConfigKey(ctx, string(msg.ActivityTypeKey))
    
    // Step 2: 构造 Activity 实体
    act := &entity.Activity{
        UserID:           msg.UserID,
        TriggerUserID:    msg.TriggerUserID,
        ObjectID:         uid.DeShortID(msg.ObjectID),
        OriginalObjectID: uid.DeShortID(msg.OriginalObjectID),
        ActivityType:     activityType,    // 数字 ID
        Cancelled:        entity.ActivityAvailable,
    }
    if len(msg.RevisionID) > 0 {
        act.RevisionID = converter.StringToInt64(msg.RevisionID)
    }
    
    // Step 3: 插入 activity 表
    ac.activityRepo.AddActivity(ctx, act)
}
```

**处理结果：** Activity 消息最终被写入 `activity` 表一条记录，包含：
- `user_id`: 操作者
- `object_id`: 标签 ID
- `activity_type`: 数字 ID（关联 config 表的 key）
- `revision_id`: 修订记录 ID（如果有）
- `cancelled`: 0（有效）

### 3.5 Activity 配置表（config 表）中的标签相关条目

**位置：** [init_data.go#L327-L331](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/migrations/init_data.go#L327-L331)

| ID | Key | Value | 说明 |
|----|-----|-------|------|
| 102 | `tag.created` | `0` | 创建标签的 rank 积分 = **0** |
| 103 | `tag.edited` | `0` | 编辑标签的 rank 积分 = **0** |
| 104 | `tag.rollback` | `0` | 回滚标签的 rank 积分 = **0** |
| 105 | `tag.deleted` | `0` | 删除标签的 rank 积分 = **0** |
| 106 | `tag.undeleted` | `0` | 恢复标签的 rank 积分 = **0** |

以及标签相关的 rank 权限：

| ID | Key | Value | 说明 |
|----|-----|-------|------|
| 51 | `rank.tag.add` | `1500` | 创建标签需要 1500 积分 |
| 52 | `rank.tag.edit` | `100` | 编辑标签需要 100 积分 |
| 53 | `rank.tag.delete` | `-1` | 删除标签（仅管理员） |
| 54 | `rank.tag.synonym` | `20000` | 设置同义词需要 20000 积分 |
| 111 | `rank.tag.edit_without_review` | `20000` | 免审核编辑标签 |
| 114 | `rank.tag.audit` | `20000` | 审核标签编辑 |
| 117 | `rank.tag.use_reserved_tag` | `-1` | 使用保留标签（仅管理员） |
| 130 | `rank.tag.undeleted` | `-1` | 恢复标签（仅管理员） |
| 4 | `tag.edit_accepted` | `2` | 标签编辑被采纳的积分 = 2 |
| 35 | `tag.follow` | `0` | 关注标签的积分 = 0 |

### 3.6 tag.created / tag.edited / tag.deleted 的订阅方与处理动作

标签 Activity 的消费**只有一个订阅方**：`ActivityCommon.HandleActivity`，它的处理动作是**写入 activity 表**。

后续读取 activity 表的消费者有：

#### 消费方 1：时间线（Timeline）

**实现位置：** [activity.go#L92-L165](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/activity/activity.go#L92-L165) `GetObjectTimeline()`

```
标签详情页 → 时间线接口
    │
    ├─ GetObjectAllActivity(objectID=tagID)  → 查该标签所有 activity
    │
    ├─ 对每条 activity:
    │     ├─ configService.GetConfigByID(activityType) → 得到 key "tag.created"
    │     ├─ strings.Cut(key, ".") → activityType = "created"
    │     ├─ formatActivity("created") → 非隐藏 → 展示
    │     └─ getTimelineActivityComment()
    │           └─ activityType == "edited" → 查 revision 表返回编辑日志
    │
    └─ 返回时间线 JSON
```

**标签 Activity 在时间线中的展示规则：**

| ActivityType | 时间线展示 | Comment 来源 |
|-------------|-----------|-------------|
| `tag.created` | ✅ "created" | 无 |
| `tag.edited` | ✅ "edited" | revision.Log（Markdown → HTML） |
| `tag.deleted` | ✅ "deleted" | 无 |
| `tag.undeleted` | ✅ "undeleted" | 无 |
| `tag.rollback` | ✅ "rollback" | 无 |

**注意：** `formatActivity()` 函数 [activity.go#L433-L449](file:///d:/fz/0601-1/solo-dogfeeding/code/64-answer/internal/service/activity/activity.go#L433-L449) 对标签类 activity 没有做特殊处理（不像 vote_up/vote_down 会被隐藏或重命名）。

#### 消费方 2：Rank 积分系统

标签 Activity 的 config Value 全部为 `0`，意味着：
- **创建标签不获得积分**
- **编辑标签不获得积分**
- **删除/恢复标签不获得积分**

但标签编辑被审核通过时，编辑者会获得 `tag.edit_accepted = 2` 的积分。这个积分不在 activity 队列中处理，而是在审核通过流程中单独计算。

#### 消费方 3：通知系统（间接）

标签 Activity 本身**不直接触发通知**。通知系统关注的是：
- 问题被回答 → 通知提问者
- 评论 → 通知被评论者
- 标签关注者 → 收到新问题通知（通过 question.asked activity，不是 tag.created）

**标签操作不会通知任何人**，包括：
- 标签被创建 → 不通知
- 标签被编辑 → 不通知
- 标签被删除 → 不通知关注者

### 3.7 Activity 完整生命周期图

```
标签操作 (Service 层)
    │
    ▼
activityQueueService.Send(ctx, &ActivityMsg{
    UserID:           操作者ID,
    ObjectID:         标签ID,
    OriginalObjectID: 标签ID,
    ActivityTypeKey:  "tag.created" / "tag.edited" / "tag.deleted",
    RevisionID:       修订ID,
})
    │
    ▼
Queue[ActivityMsg] (内存 channel, buffer=128)
    │
    ▼ (单 goroutine 消费)
    │
ActivityCommon.HandleActivity(ctx, msg)
    │
    ├─ GetActivityTypeByConfigKey("tag.created")
    │     └─ SELECT id FROM config WHERE key = "tag.created" → 102
    │
    └─ AddActivity(&Activity{
           UserID:       操作者ID,
           ObjectID:     标签ID,
           ActivityType: 102,
           RevisionID:   修订ID,
           Cancelled:    0,
       })
           │
           ▼
       activity 表 (持久化)
           │
           ├─→ Timeline 查询: SELECT * FROM activity WHERE object_id = tagID
           │     └─ 时间线展示（created / edited / deleted / undeleted）
           │
           ├─→ Rank 积分: tag.* 的 Value 均为 0，不产生积分变动
           │
           └─→ 通知: 标签操作不触发通知
```

### 3.8 标签 Activity 的特殊情况

#### 合并标签不发送 Activity

`MergeTag()` 的 5 个步骤中**没有任何 `activityQueueService.Send()` 调用**。合并操作完全在 activity 表中不留痕迹。

#### 同义词设置不发送 Activity

`UpdateTagSynonym()` 中，已有标签变为同义词或取消同义词关系时，**不发送任何 Activity**。只有新建同义词标签时发送 `tag.created`。

#### 删除/恢复标签不带 RevisionID

```go
// RemoveTag
ts.activityQueueService.Send(ctx, &schema.ActivityMsg{
    ActivityTypeKey: constant.ActTagDeleted,
    // 没有 RevisionID
})

// RecoverTag
ts.activityQueueService.Send(ctx, &schema.ActivityMsg{
    ActivityTypeKey: constant.ActTagUndeleted,
    // 没有 RevisionID
})
```

**影响：** 在时间线中，删除/恢复操作无法点击查看修订详情（因为没有关联的 revision 记录）。

---

## 四、推荐标签校验的边界场景

### 4.1 场景 1：RequiredTag 开关切换

```
初始: RequiredTag=false, 标签 A 的 Recommend=true（但前端看不到）

1. 管理员开启 RequiredTag=true
2. 但没有配置 RecommendTags 列表

结果: ExistRecommend() → !taginfo.RequiredTag (false) || len(taginfo.RecommendTags)==0 (true)
      → return true, nil（放行）

结论: 仅开启 RequiredTag 但不配置推荐标签，等于没开。
```

### 4.2 场景 2：管理员取消推荐标签但用户正在提问

```
1. 用户打开提问页，搜索到推荐标签 "bug"
2. 管理员在后台把 "bug" 从推荐列表移除
3. 用户选了 "bug" 提交提问

结果: ExistRecommend() 实时查询 tag.Recommend 字段
      → "bug".Recommend 已经被 UpdateTagsAttribute() 改为 false
      → 返回 false → 提交失败

结论: 校验是实时的，不依赖页面加载时的缓存。
```

### 4.3 场景 3：标签被合并后推荐校验

```
1. 标签 A 是推荐标签（Recommend=true）
2. 标签 A 被合并到标签 B
3. 合并后 A 的 MainTagID=B.ID，但 A 的 Recommend 字段不变

结果: 如果用户搜索到了 A 的同义词 → SearchTagLike 跳转到 B
      → B 的 Recommend=false → ExistRecommend 返回 false
      → 提问失败

风险: 合并标签时不检查推荐属性继承，推荐标签被合并后
      其"推荐"属性不会自动迁移到目标标签。
```

### 4.4 场景 4：推荐标签被删除后 RequiredTag=true

```
1. 只有标签 A 是推荐标签
2. 管理员删除标签 A（软删除，status=10）
3. RecommendTags 配置中仍有 A

结果: GetRecommendTagList() 查询条件 status=1 → A 不会出现在推荐标签列表
      → ExistRecommend 短路: len(RecommendTags)==0 → 放行
      → 但 site_info 中 RecommendTags 配置仍有 A 的 SlugName

结论: 站点配置中的推荐标签 SlugName 可能指向已删除标签，
      导致 RequiredTag 形同虚设。
```

---

## 五、改进建议

### 5.1 推荐标签改进

1. **合并标签时继承推荐属性**：若源标签是推荐标签，合并后目标标签自动变为推荐标签。或在 `MergeTag()` 中检查源标签 Recommend 状态，给出警告。

2. **删除推荐标签时清理配置**：删除标签时，若该标签在 RecommendTags 列表中，应自动从 `site_info` 配置中移除。

3. **推荐标签变更通知关注者**：标签被从推荐列表移除或新增时，关注该标签的用户应收到通知。

4. **ExistRecommend 校验同义词标签的 Recommend**：当前校验的是 `item.Recommend`，但经过 `TagsFormatRecommendAndReserved` 处理后同义词标签的 Recommend 已替换为主标签的值。应确认 `ExistRecommend` 调用前是否也执行了格式化，如果没有，同义词标签的推荐属性可能不一致。

### 5.2 Activity 消费链路改进

1. **合并标签发送 Activity**：`MergeTag()` 应发送一条 `tag.merged`（新增 ActivityType）记录，包含源标签和目标标签信息。

2. **同义词变更发送 Activity**：`UpdateTagSynonym()` 中已有标签变同义词时，应发送 activity 记录。

3. **删除/恢复标签添加 Revision**：在删除前保存标签快照到 revision 表，恢复时可以追溯。

4. **队列可靠性**：当前内存队列在进程重启时丢失消息，建议对关键 activity（如删除、合并）改为同步写入或增加持久化层。

5. **实现 tag.rollback Activity**：常量和配置都已定义，但从未使用。若标签回滚功能已实现，应补充 Activity 发送。
