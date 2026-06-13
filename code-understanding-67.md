# 前端编辑器、图片上传、文件记录与正文渲染——协作关系与责任边界

## 一、整体数据流概览

```
用户操作 (拖拽/粘贴/工具栏)
  │
  ▼
前端编辑器 (Editor) ──文件校验──▶ 上传 API (POST /answer/api/v1/file)
  │                                    │
  │  Markdown 源码                     ▼
  │                            后端 UploaderService
  │                              │         │
  │                              │         ▼
  │                              │   FileRecordService.AddFileRecord()
  │                              │     → DB: file_record 表
  │                              │     → 磁盘: UploadPath/post/ 或 files/post/
  │                              │         ▼
  │  ◀──返回 file_url────────────   返回 URL 给前端
  │
  ▼
Markdown 正文 (含 ![alt](url) / [name](url))
  │
  ├──▶ 提交问题/回答 → 后端存入 revision 表 (content 字段)
  │
  └──▶ 实时预览 → POST /answer/api/v1/post/render
                    │
                    ▼
              Markdown2HTML() (goldmark + bluemonday)
                    │
                    ▼
              前端 Viewer.tsx (dangerouslySetInnerHTML)
                    │
                    ▼
              htmlRender() 后处理 + ImgViewer 包裹
```

---

## 二、四大模块职责详析

### 2.1 前端编辑器 (Editor)

**核心文件**
- [index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/components/Editor/index.tsx) — 编辑器入口，组装工具栏、Markdown 编辑区、预览区
- [MarkdownEditor.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/components/Editor/MarkdownEditor.tsx) — CodeMirror 6 驱动的 Markdown 编辑组件
- [hooks/useImageUpload.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/components/Editor/hooks/useImageUpload.ts) — 文件校验与上传协调
- [ToolBars/image.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/components/Editor/ToolBars/image.tsx) — 图片工具栏（拖拽/粘贴/URL/本地上传）
- [ToolBars/file.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/components/Editor/ToolBars/file.tsx) — 附件工具栏

**职责边界**

| 职责 | 说明 |
|------|------|
| 文件前端校验 | `useImageUpload.verifyImageSize()` 检查文件类型和大小（图片走 `max_image_size`，附件走 `max_attachment_size`），类型不在白名单则拒绝 |
| 上传触发与降级 | 根据文件 MIME 判断走 `post`（图片）还是 `post_attachment`（附件）source，调用 `uploadImage()` API |
| 编辑器占位 | 上传期间插入 `![uploading...]() ` 占位文本，上传成功后替换为 `![name](url)` 或 `[name](url)`，失败则清除占位 |
| 插件替换 | 若存在 `EditorReplacement` 插件，整个编辑器被替换，但 `imageUploadHandler` 仍走本系统的 `verifyImageSize + uploadSingleFile` |
| 粘贴 HTML 中的 img | 粘贴含 `<img>` 的 HTML 时，提取 `src` 转为 Markdown `![alt](src)`，不会重新上传 |

**不负责**
- 不负责文件的实际存储和 URL 生成
- 不负责文件是否被正文引用的追踪
- 不负责已上传但未引用的文件的清理

### 2.2 图片/文件上传 (Uploader)

**核心文件**
- [upload.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/uploader/upload.go) — 上传服务主体
- [upload_controller.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/controller/upload_controller.go) — HTTP 入口，按 source 分发
- [storage.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/plugin/storage.go) — 插件化存储接口
- [file_type.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/pkg/checker/file_type.go) — 后端文件类型与像素校验

**上传流程详解**

