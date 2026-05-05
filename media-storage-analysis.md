# Listmonk 媒体存储与 URL 重写机制分析报告

## 1. 概述

Listmonk 采用了**接口抽象 + 多后端实现**的设计模式，实现了媒体文件在本地文件系统和 S3 存储之间的无缝切换。核心设计思想是：**数据库只存储相对路径（文件名），URL 在访问时动态生成**。

**⚠️ 重要修正**：通过二次核查发现，上述设计思想**仅适用于媒体元数据查询场景**。对于**邮件正文内联图片**，实际行为是：**完整 URL 在编辑阶段就已写入正文，发送阶段没有重写机制**。

---

## 2. 核心架构

### 2.1 媒体存储接口

定义在 `internal/media/media.go:26-32`：

```go
type Store interface {
    Put(string, string, io.ReadSeeker) (string, error)  // 上传文件
    Delete(string) error                                   // 删除文件
    GetURL(string) string                                  // 获取访问 URL
    GetBlob(string) ([]byte, error)                       // 获取文件内容（字节）
}
```

### 2.2 媒体数据模型

定义在 `internal/media/media.go:11-24`：

```go
type Media struct {
    ID          int         `db:"id" json:"id"`
    UUID        string      `db:"uuid" json:"uuid"`
    Filename    string      `db:"filename" json:"filename"`  // 只存储相对路径/文件名
    ContentType string      `db:"content_type" json:"content_type"`
    Thumb       string      `db:"thumb" json:"-"`             // 缩略图文件名
    CreatedAt   null.Time   `db:"created_at" json:"created_at"`
    ThumbURL    null.String `json:"thumb_url"`                 // 运行时生成的 URL
    Provider    string      `json:"provider"`
    Meta        models.JSON `db:"meta" json:"meta"`
    URL         string      `json:"url"`                         // 运行时生成的 URL
    Total       int         `db:"total" json:"-"`
}
```

**关键设计点**：数据库只存储 `Filename`（相对路径），`URL` 和 `ThumbURL` 是在运行时动态生成的。

---

## 3. 邮件正文内联图片 URL 处理机制（二次核查重点）

### 3.1 关键结论

| 问题 | 答案 |
|------|------|
| URL 在哪个阶段写入正文？ | **编辑阶段**（前端插入图片时） |
| URL 是相对路径还是完整路径？ | **完整 URL**（包含域名） |
| 发送阶段是否重写？ | **否**（无任何重写机制） |
| 切换存储后历史链接是否自动迁移？ | **否**（历史正文保留旧 URL） |

### 3.2 编辑阶段：URL 写入流程

#### 前端富文本编辑器插入图片

**文件位置**: `frontend/src/components/RichtextEditor.vue:329-331`

```javascript
onMediaSelect(media) {
  // 直接使用 API 返回的完整 URL
  this.imageCallack(media.url);
}
```

当用户在富文本编辑器中选择媒体库中的图片时：
1. 前端调用 `onMediaSelect(media)` 回调
2. 直接使用 `media.url` 插入 `<img src="...">`
3. **完整 URL 被写入 HTML 正文**

#### 后端 API 返回完整 URL

**文件位置**: `internal/core/media.go:34-39`（QueryMedia）和 `64-67`（GetMedia）

```go
// QueryMedia 中
for i := 0; i < len(out); i++ {
    // 动态生成完整 URL
    out[i].URL = s.GetURL(out[i].Filename)
    
    if out[i].Thumb != "" {
        out[i].ThumbURL = null.String{Valid: true, String: s.GetURL(out[i].Thumb)}
    }
}

// GetMedia 中
out.URL = s.GetURL(out.Filename)
if out.Thumb != "" {
    out.ThumbURL = null.String{Valid: true, String: s.GetURL(out.Thumb)}
}
```

**关键洞察**：
- 媒体 API 返回的是**完整 URL**（通过 `GetURL()` 动态生成）
- 前端直接将这个完整 URL 写入邮件正文
- **正文存储的是静态 URL，不是相对路径或模板变量**

#### 正文存储位置

邮件正文存储在以下数据库字段中：

| 表名 | 字段名 | 用途 |
|------|--------|------|
| `campaigns` | `body` | 邮件正文内容 |
| `campaigns` | `body_source` | 正文源码（如可视化编辑器 JSON） |
| `templates` | `body` | 模板正文 |
| `templates` | `body_source` | 模板源码 |

**示例**：假设使用 filesystem 存储，配置如下：
- `app.root_url = "https://listmonk.example.com"`
- `upload.filesystem.upload_uri = "/uploads"`

编辑阶段插入的图片 URL 是：
```html
<img src="https://listmonk.example.com/uploads/image_abc123.jpg">
```

**这个完整 URL 被直接写入 campaigns.body 字段**。

### 3.3 发送阶段：无重写机制

#### 邮件渲染流程

**文件位置**: `internal/manager/message.go:33-88`

```go
func (m *CampaignMessage) render() error {
    out := bytes.Buffer{}

    // 渲染主题（如果是模板）
    if m.Campaign.SubjectTpl != nil {
        if err := m.Campaign.SubjectTpl.ExecuteTemplate(&out, models.ContentTpl, m); err != nil {
            return err
        }
        m.subject = out.String()
        out.Reset()
    }

    // 编译主模板
    // ⚠️ 这只是执行 Go 模板语法 {{ }}，不会重写 HTML 中的 <img src="">
    if err := m.Campaign.Tpl.ExecuteTemplate(&out, models.BaseTpl, m); err != nil {
        return err
    }
    m.body = out.Bytes()

    // ... 其他渲染逻辑
    return nil
}
```

**关键发现**：
- `render()` 方法只执行 **Go 模板语法**（`{{ TrackLink }}`、`{{ .Subscriber }}` 等）
- **不会扫描或重写 HTML 中的 `<img src="">` URL**
- 正文中原封不动地使用编辑阶段写入的 URL

#### 发送时的消息构造

**文件位置**: `internal/manager/manager.go:488-519`

```go
// 构造外发消息
out := models.Message{
    From:        msg.from,
    To:          []string{msg.to},
    Subject:     msg.subject,
    ContentType: msg.Campaign.ContentType,
    Body:        msg.body,        // ⚠️ 直接使用渲染后的 body，无 URL 重写
    AltBody:     msg.altBody,
    Subscriber:  msg.Subscriber,
    Campaign:    msg.Campaign,
    Attachments: msg.Campaign.Attachments,
}
```

### 3.4 切换存储后历史链接的行为

#### 场景示例：从 filesystem 切换到 S3

**切换前（filesystem）**：
- 配置：`upload.provider = "filesystem"`
- 媒体 API 返回 URL：`https://listmonk.example.com/uploads/image.jpg`
- 正文保存：`<img src="https://listmonk.example.com/uploads/image.jpg">`

**切换后（S3）**：
- 配置：`upload.provider = "s3"`
- `upload.s3.public_url = "https://cdn.example.com"`

**历史 Campaign 的行为**：
1. **新创建的 Campaign**：插入新图片时使用新 URL（`https://cdn.example.com/...`）
2. **历史 Campaign**：body 中保存的仍然是旧 URL（`https://listmonk.example.com/uploads/...`）
3. **发送时**：旧 URL 被原样使用，**不会自动重写**

