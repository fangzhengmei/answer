# Markdown2HTML 预渲染落库模型、Post 渲染边界、评论图片孤儿隐患与 Branding/Avatar 追踪对称性

---

## 一、Markdown2HTML 在写入时预渲染落库的真实模型

### 1.1 四种对象类型的写入路径

| 对象类型 | Schema Check() | 写入目标 | Revision? | 存储字段 |
|----------|----------------|----------|-----------|----------|
| Question | `req.HTML = converter.Markdown2HTML(req.Content)` | question 表 | ✅ AddRevision | `original_text` + `parsed_text` |
| Answer | `req.HTML = converter.Markdown2HTML(req.Content)` | answer 表 | ✅ AddRevision | `original_text` + `parsed_text` |
| Comment | `req.ParsedText = converter.Markdown2HTML(req.OriginalText)` | comment 表 | ❌ 无 Revision | `original_text` + `parsed_text` |
| User Bio | `userInfo.BioHTML = converter.Markdown2HTML(basicUserInfo.Bio)` | user 表 | ❌ 无 Revision | `bio` + `bio_html` |

### 1.2 Question 写入详析

**Schema 层** [question_schema.go#L96-L97](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/schema/question_schema.go#L96-L97)

```go
type QuestionAdd struct {
    Content string `validate:"gte=0,lte=65535" json:"content"`
    HTML    string `json:"-"`   // 不序列化到 JSON，仅内部使用
}

func (req *QuestionAdd) Check() (errFields []*validator.FormErrorField, err error) {
    req.HTML = converter.Markdown2HTML(req.Content)  // 预渲染
    ...
}
```

**Service 层** [question_service.go#L376-L377](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/content/question_service.go#L376-L377)

```go
question.OriginalText = req.Content   // Markdown 原文
question.ParsedText = req.HTML        // 预渲染的 HTML
```

**落库后写 Revision** [question_service.go#L417-L426](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/content/question_service.go#L417-L426)

```go
revisionDTO := &schema.AddRevisionDTO{
    UserID:   question.UserID,
    ObjectID: question.ID,
    Title:    question.Title,
}
questionWithTagsRevision := qs.changeQuestionToRevision(ctx, question, tags)
infoJSON, _ := json.Marshal(questionWithTagsRevision)   // 整个 Question+Tags 序列化
revisionDTO.Content = string(infoJSON)                    // Revision.content = JSON 字符串
revisionID, err := qs.revisionService.AddRevision(ctx, revisionDTO, true)
```

**关键**：Question 的 Revision.content 是**整个 Question 实体（含 Tags）的 JSON 序列化**，其中 `original_text` 字段包含 Markdown 原文，**图片 URL 以 Markdown 语法嵌入**。

### 1.3 Answer 写入详析

**Schema 层** [answer_schema.go#L63-L64](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/schema/answer_schema.go#L63-L64)

```go
func (req *AnswerAddReq) Check() (errFields []*validator.FormErrorField, err error) {
    req.HTML = converter.Markdown2HTML(req.Content)
    ...
}
```

**Service 层** [answer_service.go#L267-L268](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/content/answer_service.go#L267-L268)

```go
insertData.OriginalText = req.Content   // Markdown 原文
insertData.ParsedText = req.HTML        // 预渲染 HTML
```

**落库后写 Revision** [answer_service.go#L312-L319](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/content/answer_service.go#L312-L319)

```go
revisionDTO := &schema.AddRevisionDTO{
    UserID:   insertData.UserID,
    ObjectID: insertData.ID,
    Title:    "",
}
infoJSON, _ := json.Marshal(insertData)  // 整个 Answer 实体序列化
revisionDTO.Content = string(infoJSON)
revisionID, err := as.revisionService.AddRevision(ctx, revisionDTO, true)
```

**关键**：Answer 的 Revision.content 是**整个 Answer 实体的 JSON 序列化**，包含 `original_text`（Markdown 原文）。

### 1.4 Comment 写入详析——无 Revision

**Schema 层** [comment_schema.go#L59-L60](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/schema/comment_schema.go#L59-L60)

```go
func (req *AddCommentReq) Check() (errFields []*validator.FormErrorField, err error) {
    req.ParsedText = converter.Markdown2HTML(req.OriginalText)
    ...
}
```

**Service 层** [comment_service.go#L136-L169](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/comment/comment_service.go#L136-L169)

```go
func (cs *CommentService) AddComment(ctx context.Context, req *schema.AddCommentReq) (...) {
    comment := &entity.Comment{}
    _ = copier.Copy(comment, req)  // req.OriginalText → comment.OriginalText
                                    // req.ParsedText  → comment.ParsedText
    
    err = cs.commentRepo.AddComment(ctx, comment)  // 直接写 comment 表
    // ❌ 没有调用 revisionService.AddRevision()
}
```

**关键**：Comment 只写 `comment` 表，**不写 `revision` 表**。这是一个结构性设计差异。

### 1.5 User Bio 写入详析——无 Revision

[user_center_login_service.go#L166](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/user_external_login/user_center_login_service.go#L166)

```go
userInfo.BioHTML = converter.Markdown2HTML(basicUserInfo.Bio)
```

User 表中 `bio` 字段存 Markdown 原文，`bio_html` 存预渲染 HTML，同样**不写 Revision**。

### 1.6 数据库模型对比

```
question 表          answer 表           comment 表          user 表
┌──────────────┐    ┌──────────────┐    ┌──────────────┐   ┌──────────────┐
│original_text │    │original_text │    │original_text │   │bio           │
│parsed_text   │    │parsed_text   │    │parsed_text   │   │bio_html      │
│revision_id   │    │revision_id   │    │(无)          │   │(无)          │
└──────┬───────┘    └──────┬───────┘    └──────────────┘   └──────────────┘
       │                   │
       ▼                   ▼
  revision 表         revision 表
┌──────────────┐    ┌──────────────┐
│object_id     │    │object_id     │
│content (JSON)│    │content (JSON)│
│  └─original  │    │  └─original  │
│    _text     │    │    _text     │
└──────────────┘    └──────────────┘
  ✅ 含图片 URL         ✅ 含图片 URL
```

---

## 二、Post 渲染接口只服务编辑器预览的边界

### 2.1 接口定义

[upload_controller.go#L98-L114](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/controller/upload_controller.go#L98-L114)

```go
// PostRender render post content
// @Summary render post content
// @Description render post content
// @Tags Upload
// @Accept json
// @Produce json
// @Security ApiKeyAuth
// @Param data body schema.PostRenderReq true "PostRenderReq"
// @Router /answer/api/v1/post/render [post]
func (uc *UploadController) PostRender(ctx *gin.Context) {
    req := &schema.PostRenderReq{}
    if handler.BindAndCheck(ctx, req) {
        return
    }
    handler.HandleResponse(ctx, nil, converter.Markdown2HTML(req.Content))
}
```

[render_schema.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/schema/render_schema.go)

```go
type PostRenderReq struct {
    Content string `json:"content"`
}
```

### 2.2 路由注册

[answer_api_router.go#L313](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/router/answer_api_router.go#L313)

```go
r.POST("/post/render", a.uploadController.PostRender)
```

### 2.3 边界分析

| 特征 | 说明 |
|------|------|
| 入参 | 纯 Markdown 文本（`content` 字段） |
| 出参 | 纯 HTML 字符串 |
| 副作用 | **无**——不写库、不修改任何状态 |
| 鉴权 | 需登录（`ApiKeyAuth`），但无权限细分 |
| 用途 | **仅服务前端编辑器实时预览** |

**为什么不是读取时的渲染路径**：

Question/Answer/Comment 写入时已经在 `Check()` 中预渲染了 HTML 并存入 `parsed_text`/`bio_html` 字段。**读取时直接返回 `parsed_text`，不再调用 Markdown2HTML**。

因此 `POST /post/render` 的唯一场景是：用户在编辑器中输入 Markdown，前端在预览面板实时展示效果。该接口是一个**无状态的纯函数**——输入 Markdown，输出安全 HTML，不涉及任何持久化。

```
写入路径:  Markdown → Check() → Markdown2HTML → parsed_text 存库
读取路径:  parsed_text 直接返回给前端 → Viewer.tsx 渲染
预览路径:  Markdown → POST /post/render → Markdown2HTML → 返回 HTML → Viewer.tsx 渲染
                                    ↑ 仅此接口，纯无状态
```

---

## 三、评论图片不写 Revision 致使 CleanOrphanUploadFiles 模糊搜索漏看的隐患

### 3.1 CleanOrphanUploadFiles 的搜索逻辑

[file_record_service.go#L94-L158](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/file_record/file_record_service.go#L94-L158)

```
对每个 Available 且超过 48 小时的 FileRecord:
  │
  ├── 是 Branding/Avatar 文件?
  │     → 走专用 IsBrandingFileUsed / IsAvatarFileUsed 检查
  │
  ├── ObjectID != "0"?
  │     → revisionRepo.GetLastRevisionByObjectID(ObjectID)
  │       → WHERE object_id = ?  (索引查询)
  │       → 找到 → 保留; 未找到 → 删除
  │
  └── ObjectID == "0"?
        → revisionRepo.GetLastRevisionByFileURL(FileURL)
          → WHERE content LIKE '%url%'  (全表模糊搜索 revision.content)
          → 找到 → 回填 ObjectID，保留; 未找到 → 删除
```

### 3.2 评论图片为什么被漏看

**核心原因**：`GetLastRevisionByFileURL()` 只搜索 `revision` 表，而评论**不写 revision**。

```
评论图片的完整生命周期:

1. 用户在评论中上传图片
   → 上传 API 返回 URL
   → FileRecord { ObjectID: "0", Source: "user_post", FileURL: "https://site/uploads/post/abc.jpg" }

2. 用户提交评论
   → comment.OriginalText = "![img](https://site/uploads/post/abc.jpg)"
   → comment.ParsedText = "<img src='https://site/uploads/post/abc.jpg'>"
   → 写入 comment 表
   → ❌ 不写 revision 表

3. 48 小时后，CleanOrphanUploadFiles 扫描
   → FileRecord.ObjectID == "0"
   → revisionRepo.GetLastRevisionByFileURL("https://site/uploads/post/abc.jpg")
   → WHERE content LIKE '%https://site/uploads/post/abc.jpg%'
   → 搜索 revision 表
   → ❌ 评论的 URL 在 comment 表，不在 revision 表
   → 未找到 → 判定为孤儿文件 → 删除！
```

**后果**：评论中的图片被静默删除，用户看到损坏的图片链接。

### 3.3 同样受影响的对象：User Bio 中的图片

User 的 `bio` 字段也可能包含图片 URL，同样**不写 revision**，清理时也会被误删。

### 3.4 对比：Question/Answer 为什么不受影响

| 对象 | 写 Revision? | 清理搜索位置 | 是否受影响 |
|------|-------------|-------------|-----------|
| Question | ✅ | revision.content LIKE '%url%' | ❌ 不受影响 |
| Answer | ✅ | revision.content LIKE '%url%' | ❌ 不受影响 |
| Comment | ❌ | revision.content LIKE '%url%' | ✅ **受影响** |
| User Bio | ❌ | revision.content LIKE '%url%' | ✅ **受影响** |

### 3.5 Branding/Avatar 为什么不受影响

Branding 和 Avatar 文件走了**专用检查路径**，不依赖 revision 搜索：

```go
if isBrandingOrAvatarFile(fileRecord.FilePath) {
    if strings.Contains(fileRecord.FilePath, constant.BrandingSubPath+"/") {
        if fs.siteInfoService.IsBrandingFileUsed(ctx, fileRecord.FilePath) {
            continue  // 保留
        }
    } else if strings.Contains(fileRecord.FilePath, constant.AvatarSubPath+"/") {
        if fs.userService.IsAvatarFileUsed(ctx, fileRecord.FilePath) {
            continue  // 保留
        }
    }
    // 未找到引用 → 删除
    fs.DeleteAndMoveFileRecord(ctx, fileRecord)
    continue
}
```

但**评论图片的 Source 是 `user_post`，文件路径在 `post/` 子目录**，不会被 `isBrandingOrAvatarFile()` 拦截，直接进入 revision 搜索路径——而 revision 中没有评论的 URL。

### 3.6 修复建议

**方案 A**：为评论图片增加专用搜索

在 `CleanOrphanUploadFiles` 中，对 `user_post` 类型的文件，增加在 `comment` 表中的 LIKE 搜索：

```go
// 现有逻辑：搜索 revision 表
lastRevision, exist, err := fs.revisionRepo.GetLastRevisionByFileURL(ctx, fileRecord.FileURL)
if !exist {
    // 新增：搜索 comment 表
    commentExists, err := fs.commentRepo.IsFileURLUsed(ctx, fileRecord.FileURL)
    if commentExists {
        continue  // 保留
    }
    // 新增：搜索 user.bio 字段
    userExists, err := fs.userRepo.IsBioFileUsed(ctx, fileRecord.FileURL)
    if userExists {
        continue  // 保留
    }
}
```

**方案 B**：为 Comment 增加 Revision 记录

让评论写入时也写 revision，统一搜索逻辑。但这与当前"评论不可回滚"的设计理念冲突。

**方案 C**：在 FileRecord 中增加 Source 维度的搜索路由

根据 FileRecord.Source 分别调用不同的搜索策略，而非统一搜索 revision。

---

## 四、Branding 与 Avatar 通过 site_info 和 user 字段 LIKE 引用追踪的对称机制

### 4.1 Branding 追踪

**存储位置**：`site_info` 表，`type = "branding"`，`content` 字段存储 JSON

```json
{
  "logo": "https://site/uploads/branding/abc123.png",
  "mobile_logo": "https://site/uploads/branding/def456.png",
  "square_icon": "https://site/uploads/branding/ghi789.png",
  "favicon": "https://site/uploads/branding/jkl012.ico"
}
```

**搜索实现** [siteinfo_repo.go#L107-L120](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/repo/site_info/siteinfo_repo.go#L107-L120)

```go
func (sr *siteInfoRepo) IsBrandingFileUsed(ctx context.Context, filePath string) (bool, error) {
    siteInfo := &entity.SiteInfo{}
    count, err := sr.data.DB.Context(ctx).
        Table("site_info").
        Where(builder.Eq{"type": "branding"}).           // 精确匹配 type
        And(builder.Like{"content", "%" + filePath + "%"}). // 模糊匹配 content
        Count(&siteInfo)
    return count > 0, nil
}
```

**SQL 等价**：
```sql
SELECT COUNT(*) FROM site_info 
WHERE type = 'branding' AND content LIKE '%branding/abc123.png%'
```

### 4.2 Avatar 追踪

**存储位置**：`user` 表，`avatar` 字段存储 JSON

```json
{"type":"custom","custom":"https://site/uploads/avatar/xyz789.jpg"}
```

**搜索实现** [user_repo.go#L385-L397](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/repo/user/user_repo.go#L385-L397)

```go
func (ur *userRepo) IsAvatarFileUsed(ctx context.Context, filePath string) (bool, error) {
    user := &entity.User{}
    count, err := ur.data.DB.Context(ctx).
        Table("user").
        Where(builder.Like{"avatar", "%" + filePath + "%"}). // 模糊匹配 avatar
        Count(&user)
    return count > 0, nil
}
```

**SQL 等价**：
```sql
SELECT COUNT(*) FROM user 
WHERE avatar LIKE '%avatar/xyz789.jpg%'
```

### 4.3 对称机制对比

| 维度 | Branding | Avatar |
|------|----------|--------|
| 存储表 | `site_info` | `user` |
| 存储字段 | `content` (MEDIUMTEXT, JSON) | `avatar` (TEXT, JSON) |
| 过滤条件 | `type = 'branding'` + `content LIKE` | `avatar LIKE` |
| 搜索参数 | `filePath`（相对路径，如 `branding/abc123.png`） | `filePath`（相对路径，如 `avatar/xyz789.jpg`） |
| FileRecord.Source | `admin_branding` | `user_avatar` |
| 文件子目录 | `branding/` | `avatar/` |
| 清理路径 | `isBrandingOrAvatarFile()` → `IsBrandingFileUsed()` | `isBrandingOrAvatarFile()` → `IsAvatarFileUsed()` |
| 专用入口 | `CleanUpRemovedBrandingFiles()` (管理员修改时同步清理) | 无同步清理（仅靠定时任务） |

### 4.4 Branding 的双重清理机制

**机制一：管理员修改时同步清理**

[siteinfo_service.go#L657-L699](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/siteinfo/siteinfo_service.go#L657-L699)

```go
func (s *SiteInfoService) CleanUpRemovedBrandingFiles(
    ctx context.Context,
    newBranding *schema.SiteBrandingReq,
    currentBranding *schema.SiteBrandingResp,
) error {
    currentFiles := map[string]string{
        "logo":        currentBranding.Logo,
        "mobile_logo": currentBranding.MobileLogo,
        "square_icon": currentBranding.SquareIcon,
        "favicon":     currentBranding.Favicon,
    }
    newFiles := map[string]string{
        "logo":        newBranding.Logo,
        "mobile_logo": newBranding.MobileLogo,
        "square_icon": newBranding.SquareIcon,
        "favicon":     newBranding.Favicon,
    }
    
    for key, currentFile := range currentFiles {
        newFile := newFiles[key]
        if currentFile != "" && currentFile != newFile {
            // 品牌文件被替换 → 立即删除旧文件
            fileRecord, _ := s.fileRecordService.GetFileRecordByURL(ctx, currentFile)
            s.fileRecordService.DeleteAndMoveFileRecord(ctx, fileRecord)
        }
    }
}
```

调用入口：`UpdateBranding()` 控制器在保存新配置前调用

```go
func (sc *SiteInfoController) UpdateBranding(ctx *gin.Context) {
    ...
    currentBranding, _ := sc.siteInfoService.GetSiteBranding(ctx)
    sc.siteInfoService.CleanUpRemovedBrandingFiles(ctx, req, currentBranding)  // 同步清理
    sc.siteInfoService.SaveSiteBranding(ctx, req)
}
```

**机制二：定时任务兜底**

`CleanOrphanUploadFiles()` 对 Branding 文件调用 `IsBrandingFileUsed()` 检查。

### 4.5 Avatar 缺少同步清理

与 Branding 不同，**Avatar 替换时没有同步清理旧头像文件**。旧头像只靠定时任务的 `IsAvatarFileUsed()` 清理。这意味着：

- 用户更换头像后，旧头像文件仍在磁盘上
- 最多需要等待 48 小时 + 清理周期才会被删除
- 如果清理任务异常，旧头像可能长期残留

### 4.6 LIKE 模糊匹配的可靠性分析

| 场景 | 是否可靠 | 原因 |
|------|---------|------|
| Branding 文件路径包含随机 ID | ✅ 可靠 | `branding/abc123.png` 足够唯一 |
| Avatar 文件路径包含随机 ID | ✅ 可靠 | `avatar/xyz789.jpg` 足够唯一 |
| 文件路径作为子串出现在其他 URL 中 | ⚠️ 极低风险 | 如果两个 hash 存在子串关系可能误判，但 12 位随机 ID 概率极低 |
| content/avatar 字段被清空 | ✅ 正确 | LIKE 搜索不到，判定为未使用 |

**与 Revision 搜索的对称差异**：

- Branding/Avatar 搜索的是**单个表的特定字段**，有 `type = 'branding'` 等精确前缀条件
- Revision 搜索的是**整个 revision 表的 content 字段**，无其他过滤条件
- Branding/Avatar 的 JSON 结构简单（4 个固定 key），URL 出现位置可预测
- Revision 的 content 是完整实体 JSON 序列化，URL 嵌套在 `original_text` 字段中

---

## 五、总结：追踪机制的三层架构

```
Layer 1: Branding/Avatar — 专用搜索路径
  ┌─────────────────────────────────────┐
  │ isBrandingOrAvatarFile() 分流        │
  │  ├─ Branding → site_info.content    │
  │  │   LIKE + type='branding'         │
  │  └─ Avatar → user.avatar            │
  │      LIKE                           │
  │                                     │
  │ 特点：精确表 + 特定字段 + 同步清理   │
  │ 缺陷：Avatar 无同步清理              │
  └─────────────────────────────────────┘

Layer 2: Question/Answer — Revision 搜索路径
  ┌─────────────────────────────────────┐
  │ ObjectID != "0" → 索引查询          │
  │ ObjectID == "0" → content LIKE '%url%'│
  │                                     │
  │ 特点：Revision 作为统一引用源        │
  │ 优势：一次搜索覆盖所有 Q/A 版本历史  │
  └─────────────────────────────────────┘

Layer 3: Comment/User Bio — ❌ 搜索盲区
  ┌─────────────────────────────────────┐
  │ 评论图片 → Source: user_post         │
  │  → 文件在 post/ 子目录               │
  │  → 不被 isBrandingOrAvatarFile() 拦截│
  │  → revision 搜索找不到               │
  │  → 48h 后被当作孤儿删除 ⚠️          │
  │                                     │
  │ User Bio 图片 → 同理                │
  │  → revision 搜索找不到               │
  │  → 48h 后被当作孤儿删除 ⚠️          │
  └─────────────────────────────────────┘
```

**核心设计问题**：`CleanOrphanUploadFiles` 的搜索策略基于一个隐含假设——**所有引用文件的正文都最终会出现在 `revision.content` 中**。这个假设对 Question/Answer 成立，但对 Comment 和 User Bio 不成立。Branding/Avatar 通过专用搜索路径绕过了这个假设，但 Comment/User Bio 的图片既没有专用搜索路径，也不在 revision 中，成为了追踪盲区。