```
UploadController.UploadFile()
  │  source: post | post_attachment | avatar | branding
  ▼
UploaderService.UploadPostFile / UploadPostAttachment / ...
  │
  ├──1. tryToUploadByPlugin() — 优先走 Storage 插件（如 S3）
  │     若插件返回 URL，直接返回，不走本地存储
  │
  ├──2. 本地存储路径
  │     图片: UploadPath/post/<12位随机ID>.<ext>
  │     附件: UploadPath/files/post/<12位随机ID>.<ext>
  │
  ├──3. 安全校验
  │     - MaxBytesReader 限制请求体大小
  │     - IsUnAuthorizedExtension() 检查扩展名白名单
  │     - DecodeAndCheckImageFile() 解码校验真实图片格式 + 像素上限
  │     - removeExif() 移除 JPEG/PNG 的 EXIF 数据
  │
  ├──4. URL 生成
  │     图片: {SiteUrl}/uploads/post/<filename>
  │     附件: {SiteUrl}/uploads/files/post/<id>/<url_encoded_original_name>
  │            下载时通过路径中的原始文件名做重定向
  │
  └──5. FileRecordService.AddFileRecord()
        记录 userID, filePath, fileURL, source, status=Available, objectID="0"
```

**职责边界**

| 职责 | 说明 |
|------|------|
| 文件后端校验 | 扩展名白名单、真实图片格式解码校验、像素上限检查 |
| 文件存储 | 本地磁盘存储或插件化远程存储（S3 等） |
| EXIF 脱敏 | 自动移除 JPEG/PNG 的 EXIF 元数据（隐私保护） |
| URL 生成 | 基于 SiteUrl + 子路径 + 随机文件名生成唯一访问 URL |
| 记录文件元信息 | 调用 FileRecordService 记录上传者、路径、URL、来源、初始状态 |

**不负责**
- 不负责判断文件是否被正文引用
- 不负责文件的定时清理
- 不负责 Markdown 到 HTML 的渲染

### 2.3 文件记录 (FileRecord)

**核心文件**
- [file_record_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/entity/file_record_entity.go) — 数据模型
- [file_record_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/file_record/file_record_service.go) — 业务逻辑
- [file_record_repo.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/repo/file_record/file_record_repo.go) — 数据访问层

**数据模型**

```go
type FileRecord struct {
    ID        int       // 主键
    CreatedAt time.Time // 创建时间
    UpdatedAt time.Time // 更新时间
    UserID    string    // 上传者 ID
    FilePath  string    // 相对路径 (post/xxx.jpg)
    FileURL   string    // 完整访问 URL
    ObjectID  string    // 关联的业务对象 ID (初始为 "0")
    Source    string    // 来源: user_post / user_post_attachment / user_avatar / admin_branding
    Status    int       // 1=Available, 10=Deleted
}
```

**正文引用追踪机制**

文件上传时 `ObjectID` 初始为 `"0"`，表示尚未关联到任何业务对象。追踪引用靠两种路径：

1. **通过 ObjectID 直接查找**：若 FileRecord 已关联 ObjectID（非零），调用 `revisionRepo.GetLastRevisionByObjectID()` 检查该对象是否存在最新修订版。存在则说明文件仍被引用。

2. **通过 FileURL 模糊查找**：若 ObjectID 仍为 `"0"`，调用 `revisionRepo.GetLastRevisionByFileURL()` 在所有修订版的 `content` 字段中搜索该 URL。若找到，回填 ObjectID 并保留文件。

**职责边界**

| 职责 | 说明 |
|------|------|
| 文件元信息持久化 | 存储上传者、路径、URL、来源、状态 |
| 引用关系推断 | 通过 Revision 的 content 字段搜索 FileURL 来判断文件是否被引用 |
| ObjectID 回填 | 首次发现文件被某 Revision 引用时，更新 ObjectID 字段 |
| 孤儿文件清理 | 定时扫描 Available 状态的记录，48 小时后未引用则标记删除 |
| 物理文件迁移 | 将待删除文件从原目录移至 `deleted/` 子目录 |

---

### 2.4 正文渲染 (Render)

