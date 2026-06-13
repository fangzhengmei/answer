# 提问发布与问题实体流转 - 端到端路径分析

## 一、整体架构概览

本项目是 Apache Answer 问答社区系统，采用 **Go (Gin) + React (TypeScript)** 的前后端分离架构。问题实体的流转涉及完整的 **前端表单 → API 网关 → 后端 Controller → Service 层 → Repository 层 → 数据库** 链路。

---

## 二、问题创建链路（发布提问）

### 2.1 前端入口：提问页面

**文件**: [ui/src/pages/Questions/Ask/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/pages/Questions/Ask/index.tsx)

#### 核心组件结构：
```tsx
// 表单数据结构
interface FormDataItem {
  title: FormValue<string>;      // 问题标题
  tags: FormValue<Tag[]>;        // 标签列表
  content: FormValue<string>;    // 问题内容（Markdown）
  answer_content: FormValue<string>; // 自问自答内容
  edit_summary: FormValue<string>;   // 编辑摘要
}
```

#### 关键流程：
1. **表单初始化**（`useEffect`）:
   - 支持 URL 参数预填充（`?tags=xxx&prefill=xxx`）
   - 支持从本地存储恢复草稿（`DRAFT_QUESTION_STORAGE_KEY`）
   - 编辑模式下调用 `questionDetail(qid)` 获取已有数据

2. **实时功能**:
   - 标题输入超过 10 字符时，防抖（400ms）查询相似问题：`queryQuestionByTitle(title)`
   - 草稿自动保存，触发离开页面提示

3. **提交处理**（`handleSubmit` → `submitQuestion`）:
   - 组装参数：`{ title, content, tags }`
   - 验证码校验（如启用）
   - 调用 API：
     - 普通提问：`saveQuestion(params)` → `POST /answer/api/v1/question`
     - 自问自答：`saveQuestionWithAnswer(params)` → `POST /answer/api/v1/question/answer`
   - 成功后跳转至问题详情页：`pathFactory.questionLanding(id, url_title)`

### 2.2 前端 API 层

**文件**: [ui/src/services/common.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/services/common.ts)

```typescript
// 创建问题
export const saveQuestion = (params: Type.QuestionParams) => {
  return request.post('/answer/api/v1/question', params);
};

// 获取问题详情
export const questionDetail = (id: string) => {
  return request.get<Type.QuestionDetailRes>(
    `/answer/api/v1/question/info?id=${id}`,
    { allow404: true },
  );
};

// 更新问题
export const modifyQuestion = (params) => {
  return request.put(`/answer/api/v1/question`, params);
};
```

### 2.3 后端路由层

**文件**: [internal/router/answer_api_router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/router/answer_api_router.go)

```go
// 问题相关路由
r.GET("/question/info", a.questionController.GetQuestion)        // 获取详情
r.GET("/question/page", a.questionController.QuestionPage)       // 分页列表
r.POST("/question", a.questionController.AddQuestion)            // 创建问题
r.POST("/question/answer", a.questionController.AddQuestionByAnswer) // 自问自答
r.PUT("/question", a.questionController.UpdateQuestion)          // 更新问题
r.DELETE("/question", a.questionController.RemoveQuestion)       // 删除问题
r.PUT("/question/status", a.questionController.CloseQuestion)    // 关闭问题
r.PUT("/question/operation", a.questionController.OperationQuestion) // 置顶/隐藏
r.PUT("/question/reopen", a.questionController.ReopenQuestion)   // 重开问题
```

---

## 三、后端校验与处理链路

### 3.1 Controller 层：请求接入