#### 结果分析

| 情况 | 结果 |
|------|------|
| 旧存储仍然可用（filesystem 目录保留） | 历史链接**正常工作**（旧 URL 仍可访问） |
| 旧存储已删除/不可用 | 历史链接**失效**（404 错误） |
| 使用了相对路径 public_url（如 `/s3-media`） | **可能部分工作**（取决于路由配置） |

#### 唯一的例外：相对路径 public_url

如果 S3 配置使用**相对路径**的 `public_url`：

```toml
[upload.s3]
public_url = "/s3-media"  # 相对路径，会自动拼接 RootURL
```

此时：
- 编辑阶段写入的 URL：`https://listmonk.example.com/s3-media/image.jpg`
- 如果只是**切换 bucket** 但保持相同的 `public_url` 路径，链接仍然有效
- 如果是**切换存储类型**（filesystem → s3）且路径结构不同，链接会失效

### 3.5 设计意图与限制

#### 设计意图

Listmonk 的媒体系统设计采用了**两层策略**：

| 层级 | 设计 | 适用场景 |
|------|------|----------|
| **媒体元数据层** | 数据库存文件名，URL 动态生成 | 媒体库查询、API 响应 |
| **邮件正文层** | 直接写入完整 URL | 富文本编辑器、邮件内容 |

#### 为什么没有正文 URL 重写？

可能的设计考量：
1. **性能考虑**：发送时扫描并替换每个邮件正文中的 URL 会增加延迟
2. **复杂性**：需要可靠地解析 HTML、识别媒体 URL、匹配数据库记录
3. **模板灵活性**：用户可能在正文中插入外部图片 URL，不应被误修改
4. **历史兼容性**：保持简单的 "what you see is what you send" 模型

---

## 4. 存储后端实现

### 4.1 本地文件系统存储

**文件位置**: `internal/media/providers/filesystem/filesystem.go`

#### 配置结构

```go
type Opts struct {
    UploadPath string `koanf:"upload_path"`  // 本地存储路径，如: /home/listmonk/uploads
    UploadURI  string `koanf:"upload_uri"`   // 访问 URI 前缀，如: /uploads
    RootURL    string `koanf:"root_url"`      // 应用根 URL，如: https://listmonk.example.com
}
```

#### URL 生成逻辑 (`GetURL` 方法)

```go
func (c *Client) GetURL(name string) string {
    return fmt.Sprintf("%s%s/%s", c.opts.RootURL, c.opts.UploadURI, name)
}
```

**示例**:
- `RootURL` = `https://listmonk.example.com`
- `UploadURI` = `/uploads`
- `name` = `image.jpg`
- 生成的 URL: `https://listmonk.example.com/uploads/image.jpg`

#### 核心方法

| 方法 | 功能 | 实现 |
|------|------|------|
| `Put` | 上传文件 | 直接写入本地文件系统 |
| `GetURL` | 获取访问 URL | 拼接 `RootURL + UploadURI + 文件名` |
| `GetBlob` | 获取文件内容 | 从本地文件系统读取 |
| `Delete` | 删除文件 | 删除本地文件 |

### 4.2 S3 存储

**文件位置**: `internal/media/providers/s3/s3.go`

#### 配置结构

```go
type Opt struct {
    URL        string        `koanf:"url"`           // S3 端点 URL
    PublicURL  string        `koanf:"public_url"`    // 公共访问 URL（可配置为相对路径）
    AccessKey  string        `koanf:"aws_access_key_id"`
    SecretKey  string        `koanf:"aws_secret_access_key"`
    Region     string        `koanf:"aws_default_region"`
    Bucket     string        `koanf:"bucket"`
    BucketPath string        `koanf:"bucket_path"`   // Bucket 内的路径前缀
    BucketType string        `koanf:"bucket_type"`   // "public" 或 "private"
    Expiry     time.Duration `koanf:"expiry"`        // 预签名 URL 过期时间
    RootURL    string        `koanf:"root_url"`      // 应用根 URL（用于相对路径 public_url）
}
```

#### URL 生成逻辑 (`GetURL` 方法)

这是 S3 存储的核心逻辑，支持多种 URL 模式：

```go
func (c *Client) GetURL(name string) string {
    // 私有 bucket 且未配置 public_url：生成预签名 URL
    if c.opts.BucketType == "private" && c.opts.PublicURL == "" {
        u := c.s3.GeneratePresignedURL(simples3.PresignedInput{
            Bucket:        c.opts.Bucket,
            ObjectKey:     c.makeBucketPath(name),
            Method:        "GET",
            Timestamp:     time.Now(),
            ExpirySeconds: int(c.opts.Expiry.Seconds()),
        })
        return u
    }

    // 公共 bucket 或配置了 public_url：生成普通 URL
    return c.makeFileURL(name)
}
```

#### `makeFileURL` 方法（URL 重写逻辑）

```go
func (c *Client) makeFileURL(name string) string {
    if c.opts.PublicURL != "" {
        prefix := c.opts.PublicURL
        // 如果 public_url 是相对路径（以 / 开头），则自动加上 RootURL
        if strings.HasPrefix(prefix, "/") {
            prefix = c.opts.RootURL + prefix
        }
        return prefix + "/" + c.makeBucketPath(name)
    }

    // 没有 public_url 配置，使用默认 S3 URL 格式
    return c.opts.URL + "/" + c.opts.Bucket + "/" + c.makeBucketPath(name)
}
```

#### `makeBucketPath` 方法

```go
func (c *Client) makeBucketPath(name string) string {
    // 如果路径是根 (/), 返回不带前置斜杠的文件名
    p := strings.TrimPrefix(strings.TrimSuffix(c.opts.BucketPath, "/"), "/")
    if p == "" {
        return name
    }

    // whatever/bucket/path/filename.jpg: 无前置斜杠
    return p + "/" + name
}
```

#### URL 生成示例

| 配置场景 | 生成的 URL |
|---------|-----------|
| **私有 bucket** + 无 `public_url` | `https://bucket.s3.region.amazonaws.com/path/file.jpg?X-Amz-Algorithm=...`（预签名 URL） |
| **公共 bucket** + 无 `public_url` | `https://s3.region.amazonaws.com/bucket/path/file.jpg` |
| **任意 bucket** + `public_url = "https://cdn.example.com"` | `https://cdn.example.com/path/file.jpg` |
| **任意 bucket** + `public_url = "/s3-media"` | `{RootURL}/s3-media/path/file.jpg`（相对路径自动拼接 RootURL） |

#### 核心方法

| 方法 | 功能 | 实现 |
|------|------|------|
| `Put` | 上传文件 | 调用 S3 API 上传，支持设置 ACL（public-read） |
| `GetURL` | 获取访问 URL | 根据 bucket 类型和 public_url 配置生成不同类型的 URL |
| `GetBlob` | 获取文件内容 | 调用 S3 API 下载文件 |
| `Delete` | 删除文件 | 调用 S3 API 删除文件 |

---

## 5. S3 相对路径 public_url 代理机制（二次核查重点）

### 5.1 架构概述

当 S3 的 `public_url` 配置为**相对路径**（如 `/s3-media`）时，Listmonk 会：

