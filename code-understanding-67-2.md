# Answer 仓深度技术分析——配置同步、下载鉴权、插件兜底、性能取舍与过滤设计

## 一、管理员配置修改到前后端校验的同步链路

### 1.1 完整链路全景图

```
管理员后台 Files 页面
  │
  ├─ 加载配置: getAdminFilesSetting()
  │   → GET /answer/admin/api/siteinfo/advanced
  │   → SiteInfoService.GetSiteAdvanced()
  │   → siteInfoRepo.GetByType(ctx, "advanced") 【site_info 表】
  │
  └─ 修改后保存: onSubmit()
      │
      ├─ 前端: updateAdminFilesSetting(reqParams)
      │   → PUT /answer/admin/api/siteinfo/advanced
      │   → SiteInfoService.SaveSiteAdvanced()
      │   → siteInfoRepo.SaveByType(ctx, "advanced", data)
      │     → UPDATE site_info SET content = ? WHERE type = ?
      │
      ├─ 前端 Store 同步: writeSettingStore.getState().update({ ...reqParams })
      │   → 立即更新前端内存中的上传参数
      │
      └─ 其他用户初始化时同步
          → setupApp() → initAppSettingsStore()
          → getAppSettings() → GET /answer/api/v1/siteinfo
          → writeSettingStore.getState().update({ ...site_advanced, ... })
```

### 1.2 关键代码节点分析

**前端 Store 定义** [writeSetting.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/stores/writeSetting.ts)

```typescript
const Index = create<IProps>((set) => ({
  write: {
    max_image_size: 4,                    // 默认 4MB
    max_attachment_size: 8,               // 默认 8MB
    max_image_megapixel: 40,              // 默认 40MP
    authorized_image_extensions: [],       // 空数组代表使用默认白名单
    authorized_attachment_extensions: [],  // 空数组代表使用默认白名单
    ...
  },
  update: (params) => set((state) => ({
    write: { ...state.write, ...params }
  })),
}));
```