**核心文件**
- [Viewer.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/components/Editor/Viewer.tsx) — 前端预览组件
- [utils/index.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/components/Editor/utils/index.ts) — `htmlRender()` 后处理
- [markdown.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/pkg/converter/markdown.go) — 后端 Markdown→HTML 转换
- [ImgViewer/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/components/ImgViewer/index.tsx) — 图片全屏查看器

**渲染管线**

```
Markdown 源码
  │
  ▼  POST /answer/api/v1/post/render
Markdown2HTML()
  │  goldmark (GFM + Footnote) → 原始 HTML
  │  DangerousHTMLFilterExtension:
  │    - RawHTML: 仅保留 <kbd>，其余经 bluemonday UGCPolicy 过滤
  │    - HTMLBlock: 使用 SecureWrite 转义输出
  │    - Link: 校验 URL 合法性，危险 URL 不渲染 href
  │    - AutoLink: 校验 URL 合法性
  │  bluemonday UGCPolicy 二次过滤:
  │    - AllowStyling
  │    - AllowElements("kbd")
  │    - AllowAttrs("title") 全局匹配安全正则
  │    - AllowAttrs("start") on <ol>
  │
  ▼  返回安全 HTML
前端 Viewer.tsx
  │  dangerouslySetInnerHTML={{ __html: html }}
  │
  ▼
htmlRender() 后处理
  │  1. 修复公式块中的 <br> → 换行符
  │  2. 表格添加 Bootstrap 样式 + 响应式包裹
  │  3. 外部链接添加 rel="nofollow"
  │  4. <pre> 代码块添加复制按钮
  │
  ▼
ImgViewer 包裹
  │  点击非链接内的 <img> → 全屏查看
  │  检查 naturalWidth/naturalHeight → 检测损坏图片
```

**职责边界**

| 职责 | 说明 |
|------|------|
| Markdown→HTML 转换 | goldmark 引擎支持 GFM、Footnote、自动标题 ID |
| HTML 安全过滤 | bluemonday UGCPolicy + 自定义 DangerousHTMLFilterExtension 双层过滤 |
| 危险 HTML 处理 | RawHTML 中仅保留 `<kbd>`，其余全部过滤；HTMLBlock 使用 SecureWrite 转义 |
| 链接安全 | 校验 URL 合法性（`govalidator.IsURL` 或路径格式），危险 URL 不渲染 href |
| 前端后处理 | 公式修复、表格美化、nofollow、代码复制 |
| 图片查看 | ImgViewer 提供点击放大、损坏图片检测 |

**不负责**
- 不负责图片/文件是否存在或可访问（URL 可能指向已清理的文件）
- 不负责文件 URL 的签名或时效性控制
- 不负责 XSS 之外的其他安全策略（如 CSRF、点击劫持）

---

## 三、正文引用机制——文件如何与 Revision 关联

```
上传阶段:
  FileRecord { ObjectID: "0", FileURL: "https://site/uploads/post/abc123.jpg" }
  此时文件"未关联"任何业务对象

提交问题/回答:
  revision 表插入记录 { ObjectID: "question_id", Content: "...![img](https://site/uploads/post/abc123.jpg)..." }
  文件的 URL 以纯文本形式嵌入 Markdown content 中

清理扫描阶段:
  FileRecordService.CleanOrphanUploadFiles()
    │
    ├── ObjectID != "0"?
    │     → revisionRepo.GetLastRevisionByObjectID(ObjectID)
    │       → 存在 → 文件仍被引用，保留
    │       → 不存在 → 文件可清理
    │
    └── ObjectID == "0"?
          → revisionRepo.GetLastRevisionByFileURL(FileURL)
            → 在所有 revision.content 中搜索该 URL
            → 找到 → 回填 ObjectID，保留
            → 未找到 → 文件可清理
```

**关键设计要点**：
- 文件与正文的关联是**松耦合**的，通过 URL 字符串匹配而非外键约束
- `ObjectID` 回填是**惰性**的，只在清理扫描时触发
- 48 小时宽限期确保用户编辑期间不会误删

---

