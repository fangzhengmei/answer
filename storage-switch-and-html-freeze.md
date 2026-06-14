# Storage 插件切换代价、存量 HTML 冻结与随机命名无去重

---

## 一、Storage 插件上传不写 FileRecord，插件切换后 CleanOrphanUploadFiles 双向失效

### 1.1 插件路径与本地路径的代码分支

所有上传入口（Avatar/Post/Attachment/Branding）遵循同一模式：

[upload.go#L194-L230](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/uploader/upload.go#L194-L230)

```go
func (us *uploaderService) UploadPostFile(ctx *gin.Context, userID string) (url string, err error) {
    // 第一步：尝试插件上传
    url, err = us.tryToUploadByPlugin(ctx, plugin.UserPost)
    if err != nil {
        return "", err
    }
    if len(url) > 0 {
        return url, nil    // 插件成功 → 直接返回，❌ 不写 FileRecord
    }

    // 第二步：本地存储
    ...
    url, err = us.uploadImageFile(ctx, fileHeader, avatarFilePath)
    if err != nil {
        return "", err
    }
    us.fileRecordService.AddFileRecord(ctx, userID, avatarFilePath, url, string(plugin.UserPost))  // ✅ 写 FileRecord
    return url, nil
}
```

**关键分支**：
- 插件上传成功 → `return url, nil`，**跳过 `AddFileRecord`**
- 本地存储成功 → `AddFileRecord(ctx, userID, avatarFilePath, url, source)`

### 1.2 tryToUploadByPlugin 内部逻辑

[upload.go#L365-L390](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/uploader/upload.go#L365-L390)

```go
func (us *uploaderService) tryToUploadByPlugin(ctx *gin.Context, 
    source plugin.UploadSource) (url string, err error) {
    
    cond := plugin.UploadFileCondition{...}
    
    _ = plugin.CallStorage(func(fn plugin.Storage) error {
        resp := fn.UploadFile(ctx, cond)
        if resp.OriginalError != nil {
            err = ...
        } else {
            url = resp.FullURL    // 只拿到 URL，无 FileRecord
        }
        return nil
    })
    return url, err
}
```

**插件只返回 `FullURL`**，不返回文件路径、不触发 FileRecord 写入。

### 1.3 插件切换场景的完整推演

```
阶段一：使用 S3 插件上传

  用户上传 logo.png
    → tryToUploadByPlugin() → S3 插件上传成功
    → 返回 url = "https://cdn.example.com/post/abc123.jpg"
    → ❌ 无 FileRecord
    → 用户提交帖子，revision.content 含 https://cdn.example.com/post/abc123.jpg
  
  清理扫描 CleanOrphanUploadFiles：
    → file_record 表中没有这条记录
    → 根本扫描不到 → S3 上的文件无管理

─────────────────────────────────────────────────────

阶段二：管理员关闭 S3 插件，切换到本地存储

  新上传的文件走本地存储：
    → AddFileRecord() → file_record 有记录
    → URL 变为 https://site.com/uploads/post/xyz789.jpg
  
  但旧帖子的 revision.content 中仍是 S3 URL：
    https://cdn.example.com/post/abc123.jpg
  
  清理扫描 CleanOrphanUploadFiles：
    → 扫描 file_record 表
    → 新文件有记录，可正常追踪
    → ❌ S3 URL 在 file_record 中无记录
    → CleanOrphanUploadFiles 完全不知道 S3 文件的存在
  
  问题 1：旧 S3 文件永远不会被清理（清不到）
    → 即使帖子已删除，S3 上的文件仍存在
    → S3 存储费用持续累积
    → 只能手动登录 S3 控制台清理

─────────────────────────────────────────────────────

阶段三：管理员想切回 S3 插件

  旧帖子的 revision.content 中仍有本地存储 URL：
    https://site.com/uploads/post/def456.jpg
  
  清理扫描 CleanOrphanUploadFiles：
    → file_record 表中有这条记录（本地存储时写的）
    → GetLastRevisionByObjectID 或 GetLastRevisionByFileURL 命中
    → 判定为仍被引用 → 保留文件
  
  但帖子页面现在由 S3 插件处理：
    → 访问 https://site.com/uploads/post/def456.jpg
    → 如果本地文件已被 DeleteAndMoveFileRecord 删掉 → 404
    → 如果本地文件还在 → 可以访问，但绕过了 S3 CDN
  
  问题 2：本地文件无法被正确"救回"到 S3
    → CleanOrphanUploadFiles 只管理本地文件
    → 不会把本地文件迁移到 S3
    → 已被清理的本地文件在帖子中显示为损坏图片
```

### 1.4 双向失效总结

| 方向 | 问题 | 原因 |
|------|------|------|
| S3 → 本地 | 旧 S3 文件清不到 | 插件上传无 FileRecord，清理系统不知道 S3 文件存在 |
| 本地 → S3 | 新本地文件救不回 | 清理系统只能管理本地磁盘文件，不会迁移到 S3 |
| 混合状态 | URL 混乱 | revision.content 中同时存在 S3 URL 和本地 URL，前端无法统一处理 |

### 1.5 FileRecord 的设计假设

`CleanOrphanUploadFiles` 的完整逻辑链：

```
file_record 表 → FilePath → 本地磁盘路径 → DeleteAndMoveFileRecord → 移至 deleted/
```

**设计假设**：所有需要管理的文件都在 `file_record` 表中有记录，且物理文件在本地 `UploadPath` 下。

**插件上传打破了两条假设**：
1. 插件上传的文件不在 `file_record` 表中
2. 插件上传的文件不在本地 `UploadPath` 下

### 1.6 完整的行为差异表

| 维度 | 本地存储 | Storage 插件 |
|------|---------|-------------|
| FileRecord | ✅ 写入 DB | ❌ 不写入 |
| 清理追踪 | ✅ CleanOrphanUploadFiles | ❌ 无追踪 |
| 物理文件位置 | `UploadPath/post/` | S3/OSS 等远程 |
| EXIF 移除 | ✅ 自动 | 插件自行决定 |
| 像素校验 | ✅ DecodeAndCheckImageFile | 插件自行决定 |
| 品牌文件同步清理 | ✅ CleanUpRemovedBrandingFiles | ❌ 不触发 |
| 头像同步清理 | ❌ 无 | ❌ 无 |
| 插件切换后迁移 | ❌ 不迁移 | ❌ 不迁移 |

---

## 二、存量 HTML 在 bluemonday/goldmark 升级后不重渲染的隐患

### 2.1 预渲染落库模型回顾

所有对象在写入时，Schema 的 `Check()` 方法调用 `converter.Markdown2HTML()` 预渲染，结果存入 `parsed_text` / `bio_html` 字段：

| 对象 | Schema Check() | 存储字段 | 调用点 |
|------|---------------|---------|--------|
| Question | [question_schema.go#L97](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/schema/question_schema.go#L97) | `parsed_text` | 写入时一次 |
| Answer | [answer_schema.go#L64](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/schema/answer_schema.go#L64) | `parsed_text` | 写入时一次 |
| Comment | [comment_schema.go#L60](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/schema/comment_schema.go#L60) | `parsed_text` | 写入时一次 |
| Tag | [tag_schema.go#L212](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/schema/tag_schema.go#L212) | `parsed_text` | 写入时一次 |
| User Bio | [user_schema.go#L303](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/schema/user_schema.go#L303) `Markdown2BasicHTML` | `bio_html` | 写入时一次 |

**读取时直接返回 `parsed_text`，不再调用 Markdown2HTML**。只有编辑器预览走 `POST /post/render` 实时渲染。

### 2.2 Markdown2HTML 是无状态的纯函数

[markdown.go#L40-L66](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/pkg/converter/markdown.go#L40-L66)

```go
func Markdown2HTML(source string) string {
    mdConverter := goldmark.New(
        goldmark.WithExtensions(&DangerousHTMLFilterExtension{}, extension.GFM, extension.Footnote),
        ...
    )
    var buf bytes.Buffer
    if err := mdConverter.Convert([]byte(source), &buf); err != nil {
        return source
    }
    html := buf.String()
    filter := bluemonday.UGCPolicy()
    filter.AllowStyling()
    ...
    html = strings.TrimSpace(filter.Sanitize(html))
    return html
}
```

**每次调用都创建新的 goldmark 和 bluemonday 实例**，没有版本号、没有缓存、没有迁移机制。

### 2.3 升级场景的隐患推演

**场景 A：bluemonday 升级修复了安全漏洞**

```
升级前：bluemonday v1.0.0
  用户提交评论："<script>alert(1)</script>"
  Markdown2HTML → bluemonday.Sanitize → "<script>alert(1)</script>" (漏洞，未过滤)
  comment.parsed_text = "<script>alert(1)</script>"  // 存入 DB

升级后：bluemonday v1.1.0 (修复了此漏洞)
  新提交的评论 → 正确过滤 → 安全
  ❌ 但存量 parsed_text 仍为 "<script>alert(1)</script>"
  读取时直接返回 parsed_text → XSS 仍然存在！
```

**场景 B：goldmark 升级改变了渲染行为**

```
升级前：goldmark v1.0.0
  用户提交帖子："## Title\n- item1\n- item2"
  Markdown2HTML → "<h2>Title</h2><ul><li>item1</li><li>item2</li></ul>"
  question.parsed_text 存入 DB

升级后：goldmark v1.1.0 (GFM 扩展语法变更)
  新提交的帖子 → 新渲染结果
  ❌ 存量 parsed_text 仍是旧渲染结果
  同一种 Markdown 源码，不同帖子显示效果不一致
```

**场景 C：DangerousHTMLFilterExtension 策略调整**

```
升级前：RawHTML 中 <kbd> 被保留，其他被过滤
升级后：RawHTML 中 <kbd> 和 <mark> 都被保留

  存量帖子中 "<mark>important</mark>" 被过滤为纯文本
  新帖子中 "<mark>important</mark>" 正常渲染为高亮
  用户体验不一致
```

### 2.4 影响范围

| 对象类型 | 存量 HTML 位置 | 受影响行数 | 修复难度 |
|---------|--------------|-----------|---------|
| Question | `question.parsed_text` | 全表 | 需遍历全表 |
| Answer | `answer.parsed_text` | 全表 | 需遍历全表 |
| Comment | `comment.parsed_text` | 全表 | 需遍历全表 |
| Tag | `tag.parsed_text` | 全表 | 行数少，较易 |
| User Bio | `user.bio_html` | 全表 | 需遍历全表 |
| Revision | `revision.content` JSON 中的 `parsed_text` | 全表 | 最复杂，需解析 JSON |
| Activity | `activity.content` | 部分 | 按需 |

### 2.5 重渲染的代价与不可能性

**为什么不能简单地全表 UPDATE**：

1. **原始数据源**：重渲染需要 `original_text`（Markdown 原文），Question/Answer/Comment/Tag 都保留了原文，所以理论上可以
2. **Revision 中的嵌套**：revision.content 是完整实体的 JSON 序列化，其中既有 `original_text` 又有 `parsed_text`，需要解析 JSON、重渲染 `parsed_text`、再序列化回去
3. **数据量**：大型社区可能有数十万条 question/answer/comment，全表重渲染是 O(n) 的 CPU 密集操作
4. **不可逆变更**：如果新版本渲染结果有语义差异（如标题 ID 变化导致锚点失效），无法回滚
5. **安全风险**：重渲染可能意外"修复"了之前放过的 XSS，但也可能引入新问题

**实际上没有迁移脚本**：当前代码中没有任何重渲染存量数据的功能或迁移脚本。

### 2.6 设计权衡

| 方案 | 优势 | 劣势 |
|------|------|------|
| 当前方案（预渲染落库） | 读取快，无实时渲染开销 | 升级后存量不一致 |
| 实时渲染（读取时渲染） | 始终与最新版本一致 | 读取开销大，高并发下性能差 |
| 双写+版本标记 | 可选择性重渲染 | 复杂度高，存储翻倍 |
| 惰性重渲染（读取时比较版本号） | 按需重渲染 | 需要版本号机制，首次读取慢 |

---

## 三、同图通过 uid.IDStr12 随机命名不做去重，多次上传后清理只能逐条针对

### 3.1 随机命名机制

[id.go#L51-L54](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/pkg/uid/id.go#L51-L54)

```go
func IDStr12() string {
    id := snowFlakeIDGenerator.Generate()
    return id.Base58()
}
```

每次上传都生成一个全新的 12 位 Base58 随机 ID，作为文件名的一部分：

[upload.go#L222](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/service/uploader/upload.go#L222)

```go
newFilename := fmt.Sprintf("%s%s", uid.IDStr12(), fileExt)
// 例如: aB3xK9mN2pQr.jpg
```

**Base58 字符集**：`123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz`（58 个字符）

**12 位 Base58 的空间**：58^12 ≈ 1.95 × 10^21 种组合

### 3.2 同图多次上传的实况推演

```
用户 A 上传 logo.png (MD5: d41d8cd98f00b204e9800998ecf8427e)
  → newFilename = "aB3xK9mN2pQr.jpg"
  → 本地路径: UploadPath/post/aB3xK9mN2pQr.jpg
  → FileRecord { FilePath: "post/aB3xK9mN2pQr.jpg", FileURL: "https://site/uploads/post/aB3xK9mN2pQr.jpg" }
  → 帖子 revision v1: original_text = "![logo](https://site/uploads/post/aB3xK9mN2pQr.jpg)"

用户 A 再次上传同一张 logo.png (相同 MD5)
  → newFilename = "zY8wL5tM1nVp.jpg"    ← 全新的随机 ID
  → 本地路径: UploadPath/post/zY8wL5tM1nVp.jpg
  → FileRecord { FilePath: "post/zY8wL5tM1nVp.jpg", FileURL: "https://site/uploads/post/zY8wL5tM1nVp.jpg" }
  → 帖子 revision v2: original_text = "![logo](https://site/uploads/post/zY8wL5tM1nVp.jpg)"

磁盘上现在有两份完全相同的图片：
  UploadPath/post/aB3xK9mN2pQr.jpg  (旧)
  UploadPath/post/zY8wL5tM1nVp.jpg  (新)

file_record 表中有两条记录，分别指向不同的 URL。
```

### 3.3 清理必须逐条处理

```
CleanOrphanUploadFiles 扫描：

  记录 1: aB3xK9mN2pQr.jpg
    → GetLastRevisionByFileURL("https://site/uploads/post/aB3xK9mN2pQr.jpg")
    → 命中 revision v1 → 保留（即使 revision v2 已经不再引用这个 URL）

  记录 2: zY8wL5tM1nVp.jpg
    → GetLastRevisionByFileURL("https://site/uploads/post/zY8wL5tM1nVp.jpg")
    → 命中 revision v2 → 保留

结果: 两份完全相同的图片都保留，无法去重
```

### 3.4 去重不可能的原因链

**原因 1：上传时无内容哈希**

`UploadPostFile` 只检查文件扩展名和大小，不计算文件内容的 MD5/SHA256。即使两个文件内容完全相同，也会生成不同的文件名和 FileRecord。

**原因 2：FileRecord 无内容指纹**

[file_record_entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/67-answer/internal/entity/file_record_entity.go) 的字段中只有 `FilePath` 和 `FileURL`，没有 `ContentHash` 或 `MD5` 字段。无法通过哈希值找到重复文件。

**原因 3：URL 是唯一标识**

整个系统以 URL 作为文件的唯一引用标识。同一图片的不同上传会产生不同 URL，系统无法知道它们是同一内容。

**原因 4：清理基于 URL 匹配**

`GetLastRevisionByFileURL` 按 URL 字符串匹配，不同 URL 即使内容相同也被视为不同文件。

### 3.5 存储浪费的实际影响

| 场景 | 浪费程度 | 说明 |
|------|---------|------|
| 单用户重复上传 | 低 | 通常不会重复上传同一文件 |
| 多用户上传相同图片 | 中 | 热门图片（如 logo、icon）可能被多人上传 |
| 编辑替换图片 | 高 | 每次编辑换图都产生新文件，旧文件因 revision 保留不清 |
| 拖拽+粘贴重复 | 高 | 用户拖拽上传后又在粘贴区粘贴同一图片，产生两份 |

### 3.6 与去重系统的对比

| 维度 | 当前实现 | 理想去重系统 |
|------|---------|------------|
| 文件标识 | URL (随机) | Content Hash (确定性) |
| 上传时计算 | ❌ 不算哈希 | ✅ 计算 MD5/SHA256 |
| FileRecord 字段 | FilePath + FileURL | + ContentHash |
| 去重策略 | 无 | 相同哈希返回已有 URL |
| 清理策略 | 按 URL 逐条 | 按哈希引用计数 |
| 存储开销 | O(上传次数) | O(唯一内容数) |

### 3.7 为什么不做去重——设计权衡

| 不去重的理由 | 分析 |
|-------------|------|
| 简单性 | 不需要哈希计算、引用计数、冲突处理 |
| 安全性 | 不同用户上传相同内容的文件应隔离，避免 A 删图影响 B |
| 独立性 | 每个上传是独立操作，不依赖其他上传的状态 |
| 随机性 | IDStr12 避免文件名冲突和路径遍历攻击 |

| 去重的潜在风险 | 分析 |
|---------------|------|
| 删除级联 | A 上传的图片被 B 复用，A 删帖后 B 的图片也失效 |
| 哈希冲突 | MD5 碰撞导致不同文件共享同一 URL |
| 复杂度 | 引用计数、原子操作、并发安全 |

**核心决策**：选择了**简单 + 安全**，放弃了**存储效率**。

---

## 四、三个问题的关联性

这三个问题看似独立，但共享一个根本设计理念：**写入时决定一切，读取时不干预**。

| 问题 | 写入时决策 | 读取时后果 |
|------|-----------|-----------|
| 插件切换 | 插件上传不写 FileRecord | 旧文件无追踪，新文件不迁移 |
| HTML 冻结 | 预渲染落库，不存版本号 | 升级后存量不一致 |
| 无去重 | 每次上传生成独立文件 | 存储浪费，清理逐条 |

**设计哲学**：以写入的简单性换取系统的可维护性。每次上传、每次提交都是独立、幂等、无副作用的操作。不引入跨上传的依赖关系、不引入版本号机制、不引入引用计数。

**代价**：
1. 存储只增不减（旧文件不清理、重复文件不去重）
2. 数据一致性随时间退化（HTML 不重渲染）
3. 运维复杂度外移（插件切换需手动迁移、安全修复需手动重渲染）