1. **URL 生成**：`makeFileURL` 自动将 `RootURL` 拼接到前面
2. **路由注册**：注册一个 HTTP 路由来代理这些请求
3. **代理服务**：`ServeS3Media` 处理器从 S3 获取文件并返回给客户端

### 5.2 路由注册

**文件位置**: `cmd/init.go:954-964`

```go
var (
    uploadProvider = ko.String("upload.provider")
    uploadFsURI    = ko.String("upload.filesystem.upload_uri")
    publicURL      = ko.String("upload.s3.public_url")
)
switch {
case uploadProvider == "filesystem" && uploadFsURI != "":
    // 文件系统：直接静态服务
    srv.Static(uploadFsURI, ko.String("upload.filesystem.upload_path"))
case uploadProvider == "s3" && strings.HasPrefix(publicURL, "/"):
    // ⚠️ S3 相对路径：注册代理路由
    // 使用 Echo 的单段路由参数 :filepath
    srv.GET(path.Join(publicURL, "/:filepath"), app.ServeS3Media)
}
```

### 5.3 代理处理器

**文件位置**: `cmd/media.go:195-208`

```go
func (a *App) ServeS3Media(c echo.Context) error {
    // ⚠️ 从路由参数获取 filepath（单段参数）
    key := c.Param("filepath")
    if key == "" {
        return echo.NewHTTPError(http.StatusBadRequest, "missing media file path")
    }

    // 从 S3 获取文件内容
    b, err := a.media.GetBlob(key)
    if err != nil {
        a.log.Printf("error fetching media from s3 %s: %v", key, err)
        return echo.NewHTTPError(http.StatusInternalServerError, "error fetching media")
    }

    // 流式返回给客户端
    return c.Stream(http.StatusOK, http.DetectContentType(b), bytes.NewReader(b))
}
```

### 5.4 GetBlob 方法的路径处理

**文件位置**: `internal/media/providers/s3/s3.go:111-136`

```go
func (c *Client) GetBlob(uurl string) ([]byte, error) {
    // 解析 URL，提取路径
    if p, err := url.Parse(uurl); err != nil {
        // ⚠️ 如果不是有效 URL，使用 filepath.Base 获取文件名
        uurl = filepath.Base(uurl)
    } else {
        // ⚠️ 如果是有效 URL，从路径中提取文件名
        uurl = filepath.Base(p.Path)
    }

    // 从 S3 下载文件
    file, err := c.s3.FileDownload(simples3.DownloadInput{
        Bucket:    c.opts.Bucket,
        // ⚠️ 再次使用 filepath.Base，确保只使用文件名
        ObjectKey: c.makeBucketPath(filepath.Base(uurl)),
    })
    if err != nil {
        return nil, err
    }

    // 读取为字节
    b, err := io.ReadAll(file)
    if err != nil {
        return nil, err
    }
    defer file.Close()

    return b, nil
}
```

### 5.5 行为边界与潜在失败场景（深度分析）

#### 设计缺陷总览

通过代码分析，发现**两个相互关联的设计缺陷**：

| 缺陷 | 位置 | 影响 |
|------|------|------|
| 路由参数限制 | `cmd/init.go:963` | 无法匹配多级路径 |
| filepath.Base 处理 | `internal/media/providers/s3/s3.go:114-122` | 丢弃目录层级 |

#### 场景 1：bucket_path 为空（正常工作）

**配置**：
```toml
[upload.s3]
bucket_path = ""      # 空路径
public_url = "/s3-media"
```

**流程**：
1. 上传文件 `image.jpg`
2. 实际 S3 对象键：`image.jpg`（`makeBucketPath("image.jpg")` = `"image.jpg"`）
3. URL 生成：`makeFileURL("image.jpg")` = `{RootURL}/s3-media/image.jpg`
4. 路由匹配：`:filepath` = `"image.jpg"` ✓
5. `ServeS3Media` 接收：`key = "image.jpg"`
6. `GetBlob("image.jpg")`：
   - `filepath.Base("image.jpg")` = `"image.jpg"`
   - `makeBucketPath("image.jpg")` = `"image.jpg"`
7. S3 下载：`ObjectKey = "image.jpg"` ✓

**结果**：✅ 正常工作

#### 场景 2：bucket_path 不为空（失败场景 1）

**配置**：
```toml
[upload.s3]
bucket_path = "assets"   # 非空路径
public_url = "/s3-media"
```

**流程**：
1. 上传文件 `image.jpg`
2. 实际 S3 对象键：`assets/image.jpg`（`makeBucketPath("image.jpg")` = `"assets/image.jpg"`）
3. URL 生成：`makeFileURL("image.jpg")` = `{RootURL}/s3-media/assets/image.jpg`
4. **路由匹配问题**：
   - Echo 路由：`/s3-media/:filepath`
   - 请求 URL：`/s3-media/assets/image.jpg`
   - `:filepath` 只能匹配单段路径 → **只匹配到 `"assets"`**
   - **`image.jpg` 部分被丢弃！**
5. `ServeS3Media` 接收：`key = "assets"`（不完整！）
6. `GetBlob("assets")`：
   - `filepath.Base("assets")` = `"assets"`
   - `makeBucketPath("assets")` = `"assets/assets"`（错误！）
7. S3 下载：`ObjectKey = "assets/assets"` ❌
8. **实际对象键是 `assets/image.jpg`** → **404 错误**

**结果**：❌ 失败

#### 场景 3：更深的目录层级（失败场景 2）

**配置**：
```toml
[upload.s3]
bucket_path = "assets/images/2024"
public_url = "/s3-media"
```

**流程**：
1. 上传文件 `photo.jpg`
2. 实际 S3 对象键：`assets/images/2024/photo.jpg`
3. URL 生成：`{RootURL}/s3-media/assets/images/2024/photo.jpg`
4. **路由匹配问题**：
   - Echo 路由：`/s3-media/:filepath`
   - 请求 URL 包含 4 个斜杠后的段
   - `:filepath` 只能匹配第一个段 `"assets"`
   - **`images/2024/photo.jpg` 全部丢失！**
5. `ServeS3Media` 接收：`key = "assets"`
6. `GetBlob("assets")`：
   - `filepath.Base("assets")` = `"assets"`
   - `makeBucketPath("assets")` = `"assets/images/2024/assets"`
7. S3 下载：`ObjectKey = "assets/images/2024/assets"` ❌
8. **实际对象键是 `assets/images/2024/photo.jpg`** → **404 错误**

**结果**：❌ 失败

#### 场景 4：即使路由修复，filepath.Base 仍有问题

假设路由被修复为使用通配符参数（Echo 的 `*` 语法）：

```go
// 假设修复后的路由
srv.GET(path.Join(publicURL, "/*"), app.ServeS3Media)
```

**请求**：`/s3-media/assets/image.jpg`
**修复后 key**：`"assets/image.jpg"`

**但 `GetBlob` 仍然失败**：
```go
func (c *Client) GetBlob(uurl string) ([]byte, error) {
    // uurl = "assets/image.jpg"
    if p, err := url.Parse(uurl); err != nil {
        // 不是有效 URL，走这个分支
        uurl = filepath.Base(uurl)  // ⚠️ filepath.Base("assets/image.jpg") = "image.jpg"
    }
    // ...
    ObjectKey: c.makeBucketPath(filepath.Base(uurl)),  // ⚠️ 再次使用 filepath.Base
}
```