## 四、清理策略

**核心文件**
- [file_record_service.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/file_record/file_record_service.go#L94-L196) — `CleanOrphanUploadFiles()` + `PurgeDeletedFiles()`
- [cron.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/base/cron/cron.go#L98-L117) — 定时任务调度
- [service_config.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/service_config/service_config.go) — 清理配置

**两阶段清理**

| 阶段 | 方法 | 频率 | 行为 |
|------|------|------|------|
| 孤儿检测 | `CleanOrphanUploadFiles()` | 每 N 小时（`clean_orphan_uploads_period_hours`） | 扫描 Available 记录，跳过 48h 内新建的；对 Branding/Avatar 文件走专用检查；对 Post 文件查 Revision 引用；无引用则标记 Deleted + 移至 `deleted/` 目录 |
| 物理清除 | `PurgeDeletedFiles()` | 每 N 天（`purge_deleted_files_period_days`） | 直接 `os.RemoveAll(deleted/)` 整个目录，然后重建空目录 |

**开关控制**：`service_config.CleanUpUploads` 为 `true` 时才注册清理定时任务。

**Branding/Avatar 特殊处理**：
- Branding 文件：调用 `siteInfoService.IsBrandingFileUsed()` 检查是否仍在使用
- Avatar 文件：调用 `userService.IsAvatarFileUsed()` 检查是否仍在使用
- 不走 Revision 引用检查，走独立的业务逻辑

**软删除 → 硬删除**：
- `DeleteFileRecord()` 仅将 Status 更新为 `FileRecordStatusDeleted(10)`，不物理删除记录
- `DeleteAndMoveFileRecord()` 将文件从原路径移至 `deleted/` 子目录
- `PurgeDeletedFiles()` 才真正删除磁盘文件

---

## 五、渲染安全——多层防线

### 5.1 后端防线（Markdown2HTML）

| 层级 | 机制 | 代码位置 |
|------|------|----------|
| L1: goldmark 解析 | 不渲染原始 HTML 块，`DangerousHTMLFilterExtension` 拦截 | [markdown.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/pkg/converter/markdown.go#L79-L119) |
| L2: RawHTML 过滤 | 仅保留 `<kbd>` 标签，其余经 bluemonday 过滤 | [markdown.go#L104-L119](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/pkg/converter/markdown.go#L104-L119) |
| L3: HTMLBlock 转义 | `SecureWrite` 方法输出，自动转义危险内容 | [markdown.go#L121-L134](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/pkg/converter/markdown.go#L121-L134) |
| L4: Link/AutoLink 校验 | `govalidator.IsURL()` + 路径正则校验，危险 URL 不渲染 href | [markdown.go#L136-L189](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/pkg/converter/markdown.go#L136-L189) |
| L5: bluemonday 二次过滤 | UGCPolicy 全局过滤，白名单元素 + 安全属性 | [markdown.go#L56-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/pkg/converter/markdown.go#L56-L65) |

### 5.2 前端防线（htmlRender + 浏览器）

| 层级 | 机制 | 代码位置 |
|------|------|----------|
| L6: nofollow | 外部链接自动添加 `rel="nofollow"` | [utils/index.ts#L74-L81](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/components/Editor/utils/index.ts#L74-L81) |
| L7: 公式修复 | 替换公式块中 `<br>` 为换行符，防止 HTML 注入破坏公式渲染 | [utils/index.ts#L48-L53](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/components/Editor/utils/index.ts#L48-L53) |
| L8: React dangerouslySetInnerHTML | 仅接受后端返回的已过滤 HTML，前端不再做二次拼接 | [Viewer.tsx#L82](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/components/Editor/Viewer.tsx#L82) |

### 5.3 上传安全防线

| 层级 | 机制 | 代码位置 |
|------|------|----------|
| 前端类型校验 | `verifyImageSize()` MIME 类型 + 扩展名白名单 | [useImageUpload.ts#L35-L99](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/components/Editor/hooks/useImageUpload.ts#L35-L99) |
| 后端大小限制 | `http.MaxBytesReader()` 限制请求体 | [upload.go#L117](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/uploader/upload.go#L117) |
| 后端扩展名校验 | `IsUnAuthorizedExtension()` 二次检查扩展名 | [upload.go#L217-L219](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/uploader/upload.go#L217-L219) |
| 后端图片解码校验 | `DecodeAndCheckImageFile()` 真实解码 + 像素上限 | [file_type.go#L47-L66](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/pkg/checker/file_type.go#L47-L66) |
| EXIF 脱敏 | `removeExif()` 移除 JPEG/PNG 的 EXIF 隐私数据 | [upload.go#L394-L408](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/uploader/upload.go#L394-L408) |
| 随机文件名 | `uid.IDStr12()` 生成 12 位随机 ID，避免文件名注入 | [upload.go#L130](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/uploader/upload.go#L130) |

---

## 六、责任边界总结

```
┌──────────────────────────────────────────────────────────────────────┐
│                        责任矩阵                                      │
├────────────────────┬───────┬───────┬───────────┬──────────┤
│ 职责               │ 编辑器 │ 上传   │ 文件记录   │ 渲染     │
├────────────────────┼───────┼───────┼───────────┼──────────┤
│ 前端文件校验       │  ✅   │       │           │          │
│ 上传触发与占位     │  ✅   │       │           │          │
│ 后端文件校验       │       │  ✅   │           │          │
│ 文件存储           │       │  ✅   │           │          │
│ EXIF 脱敏          │       │  ✅   │           │          │
│ URL 生成           │       │  ✅   │           │          │
│ 文件元信息持久化   │       │       │  ✅       │          │
│ 引用关系推断       │       │       │  ✅       │          │
│ 孤儿文件检测       │       │       │  ✅       │          │
│ 物理文件迁移/清除  │       │       │  ✅       │          │
│ Markdown→HTML      │       │       │           │  ✅      │
│ HTML 安全过滤      │       │       │           │  ✅      │
│ 链接安全 (nofollow)│       │       │           │  ✅      │
│ 前端后处理         │       │       │           │  ✅      │
│ 图片查看交互       │       │       │           │  ✅      │
└────────────────────┴───────┴───────┴───────────┴──────────┘
```

### 关键协作接口

| 接口 | 提供方 | 消费方 | 数据流 |
|------|--------|--------|--------|
| `POST /answer/api/v1/file` | UploaderService | 前端 Editor | FormData → URL 字符串 |
| `POST /answer/api/v1/post/render` | Markdown2HTML | 前端 Viewer | Markdown → 安全 HTML |
| `AddFileRecord()` | FileRecordService | UploaderService | 文件元信息 → DB |
| `GetLastRevisionByFileURL()` | RevisionRepo | FileRecordService | URL → Revision 存在性 |
| `writeSettingStore` | Admin Files 页面 | 前端 Editor | 校验参数 (max_size, extensions) |

### 设计哲学

1. **上传与引用解耦**：文件先上传获得 URL，再通过 Markdown 嵌入正文。FileRecord 不直接持有与 Revision 的外键，而是通过 URL 字符串在清理时做模糊匹配。

2. **延迟绑定**：ObjectID 初始为 `"0"`，在清理扫描时才回填。这简化了写入路径，但增加了清理的复杂度。

3. **两阶段删除**：软删除（Status=Deleted + 移至 deleted/）→ 硬删除（RemoveAll）。给误删留出恢复窗口。

4. **渲染安全纵深防御**：goldmark 不渲染危险 HTML → bluemonday 白名单过滤 → 前端 nofollow + 公式修复。三层防线确保即使某一层被绕过，下一层仍能兜底。

5. **插件化存储**：Storage 插件可替换本地存储为 S3 等远程存储，FileRecord 仍记录 URL，清理逻辑不变。
