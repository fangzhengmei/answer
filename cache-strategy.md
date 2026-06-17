# Answer 站点缓存策略分析

## 一、缓存基础架构

### 1.1 缓存抽象层

系统通过插件机制提供缓存抽象接口，定义于 [plugin/cache.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/plugin/cache.go#L27-L38)：

```go
type Cache interface {
    GetString(ctx context.Context, key string) (data string, exist bool, err error)
    SetString(ctx context.Context, key, value string, ttl time.Duration) (err error)
    GetInt64(ctx context.Context, key string) (data int64, exist bool, err error)
    SetInt64(ctx context.Context, key string, value int64, ttl time.Duration) (err error)
    Increase(ctx context.Context, key string, value int64) (data int64, err error)
    Decrease(ctx context.Context, key string, value int64) (data int64, err error)
    Del(ctx context.Context, key string) (err error)
    Flush(ctx context.Context) (err error)
}
```

### 1.2 缓存初始化与实现

缓存初始化在 [internal/base/data/data.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/base/data/data.go#L97-L138) 的 `NewCache` 函数中完成：

- **插件优先**：首先尝试加载注册的 Cache 插件（如 Redis）
- **内存缓存兜底**：无插件时使用 `github.com/segmentfault/pacman/contrib/cache/memory` 内存缓存
- **持久化支持**：配置 `FilePath` 时，每分钟自动将内存缓存持久化到磁盘，启动时加载
- **优雅关闭**：进程退出时自动保存缓存到磁盘

### 1.3 缓存键规范

所有缓存键统一在 [internal/base/constant/cache_key.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/base/constant/cache_key.go) 中定义，采用 `answer:模块:业务:标识` 格式：

| 缓存键前缀 | 用途 | TTL |
|-----------|------|-----|
| `answer:config:id:` | 配置ID到内容映射 | 1小时 |
| `answer:config:key:` | 配置Key到内容映射 | 1小时 |
| `answer:site-info:` | 站点信息 | 1小时 |
| `answer:sitemap:question:%d` | Sitemap分页数据 | 1小时 |
| `answer:user:token:` | 用户登录Token | 7天 |
| `answer:red-dot:%s:%s` | 通知红点计数 | 30天 |

---

## 二、缓存命中策略（Cache-Aside 模式）

系统全面采用 **Cache-Aside（旁路缓存）** 模式，读流程统一为：**查缓存 → 未命中查DB → 回写缓存**。

### 2.1 配置缓存命中

[internal/repo/config/config_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/config/config_repo.go#L48-L100) 中 `GetConfigByID` 和 `GetConfigByKey` 实现：

```go
func (cr configRepo) GetConfigByKey(ctx context.Context, key string) (c *entity.Config, err error) {
    cacheKey := constant.ConfigKEY2ContentCacheKeyPrefix + key
    cacheData, exist, err := cr.data.Cache.GetString(ctx, cacheKey)
    if err == nil && exist && len(cacheData) > 0 {
        c = &entity.Config{}
        c.BuildByJSON([]byte(cacheData))
        if c.ID > 0 {
            return c, nil  // 缓存命中
        }
    }
    // 缓存未命中，查数据库
    c = &entity.Config{Key: key}
    exist, err = cr.data.DB.Context(ctx).Get(c)
    // 回写缓存
    if err == nil && exist {
        cr.data.Cache.SetString(ctx, cacheKey, c.JsonString(), constant.ConfigCacheTime)
    }
    return c, err
}
```

**要点**：
- 缓存数据做有效性校验（`c.ID > 0`），防止无效缓存
- 缓存出错时自动降级到数据库查询，不阻塞业务

### 2.2 站点信息缓存命中

[internal/repo/site_info/siteinfo_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/site_info/siteinfo_repo.go#L65-L105) 中 `GetByType` 实现：

- 支持 `withoutCache` 参数绕过缓存（用于管理后台强制刷新）
- 缓存序列化使用 JSON，反序列化失败降级到 DB
- `getCache`/`setCache` 封装为独立方法，便于统一处理

### 2.3 Sitemap 列表缓存命中

[internal/repo/question/question_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/question/question_repo.go#L345-L392) 中 `SitemapQuestions` 实现：

```go
func (qr *questionRepo) SitemapQuestions(ctx context.Context, page, pageSize int) (
    questionIDList []*schema.SiteMapQuestionInfo, err error) {
    
    cacheKey := fmt.Sprintf(constant.SiteMapQuestionCacheKeyPrefix, page)
    cacheData, exist, err := qr.data.Cache.GetString(ctx, cacheKey)
    if err == nil && exist {
        _ = json.Unmarshal([]byte(cacheData), &questionIDList)
        return questionIDList, nil  // 缓存命中直接返回
    }
    
    // 查DB并组装数据...
    
    // 回写缓存
    cacheDataByte, _ := json.Marshal(questionIDList)
    qr.data.Cache.SetString(ctx, cacheKey, string(cacheDataByte), constant.SiteMapQuestionCacheTime)
    return questionIDList, nil
}
```

**要点**：
- 按页码分片缓存，避免单条缓存过大
- 缓存有效期 1 小时，由定时任务 `SitemapCron` 主动预热
- 反序列化出错不返回错误，继续走 DB 查询

### 2.4 仪表盘统计缓存命中

[internal/service/dashboard/dashboard_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/dashboard/dashboard_service.go#L102-L173) 中 `Statistical` 实现：

- 部分字段（问题数、未回答数等）实时查询，不缓存
- 部分字段（回答数、评论数、用户数等）走缓存
- 采用**部分缓存**策略，平衡实时性与性能

### 2.5 通知红点缓存命中

[internal/service/notification_common/notification.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/notification_common/notification.go#L270-L325) 中红点计数缓存：

- 使用 `Increase`/`Decrease` 原子操作维护计数
- 支持徽章成就的结构化缓存（JSON 存储徽章列表）
- 计数归零时主动删除缓存（`DeleteRedDot`）

---

## 三、写时失效机制（Write-Through + Write-Invalidate）

系统采用 **写时更新** + **写时失效** 混合策略，根据业务场景选择。

### 3.1 配置更新：写时更新（Write-Through）

[internal/repo/config/config_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/config/config_repo.go#L114-L143) 中 `UpdateConfig`：

```go
func (cr configRepo) UpdateConfig(ctx context.Context, key string, value string) (err error) {
    // 1. 更新数据库
    _, err = cr.data.DB.Context(ctx).ID(oldConfig.ID).Update(&entity.Config{Value: value})
    
    // 2. 同时更新两份缓存（ID映射 和 Key映射）
    cacheVal := oldConfig.JsonString()
    cr.data.Cache.SetString(ctx, constant.ConfigKEY2ContentCacheKeyPrefix+key, cacheVal, constant.ConfigCacheTime)
    cr.data.Cache.SetString(ctx, fmt.Sprintf("%s%d", constant.ConfigID2KEYCacheKeyPrefix, oldConfig.ID), cacheVal, constant.ConfigCacheTime)
    
    return nil
}
```

**适用场景**：配置数据更新频率低，读取频率高，写时更新避免缓存击穿。

### 3.2 站点信息更新：写时更新

[internal/repo/site_info/siteinfo_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/site_info/siteinfo_repo.go#L46-L63) 中 `SaveByType`：

- 数据库更新成功后立即调用 `setCache` 更新缓存
- 缓存与数据库保持强一致

### 3.3 通知红点：写时失效（Write-Invalidate）

[internal/service/notification_common/notification.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/notification_common/notification.go#L270-L282) 中 `DeleteRedDot`：

```go
func (ns *NotificationCommon) DeleteRedDot(ctx context.Context, userID string, notificationType int) error {
    key := fmt.Sprintf(constant.RedDotCacheKey, notificationType, userID)
    return ns.data.Cache.Del(ctx, key)  // 直接删除缓存
}
```

**适用场景**：红点计数实时性要求高，且可能被并发修改，删除缓存比更新更安全。

### 3.4 徽章成就缓存：读写结合

[internal/service/notification_common/notification.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/notification_common/notification.go#L284-L325)：

- `AddBadgeAwardAlertCache`：读-改-写模式，先查缓存，修改后回写
- `RemoveBadgeAwardAlertCache`：列表为空时主动删除缓存

---

## 四、失效广播与事件监听机制

系统通过**异步队列** + **事件驱动** 实现跨模块缓存失效和数据同步。

### 4.1 通用队列框架

[internal/base/queue/queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/base/queue/queue.go#L40-L130) 提供泛型队列实现：

```go
type Queue[T any] struct {
    name    string
    queue   chan T          // 带缓冲通道
    handler func(ctx context.Context, msg T) error
    mu      sync.RWMutex    // 保护 handler 并发安全
    closed  bool
    wg      sync.WaitGroup  // 优雅关闭等待
}
```

**核心特性**：
- 异步处理：后台 goroutine 消费消息，不阻塞主流程
- 缓冲队列：默认缓冲区 128，防止突发流量打满
- 优雅关闭：`Close()` 方法等待所有消息处理完成
- 错误隔离：单条消息处理失败不影响其他消息
- 处理上下文：使用 `context.TODO()` 确保异步处理不被请求上下文取消

### 4.2 四类专用队列

系统在 [internal/service/provider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/provider.go#L122-L140) 初始化四类队列：

| 队列 | 定义位置 | 消息类型 | 用途 |
|------|---------|---------|------|
| Activity 队列 | [activityqueue/activity_queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/activityqueue/activity_queue.go) | `*schema.ActivityMsg` | 活动记录、声望计算 |
| Event 队列 | [eventqueue/event_queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/eventqueue/event_queue.go) | `*schema.EventMsg` | 跨模块事件广播 |
| Notification 队列 | [noticequeue/notice_queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/noticequeue/notice_queue.go) | `*schema.NotificationMsg` | 站内通知 |
| VectorSync 队列 | [vector_sync/vector_sync.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/vector_sync/vector_sync.go) | `*Task` | 向量搜索索引同步 |

### 4.3 事件类型定义

[internal/base/constant/event.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/base/constant/event.go) 定义了完整的事件类型，采用 `对象.动作` 格式：

```go
const (
    EventQuestionCreate EventType = "question.create"
    EventQuestionUpdate EventType = "question.update"
    EventQuestionDelete EventType = "question.delete"
    EventQuestionVote   EventType = "question.vote"
    EventQuestionAccept EventType = "question.accept"
    EventAnswerCreate   EventType = "answer.create"
    EventAnswerUpdate   EventType = "answer.update"
    EventAnswerDelete   EventType = "answer.delete"
    // ... 更多事件
)
```

### 4.4 事件广播流程

以问题创建为例，[internal/service/content/question_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/content/question_service.go#L442-L465)：

```go
// 1. 写入数据库成功后
qs.activityQueueService.Send(ctx, &schema.ActivityMsg{
    UserID:           question.UserID,
    ObjectID:         question.ID,
    ActivityTypeKey:  constant.ActQuestionAsked,
    RevisionID:       revisionID,
})

// 2. 发送事件广播
qs.eventQueueService.Send(ctx, schema.NewEvent(constant.EventQuestionCreate, req.UserID).
    TID(question.ID).
    QID(question.ID, question.UserID))

// 3. 发送向量同步任务
qs.vectorSyncService.Send(ctx, &vector_sync.Task{
    Action:     vector_sync.ActionUpsert,
    ObjectType: vector_sync.ObjectTypeQuestion,
    ObjectID:   question.ID,
})
```

**事件消息结构** [internal/schema/event_schema.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/schema/event_schema.go#L27-L44)：

```go
type EventMsg struct {
    EventType       constant.EventType  // 事件类型
    UserID          string              // 触发用户
    TriggerObjectID string              // 触发对象ID
    QuestionID      string              // 关联问题ID
    QuestionUserID  string              // 问题作者ID
    AnswerID        string              // 关联回答ID
    AnswerUserID    string              // 回答作者ID
    CommentID       string              // 关联评论ID
    CommentUserID   string              // 评论作者ID
    ExtraInfo       map[string]string   // 扩展信息
}
```

### 4.5 事件监听与处理器注册

通过 `RegisterHandler` 注册处理器，以徽章事件为例 [internal/service/badge/badge_event_handler.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/badge/badge_event_handler.go#L46-L77)：

```go
func NewBadgeEventService(...) *BadgeEventService {
    n := &BadgeEventService{...}
    // 注册事件处理器
    eventQueueService.RegisterHandler(n.Handler)
    return n
}

func (ns *BadgeEventService) Handler(ctx context.Context, msg *schema.EventMsg) error {
    // 根据事件类型匹配徽章规则，自动颁发徽章
    awards := ns.eventRuleRepo.HandleEventWithRule(ctx, msg)
    for _, award := range awards {
        ns.badgeAwardService.Award(ctx, award.BadgeID, award.UserID, award.AwardKey)
    }
    return nil
}
```

**活动队列处理器** [internal/service/activity_common/activity.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/activity_common/activity.go#L68-L91)：

```go
func (ac *ActivityCommon) HandleActivity(ctx context.Context, msg *schema.ActivityMsg) error {
    // 查询活动类型配置
    activityType, err := ac.activityRepo.GetActivityTypeByConfigKey(ctx, string(msg.ActivityTypeKey))
    
    // 写入活动记录（用于声望计算、时间线展示）
    act := &entity.Activity{
        UserID:           msg.UserID,
        TriggerUserID:    msg.TriggerUserID,
        ObjectID:         uid.DeShortID(msg.ObjectID),
        ActivityType:     activityType,
        Cancelled:        entity.ActivityAvailable,
    }
    return ac.activityRepo.AddActivity(ctx, act)
}
```

---

## 五、异步刷新与回源降级策略

### 5.1 向量搜索异步刷新

[internal/service/vector_sync/vector_sync.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/vector_sync/vector_sync.go#L59-L115) 实现带重试的异步刷新：

```go
func handle(ctx context.Context, data *data.Data, msg *Task) error {
    // 检查向量搜索插件是否启用
    var vectorSearch plugin.VectorSearch
    _ = plugin.CallVectorSearch(func(vs plugin.VectorSearch) error {
        vectorSearch = vs
        return nil
    })
    if vectorSearch == nil {
        return nil  // 插件未启用，直接跳过（降级）
    }

    // 最多重试 3 次
    var lastErr error
    for attempt := 1; attempt <= maxRetry; attempt++ {
        err := handleOnce(ctx, data, vectorSearch, msg.Action, msg.ObjectType, objectID)
        if err == nil {
            return nil
        }
        lastErr = err
        log.Warnf("vector sync failed: attempt=%d err=%v", attempt, err)
    }
    return lastErr  // 重试耗尽后返回错误
}

func handleOnce(...) error {
    if action == ActionDelete {
        return vectorSearch.DeleteContent(ctx, objectID)
    }
    // 从数据库回源构建索引内容
    switch objectType {
    case ObjectTypeQuestion:
        content, err = vector_search_sync.BuildQuestionContentByID(ctx, data, objectID)
    case ObjectTypeAnswer:
        content, err = vector_search_sync.BuildAnswerContentByID(ctx, data, objectID)
    }
    if content == nil {
        return vectorSearch.DeleteContent(ctx, objectID)  // 内容不存在，清理索引
    }
    return vectorSearch.UpdateContent(ctx, content)
}
```

**要点**：
- **插件可选**：向量搜索插件未启用时自动降级为空操作
- **回源构建**：从数据库查询最新数据构建索引内容
- **重试机制**：3 次重试，避免临时网络问题导致索引不一致
- **空值处理**：内容已删除时主动清理索引

### 5.2 缓存回源降级策略

所有缓存查询都遵循以下降级顺序：

```
1. 查缓存 → 命中 → 返回数据
   ↓ 未命中或出错
2. 查数据库 → 成功 → 回写缓存 → 返回数据
   ↓ 数据库也出错
3. 返回错误，由上层处理
```

**代码示例** [internal/repo/config/config_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/config/config_repo.go#L48-L73)：

```go
cacheData, exist, err := cr.data.Cache.GetString(ctx, cacheKey)
if err == nil && exist && len(cacheData) > 0 {
    // 缓存命中，校验有效性后返回
    c = &entity.Config{}
    c.BuildByJSON([]byte(cacheData))
    if c.ID > 0 {
        return c, nil
    }
}
// 缓存失效/出错 → 降级到DB
c = &entity.Config{}
exist, err = cr.data.DB.Context(ctx).ID(id).Get(c)
if err == nil && exist {
    // 回写缓存
    cr.data.Cache.SetString(ctx, cacheKey, c.JsonString(), constant.ConfigCacheTime)
}
return c, err
```

### 5.3 缓存穿透防护

- **空值不缓存**：数据库查询结果不存在时不写入缓存（由业务代码控制）
- **有效性校验**：缓存读取后校验关键字段（如 `c.ID > 0`），无效则穿透到 DB
- **JSON 容错**：反序列化失败不抛出错误，继续走 DB 查询

### 5.4 缓存预热机制

[Sitemap 预热](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/question_common/question.go#L601-L623) 由定时任务触发：

```go
func (qs *QuestionCommon) SitemapCron(ctx context.Context) {
    questionNum, _ := qs.questionRepo.GetQuestionCount(ctx)
    totalPages := int(math.Ceil(float64(questionNum) / float64(constant.SitemapMaxSize)))
    for i := 1; i <= totalPages; i++ {
        // 调用查询方法，自动填充缓存
        _, _ = qs.questionRepo.SitemapQuestions(ctx, i, constant.SitemapMaxSize)
    }
}
```

---

## 六、核心业务写操作完整链路

以**删除问题**为例，完整链路如下 [internal/service/content/question_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/content/question_service.go#L556-L664)：

```
用户请求删除问题
    ↓
1. 权限校验（是否作者/管理员、是否有采纳回答等）
    ↓
2. 更新数据库问题状态为已删除
    ↓
3. 更新关联数据计数（用户问题数、标签问题数）
    ↓
4. 发送活动消息 → Activity 队列 → 写入活动记录
    ↓
5. 发送事件广播 → Event 队列 → 徽章判定、其他监听
    ↓
6. 发送向量同步 → VectorSync 队列 → 删除向量索引（带重试）
    ↓
返回成功
```

**关键说明**：
- 数据库更新是同步的，确保数据一致性
- 活动记录、事件广播、索引同步都是异步的，不影响响应时间
- 各队列相互独立，某一队列阻塞不影响其他

---

## 七、缓存策略总结

### 7.1 设计优点

1. **分层抽象**：Cache 接口 → 内存/插件实现 → 业务封装，层次清晰
2. **异步解耦**：通过队列实现写后操作与主流程解耦，提升响应速度
3. **容错设计**：缓存出错自动降级，队列消费失败不影响主业务
4. **重试机制**：向量同步等关键异步操作带重试，保证最终一致性
5. **插件扩展**：缓存和向量搜索都支持插件替换，便于部署不同规模环境

### 7.2 关键权衡

| 策略 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| Cache-Aside | 所有读场景 | 简单灵活，缓存故障不影响DB | 首次查询必查DB |
| Write-Through | 配置、站点信息 | 缓存与DB强一致 | 写操作耗时增加 |
| Write-Invalidate | 红点、计数 | 并发安全，避免竞态 | 下次查询需回源 |
| 异步队列 | 活动、事件、索引 | 主流程快，解耦彻底 | 最终一致性，有延迟 |
| 内存缓存 | 单机部署 | 高性能，无需额外组件 | 多实例部署数据不一致 |

### 7.3 代码要点速查表

| 功能 | 关键文件 | 关键函数 |
|------|---------|---------|
| 缓存初始化 | [data.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/base/data/data.go) | `NewCache` |
| 配置缓存 | [config_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/config/config_repo.go) | `GetConfigByKey`, `UpdateConfig` |
| 站点信息缓存 | [siteinfo_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/site_info/siteinfo_repo.go) | `GetByType`, `SaveByType` |
| Sitemap缓存 | [question_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/question/question_repo.go) | `SitemapQuestions` |
| 仪表盘缓存 | [dashboard_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/dashboard/dashboard_service.go) | `Statistical`, `getFromCache` |
| 通用队列 | [queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/base/queue/queue.go) | `New`, `Send`, `RegisterHandler` |
| 事件广播 | [question_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/content/question_service.go) | `AddQuestion`, `RemoveQuestion` |
| 向量同步 | [vector_sync.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/vector_sync/vector_sync.go) | `handle`, `handleOnce` |
| 活动处理 | [activity.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/activity_common/activity.go) | `HandleActivity` |
| 事件监听 | [badge_event_handler.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/badge/badge_event_handler.go) | `Handler` |