**流程**：
1. `uurl = "assets/image.jpg"`
2. `filepath.Base("assets/image.jpg")` = `"image.jpg"`（**目录被丢弃！**）
3. `makeBucketPath("image.jpg")` = `"assets/image.jpg"`（假设 bucket_path = "assets"）
4. 这看起来是对的，但**如果原始路径包含更多层级**？

**更深层级的问题**：
- `bucket_path = "assets/images"`
- 实际对象键：`assets/images/photo.jpg`
- URL：`/s3-media/assets/images/photo.jpg`
- 修复后 key：`"assets/images/photo.jpg"`
- `filepath.Base("assets/images/photo.jpg")` = `"photo.jpg"`
- `makeBucketPath("photo.jpg")` = `"assets/images/photo.jpg"`
- **这个场景恰好正确**，但逻辑是巧合

**真正的问题场景**：
- `bucket_path = "assets"`
- 用户上传了一个**本身包含路径的文件名**（虽然不太可能）
- 或者 S3 中已有**手动上传**的带路径文件
- `GetBlob` 会丢弃所有目录层级

### 5.6 设计假设与限制

#### 隐含的设计假设

通过代码分析，S3 相对路径代理机制**隐含了以下假设**：

| 假设 | 说明 |
|------|------|
| `bucket_path` 为空 | 对象键就是简单的文件名，无目录层级 |
| 文件名不包含斜杠 | `filepath.Base()` 可以安全使用 |
| URL 路径只有一段 | 路由的 `:filepath` 单段参数足够 |

#### 实际影响

| 配置情况 | 是否支持 |
|----------|----------|
| `bucket_path = ""` | ✅ 支持 |
| `bucket_path` 不为空 + 相对路径 `public_url` | ⚠️ **不支持**（设计缺陷） |
| `bucket_path` 不为空 + 绝对路径 `public_url`（CDN） | ✅ 支持（CDN 直接访问 S3） |
| `bucket_path` 不为空 + 私有 bucket + 预签名 URL | ✅ 支持（URL 直接指向 S3） |

#### 建议的使用方式

为了避免上述问题，建议：

1. **使用相对路径 `public_url` 时**：
   ```toml
   [upload.s3]
   bucket_path = ""  # 必须为空！
   public_url = "/s3-media"
   ```

2. **需要 `bucket_path` 时**：
   - 使用绝对路径的 `public_url`（CDN）
   - 或者使用私有 bucket + 预签名 URL
   ```toml
   [upload.s3]
   bucket_path = "assets/images"
   public_url = "https://cdn.example.com/media"  # 绝对路径，CDN 负责代理
   # 或者
   bucket_type = "private"
   public_url = ""  # 使用预签名 URL
   ```

### 5.7 可能的修复方案

如果确实需要在相对路径代理场景下使用 `bucket_path`，需要修复以下两点：

#### 修复 1：路由使用通配符参数

```go
// 修复前（当前实现）
srv.GET(path.Join(publicURL, "/:filepath"), app.ServeS3Media)

// 修复后（使用 Echo 的通配符路由）
srv.GET(path.Join(publicURL, "/*"), app.ServeS3Media)
```

#### 修复 2：GetBlob 移除 filepath.Base 处理

```go
// 修复前（当前实现）
func (c *Client) GetBlob(uurl string) ([]byte, error) {
    if p, err := url.Parse(uurl); err != nil {
        uurl = filepath.Base(uurl)  // ⚠️ 丢弃目录
    } else {
        uurl = filepath.Base(p.Path)  // ⚠️ 丢弃目录
    }

    file, err := c.s3.FileDownload(simples3.DownloadInput{
        Bucket:    c.opts.Bucket,
        ObjectKey: c.makeBucketPath(filepath.Base(uurl)),  // ⚠️ 再次丢弃
    })
    // ...
}

// 修复后（保留完整路径）
func (c *Client) GetBlob(uurl string) ([]byte, error) {
    // 如果是完整 URL，提取路径部分
    if p, err := url.Parse(uurl); err == nil && p.Scheme != "" {
        uurl = p.Path
        // 移除可能的前导斜杠
        uurl = strings.TrimPrefix(uurl, "/")
    }

    // 直接使用完整路径作为 ObjectKey
    // 注意：这里假设调用方已经知道完整的对象键
    // 或者需要根据 public_url 前缀来判断是否需要拼接 bucket_path
    // 这需要更复杂的逻辑...
    
    file, err := c.s3.FileDownload(simples3.DownloadInput{
        Bucket:    c.opts.Bucket,
        ObjectKey: uurl,  // 直接使用传入的路径
    })
    // ...
}
```

**注意**：完整的修复需要仔细考虑：
- 如何区分 "相对路径代理场景" 和 "普通 URL 场景"
- `GetBlob` 的调用方可能传入完整 URL、相对路径或纯文件名
- 需要保持向后兼容性

---

## 6. 媒体上传流程

### 6.1 完整上传流程

**文件位置**: `cmd/media.go:26-140`

```
1. 接收上传文件 (FormFile)
        ↓
2. 验证文件扩展名 (检查配置的 upload.extensions)
        ↓
3. 生成唯一文件名 (makeFilename + 检查数据库是否存在)
        ↓
4. 调用存储后端的 Put 方法 (a.media.Put())
        ↓
5. 如果是图片文件，生成缩略图
        ↓
6. 上传缩略图到存储后端
        ↓
7. 将媒体元数据写入数据库 (core.InsertMedia)
        ↓
8. 返回包含动态生成 URL 的 Media 对象
```

### 6.2 关键代码片段

```go
// UploadMedia handles media file uploads.
func (a *App) UploadMedia(c echo.Context) error {
    // 1. 接收文件
    file, err := c.FormFile("file")
    // ... 错误处理
    
    // 2. 验证扩展名
    ext := strings.TrimPrefix(strings.ToLower(filepath.Ext(file.Filename)), ".")
    if !inArray("*", a.cfg.MediaUpload.Extensions) {
        if ok := inArray(ext, a.cfg.MediaUpload.Extensions); !ok {
            return echo.NewHTTPError(http.StatusBadRequest,
                a.i18n.Ts("media.unsupportedFileType", "type", ext))
        }
    }
    
    // 3. 生成唯一文件名
    fName := makeFilename(file.Filename)
    if _, err := a.core.GetMedia(0, "", fName, a.media); err == nil {
        // 文件名已存在，添加随机后缀
        suffix, _ := generateRandomString(6)
        fName = appendSuffixToFilename(fName, suffix)
    }
    
    // 4. 上传到存储后端
    fName, err = a.media.Put(fName, contentType, src)
    // ... 错误处理
    
    // 5. 生成缩略图（图片文件）
    if isImage {
        thumbFile, width, height, err := processImage(file)
        // ... 错误处理
        
        // 6. 上传缩略图
        tf, err := a.media.Put(thumbPrefix+fName, contentType, thumbFile)
        thumbfName = tf
    }
    
    // 7. 写入数据库
    m, err := a.core.InsertMedia(fName, thumbfName, contentType, meta, a.cfg.MediaUpload.Provider, a.media)
    // ... 错误处理
    
    // 8. 返回结果（包含动态生成的 URL）
    return c.JSON(http.StatusOK, okResp{m})
}
```