**前端修改成功后立即同步** [Admin/Files/index.tsx#L94-L100](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/pages/Admin/Files/index.tsx#L94-L100)

```typescript
updateAdminFilesSetting(reqParams)
  .then(() => {
    Toast.onShow({ msg: t('update', { keyPrefix: 'toast' }), variant: 'success' });
    writeSettingStore.getState().update({ ...reqParams });  // 立即更新本地 Store
  })
```

**应用初始化时拉取配置** [guard.ts#L367-L394](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/ui/src/utils/guard.ts#L367-L394)

```typescript
export const initAppSettingsStore = async () => {
  const appSettings = await getAppSettings();  // GET /answer/api/v1/siteinfo
  if (appSettings) {
    writeSettingStore.getState().update({
      ...appSettings.site_advanced,   // 上传配置
      ...appSettings.site_questions,  // 提问配置
      ...appSettings.site_tags,       // 标签配置
    });
  }
};
```

**后端每次上传动态读取配置** [upload.go#L204-L209](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/uploader/upload.go#L204-L209)

```go
func (us *uploaderService) UploadPostFile(ctx *gin.Context, userID string) (url string, err error) {
    // 先尝试插件上传
    url, err = us.tryToUploadByPlugin(ctx, plugin.UserPost)
    if err != nil || len(url) > 0 {
        return url, err
    }
    
    // 每次上传都重新读取最新配置
    siteAdvanced, err := us.siteInfoService.GetSiteAdvanced(ctx)
    if err != nil {
        return "", err
    }
    
    // 应用最新配置
    ctx.Request.Body = http.MaxBytesReader(
        ctx.Writer, ctx.Request.Body, 
        siteAdvanced.GetMaxImageSize(),  // 大小限制
    )
    ...
    if checker.IsUnAuthorizedExtension(
        fileHeader.Filename, 
        siteAdvanced.AuthorizedImageExtensions,  // 扩展名限制
    ) {
        return "", errors.BadRequest(...)
    }
}
```

### 1.3 同步链路设计要点

| 层级 | 触发时机 | 数据来源 | 缓存策略 |
|------|----------|----------|----------|
| 前端 Store (管理员修改后) | 保存成功回调 | 请求参数 | 内存实时更新 |
| 前端 Store (用户初始化) | 应用启动 | API 返回的 site_advanced | 内存，页面刷新失效 |
| 前端上传校验 | 拖拽/粘贴文件时 | writeSettingStore | 读取内存值 |
| 后端上传校验 | 每次上传请求 | site_info 表实时查询 | 无缓存，动态读取 |

**设计权衡**：
- 前端 Store 实时更新确保管理员修改后立即生效（当前会话）
- 后端每次上传动态查询数据库，确保即使前端缓存未更新也能强制执行最新配置
- 其他用户下次刷新页面或重新登录后自动获取最新配置

---

## 二、附件下载控制器的鉴权与原文件名重定向

### 2.1 附件 URL 生成机制

上传时生成特殊格式的 URL [upload.go#L354-L362](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/uploader/upload.go#L354-L362)

```go
func (us *uploaderService) uploadAttachmentFile(ctx *gin.Context, 
    file *multipart.FileHeader, originalFilename, fileSubPath string) (
    downloadUrl string, err error) {
    
    // URL 编码原文件名，避免特殊字符破坏 Markdown 语法
    originalFilename = url.QueryEscape(originalFilename)
    
    // fileSubPath 格式: files/post/<12位hash>.pdf
    // 拼接成: files/post/<12位hash>/<原文件名.pdf>
    downloadPath := strings.TrimSuffix(fileSubPath, filepath.Ext(fileSubPath)) + 
        "/" + originalFilename
    downloadUrl = fmt.Sprintf("%s/uploads/%s", siteGeneral.SiteUrl, downloadPath)
    return downloadUrl, nil
}
```

**实际 URL 示例**：
```
https://example.com/uploads/files/post/abc123def456/%E6%96%87%E6%A1%A3.pdf
                                └── 12位hash ──┘   └── URL 编码的原文件名.pdf
```

**磁盘存储路径**：
```
<UploadPath>/files/post/abc123def456.pdf
```

### 2.2 下载控制器路由与重定向逻辑

[static_router.go#L51-L66](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/router/static_router.go#L51-L66)

```go
r.GET("/uploads/"+constant.FilesPostSubPath+"/*filepath", func(c *gin.Context) {
    // filepath: abc123def456/%E6%96%87%E6%A1%A3.pdf
    filePath := c.Param("filepath")
    
    // 原文件名: 文档.pdf (自动 URL 解码)
    originalFilename := filepath.Base(filePath)
    
    // 真实文件名: abc123def456.pdf
    realFilename := strings.TrimSuffix(filePath, "/"+originalFilename) + 
        filepath.Ext(originalFilename)
    
    // 真实本地路径: <UploadPath>/files/post/abc123def456.pdf
    fileLocalPath := filepath.Join(
        a.serviceConfig.UploadPath, 
        constant.FilesPostSubPath, 
        realFilename,
    )
    
    // 文件不存在重定向到 404
    if !dir.CheckFileExist(fileLocalPath) {
        c.Redirect(http.StatusFound, "/404")
        return
    }
    
    // 设置 Content-Disposition: attachment; filename="文档.pdf"
    c.FileAttachment(fileLocalPath, originalFilename)
})
```

### 2.3 鉴权分析

**结论：附件下载无鉴权**

| 设计点 | 说明 |
|--------|------|
| 路由级别 | `gin.RouterGroup` 无中间件装饰，直接注册在根路由 |
| 权限检查 | 无用户登录校验、无权限判断 |
| 访问控制 | 依赖 12 位随机 hash 的不可预测性 |
| 文件枚举 | 12 位随机 ID 有 `62^12 ≈ 3.2e21` 种组合，暴力枚举不可行 |

**设计考量**：
- 附件 URL 是公开的，任何人持有链接即可下载
- 安全性依赖于 URL 的不可预测性（12 位随机 ID）
- 与图片的处理方式一致（图片也是直接通过 `/uploads/post/xxx.jpg` 公开访问）
- 如需鉴权，需要额外添加中间件并修改 URL 生成逻辑

### 2.4 原文件名重定向的设计目的

1. **用户体验**：下载时保留原始文件名，而非 12 位随机 hash
2. **Markdown 语法安全**：URL 编码原文件名，避免空格、括号等特殊字符破坏 `[name](url)` 语法
3. **存储隔离**：磁盘上使用随机文件名，避免：
   - 路径遍历攻击（`../etc/passwd`）
   - 文件名冲突（多人上传同名文件）
   - 特殊字符导致的文件系统问题
4. **SEO 友好**：原文件名出现在 URL 中，搜索引擎可索引

---

## 三、Storage 插件 Hook 时机与上传失败的本地兜底

### 3.1 Hook 时机与调用顺序

[upload.go#L194-L202](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/uploader/upload.go#L194-L202)

```go
func (us *uploaderService) UploadPostFile(ctx *gin.Context, userID string) (
    url string, err error) {
    
    // 第一步：优先调用 Storage 插件
    url, err = us.tryToUploadByPlugin(ctx, plugin.UserPost)
    if err != nil {
        return "", err          // 插件报错 → 直接返回错误，不降级
    }
    if len(url) > 0 {
        return url, nil         // 插件成功 → 直接返回 URL，不记录 FileRecord
    }
    
    // 第二步：无插件或插件返回空 URL → 本地存储
    return us.localUploadPostFile(ctx, userID)
}
```

### 3.2 tryToUploadByPlugin 完整逻辑

[upload.go#L365-L390](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/uploader/upload.go#L365-L390)

```go
func (us *uploaderService) tryToUploadByPlugin(ctx *gin.Context, 
    source plugin.UploadSource) (url string, err error) {
    
    // 插件也需要最新的配置参数
    siteAdvanced, err := us.siteInfoService.GetSiteAdvanced(ctx)
    if err != nil {
        return "", err
    }
    
    // 组装上传条件传递给插件
    cond := plugin.UploadFileCondition{
        Source:                         source,
        MaxImageSize:                   siteAdvanced.MaxImageSize,
        MaxAttachmentSize:              siteAdvanced.MaxAttachmentSize,
        MaxImageMegapixel:              siteAdvanced.MaxImageMegapixel,
        AuthorizedImageExtensions:      siteAdvanced.AuthorizedImageExtensions,
        AuthorizedAttachmentExtensions: siteAdvanced.AuthorizedAttachmentExtensions,
    }
    
    // 调用所有注册的 Storage 插件（只能注册一个）
    _ = plugin.CallStorage(func(fn plugin.Storage) error {
        resp := fn.UploadFile(ctx, cond)
        if resp.OriginalError != nil {
            log.Errorf("upload file by plugin failed, err: %v", resp.OriginalError)
            // 插件返回错误 → 包装后直接返回
            err = errors.BadRequest("").
                WithMsg(resp.DisplayErrorMsg.Translate(ctx)).
                WithError(err)
        } else {
            // 插件返回成功 → 直接使用插件返回的 URL
            url = resp.FullURL
        }
        return nil
    })
    return url, err
}
```

### 3.3 插件与本地存储的行为差异对比

| 行为 | Storage 插件 | 本地存储 |
|------|--------------|----------|
| 配置读取 | 由 tryToUploadByPlugin 传递 | 由 UploadPostFile 直接读取 |
| 文件存储位置 | 插件决定（S3、OSS 等） | 本地 `UploadPath` 目录 |
| FileRecord 记录 | ❌ 不记录 | ✅ `AddFileRecord()` 写入 DB |
| URL 格式 | 插件决定（可能是 CDN 域名） | 本站 `SiteUrl/uploads/...` |
| EXIF 移除 | 插件自行决定 | ✅ `removeExif()` 自动移除 |
| 图片像素校验 | 插件自行决定 | ✅ `DecodeAndCheckImageFile()` |
| 孤儿文件清理 | 插件自行管理 | ✅ `CleanOrphanUploadFiles()` 清理 |
| 失败降级 | ❌ 无（直接返回错误） | ✅ 就是兜底路径本身 |

### 3.4 本地兜底的设计边界

**关键设计**：
- 插件优先级高于本地存储
- 插件报错时**不会**自动降级到本地存储，而是直接返回错误
- 只有当没有注册 Storage 插件，或插件返回空 URL 时，才会走本地存储

**设计意图**：
1. **插件是一等公民**：管理员配置了 S3 插件，就应该始终使用 S3，失败就是失败
2. **避免数据分散**：部分文件存 S3，部分文件存本地会造成管理混乱
3. **错误透明**：让用户明确知道插件上传失败，而不是静默降级

**潜在问题**：
- 插件临时故障（网络抖动）时无法自动重试或降级
- 插件上传的文件不记录 FileRecord，无法通过管理界面统一管理

---

## 四、FileRecord 靠 URL 模糊匹配 Revision Content 的全表搜索性能取舍

### 4.1 搜索代码实现

[revision_repo.go#L166-L173](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/repo/revision/revision_repo.go#L166-L173)

```go
func (rr *revisionRepo) GetLastRevisionByFileURL(ctx context.Context, 
    fileURL string) (revision *entity.Revision, exist bool, err error) {
    
    revision = &entity.Revision{}
    exist, err = rr.data.DB.Context(ctx).
        Where("content LIKE ?", "%"+fileURL+"%").  // 全表模糊匹配
        Desc("created_at").                        // 取最新的
        Get(revision)
    return
}
```

### 4.2 调用时机与前置条件

[file_record_service.go#L128-L148](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/file_record/file_record_service.go#L128-L148)

```go
if checker.IsNotZeroString(fileRecord.ObjectID) {
    // ObjectID 已回填 → 走索引查询
    _, exist, err := fs.revisionRepo.GetLastRevisionByObjectID(
        ctx, fileRecord.ObjectID,
    )  // WHERE object_id = ? 走索引
    if exist {
        continue  // 仍被引用
    }
} else {
    // ObjectID 仍为 "0" → 走全表模糊搜索
    lastRevision, exist, err := fs.revisionRepo.GetLastRevisionByFileURL(
        ctx, fileRecord.FileURL,
    )  // WHERE content LIKE '%url%' 全表扫描
    if exist {
        // 找到引用 → 回填 ObjectID，下次走索引
        fileRecord.ObjectID = lastRevision.ObjectID
        if err := fs.fileRecordRepo.UpdateFileRecord(ctx, fileRecord); err != nil {
            log.Error(err)
        }
        continue  // 仍被引用
    }
}
// 未找到引用 → 删除文件
```

### 4.3 性能分析

| 指标 | ObjectID 索引查询 | URL 全表模糊搜索 |
|------|------------------|------------------|
| SQL | `WHERE object_id = ?` | `WHERE content LIKE '%url%'` |
| 索引 | ✅ 命中 `object_id` 索引 | ❌ 全表扫描，`content` 是 MEDIUMTEXT |
| 复杂度 | O(log n) | O(n)，n 是 revision 表总行数 |
| 平均耗时 | 毫秒级 | 取决于表大小，万级以上明显变慢 |
| 触发频率 | 大部分 FileRecord 回填后走此路径 | 仅 ObjectID 为 "0" 的文件触发 |

### 4.4 性能缓解机制

1. **48 小时宽限期** [file_record_service.go#L110-L112](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/file_record/file_record_service.go#L110-L112)
   ```go
   if fileRecord.CreatedAt.AddDate(0, 0, 2).After(time.Now()) {
       continue  // 48 小时内的文件不扫描
   }
   ```
   用户正在编辑的内容不会触发扫描

2. **分页扫描** [file_record_service.go#L95-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/file_record/file_record_service.go#L95-L107)
   ```go
   page, pageSize := 1, 1000
   for {
       fileRecordList, total, err := fs.fileRecordRepo.GetFileRecordPage(
           ctx, page, pageSize, &entity.FileRecord{Status: Available},
       )
       if len(fileRecordList) == 0 { break }
       // 逐批处理
   }
   ```
   避免一次性加载所有 FileRecord 到内存

3. **惰性回填**
   一旦找到引用，立即回填 ObjectID，下次扫描走索引查询

### 4.5 设计权衡

**选择模糊匹配的原因**：
- **写入性能优先**：提交正文时不需要解析 Markdown 提取 URL，不影响发帖速度
- **解耦设计**：FileRecord 和 Revision 之间没有强外键约束，修改任一方不影响另一方
- **灵活性**：支持用户在多个帖子中引用同一个文件 URL，或者在正文中手动粘贴 URL

**代价**：
- 清理时需要全表扫描，大数据量下性能较差
- 可能误判：URL 出现在普通文本中（不是 `![](url)` 或 `[](url)`）也会被判定为被引用
- 无法反向查询：不知道一个帖子引用了哪些文件

**优化空间**（当前未实现）：
- 在 Revision 表增加 `file_urls` JSON 字段，发帖时解析并存储引用的 URL
- 建立 MySQL 全文索引或使用 Elasticsearch 搜索 content 字段
- 清理任务在低峰期（凌晨）执行，减少对用户的影响

---

## 五、goldmark 与 bluemonday 双层过滤为何不合并的设计权衡

### 5.1 过滤管线分层架构

```
Markdown 源码
  │
  ▼  goldmark 解析 ── AST 节点渲染
      │
      ├─ Node: Paragraph       → <p>...</p>
      ├─ Node: Link            → [DangerousHTMLFilterExtension] 校验 URL
      ├─ Node: AutoLink        → [DangerousHTMLFilterExtension] 校验 URL
      ├─ Node: RawHTML         → [DangerousHTMLFilterExtension] 仅保留 <kbd>
      ├─ Node: HTMLBlock       → [SecureWrite] 转义输出
      │
      ▼  初步安全 HTML
          │
          ▼  bluemonday.UGCPolicy 全局过滤
              │
              ├─ AllowStyling          允许内联样式
              ├─ AllowElements("kbd")   允许 <kbd> 标签
              ├─ AllowAttrs("title")    允许 title 属性
              ├─ AllowAttrs("start") on <ol>
              │
              ▼  最终安全 HTML
```

### 5.2 goldmark 层的细粒度控制

[markdown.go#L79-L189](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/pkg/converter/markdown.go#L79-L189)

```go
// 自定义 AST 渲染扩展
type DangerousHTMLFilterExtension struct{}

func (e *DangerousHTMLFilterExtension) Extend(m goldmark.Markdown) {
    m.Renderer().AddOptions(renderer.WithNodeRenderers(
        renderer.Prioritized(
            renderer.NewNodeRendererMap(
                nodeTypeRawHTML: &RawHTMLRenderer{},      // 拦截 RawHTML
                nodeTypeHTMLBlock: &HTMLBlockRenderer{}, // 拦截 HTMLBlock
                nodeTypeLink: &LinkRenderer{},           // 拦截 Link
                nodeTypeAutoLink: &AutoLinkRenderer{},   // 拦截 AutoLink
            ),
            100,  // 高优先级，覆盖默认渲染器
        ),
    ))
}
```

**RawHTMLRenderer 的处理**：
```go
func (r *RawHTMLRenderer) Render(w util.BufWriter, 
    source []byte, node ast.Node, entering bool) (ast.WalkStatus, error) {
    
    n := node.(*ast.RawHTML)
    for i := 0; i < n.Segments.Len(); i++ {
        segment := n.Segments.At(i)
        html := string(segment.Value(source))
        if isKbdTag(html) {
            // <kbd> 标签 → 直接输出
            _, _ = w.WriteString(html)
        } else {
            // 其他标签 → 交给 bluemonday 过滤
            policy := bluemonday.UGCPolicy()
            policy.AllowElements("kbd")
            filtered := policy.Sanitize(html)
            _, _ = w.WriteString(filtered)
        }
    }
    return ast.WalkContinue, nil
}
```

**HTMLBlock 的 SecureWrite**：
```go
func SecureWrite(w util.BufWriter, content string) {
    // 将 HTML 转义为纯文本输出
    esc := html.EscapeString(content)
    _, _ = w.WriteString(esc)
}
```

**Link/AutoLink 的 URL 校验**：
```go
func renderLink(w util.BufWriter, dest string, childHTML string) {
    if !govalidator.IsURL(dest) && !isSafePath(dest) {
        // 危险 URL → 只渲染文本，不渲染 href
        _, _ = w.WriteString(childHTML)
        return
    }
    // 安全 URL → 正常渲染 <a href="...">
    _, _ = fmt.Fprintf(w, `<a href="%s">%s</a>`, 
        html.EscapeString(dest), childHTML)
}
```

### 5.3 bluemonday 层的全局兜底

[markdown.go#L56-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/pkg/converter/markdown.go#L56-L65)

```go
// 初始化 bluemonday 策略
var markdownPolicy = func() *bluemonday.Policy {
    p := bluemonday.UGCPolicy()
    p.AllowStyling()                              // 允许内联样式
    p.AllowElements("kbd")                         // 允许 <kbd> 标签
    p.AllowAttrs("title").Matching(regexp.MustCompile( // 允许 title 属性
        `^[\p{L}\p{N}\s\.,!?'"\-()&@#%:;/]+$`,
    )).Globally()
    p.AllowAttrs("start").OnElements("ol")         // 允许 <ol start="...">
    return p
}()

// Markdown2HTML 主函数
func Markdown2HTML(md string) string {
    // 第一步：goldmark 解析 + AST 级过滤
    var buf bytes.Buffer
    if err := markdownRenderer.Convert([]byte(md), &buf); err != nil {
        return ""
    }
    
    // 第二步：bluemonday 全局二次过滤
    return markdownPolicy.Sanitize(buf.String())
}
```

### 5.4 双层过滤的设计权衡

| 维度 | goldmark 层 | bluemonday 层 |
|------|------------|--------------|
| 过滤时机 | AST 节点渲染时，逐节点处理 | 完整 HTML 生成后，全局扫描 |
| 粒度 | 细粒度，可针对不同节点类型定制逻辑 | 粗粒度，基于标签/属性白名单 |
| 能力 | 可访问 Markdown 源码上下文，可做语义判断 | 仅能看到 HTML 结构，无上下文 |
| 性能 | O(n)，与解析过程合并，无额外开销 | O(m)，额外扫描整个 HTML |
| 可靠性 | 依赖 goldmark 正确解析所有节点 | 独立于解析器，安全边界更清晰 |
| 维护成本 | 需理解 goldmark 扩展机制，代码复杂度高 | API 成熟，配置简单 |

**为什么不合并成单层**：

1. **goldmark 无法替代 bluemonday**：
   - goldmark 主要职责是 Markdown→HTML 转换，安全过滤是附加功能
   - goldmark 的扩展机制复杂，每个节点类型需要单独实现 Renderer
   - 未来升级 goldmark 版本时，节点类型或 API 可能变化，安全逻辑易失效

2. **bluemonday 无法替代 goldmark 层**：
   - bluemonday 看不到 Markdown 源码，无法区分用户输入的 HTML 还是生成的 HTML
   - 无法实现 `SecureWrite` 这种语义级转义（HTMLBlock 转成纯文本）
   - 无法实现 URL 的语义校验（区分 `![img](url)` 和 `[text](url)`）

3. **纵深防御**：
   - 两层过滤使用不同的技术栈（goldmark AST vs bluemonday HTML 解析）
   - 即使某一层被绕过（如 goldmark 解析漏洞），另一层仍能拦截
   - 符合安全设计的"最小权限"和"深度防御"原则

4. **性能代价可接受**：
   - goldmark 过滤在解析时完成，无额外解析开销
   - bluemonday 扫描 HTML 字符串的开销远小于数据库查询等操作
   - 渲染 API 本身不是高频接口（预览、查看帖子时调用）

**潜在问题**：
- 两层过滤的规则可能不一致（如 goldmark 允许的标签被 bluemonday 过滤）
- 维护者需要同时理解两个库的 API 和行为
- `RawHTMLRenderer` 内部也调用了 `bluemonday.UGCPolicy().Sanitize()`，存在重复过滤

---

## 六、总结与设计洞察

| 技术点 | 设计原则 | 权衡结果 |
|--------|----------|----------|
| 配置同步 | 读写分离，后端为准 | 前端实时更新体验好，后端动态读取一致性强 |
| 附件下载 | 安全靠不可预测性，体验靠原文件名 | 无鉴权简化架构，12 位 hash 防枚举 |
| 插件兜底 | 插件优先，无静默降级 | 避免数据分散，错误透明但缺少容错 |
| URL 匹配 | 写入简单，清理复杂 | 发帖无额外开销，清理时全表扫描 |
| 双层过滤 | 深度防御，分层控制 | 安全性高，性能代价小，维护复杂度上升 |

**核心设计哲学**：
- 写入路径尽量简单，把复杂度移到异步/后台任务
- 安全是分层的，不依赖单一防线
- 插件是一等公民，本地存储是默认实现而非降级方案
- 配置最终一致性优于强一致，避免阻塞用户操作
