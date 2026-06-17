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

### 2.3 核心列表页（问题/标签分页）：不使用缓存

需要特别指出，**核心业务列表页和详情页均不使用缓存**，每次请求直接查询数据库：

**问题列表分页** [internal/repo/question/question_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/question/question_repo.go#L394-L457) 的 `GetQuestionPage` 方法：

```go
func (qr *questionRepo) GetQuestionPage(ctx context.Context, page, pageSize int,
    tagIDs []string, userID, orderCond string, inDays int, showHidden, showPending bool) (
    questionList []*entity.Question, total int64, err error) {
    
    questionList = make([]*entity.Question, 0)
    session := qr.data.DB.Context(ctx)
    // ... 构建复杂查询条件（标签过滤、时间范围、多种排序）
    total, err = pager.Help(page, pageSize, &questionList, &entity.Question{}, session)
    return questionList, total, err
}
```

**标签列表分页** [internal/service/tag_common/tag_common.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/tag_common/tag_common.go#L427-L433) 的 `GetTagPage` 方法同样直接查 DB。

**不缓存的原因**：
- 查询维度多（标签过滤、排序方式、时间范围、用户维度等组合爆炸），缓存键管理困难
- 数据实时性要求高，用户发布/操作后需要立即看到变化
- 写操作频繁（投票、回答等都会影响排序），写时失效成本高

### 2.4 详情页（问题/回答详情）：不使用缓存

**问题详情** [internal/service/question_common/question.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/question_common/question.go#L254-L362) 的 `Info` 方法：

```go
func (qs *QuestionCommon) Info(ctx context.Context, questionID string, loginUserID string) (
    resp *schema.QuestionInfoResp, err error) {
    
    questionInfo, has, err := qs.questionRepo.GetQuestion(ctx, questionID)  // 直接查DB
    if !has {
        return resp, errors.NotFound(reason.QuestionNotFound)
    }
    resp = qs.ShowFormat(ctx, questionInfo)
    // ... 组装标签、用户信息、投票状态、关注状态、回答状态、收藏状态
    return resp, nil
}
```

**回答详情** [internal/service/content/answer_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/content/answer_service.go#L599-L609) 的 `GetDetail` 方法同样直接查 DB。

**不缓存的原因**：
- 详情页包含大量用户个性化数据（投票状态、关注状态、收藏状态），与登录用户绑定
- 问题/回答内容可能被编辑，需要保证实时性
- 查询路径上还有 DB 层（XORM）的会话缓存，能提供一定的性能优化

### 2.5 Sitemap 列表缓存命中

[internal/repo/question/question_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/question/question_repo.go#L345-L392) 中 `SitemapQuestions` 实现：

```go
func (qr *questionRepo) SitemapQuestions(ctx context.Context, page, pageSize int) (
    questionIDList []*schema.SiteMapQuestionInfo, err error) {
    page--
    questionIDList = make([]*schema.SiteMapQuestionInfo, 0)  // 初始化空切片
    
    // try to get sitemap data from cache
    cacheKey := fmt.Sprintf(constant.SiteMapQuestionCacheKeyPrefix, page)
    cacheData, exist, err := qr.data.Cache.GetString(ctx, cacheKey)
    if err == nil && exist {
        _ = json.Unmarshal([]byte(cacheData), &questionIDList)  // 用下划线忽略错误
        return questionIDList, nil  // ⚠️ 反序列化失败也直接返回
    }
    
    // 查DB并组装数据...
    
    // 回写缓存
    cacheDataByte, _ := json.Marshal(questionIDList)
    if err := qr.data.Cache.SetString(ctx, cacheKey, string(cacheDataByte), 
        constant.SiteMapQuestionCacheTime); err != nil {
        log.Error(err)
    }
    return questionIDList, nil
}
```

**要点**：
- 按页码分片缓存，避免单条缓存过大
- 缓存有效期 1 小时，由定时任务 `SitemapCron` 主动预热
- **反序列化失败处理**：`_ = json.Unmarshal(...)` 用下划线丢弃错误。如果缓存数据损坏或格式不兼容，不会降级到 DB，而是返回初始化的空切片（`[]*schema.SiteMapQuestionInfo{}`），同时 `err=nil`。调用方会得到一个空列表而非错误。
- **写时不失效**：问题创建/更新/删除时不会主动删除 Sitemap 缓存，完全依赖 TTL 自动过期（1小时）和定时任务预热。

### 2.6 仪表盘统计缓存命中

[internal/service/dashboard/dashboard_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/dashboard/dashboard_service.go#L102-L173) 中 `Statistical` 实现：

- 部分字段（问题数、未回答数等）实时查询，不缓存
- 部分字段（回答数、评论数、用户数等）走缓存
- 采用**部分缓存**策略，平衡实时性与性能

### 2.7 通知红点缓存命中

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

## 四、事件广播、索引刷新与缓存失效的区分

用户写入操作后，系统会触发三类完全不同的动作，必须严格区分：

| 机制 | 目的 | 操作对象 | 执行时机 |
|------|------|---------|---------|
| **缓存失效（Cache.Del）** | 删除陈旧缓存键，下次查询回源DB | 内存/Redis缓存键 | **同步**，写DB后立即执行 |
| **事件广播（Event 队列）** | 通知其他模块发生了某业务事件 | 业务事件消息 | **异步**，放入队列后返回 |
| **索引刷新（VectorSync 队列）** | 保持向量搜索索引与DB一致 | 向量搜索插件索引 | **异步**，放入队列+3次重试 |

### 4.1 缓存失效：同步删除缓存键

缓存失效通过 `Cache.Del(ctx, key)` **同步**执行，仅针对明确使用了缓存的业务场景。

搜索全局 `Cache.Del` 调用发现，缓存失效仅用于以下场景：

| 场景 | 代码位置 | 缓存键 |
|------|---------|--------|
| 删除通知红点 | [notification.go:277](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/notification_common/notification.go#L277) | `answer:red-dot:%d:%s` |
| 移除徽章缓存 | [notification.go:322](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/notification_common/notification.go#L322) | `answer:badge-award:%s` |
| 清理限流记录 | [limit.go:64](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/limit/limit.go#L64) | `answer:limit:%s` |
| 删除邮箱验证码 | [email_repo.go:75](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/export/email_repo.go#L75) | `answer:user:email:code:%s` |
| 删除操作频率记录 | [captcha.go:83](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/captcha/captcha.go#L83) | `ActionRecord:%s` |
| 删除验证码 | [captcha.go:112](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/captcha/captcha.go#L112) | `answer:captcha:%s` |
| 用户登出（删除Token） | [auth.go:102](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/auth/auth.go#L102) | `answer:user:token:%s` |
| 删除用户状态缓存 | [auth.go:148](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/auth/auth.go#L148) | `answer:user:status-changed:%s` |
| 管理员登出 | [auth.go:187](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/auth/auth.go#L187) | `answer:admin:token:%s` |

**重要结论**：
- 问题创建/更新/删除、回答创建/更新/删除等核心业务写操作 **不会** 触发 `Cache.Del`，因为核心列表页和详情页根本没有做缓存。
- Sitemap 缓存也 **不会** 写时失效，完全依赖 TTL（1小时）自动过期。
- 配置和站点信息使用的是 **Write-Through**（写时更新）而非写时失效。

### 4.2 事件广播：异步通知各业务模块

系统通过**异步队列** + **事件驱动** 实现跨模块数据同步。

#### 4.2.1 通用队列框架

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

#### 4.2.2 四类专用队列

系统在 [internal/service/provider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/provider.go#L122-L140) 初始化四类队列：

| 队列 | 定义位置 | 消息类型 | 用途 |
|------|---------|---------|------|
| Activity 队列 | [activityqueue/activity_queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/activityqueue/activity_queue.go) | `*schema.ActivityMsg` | 活动记录、声望计算 |
| Event 队列 | [eventqueue/event_queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/eventqueue/event_queue.go) | `*schema.EventMsg` | 跨模块事件广播 |
| Notification 队列 | [noticequeue/notice_queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/noticequeue/notice_queue.go) | `*schema.NotificationMsg` | 站内通知 |
| VectorSync 队列 | [vector_sync/vector_sync.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/vector_sync/vector_sync.go) | `*Task` | 向量搜索索引同步 |

#### 4.2.3 事件类型定义

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

#### 4.2.4 事件广播流程

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

#### 4.2.5 事件监听与处理器注册

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
- **JSON 容错**：反序列化失败不抛出错误，继续走 DB 查询（注意：Sitemap 除外，其反序列化失败返回空列表而非降级DB）

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

## 六、核心业务写操作完整链路（区分三类机制）

以**删除问题**为例 [internal/service/content/question_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/content/question_service.go#L556-L664)，链路中标注了每一步属于哪类机制：

```
用户请求删除问题
    ↓
1. 权限校验（是否作者/管理员、是否有采纳回答等）
    ↓
2. 【同步】更新数据库问题状态为已删除（DB 写操作）
    ↓
3. 【同步】更新关联数据：用户问题数、标签关联、标签问题数计数
    ↓
4. 【同步】⚠️ 注意：此处**没有**调用 Cache.Del 做缓存失效
    ↓
5. 【异步→事件广播】发送活动消息 → Activity 队列 → 写入活动记录/声望计算
    ↓
6. 【异步→事件广播】发送事件广播 → Event 队列 → 徽章判定、其他模块监听
    ↓
7. 【异步→索引刷新】发送向量同步 → VectorSync 队列 → 删除向量索引（3次重试）
    ↓
返回成功
```

**三类机制对比（以删除问题为例）**：

| 步骤 | 机制类型 | 是否同步 | 代码实现 | 失败影响 |
|------|---------|---------|---------|---------|
| 2 | DB写入 | 同步 | `questionRepo.UpdateQuestionStatusWithOutUpdateTime` | 请求失败，返回错误 |
| 3 | DB写入 | 同步 | `userCommon.UpdateQuestionCount`, `tagCommon.RemoveTagRelListByObjectID` | 请求失败，回滚失败 |
| 4 | **缓存失效** | — | **未执行**（无相关缓存） | 无影响，核心列表/详情不缓存 |
| 5 | **事件广播** | 异步 | `activityQueueService.Send` | 活动记录丢失，不影响主流程 |
| 6 | **事件广播** | 异步 | `eventQueueService.Send(EventQuestionDelete)` | 徽章等未触发，不影响主流程 |
| 7 | **索引刷新** | 异步+重试 | `vectorSyncService.Send(ActionDelete)` | 搜索索引暂时不一致，3次重试后仍失败则记录日志 |

**关键说明**：
- 删除问题时 **没有任何缓存失效操作**，因为核心列表页、详情页均不使用缓存
- Sitemap 缓存不做写时失效，依赖 TTL（1小时）自动过期
- 数据库更新是同步的，确保核心数据一致性
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
| 问题列表（**不缓存**） | [question_repo.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/repo/question/question_repo.go) | `GetQuestionPage` |
| 问题详情（**不缓存**） | [question.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/question_common/question.go) | `Info` |
| 回答详情（**不缓存**） | [answer_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/content/answer_service.go) | `GetDetail` |
| 标签列表（**不缓存**） | [tag_common.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/tag_common/tag_common.go) | `GetTagPage` |
| 仪表盘缓存 | [dashboard_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/dashboard/dashboard_service.go) | `Statistical`, `getFromCache` |
| 通知红点缓存+失效 | [notification.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/notification_common/notification.go) | `DeleteRedDot` |
| 通用队列 | [queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/base/queue/queue.go) | `New`, `Send`, `RegisterHandler` |
| 事件广播+索引刷新 | [question_service.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/content/question_service.go) | `AddQuestion`, `RemoveQuestion` |
| 向量同步（索引刷新） | [vector_sync.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/vector_sync/vector_sync.go) | `handle`, `handleOnce` |
| 活动处理（事件广播） | [activity.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/activity_common/activity.go) | `HandleActivity` |
| 徽章监听（事件广播） | [badge_event_handler.go](file:///d:/fz/0601-2/solo-dogfeeding/code/17-answer/internal/service/badge/badge_event_handler.go) | `Handler` |

### 7.4 三类写后机制核心区别（再次强调）

| 维度 | 缓存失效（Cache.Del） | 事件广播（Event/Activity队列） | 索引刷新（VectorSync队列） |
|------|---------------------|------------------------------|--------------------------|
| **核心目标** | 删除缓存键，下次查回源DB | 通知其他模块业务事件发生 | 更新向量搜索索引数据 |
| **执行方式** | **同步**调用 | **异步**入队即返回 | **异步**入队+3次重试 |
| **操作对象** | `plugin.Cache` 接口 | 业务消息（ActivityMsg/EventMsg） | 向量搜索插件 |
| **适用数据** | 红点、Token、验证码、徽章等**有缓存的数据** | 活动记录、声望、徽章、通知等**跨模块业务** | 问题/回答的**搜索索引** |
| **核心业务是否使用** | ❌ 问题/回答写操作**不触发**（无缓存） | ✅ 每次写操作都发送 | ✅ 状态为Available/Deleted时发送 |
| **Sitemap缓存** | ❌ 写时不失效，靠TTL过期 | — | — |
| **失败影响** | 下次查询多读一次DB | 活动/徽章/通知可能丢失（无补偿） | 搜索暂时不准（有3次重试） |