---

## 7. 存储后端切换机制

### 7.1 初始化逻辑

**文件位置**: `cmd/init.go:750-783`

```go
func initMediaStore(ko *koanf.Koanf) media.Store {
    switch provider := ko.String("upload.provider"); provider {
    case "s3":
        var o s3.Opt
        ko.Unmarshal("upload.s3", &o)
        o.RootURL = ko.String("app.root_url")
        
        up, err := s3.NewS3Store(o)
        if err != nil {
            lo.Fatalf("error initializing s3 upload provider %s", err)
        }
        lo.Println("media upload provider: s3")
        return up

    case "filesystem":
        var o filesystem.Opts

        ko.Unmarshal("upload.filesystem", &o)
        o.RootURL = ko.String("app.root_url")
        o.UploadPath = filepath.Clean(o.UploadPath)
        o.UploadURI = filepath.Clean(o.UploadURI)
        up, err := filesystem.New(o)
        if err != nil {
            lo.Fatalf("error initializing filesystem upload provider %s", err)
        }
        lo.Println("media upload provider: filesystem")
        return up

    default:
        lo.Fatalf("unknown provider. select filesystem or s3")
    }
    return nil
}
```

### 7.2 配置示例

#### 文件系统存储配置

```toml
[upload]
provider = "filesystem"
extensions = ["jpg", "jpeg", "png", "gif", "svg"]

[upload.filesystem]
upload_path = "/home/listmonk/uploads"
upload_uri = "/uploads"
```

#### S3 存储配置（公共 CDN）

```toml
[upload]
provider = "s3"
extensions = ["jpg", "jpeg", "png", "gif", "svg"]

[upload.s3]
aws_default_region = "ap-south-1"
aws_access_key_id = "AKIA..."
aws_secret_access_key = "..."
bucket = "listmonk-media"
bucket_path = "/"
bucket_type = "public"
url = "https://s3.ap-south-1.amazonaws.com"
public_url = "https://cdn.example.com"  # 绝对 URL，直接使用
```

#### S3 存储配置（相对路径代理）

```toml
[upload]
provider = "s3"

[upload.s3]
aws_default_region = "ap-south-1"
aws_access_key_id = "AKIA..."
aws_secret_access_key = "..."
bucket = "listmonk-media"
bucket_path = ""  # ⚠️ 必须为空！否则代理机制无法工作
bucket_type = "private"
public_url = "/s3-media"  # 相对路径，会自动拼接 app.root_url
expiry = "14d"
```

### 7.3 后端切换的无缝性（修正版）

#### 媒体元数据查询：无缝切换

由于 Listmonk 的设计，**媒体元数据查询**的切换非常简单：

1. **不需要修改数据库**：`media` 表的 `filename` 字段只存储相对路径
2. **只需要修改配置**：修改 `upload.provider` 和对应的后端配置
3. **URL 动态生成**：所有媒体查询都会通过当前配置的后端的 `GetURL` 方法生成 URL

**适用场景**：
- 媒体库列表查询
- 媒体详情查询
- 新上传媒体的 URL 生成

#### 邮件正文内联图片：不无缝切换

**⚠️ 重要修正**：对于已保存的邮件正文，切换存储后端**不无缝**：

1. **历史数据问题**：`campaigns.body` 和 `templates.body` 中保存的是**完整 URL**
2. **无重写机制**：发送时不会重写这些 URL
3. **需要手动迁移**：如果旧存储不可用，历史链接会失效

#### 切换检查表

| 检查项 | 说明 |
|--------|------|
| 媒体文件迁移 | 需要手动将文件从旧存储复制到新存储 |
| 历史 Campaign 正文 | 需要手动更新或接受旧 URL（如果旧存储仍可用） |
| 历史 Template 正文 | 同上 |
| 新 Campaign | 自动使用新存储的 URL |
| 媒体库 API | 自动使用新存储的 URL |

---

## 8. URL 生成机制详解

### 8.1 设计思想：相对路径存储 + 动态 URL 生成（适用场景）

Listmonk 的核心设计思想是：**数据库永远只存储相对路径（文件名），URL 在需要访问时才动态生成**。

**⚠️ 适用范围**：
- ✅ `media` 表的查询（媒体库）
- ✅ 新上传媒体的 API 响应
- ❌ `campaigns.body` 和 `templates.body`（已写入完整 URL）

### 8.2 URL 生成的触发时机

#### 1. 查询媒体列表时

**文件位置**: `internal/core/media.go:17-44`

```go
func (c *Core) QueryMedia(provider string, s media.Store, query string, offset, limit int) ([]media.Media, int, error) {
    // ... 数据库查询 ...
    
    for i := 0; i < len(out); i++ {
        // 动态生成 URL
        out[i].URL = s.GetURL(out[i].Filename)
        
        if out[i].Thumb != "" {
            out[i].ThumbURL = null.String{Valid: true, String: s.GetURL(out[i].Thumb)}
        }
    }
    
    return out, total, nil
}
```

#### 2. 查询单个媒体时

**文件位置**: `internal/core/media.go:47-70`

```go
func (c *Core) GetMedia(id int, uuid, fileName string, s media.Store) (media.Media, error) {
    // ... 数据库查询 ...
    
    // 动态生成 URL
    out.URL = s.GetURL(out.Filename)
    if out.Thumb != "" {
        out.ThumbURL = null.String{Valid: true, String: s.GetURL(out.Thumb)}
    }
    
    return out, nil
}
```

#### 3. 插入新媒体后返回时

**文件位置**: `internal/core/media.go:73-90`

```go
func (c *Core) InsertMedia(fileName, thumbName, contentType string, meta models.JSON, provider string, s media.Store) (media.Media, error) {
    // ... 写入数据库 ...
    
    // 通过 GetMedia 获取包含动态 URL 的对象
    return c.GetMedia(newID, "", "", s)
}
```

### 8.3 S3 的特殊 URL 重写逻辑

S3 存储的 `makeFileURL` 方法实现了灵活的 URL 重写：

```
┌─────────────────────────────────────────────────────────────────┐
│                      makeFileURL 执行流程                         │
├─────────────────────────────────────────────────────────────────┤
│  1. 检查是否配置了 public_url                                      │
│     ├── 是 → 进入重写逻辑                                          │
│     │     ├── 检查 public_url 是否以 "/" 开头                      │
│     │     │     ├── 是 → 相对路径，需要拼接 RootURL               │
│     │     │     │     示例: "/s3-media" → "{RootURL}/s3-media"  │
│     │     │     └── 否 → 绝对 URL，直接使用                        │
│     │     │           示例: "https://cdn.example.com"             │
│     │     └── 拼接 bucket_path 和 filename                         │
│     └── 否 → 使用默认 S3 URL 格式                                   │
│           格式: {s3_url}/{bucket}/{bucket_path}/{filename}        │
└─────────────────────────────────────────────────────────────────┘
```

### 8.4 相对路径 public_url 的工作原理

当 S3 的 `public_url` 配置为相对路径（如 `/s3-media`）时：