**文件**: [internal/controller/question_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/controller/question_controller.go#L384-L487)

**核心方法**: `AddQuestion(ctx *gin.Context)`

#### 处理步骤：

1. **参数绑定与校验**（`handler.BindAndCheckReturnErr`）:
   ```go
   req := &schema.QuestionAdd{}
   errFields := handler.BindAndCheckReturnErr(ctx, req)
   ```
   - 调用 `ctx.ShouldBind(data)` 绑定 JSON
   - 调用 `validator.Check(data)` 执行结构化校验

2. **防重复提交**（`DuplicateRequestRejection`）:
   - 基于请求内容生成唯一键，防止重复提交

3. **权限校验**（`rankService.CheckOperationPermissionsForRanks`）:
   ```go
   canList, requireRanks, err := qc.rankService.CheckOperationPermissionsForRanks(ctx, req.UserID, []string{
       permission.QuestionAdd,          // 提问权限
       permission.QuestionEdit,         // 编辑权限
       permission.QuestionDelete,       // 删除权限
       permission.TagUseReservedTag,    // 使用保留标签权限
       permission.TagAdd,               // 新建标签权限
       permission.LinkUrlLimit,         // 链接限制
   })
   ```

4. **验证码校验**（`ActionRecordVerifyCaptcha`）:
   - 非管理员用户需验证图形验证码

5. **业务校验**（`questionService.CheckAddQuestion`）:
   - 标签数量校验
   - 内容长度校验
   - 推荐标签存在性校验
   - 保留标签使用权限校验

6. **调用 Service 层**:
   ```go
   req.UserAgent = ctx.GetHeader("User-Agent")
   req.IP = ctx.ClientIP()
   resp, err := qc.questionService.AddQuestion(ctx, req)
   ```

### 3.2 Schema 层：请求/响应定义与参数校验

**文件**: [internal/schema/question_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/schema/question_schema.go)

#### 创建请求结构：
```go
type QuestionAdd struct {
    Title   string     `validate:"required,notblank,gte=6,lte=150" json:"title"`
    Content string     `validate:"gte=0,lte=65535" json:"content"`
    HTML    string     `json:"-"` // Markdown 转 HTML 后的值
    Tags    []*TagItem `validate:"dive" json:"tags"`
    UserID  string     `json:"-"`
    QuestionPermission            // 权限字段
    CaptchaID   string `json:"captcha_id"`
    CaptchaCode string `json:"captcha_code"`
    IP          string `json:"-"`
    UserAgent   string `json:"-"`
}

// 自定义校验方法
func (req *QuestionAdd) Check() (errFields []*validator.FormErrorField, err error) {
    req.HTML = converter.Markdown2HTML(req.Content)  // Markdown 转 HTML
    for _, tag := range req.Tags {
        if len(tag.OriginalText) > 0 {
            tag.ParsedText = converter.Markdown2HTML(tag.OriginalText)
        }
    }
    return nil, nil
}
```

#### 校验规则说明：
- `required` - 必填
- `notblank` - 非空白
- `gte=6,lte=150` - 长度 6-150 字符
- `dive` - 递归校验数组元素

### 3.3 通用参数绑定与校验

**文件**: [internal/base/handler/handler.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/handler/handler.go#L63-L78)

```go
func BindAndCheck(ctx *gin.Context, data any) bool {
    lang := GetLangByCtx(ctx)
    // 参数绑定
    if err := ctx.ShouldBind(data); err != nil {
        HandleResponse(ctx, myErrors.New(http.StatusBadRequest, reason.RequestFormatError), nil)
        return true
    }
    // 结构化校验
    errField, err := validator.GetValidatorByLang(lang).Check(data)
    if err != nil {
        HandleResponse(ctx, err, errField)
        return true
    }
    return false
}
```

---

## 四、Service 层：核心业务逻辑

**文件**: [internal/service/content/question_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/content/question_service.go)

### 4.1 业务校验：`CheckAddQuestion`

```go
func (qs *QuestionService) CheckAddQuestion(ctx context.Context, req *schema.QuestionAdd) (errorlist any, err error) {
    // 1. 最少标签数校验
    minimumTags, _ := qs.tagCommon.GetMinimumTags(ctx)
    if len(req.Tags) < minimumTags { ... }
    
    // 2. 最少内容长度校验
    minimumContentLength, _ := qs.questioncommon.GetMinimumContentLength(ctx)
    if len(req.Content) < minimumContentLength { ... }
    
    // 3. 推荐标签存在性校验
    recommendExist, _ := qs.tagCommon.ExistRecommend(ctx, req.Tags)
    if !recommendExist { ... }
    
    // 4. 保留标签使用权限校验
    if !req.CanUseReservedTag {
        taglist, err := qs.AddQuestionCheckTags(ctx, tags)
        if err != nil { ... }
    }
    return nil, nil
}
```

### 4.2 核心创建逻辑：`AddQuestion`

**完整流程**（第 307-469 行）：

```go
func (qs *QuestionService) AddQuestion(ctx context.Context, req *schema.QuestionAdd) (questionInfo any, err error) {
    // --- 前置校验（重复调用 CheckAddQuestion 逻辑）---
    // 1. 最少标签数、最少内容长度、推荐标签、保留标签校验
    
    // 2. 标签处理
    tagNameList := make([]string, 0)
    for _, tag := range req.Tags {
        tag.SlugName = strings.ReplaceAll(tag.SlugName, " ", "-")
        tagNameList = append(tagNameList, tag.SlugName)
    }
    tags, _ := qs.tagCommon.GetTagListByNames(ctx, tagNameList)
    
    // 3. 构建 Question 实体
    question := &entity.Question{
        UserID:         req.UserID,
        Title:          req.Title,
        OriginalText:   req.Content,      // Markdown 原文
        ParsedText:     req.HTML,         // 解析后的 HTML
        Status:         entity.QuestionStatusPending, // 初始状态：待审核
        CreatedAt:      time.Now(),
        PostUpdateTime: time.Now(),
        Pin:            entity.QuestionUnPin,
        Show:           entity.QuestionShow,
        // 其他字段默认值初始化...
    }
    
    // 4. 持久化到数据库
    err = qs.questionRepo.AddQuestion(ctx, question)
    
    // 5. 审核流程
    question.Status = qs.reviewService.AddQuestionReview(ctx, question, req.Tags, req.IP, req.UserAgent)
    // 根据审核结果更新状态（Available / Pending）
    qs.questionRepo.UpdateQuestionStatus(ctx, question.ID, question.Status)
    
    // 6. 问题链接处理（如果审核通过）
    if question.Status == entity.QuestionStatusAvailable {
        question.ParsedText, _ = qs.questioncommon.UpdateQuestionLink(...)
        qs.questionRepo.UpdateQuestion(ctx, question, []string{"parsed_text"})
    }
    
    // 7. 标签关联
    objectTagData := schema.TagChange{
        ObjectID: question.ID,
        Tags:     req.Tags,
        UserID:   req.UserID,
    }
    qs.ChangeTag(ctx, &objectTagData)  // 创建 tag_rel 关联
    
    // 8. 更新搜索索引
    qs.questionRepo.UpdateSearch(ctx, question.ID)
    
    // 9. 创建版本记录（Revision）
    revisionDTO := &schema.AddRevisionDTO{...}
    revisionID, _ := qs.revisionService.AddRevision(ctx, revisionDTO, true)
    
    // 10. 更新用户提问计数
    userQuestionCount, _ := qs.questioncommon.GetUserQuestionCount(ctx, question.UserID)
    qs.userCommon.UpdateQuestionCount(ctx, question.UserID, userQuestionCount)
    
    // 11. 发送活动事件（Activity Queue）
    qs.activityQueueService.Send(ctx, &schema.ActivityMsg{
        UserID:          question.UserID,
        ObjectID:        question.ID,
        ActivityTypeKey: constant.ActQuestionAsked,
        RevisionID:      revisionID,
    })
    
    // 12. 发送通知事件（Notification Queue）
    if question.Status == entity.QuestionStatusAvailable {
        qs.externalNotificationQueueService.Send(ctx,
            schema.CreateNewQuestionNotificationMsg(...))
    }
    
    // 13. 发送领域事件（Event Queue）
    qs.eventQueueService.Send(ctx, schema.NewEvent(constant.EventQuestionCreate, req.UserID)...)
    
    // 14. 向量索引同步（用于 AI 搜索）
    if question.Status == entity.QuestionStatusAvailable {
        qs.vectorSyncService.Send(ctx, &vector_sync.Task{...})
    }
    
    // 15. 返回问题详情
    return qs.GetQuestion(ctx, question.ID, question.UserID, req.QuestionPermission)
}
```

---

## 五、持久化层：Repository 与 Entity

### 5.1 实体定义

**文件**: [internal/entity/question_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/entity/question_entity.go)

```go
// 问题状态常量
const (
    QuestionStatusAvailable = 1   // 正常
    QuestionStatusClosed    = 2   // 已关闭
    QuestionStatusDeleted   = 10  // 已删除
    QuestionStatusPending   = 11  // 待审核
)

// 问题实体
type Question struct {
    ID               string    `xorm:"not null pk BIGINT(20) id"`
    CreatedAt        time.Time `xorm:"not null default CURRENT_TIMESTAMP TIMESTAMP created_at"`
    UpdatedAt        time.Time `xorm:"updated_at TIMESTAMP"`
    UserID           string    `xorm:"not null default 0 BIGINT(20) INDEX user_id"`
    LastEditUserID   string    `xorm:"not null default 0 BIGINT(20) last_edit_user_id"`
    Title            string    `xorm:"not null default '' VARCHAR(150) title"`
    OriginalText     string    `xorm:"not null MEDIUMTEXT original_text"`  // Markdown 原文
    ParsedText       string    `xorm:"not null MEDIUMTEXT parsed_text"`    // HTML 内容
    Pin              int       `xorm:"not null default 1 INT(11) pin"`     // 1:未置顶 2:置顶
    Show             int       `xorm:"not null default 1 INT(11) show"`    // 1:显示 2:隐藏
    Status           int       `xorm:"not null default 1 INT(11) status"`  // 状态
    ViewCount        int       `xorm:"not null default 0 INT(11) view_count"`
    UniqueViewCount  int       `xorm:"not null default 0 INT(11) unique_view_count"`
    VoteCount        int       `xorm:"not null default 0 INT(11) vote_count"`
    AnswerCount      int       `xorm:"not null default 0 INT(11) answer_count"`
    HotScore         int       `xorm:"not null default 0 INT(11) hot_score"`
    CollectionCount  int       `xorm:"not null default 0 INT(11) collection_count"`
    FollowCount      int       `xorm:"not null default 0 INT(11) follow_count"`
    AcceptedAnswerID string    `xorm:"not null default 0 BIGINT(20) accepted_answer_id"`
    LastAnswerID     string    `xorm:"not null default 0 BIGINT(20) last_answer_id"`
    PostUpdateTime   time.Time `xorm:"post_update_time TIMESTAMP"`
    RevisionID       string    `xorm:"not null default 0 BIGINT(20) revision_id"`
    LinkedCount      int       `xorm:"not null default 0 INT(11) linked_count"`
}

func (Question) TableName() string {
    return "question"
}
```

### 5.2 Repository 层

**文件**: [internal/repo/question/question_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/repo/question/question_repo.go)

```go
type questionRepo struct {
    data         *data.Data      // 数据库连接
    uniqueIDRepo unique.UniqueIDRepo // ID 生成器
}

// 新增问题
func (qr *questionRepo) AddQuestion(ctx context.Context, question *entity.Question) (err error) {
    // 1. 生成唯一 ID（雪花算法）
    question.ID, err = qr.uniqueIDRepo.GenUniqueIDStr(ctx, question.TableName())
    
    // 2. 插入数据库（XORM）
    _, err = qr.data.DB.Context(ctx).Insert(question)
    
    // 3. 如果启用短 ID，转换为短 ID 格式
    if handler.GetEnableShortID(ctx) {
        question.ID = uid.EnShortID(question.ID)
    }
    return
}

// 更新问题
func (qr *questionRepo) UpdateQuestion(ctx context.Context, question *entity.Question, cols []string) (err error) {
    question.ID = uid.DeShortID(question.ID)  // 短 ID 转长 ID
    _, err = qr.data.DB.Context(ctx).
        Where("id =?", question.ID).
        Cols(cols...).  // 只更新指定列
        Update(question)
    // ...
}

// 更新问题状态
func (qr *questionRepo) UpdateQuestionStatus(ctx context.Context, questionID string, status int) (err error) {
    questionID = uid.DeShortID(questionID)
    question := &entity.Question{Status: status}
    _, err = qr.data.DB.Context(ctx).
        Where("id =?", questionID).
        Cols("status", "updated_at").
        Update(question)
    return
}

// 获取单条问题
func (qr *questionRepo) GetQuestion(ctx context.Context, id string) (*entity.Question, bool, error) {
    id = uid.DeShortID(id)
    question := &entity.Question{}
    has, err := qr.data.DB.Context(ctx).Where("id = ?", id).Get(question)
    if has && handler.GetEnableShortID(ctx) {
        question.ID = uid.EnShortID(question.ID)
    }
    return question, has, err
}

// 分页查询
func (qr *questionRepo) GetQuestionPage(ctx context.Context, page, pageSize int,
    tagIDs []string, userID string, orderCond string, inDays int,
    showHidden bool, showPending bool) ([]*entity.Question, int64, error) {
    // 复杂的动态查询条件构建...
}
```

---

## 六、问题展示链路

### 6.1 问题列表页

**前端入口**: [ui/src/pages/Questions/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/pages/Questions/index.tsx)

```tsx
const Questions: FC = () => {
    // 请求参数
    const reqParams: Type.QueryQuestionsReq = {
        page_size: 20,
        page: curPage,
        order: curOrder,  // newest/active/hot/score/unanswered/recommend
    };
    
    // 使用 SWR 进行数据请求与缓存
    const { data: listData, isLoading: listLoading } =
        curOrder === 'recommend'
            ? useQuestionRecommendList(reqParams)
            : useQuestionList(reqParams);
    
    return (
        <QuestionList
            source="questions"
            data={listData}
            order={curOrder}
            isLoading={listLoading}
        />
    );
};
```

**前端数据请求**: [ui/src/services/client/question.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/services/client/question.ts)

```typescript
export const useQuestionList = (params: Type.QueryQuestionsReq) => {
    const apiUrl = `/answer/api/v1/question/page?${qs.stringify(params)}`;
    const { data, error } = useSWR<Type.ListResult, Error>(apiUrl, (url) =>
        request.get(url, { allow404: true }),
    );
    return { data, isLoading: !data && !error, error };
};
```

**列表组件**: [ui/src/components/QuestionList/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/components/QuestionList/index.tsx)

- 支持排序切换：newest/active/unanswered/recommend/frequent/score
- 支持视图切换：card/compact
- 置顶问题单独展示（最多 3 条）
- 骨架屏加载

**后端列表查询 Service**: `GetQuestionPage` [question_service.go#L1470-L1530](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/content/question_service.go#L1470-L1530)

```go
func (qs *QuestionService) GetQuestionPage(ctx context.Context, req *schema.QuestionPageReq) (
    questions []*schema.QuestionPageResp, total int64, err error) {
    
    // 1. 权限判断：是否显示隐藏的问题（作者/管理员/版主）
    showHidden := req.LoginUserID == req.UserIDBeSearched || userRole == admin/moderator
    
    // 2. 标签条件处理（支持主标签与同义词标签）
    var tagIDs = make([]string, 0)
    if len(req.Tag) > 0 {
        tagInfo, exist, _ := qs.tagCommon.GetTagBySlugName(ctx, req.Tag)
        synTagIds, _ := qs.tagCommon.GetTagIDsByMainTagID(ctx, tagInfo.ID)
        tagIDs = append(tagIDs, synTagIds...)
        tagIDs = append(tagIDs, tagInfo.ID)
    }
    
    // 3. 用户条件处理
    if req.Username != "" {
        userinfo, exist, _ := qs.userCommon.GetUserBasicInfoByUserName(ctx, req.Username)
        req.UserIDBeSearched = userinfo.ID
    }
    
    // 4. 热门排序限定 90 天内
    if req.OrderCond == schema.QuestionOrderCondHot {
        req.InDays = schema.HotInDays
    }
    
    // 5. 调用 Repository 层查询
    questionList, total, err := qs.questionRepo.GetQuestionPage(...)
    
    // 6. 格式化输出（补充用户信息、标签信息等）
    questions, err = qs.questioncommon.FormatQuestionsPage(ctx, questionList, req.LoginUserID, req.OrderCond)
    
    return questions, total, nil
}
```

### 6.2 问题详情页

**前端入口**: [ui/src/pages/Questions/Detail/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/pages/Questions/Detail/index.tsx)

```tsx
const Index = () => {
    const { qid = '' } = useParams();
    
    // 获取问题详情
    const getDetail = async () => {
        const res = await questionDetail(qid);
        if (res) {
            setQuestion(res);
            // 补充页面用户信息用于 SEO
            setUsers([{
                displayName: res.user_info?.display_name,
                userName: res.user_info?.username,
            }]);
        }
        setIsLoading(false);
    };
    
    // 获取回答列表
    const requestAnswers = async () => {
        const res = await getAnswers({
            order: 'default',
            question_id: qid,
            page: 1,
            page_size: 999,
        });
        setAnswers({ ...res, count: res.list.length });
    };
};
```

**后端详情查询**: `GetQuestion` [question_service.go#L1088-L1143](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/content/question_service.go#L1088-L1143)

```go
func (qs *QuestionService) GetQuestion(ctx context.Context, questionID, userID string,
    per schema.QuestionPermission) (resp *schema.QuestionInfoResp, err error) {
    
    // 1. 获取问题基本信息
    question, err := qs.questioncommon.Info(ctx, questionID, userID)
    
    // 2. 可见性校验
    if (question.Status == Deleted || Pending) && !per.CanReopen && question.UserID != userID {
        return nil, errors.NotFound(reason.QuestionNotFound)
    }
    if question.Show == Hide && !per.IsAdminModerator && question.UserID != userID {
        return nil, errors.NotFound(reason.QuestionNotFound)
    }
    
    // 3. 根据状态调整权限（如已关闭的问题不能再关闭）
    if question.Status == Closed { per.CanClose = false }
    
    // 4. 添加状态提示（已删除/待审核）
    if question.Status == Deleted {
        operation.Msg = "Question already deleted"
        operation.Level = Danger
    }
    
    // 5. 生成描述摘要（用于 SEO）
    question.Description = htmltext.FetchExcerpt(question.HTML, "...", 240)
    
    // 6. 计算用户可操作按钮
    question.MemberActions = permission.GetQuestionPermission(...)
    
    return question, nil
}
```

---

## 七、完整端到端流转图

### 7.1 问题创建流程

```
用户在提问页面输入内容
        ↓
[前端] ui/src/pages/Questions/Ask/index.tsx
  - 表单校验 + 相似问题查询
  - submitQuestion() → saveQuestion()
        ↓
[前端 API] ui/src/services/common.ts
  - POST /answer/api/v1/question
        ↓
[路由] internal/router/answer_api_router.go
  - r.POST("/question", AddQuestion)
        ↓
[Controller] internal/controller/question_controller.go
  - AddQuestion()
    ├─ 参数绑定与校验 (BindAndCheck)
    ├─ 防重复提交
    ├─ 权限校验 (rankService)
    ├─ 验证码校验
    ├─ 业务校验 (CheckAddQuestion)
    └─ 调用 Service.AddQuestion
        ↓
[Service] internal/service/content/question_service.go
  - AddQuestion()
    ├─ 构建 Question Entity
    ├─ Repository.AddQuestion (DB Insert)
    ├─ 审核流程 (reviewService)
    ├─ 标签关联 (ChangeTag)
    ├─ 创建版本记录 (revisionService)
    ├─ 更新用户计数
    ├─ 发送活动事件 (Activity Queue)
    ├─ 发送通知事件 (Notification Queue)
    ├─ 发送领域事件 (Event Queue)
    └─ 向量索引同步 (Vector Sync)
        ↓
[Repository] internal/repo/question/question_repo.go
  - AddQuestion()
    ├─ 生成唯一 ID (GenUniqueIDStr)
    ├─ XORM Insert
    └─ 短 ID 转换 (EnShortID)
        ↓
[数据库] question 表
  - INSERT INTO question (...) VALUES (...)
        ↓
[异步队列]
  ├─ Activity Queue → 用户活跃度、积分
  ├─ Notification Queue → 站内通知、邮件通知
  ├─ Event Queue → 领域事件处理
  └─ Vector Sync → AI 向量搜索索引
```

### 7.2 问题展示流程

```
用户访问问题列表或详情页
        ↓
[前端] useQuestionList() / questionDetail()
        ↓
[API] GET /answer/api/v1/question/page 或 /question/info
        ↓
[Controller] QuestionPage() / GetQuestion()
  - 参数解析
  - 登录用户信息注入
        ↓
[Service] GetQuestionPage() / GetQuestion()
  ├─ 权限判断
  ├─ 条件构建（标签、用户、排序）
  ├─ Repository 查询
  └─ 数据格式化（用户信息、标签、操作权限）
        ↓
[Repository] GetQuestionPage() / GetQuestion()
  - XORM 查询
  - 短 ID 转换
        ↓
[数据库] SELECT ... FROM question WHERE ...
        ↓
[前端渲染] QuestionList 组件 / Question 详情组件
  ├─ 列表渲染（卡片/紧凑模式）
  ├─ 置顶问题优先展示
  ├─ 详情页展示（标题、内容、标签、操作按钮）
  └─ 回答列表展示
```

---

## 八、关键设计要点

### 8.1 ID 设计
- 使用雪花算法生成全局唯一 ID（长 ID）
- 对外暴露短 ID（Base62 编码）
- Repository 层自动处理长短 ID 转换

### 8.2 内容双存储
- `OriginalText`: 存储 Markdown 原文，用于编辑
- `ParsedText`: 存储渲染后的 HTML，用于展示

### 8.3 状态机
```
Pending (待审核) → Available (正常) → Closed (已关闭)
                       ↓                ↓
                    Deleted (已删除)  Reopen (重开)
```

### 8.4 异步解耦
核心操作通过异步队列解耦：
- **Activity Queue**: 用户活跃度、积分统计
- **Notification Queue**: 通知推送
- **Event Queue**: 领域事件（插件扩展点）
- **Vector Sync**: AI 向量索引同步

### 8.5 审核机制
问题创建后默认进入 `Pending` 状态，由 `reviewService` 根据用户信誉、内容质量等决定是否直接放行或进入人工审核队列。

---

## 九、核心文件索引

| 层级 | 文件路径 | 核心职责 |
|------|---------|---------|
| 前端页面 | [ui/src/pages/Questions/Ask/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/pages/Questions/Ask/index.tsx) | 提问表单页面 |
| 前端页面 | [ui/src/pages/Questions/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/pages/Questions/index.tsx) | 问题列表页 |
| 前端页面 | [ui/src/pages/Questions/Detail/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/pages/Questions/Detail/index.tsx) | 问题详情页 |
| 前端 API | [ui/src/services/common.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/services/common.ts) | 通用 API 定义 |
| 前端 API | [ui/src/services/client/question.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/services/client/question.ts) | 问题相关 SWR Hooks |
| 前端组件 | [ui/src/components/QuestionList/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/ui/src/components/QuestionList/index.tsx) | 问题列表组件 |
| 路由层 | [internal/router/answer_api_router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/router/answer_api_router.go) | API 路由定义 |
| Controller | [internal/controller/question_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/controller/question_controller.go) | 问题 HTTP 接口 |
| Service | [internal/service/content/question_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/service/content/question_service.go) | 问题业务逻辑 |
| Repository | [internal/repo/question/question_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/repo/question/question_repo.go) | 问题数据访问 |
| Entity | [internal/entity/question_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/entity/question_entity.go) | 问题实体定义 |
| Schema | [internal/schema/question_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/schema/question_schema.go) | 请求/响应 DTO |
| 通用处理 | [internal/base/handler/handler.go](file:///d:/fz/0601-1/solo-dogfeeding/code/61-answer/internal/base/handler/handler.go) | 参数绑定与响应处理 |