1. **URL 生成时**：`makeFileURL` 会自动将 `RootURL` 拼接到前面
2. **HTTP 服务端**：Listmonk 会注册一个路由来代理这些请求

**服务端路由注册**（`cmd/init.go:954-964`）：

```go
var (
    uploadProvider = ko.String("upload.provider")
    uploadFsURI    = ko.String("upload.filesystem.upload_uri")
    publicURL      = ko.String("upload.s3.public_url")
)
switch {
case uploadProvider == "filesystem" && uploadFsURI != "":
    // 文件系统：直接静态服务
    srv.Static(uploadFsURI, ko.String("upload.filesystem.upload_path"))
case uploadProvider == "s3" && strings.HasPrefix(publicURL, "/"):
    // S3 相对路径：注册代理路由
    srv.GET(path.Join(publicURL, "/:filepath"), app.ServeS3Media)
}
```

**S3 媒体代理处理器**（`cmd/media.go:195-208`）：

```go
func (a *App) ServeS3Media(c echo.Context) error {
    key := c.Param("filepath")
    if key == "" {
        return echo.NewHTTPError(http.StatusBadRequest, "missing media file path")
    }

    // 从 S3 获取文件内容
    b, err := a.media.GetBlob(key)
    if err != nil {
        a.log.Printf("error fetching media from s3 %s: %v", key, err)
        return echo.NewHTTPError(http.StatusInternalServerError, "error fetching media")
    }

    // 流式返回给客户端
    return c.Stream(http.StatusOK, http.DetectContentType(b), bytes.NewReader(b))
}
```

---

## 9. 邮件正文里的图片 URL 处理（二次核查总结）

### 9.1 媒体附件的处理

邮件发送时，媒体附件通过 `attachMedia` 方法加载：

**文件位置**: `internal/manager/manager.go:663-679`

```go
func (m *Manager) attachMedia(c *models.Campaign) error {
    if len(c.Attachments) > 0 {
        return nil
    }

    // 加载媒体/附件
    for _, mid := range []int64(c.MediaIDs) {
        // 从 store 获取附件（包含从存储后端读取字节）
        a, err := m.store.GetAttachment(int(mid))
        if err != nil {
            return fmt.Errorf("error fetching attachment %d on campaign %s: %v", mid, c.Name, err)
        }

        c.Attachments = append(c.Attachments, a)
    }

    return nil
}
```

**获取附件的实现**（`cmd/manager_store.go:89-105`）：

```go
func (s *store) GetAttachment(mediaID int) (models.Attachment, error) {
    // 1. 获取媒体元数据（包含动态生成的 URL）
    m, err := s.core.GetMedia(mediaID, "", "", s.media)
    if err != nil {
        return models.Attachment{}, err
    }

    // 2. 通过 GetBlob 获取文件内容
    b, err := s.media.GetBlob(m.URL)
    if err != nil {
        return models.Attachment{}, err
    }

    return models.Attachment{
        Name:    m.Filename,
        Content: b,
        Header:  manager.MakeAttachmentHeader(m.Filename, "base64", m.ContentType),
    }, nil
}
```

**关键区别**：
- 附件通过 `MediaIDs` 关联，发送时**动态获取**文件内容
- 附件内容**不依赖 URL**，直接通过 `GetBlob` 读取
- 切换存储后端后，**附件仍然可以正常工作**（只要文件已迁移）

### 9.2 邮件正文内联图片 vs 附件：处理差异

| 特性 | 邮件正文内联图片 | 媒体附件 |
|------|-----------------|----------|
| 存储位置 | `campaigns.body`（HTML 字符串） | `campaign_media` 关联表 |
| URL 写入时机 | 编辑阶段（完整 URL） | 发送阶段动态生成 |
| 切换存储后 | 历史 URL 不自动更新 | 自动使用新存储 |
| 依赖旧存储 | 是（如果 URL 指向旧存储） | 否（只要文件已迁移） |
| 处理方式 | 直接使用保存的 URL | 通过 `GetBlob` 读取内容 |

### 9.3 邮件正文内联图片的 URL 生成流程图

```
┌──────────────────────────────────────────────────────────────────────┐
│              邮件正文内联图片 URL 处理流程                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  【编辑阶段】                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  1. 用户在富文本编辑器中点击 "插入图片"                            │  │
│  │         ↓                                                         │  │
│  │  2. 前端调用媒体库 API: GET /api/media                            │  │
│  │         ↓                                                         │  │
│  │  3. 后端 QueryMedia() 执行:                                       │  │
│  │     - 从数据库查询 media 记录（只有 filename）                     │  │
│  │     - 调用 s.GetURL(filename) 生成完整 URL                        │  │
│  │     - 返回包含 URL 的媒体列表                                      │  │
│  │         ↓                                                         │  │
│  │  4. 用户选择图片，onMediaSelect(media) 被调用                     │  │
│  │         ↓                                                         │  │
│  │  5. 前端执行: this.imageCallack(media.url)                        │  │
│  │         ↓                                                         │  │
│  │  6. 编辑器插入: <img src="https://完整URL/image.jpg">             │  │
│  │         ↓                                                         │  │
│  │  7. 用户保存 Campaign                                              │  │
│  │         ↓                                                         │  │
│  │  8. 完整 HTML 写入 campaigns.body 字段                             │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                              │                                          │
│                              ▼                                          │
│  【发送阶段】                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  1. Campaign 状态变为 "running"                                   │  │
│  │         ↓                                                         │  │
│  │  2. pipe.newMessage() 创建邮件消息                                 │  │
│  │         ↓                                                         │  │
│  │  3. manager.NewCampaignMessage() 执行:                            │  │
│  │     - 调用 msg.render()                                            │  │
│  │     - 执行 Go 模板语法: {{ TrackLink }}, {{ .Subscriber }} 等    │  │
│  │     - ⚠️ 不会扫描或修改 <img src=""> 中的 URL                     │  │
│  │         ↓                                                         │  │
│  │  4. msg.body = 渲染后的 HTML（包含原始 URL）                       │  │
│  │         ↓                                                         │  │
│  │  5. 构造外发消息:                                                  │  │
│  │     out := models.Message{                                        │  │
│  │         Body: msg.body,  // 直接使用，无 URL 重写                 │  │
│  │         ...                                                       │  │
│  │     }                                                              │  │
│  │         ↓                                                         │  │
│  │  6. 邮件发送: <img src="编辑阶段写入的完整URL">                    │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 10. 配置选项详解

### 10.1 文件系统存储配置

| 配置项 | 类型 | 说明 | 示例 |
|-------|------|------|------|
| `upload.provider` | string | 存储后端类型 | `"filesystem"` |
| `upload.extensions` | array | 允许的文件扩展名 | `["jpg", "png", "gif"]` |
| `upload.filesystem.upload_path` | string | 本地存储路径 | `"/home/listmonk/uploads"` |
| `upload.filesystem.upload_uri` | string | 访问 URI 前缀 | `"/uploads"` |
| `app.root_url` | string | 应用根 URL | `"https://listmonk.example.com"` |

### 10.2 S3 存储配置

| 配置项 | 类型 | 说明 | 示例 |
|-------|------|------|------|
| `upload.provider` | string | 存储后端类型 | `"s3"` |
| `upload.s3.aws_default_region` | string | AWS 区域 | `"ap-south-1"` |
| `upload.s3.aws_access_key_id` | string | AWS 访问密钥 ID | `"AKIA..."` |
| `upload.s3.aws_secret_access_key` | string | AWS 密钥 | `"..."` |
| `upload.s3.bucket` | string | Bucket 名称 | `"listmonk-media"` |
| `upload.s3.bucket_path` | string | Bucket 内路径前缀 | `"/assets"` ⚠️ |
| `upload.s3.bucket_type` | string | Bucket 类型 | `"public"` 或 `"private"` |
| `upload.s3.url` | string | S3 端点 URL | `"https://s3.region.amazonaws.com"` |
| `upload.s3.public_url` | string | 公共访问 URL（可相对路径） | `"https://cdn.example.com"` 或 `"/s3-media"` |
| `upload.s3.expiry` | string | 预签名 URL 过期时间 | `"14d"` |
| `app.root_url` | string | 应用根 URL（用于相对路径 public_url） | `"https://listmonk.example.com"` |

**⚠️ 注意**: `bucket_path` 与相对路径 `public_url` 组合使用时存在设计缺陷，详见第 5 节。

---

## 11. 关键代码位置索引

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 媒体存储接口定义 | `internal/media/media.go` | 26-32 |
| 媒体数据模型 | `internal/media/media.go` | 11-24 |
| 文件系统存储实现 | `internal/media/providers/filesystem/filesystem.go` | 全部 |
| S3 存储实现 | `internal/media/providers/s3/s3.go` | 全部 |
| S3 URL 生成逻辑 | `internal/media/providers/s3/s3.go` | 91-171 |
| S3 GetBlob 方法 | `internal/media/providers/s3/s3.go` | 111-136 |
| 媒体上传处理 | `cmd/media.go` | 26-140 |
| S3 媒体代理服务 | `cmd/media.go` | 195-208 |
| 存储后端初始化 | `cmd/init.go` | 750-783 |
| 静态路由注册 | `cmd/init.go` | 954-964 |
| 媒体查询（含 URL 生成） | `internal/core/media.go` | 17-70 |
| 邮件消息渲染 | `internal/manager/message.go` | 33-88 |
| 邮件附件加载 | `internal/manager/manager.go` | 663-679 |
| 富文本编辑器媒体选择 | `frontend/src/components/RichtextEditor.vue` | 329-331 |

---

## 12. 设计亮点与限制（二次核查更新）

### 12.1 设计亮点

1. **接口抽象**：`media.Store` 接口完美地隔离了存储实现细节
2. **相对路径存储**：`media` 表只存储文件名，实现了存储后端的完全解耦
3. **动态 URL 生成**：URL 在访问时生成，确保媒体库查询总是使用当前配置
4. **灵活的 S3 URL 重写**：支持 CDN 域名、相对路径代理、预签名 URL 等多种模式
5. **统一的错误处理**：所有存储操作的错误处理一致

### 12.2 设计限制与注意事项

#### 限制 1：邮件正文 URL 不重写

- **问题**：邮件正文内联图片的 URL 在编辑阶段就已写入，发送时不会重写
- **影响**：切换存储后端后，历史 Campaign 的图片链接不会自动更新
- **建议**：
  - 切换存储前评估历史 Campaign 的重要性
  - 如果需要保留历史链接，考虑：
    - 保持旧存储可用（作为只读）
    - 或使用 CDN 作为中间层（URL 不变，后端切换）
    - 或手动迁移（数据库更新 + 文件迁移）

#### 限制 2：S3 相对路径代理不支持目录层级

- **问题**：
  - 路由使用单段参数 `:filepath`，无法匹配多级路径
  - `GetBlob` 使用 `filepath.Base()` 丢弃目录层级
- **影响**：
  - `bucket_path` 不为空时，相对路径 `public_url` 无法正确工作
  - 带目录的对象键会导致 404 错误
- **建议**：
  - 使用相对路径 `public_url` 时，`bucket_path` 必须为空
  - 需要 `bucket_path` 时，使用绝对路径 `public_url`（CDN）或预签名 URL

#### 限制 3：媒体文件需要手动迁移

- **问题**：切换存储后端时，Listmonk 不会自动迁移媒体文件
- **影响**：
  - 新上传的文件会写入新存储
  - 历史文件仍在旧存储
- **建议**：
  - 切换前规划迁移策略
  - 文件迁移工具：
    - 对于小批量：手动下载上传
    - 对于大批量：使用云服务商的迁移工具（如 AWS DataSync）
    - 或编写脚本：遍历 `media` 表，从旧存储 `GetBlob`，到新存储 `Put`

### 12.3 最佳实践参考

#### 存储后端选择建议

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| 小型部署、单服务器 | filesystem | 简单、无额外成本 |
| 多实例部署、高可用 | S3 + CDN | 共享存储、横向扩展 |
| 隐私敏感数据 | S3 私有 bucket + 预签名 URL | 安全、可控的访问 |
| 需要历史链接永久有效 | CDN 作为中间层 | URL 不变，后端可切换 |

#### 切换存储后端的检查清单

```
□ 1. 备份数据库（特别是 campaigns 表和 templates 表）
□ 2. 备份所有媒体文件
□ 3. 在测试环境验证新存储配置
□ 4. 评估历史 Campaign 的处理方式：
   □ 保持旧存储可用（只读）
   □ 或使用 CDN 中间层
   □ 或手动更新数据库中的 URL
□ 5. 规划文件迁移：
   □ 小批量：手动迁移
   □ 大批量：使用工具或脚本
□ 6. 更新配置并重启
□ 7. 验证：
   □ 新媒体上传正常
   □ 媒体库查询正常
   □ 历史 Campaign（根据策略）正常
□ 8. 监控一段时间，确认无问题
```

---

## 13. 二次核查核心结论总结

### 13.1 邮件正文内联图片 URL 处理

| 问题 | 答案 | 证据位置 |
|------|------|----------|
| URL 在哪个阶段写入？ | **编辑阶段** | `RichtextEditor.vue:329-331` |
| 写入的是相对路径还是完整 URL？ | **完整 URL** | `internal/core/media.go:35,64` |
| 发送阶段是否重写？ | **否** | `internal/manager/message.go:33-88` |
| 切换存储后历史链接是否自动迁移？ | **否** | （无重写逻辑） |

### 13.2 S3 相对路径 public_url 代理的行为边界

| 配置场景 | 是否支持 | 问题描述 |
|----------|----------|----------|
| `bucket_path = ""` + 相对路径 `public_url` | ✅ 支持 | 正常工作 |
| `bucket_path` 不为空 + 相对路径 `public_url` | ❌ **不支持** | 路由参数限制 + `filepath.Base` 处理导致 404 |
| `bucket_path` 不为空 + 绝对路径 `public_url` | ✅ 支持 | CDN 直接访问 S3，无代理问题 |
| `bucket_path` 不为空 + 私有 bucket + 预签名 URL | ✅ 支持 | URL 直接指向 S3，无代理问题 |

### 13.3 存储后端切换的实际影响

| 组件 | 切换后的行为 |
|------|-------------|
| 媒体库（API 查询） | ✅ 自动使用新存储的 URL |
| 新上传媒体 | ✅ 自动使用新存储 |
| 媒体附件（通过 MediaIDs） | ✅ 自动使用新存储（只要文件已迁移） |
| 历史 Campaign 正文内联图片 | ⚠️ 使用旧 URL（如果旧存储不可用则失效） |
| 历史 Template 正文内联图片 | ⚠️ 使用旧 URL（如果旧存储不可用则失效） |

---

## 14. 架构总结（修正版）

### 14.1 核心设计模式

Listmonk 采用了以下设计模式来实现跨存储后端的适配：

1. **接口抽象（Store Interface）**：统一的 `media.Store` 接口定义了所有存储操作
2. **依赖注入（Dependency Injection）**：`media.Store` 实例在初始化时创建，然后注入到需要使用的组件中
3. **策略模式（Strategy Pattern）**：不同的存储后端实现相同的接口，可以在运行时切换
4. **延迟计算（Lazy Evaluation）**：URL 不在存储时生成，而是在访问时动态计算（仅适用于媒体元数据查询）

### 14.2 组件关系图

```
┌──────────────────────────────────────────────────────────────────────┐
│                           应用层 (Application)                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │  HTTP Handlers│  │ Campaign     │  │ Manager (邮件发送)        │  │
│  │  (media.go)  │  │ Core         │  │                          │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────────┘  │
│         │                 │                      │                   │
│         │  媒体库查询      │  正文存储             │  邮件发送         │
│         │  (动态URL)       │  (静态URL)            │  (无重写)         │
│         ▼                 ▼                      ▼                   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                         数据存储层                              │   │
│  │  ┌─────────────────┐  ┌───────────────────────────────────┐   │   │
│  │  │   media 表      │  │  campaigns 表 / templates 表       │   │   │
│  │  │  - filename     │  │  - body (包含完整 URL)             │   │   │
│  │  │  - thumb        │  │  - body_source                      │   │   │
│  │  │  (相对路径)      │  │  (静态 URL，编辑阶段写入)           │   │   │
│  │  └─────────────────┘  └───────────────────────────────────┘   │   │
│  └──────────────────────────┬─────────────────────────────────────┘   │
│                             │                                          │
│                             ▼                                          │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                   media.Store 接口                                 │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │ │
│  │  │ Put()   │  │ Delete()│  │ GetURL()│  │GetBlob()│           │ │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘           │ │
│  └──────────────────────────┬───────────────────────────────────────┘ │
│                             │                                          │
│           ┌─────────────────┼─────────────────┐                      │
│           │                 │                 │                      │
│           ▼                 ▼                 ▼                      │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐          │
│  │  filesystem    │ │      S3        │ │  (未来可扩展)   │          │
│  │   Provider     │ │   Provider     │ │                │          │
│  └────────────────┘ └────────────────┘ └────────────────┘          │
└──────────────────────────────────────────────────────────────────────┘
```

### 14.3 两种 URL 处理策略的对比

| 策略 | 适用场景 | URL 存储 | 生成时机 | 切换存储后 |
|------|----------|----------|----------|------------|
| **动态 URL** | 媒体元数据查询 | 相对路径（filename） | 访问时动态生成 | ✅ 自动更新 |
| **静态 URL** | 邮件正文、模板正文 | 完整 URL（包含域名） | 编辑阶段写入 | ❌ 不更新 |

---

## 15. 故障排查指南

### 15.1 切换存储后图片无法显示

**症状**：
- 新媒体上传正常
- 媒体库查询正常
- 历史 Campaign 邮件中的图片无法显示

**可能原因**：
1. 邮件正文保存的是旧存储的完整 URL
2. 旧存储已删除或不可用
3. 旧存储的路径配置已更改

**排查步骤**：
```
1. 查看数据库中 campaigns.body 的内容：
   SELECT body FROM campaigns WHERE id = <历史 campaign ID>;
   
2. 检查 <img src=""> 中的 URL：
   - 是否指向旧存储？
   - 旧存储是否仍可访问？

3. 验证当前配置：
   - 当前 upload.provider 是什么？
   - 旧配置是什么？
```

**解决方案**：
| 方案 | 适用场景 | 操作 |
|------|----------|------|
| 恢复旧存储 | 旧存储数据仍在 | 恢复旧存储的访问（作为只读） |
| CDN 中间层 | 规划阶段 | 使用 CDN，URL 不变，后端可切换 |
| 手动迁移 | 必须更新 URL | 编写脚本更新数据库中的 URL + 迁移文件 |
| 忽略历史 | 历史 Campaign 不重要 | 不处理，仅确保新 Campaign 正常 |

### 15.2 S3 相对路径代理返回 404

**症状**：
- `public_url` 配置为相对路径（如 `/s3-media`）
- 媒体库显示正确的 URL
- 访问图片时返回 404

**可能原因**：
1. `bucket_path` 不为空（设计缺陷）
2. 路由参数匹配问题
3. `GetBlob` 中的 `filepath.Base` 处理

**排查步骤**：
```
1. 检查配置：
   [upload.s3]
   bucket_path = ?  # 是否为空？
   public_url = "/s3-media"

2. 查看应用日志：
   # 是否有 "error fetching media from s3" 错误？
   # 错误日志中的 key 是什么？

3. 验证 S3 对象键：
   - 实际存储的对象键是什么？
   - 代码尝试访问的对象键是什么？
```

**解决方案**：

**方案 1（推荐）**：`bucket_path` 设为空
```toml
[upload.s3]
bucket_path = ""  # 必须为空
public_url = "/s3-media"
```

**方案 2**：使用绝对路径 `public_url`（CDN）
```toml
[upload.s3]
bucket_path = "assets/images"  # 可以不为空
public_url = "https://cdn.example.com/media"  # 绝对路径，CDN 负责代理
```

**方案 3**：使用私有 bucket + 预签名 URL
```toml
[upload.s3]
bucket_path = "assets/images"  # 可以不为空
bucket_type = "private"
public_url = ""  # 不使用代理，使用预签名 URL
expiry = "14d"
```

### 15.3 媒体上传失败

**症状**：
- 上传文件时返回错误
- 或上传成功但无法访问

**排查步骤**：
```
1. 检查存储后端配置：
   - filesystem: upload_path 是否存在？权限是否正确？
   - S3: 凭证是否有效？Bucket 是否存在？权限是否正确？

2. 查看应用日志：
   - 是否有 "error uploading file" 错误？
   - 是否有存储后端的具体错误信息？

3. 验证配置：
   - upload.extensions 是否包含文件扩展名？
   - 文件大小是否超出限制？
```

---

## 16. 附录：数据库表结构

### 16.1 media 表（媒体元数据）

```sql
-- 核心字段（简化版）
CREATE TABLE media (
    id          SERIAL PRIMARY KEY,
    uuid        UUID NOT NULL UNIQUE,
    filename    TEXT NOT NULL,        -- ⚠️ 只存储相对路径/文件名
    content_type TEXT NOT NULL,
    thumb       TEXT,                 -- 缩略图文件名（相对路径）
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    provider    TEXT NOT NULL,        -- "filesystem" 或 "s3"
    meta        JSONB DEFAULT '{}'    -- 元数据（如图片宽高）
);
```

### 16