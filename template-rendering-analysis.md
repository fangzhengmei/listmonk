# Listmonk 模板渲染机制分析报告

## 一、概述

Listmonk 使用 Go 语言标准库的 `html/template` 和 `text/template` 包实现邮件模板渲染系统。该系统支持：

- 订阅者个性化字段替换
- 活动（Campaign）元数据访问
- 强大的模板函数（内置 + Sprig 库）
- 简化的模板语法预处理
- Markdown 内容自动转换

## 二、核心数据结构

### 2.1 Campaign（活动）模型

**文件位置**: `models/campaigns.go:38-83`

```go
type Campaign struct {
    Base
    CampaignMeta

    UUID        string          // 活动唯一标识
    Name        string          // 活动内部名称
    Subject     string          // 邮件主题（支持模板表达式）
    FromEmail   string          // 发件人邮箱
    Body        string          // 邮件主体内容
    AltBody     null.String     // 纯文本备选内容
    ContentType string          // 内容类型 (richtext/html/markdown/plain/visual)
    TemplateID  null.Int        // 关联的模板ID

    // 预编译的模板
    Tpl         *template.Template       // 主体模板
    SubjectTpl  *txttpl.Template         // 主题模板
    AltBodyTpl  *template.Template       // 纯文本模板
    HeaderTpls  []map[string]*txttpl.Template // 自定义邮件头模板
}
```

### 2.2 Subscriber（订阅者）模型

**文件位置**: `models/subscribers.go:28-37`

```go
type Subscriber struct {
    Base
    UUID    string         // 订阅者唯一标识
    Email   string         // 邮箱地址
    Name    string         // 姓名
    Attribs JSON           // 自定义属性（任意 JSON 对象）
    Status  string         // 状态 (enabled/disabled/blocklisted)
    Lists   types.JSONText // 所属邮件列表
}
```

**订阅者辅助方法**:
- `FirstName()`: 从姓名字段中提取名
- `LastName()`: 从姓名字段中提取姓

### 2.3 CampaignMessage（活动消息）

**文件位置**: `internal/manager/manager.go:99-112`

```go
type CampaignMessage struct {
    Campaign   *models.Campaign   // 关联的活动
    Subscriber models.Subscriber  // 关联的订阅者

    from     string
    to       string
    subject  string              // 渲染后的主题
    body     []byte              // 渲染后的主体
    altBody  []byte              // 渲染后的纯文本
    unsubURL string              // 退订链接
    headers  models.Headers      // 邮件头
}
```

## 三、模板编译流程

### 3.1 编译触发时机

模板编译在以下场景触发：

1. **活动开始发送时**: `internal/manager/pipe.go:35`
   ```go
   // newPipe 函数中
   if err := c.CompileTemplate(m.TemplateFuncs(c)); err != nil {
       return nil, err
   }
   ```

2. **预览活动时**: `cmd/campaigns.go:176`
   ```go
   if err := camp.CompileTemplate(a.manager.TemplateFuncs(&camp)); err != nil {
       // ...
   }
   ```

### 3.2 编译过程详解

**文件位置**: `models/campaigns.go:141-242`

```go
func (c *Campaign) CompileTemplate(f template.FuncMap) error {
    // 1. 编译主题模板（如果包含模板表达式）
    if hasTplExpr(c.Subject) {
        // 预处理：替换简化语法
        subj := c.Subject
        for _, r := range regTplFuncs {
            subj = r.regExp.ReplaceAllString(subj, r.replace)
        }
        // 编译主题
        subjTpl, err := txttpl.New(ContentTpl).Funcs(txtFuncs).Parse(subj)
        // ...
        c.SubjectTpl = subjTpl
    }

    // 2. 编译基础模板 (base template)
    body := c.TemplateBody  // 从关联模板获取
    if body == "" || c.ContentType == CampaignContentTypeVisual {
        body = `{{ template "content" . }}`
    }
    // 预处理
    for _, r := range regTplFuncs {
        body = r.regExp.ReplaceAllString(body, r.replace)
    }
    baseTPL, err := template.New(BaseTpl).Funcs(f).Parse(body)

    // 3. 处理 Markdown 格式
    if c.ContentType == CampaignContentTypeMarkdown {
        // 转换 Markdown 为 HTML
        var b bytes.Buffer
        if err := markdown.Convert([]byte(c.Body), &b); err != nil {
            return err
        }
        body = b.String()
    } else {
        body = c.Body
    }

    // 4. 编译内容模板 (content template)
    for _, r := range regTplFuncs {
        body = r.regExp.ReplaceAllString(body, r.replace)
    }
    msgTpl, err := template.New(ContentTpl).Funcs(f).Parse(body)

    // 5. 合并 base 和 content 模板
    out, err := baseTPL.AddParseTree(ContentTpl, msgTpl.Tree)
    c.Tpl = out

    // 6. 编译纯文本备选内容
    if hasTplExpr(c.AltBody.String) {
        // ... 编译 AltBodyTpl
    }

    // 7. 编译自定义邮件头（如果包含模板表达式）
    for _, set := range c.Headers {
        for _, val := range set {
            if hasTplExpr(val) {
                // 编译 HeaderTpls
            }
        }
    }
}
```

### 3.3 模板语法预处理

**文件位置**: `models/common.go:35-66`

Listmonk 使用正则表达式对用户编写的简化模板语法进行预处理，转换为标准 Go 模板语法：

| 原始语法 | 转换后语法 | 说明 |
|---------|-----------|------|
| `{{ TrackLink "http://link.com" }}` | `{{ TrackLink "http://link.com" . }}` | 自动添加上下文参数 |
| `https://link.com@TrackLink` | `{{ TrackLink "https://link.com" . }}` | WYSIWYG 编辑器友好格式 |
| `{{ TrackView }}` | `{{ TrackView . }}` | 自动添加上下文参数 |
| `{{ UnsubscribeURL }}` | `{{ UnsubscribeURL . }}` | 自动添加上下文参数 |
| `{{ ManageURL }}` | `{{ ManageURL . }}` | 自动添加上下文参数 |
| `{{ OptinURL }}` | `{{ OptinURL . }}` | 自动添加上下文参数 |
| `{{ MessageURL }}` | `{{ MessageURL . }}` | 自动添加上下文参数 |

**预处理正则表达式定义** (`models/common.go:43-66`):
```go
var regTplFuncs = []regTplFunc{
    // TrackLink 无参数形式
    {
        regExp:  regexp.MustCompile(`{{\s*TrackLink\s+"([^"]+)"\s*}}`),
        replace: `{{ TrackLink "$1" . }}`,
    },
    // URL@TrackLink 简化形式
    {
        regExp:  regexp.MustCompile(`(https?://[\p{L}\p{N}_\-\.~!#$&'()*+,/:;=?@\[\]%]*)@TrackLink`),
        replace: `{{ TrackLink "$1" . }}`,
    },
    // 其他无参函数
    {
        regExp:  regexp.MustCompile(`{{(\s+)?(TrackView|UnsubscribeURL|ManageURL|OptinURL|MessageURL)(\s+)?}}`),
        replace: `{{ $2 . }}`,
    },
}
```

## 四、模板渲染流程

### 4.1 渲染入口

**文件位置**: `internal/manager/message.go:13-29`

```go
func (m *Manager) NewCampaignMessage(c *models.Campaign, s models.Subscriber) (CampaignMessage, error) {
    msg := CampaignMessage{
        Campaign:   c,
        Subscriber: s,
        subject:    c.Subject,
        from:       c.FromEmail,
        to:         s.Email,
        unsubURL:   fmt.Sprintf(m.cfg.UnsubURL, c.UUID, s.UUID),
    }

    if err := msg.render(); err != nil {
        return msg, err
    }
    return msg, nil
}
```

### 4.2 渲染过程详解

**文件位置**: `internal/manager/message.go:33-88`

```go
func (m *CampaignMessage) render() error {
    out := bytes.Buffer{}

    // 1. 渲染主题
    if m.Campaign.SubjectTpl != nil {
        if err := m.Campaign.SubjectTpl.ExecuteTemplate(&out, models.ContentTpl, m); err != nil {
            return err
        }
        m.subject = out.String()
        out.Reset()
    }

    // 2. 渲染主体内容（核心步骤）
    if err := m.Campaign.Tpl.ExecuteTemplate(&out, models.BaseTpl, m); err != nil {
        return err
    }
    m.body = out.Bytes()

    // 3. 渲染纯文本备选内容
    if m.Campaign.ContentType != models.CampaignContentTypePlain && m.Campaign.AltBody.Valid {
        if m.Campaign.AltBodyTpl != nil {
            b := bytes.Buffer{}
            if err := m.Campaign.AltBodyTpl.ExecuteTemplate(&b, models.ContentTpl, m); err != nil {
                return err
            }
            m.altBody = b.Bytes()
        } else {
            m.altBody = []byte(m.Campaign.AltBody.String)
        }
    }

    // 4. 渲染自定义邮件头
    if m.Campaign.HeaderTpls != nil {
        hdrOut := bytes.Buffer{}
        m.headers = make(models.Headers, len(m.Campaign.Headers))
        for i, set := range m.Campaign.Headers {
            m.headers[i] = make(map[string]string, len(set))
            for hdr, val := range set {
                tpl := m.Campaign.HeaderTpls[i][hdr]
                if tpl == nil {
                    m.headers[i][hdr] = val
                    continue
                }
                hdrOut.Reset()
                if err := tpl.ExecuteTemplate(&hdrOut, models.ContentTpl, m); err != nil {
                    return fmt.Errorf("error rendering header %q: %v", hdr, err)
                }
                m.headers[i][hdr] = hdrOut.String()
            }
        }
    } else {
        m.headers = m.Campaign.Headers
    }

    return nil
}
```

### 4.3 渲染上下文对象

模板渲染时，`CampaignMessage` 对象作为上下文（`.`）传递给模板。模板中可访问的字段：

| 表达式 | 类型 | 说明 |
|--------|------|------|
| `.` | `CampaignMessage` | 完整的消息对象 |
| `.Campaign` | `*Campaign` | 活动元数据 |
| `.Subscriber` | `Subscriber` | 订阅者数据 |

**订阅者个性化字段** (`.Subscriber`):
```go
// 标准字段
.Subscriber.UUID       // string - 订阅者唯一标识
.Subscriber.Email      // string - 邮箱地址
.Subscriber.Name       // string - 完整姓名
.Subscriber.Status     // string - 状态
.Subscriber.CreatedAt  // time.Time - 创建时间
.Subscriber.UpdatedAt  // time.Time - 更新时间

// 辅助方法
.Subscriber.FirstName() // string - 名（自动提取）
.Subscriber.LastName()  // string - 姓（自动提取）

// 自定义属性
.Subscriber.Attribs     // map[string]any - 任意 JSON 数据
// 例如: .Subscriber.Attribs.city, .Subscriber.Attribs.age
```

**活动元数据** (`.Campaign`):
```go
.Campaign.UUID       // string - 活动唯一标识
.Campaign.Name       // string - 活动内部名称
.Campaign.Subject    // string - 邮件主题
.Campaign.FromEmail  // string - 发件人邮箱
.Campaign.Tags       // []string - 标签
.Campaign.CreatedAt  // time.Time - 创建时间
.Campaign.UpdatedAt  // time.Time - 更新时间
```

## 五、模板函数系统

### 5.1 函数注册

**文件位置**: `internal/manager/manager.go:347-403`

模板函数分为两部分：活动特定函数和通用函数。

```go
// 活动特定函数（每个活动动态创建）
func (m *Manager) TemplateFuncs(c *models.Campaign) template.FuncMap {
    f := template.FuncMap{
        "TrackLink": func(url string, msg *CampaignMessage) string { ... },
        "TrackView": func(msg *CampaignMessage) template.HTML { ... },
        "UnsubscribeURL": func(msg *CampaignMessage) string { ... },
        "ManageURL": func(msg *CampaignMessage) string { ... },
        "OptinURL": func(msg *CampaignMessage) string { ... },
        "MessageURL": func(msg *CampaignMessage) string { ... },
        "ArchiveURL": func() string { ... },
        "RootURL": func() string { ... },
    }
    // 合并通用函数
    maps.Copy(f, m.tplFuncs)
    return f
}
```

### 5.2 通用模板函数

**文件位置**: `internal/manager/manager.go:632-659`

```go
func (m *Manager) makeGnericFuncMap() template.FuncMap {
    funcs := template.FuncMap{
        "Date": func(layout string) string {
            if layout == "" {
                layout = time.ANSIC
            }
            return time.Now().Format(layout)
        },
        "L": func() *i18n.I18n {
            return m.i18n  // 国际化支持
        },
        "Safe": func(safeHTML string) template.HTML {
            return template.HTML(safeHTML)  // 输出原始 HTML
        },
    }

    // 合并 Sprig 函数库（100+ 函数）
    sprigFuncs := sprig.GenericFuncMap()
    // 移除危险函数
    delete(sprigFuncs, "env")
    delete(sprigFuncs, "expandenv")
    delete(sprigFuncs, "getHostByName")
    
    maps.Copy(funcs, sprigFuncs)
    return funcs
}
```

### 5.3 模板函数详细说明

#### 活动追踪函数

| 函数 | 签名 | 说明 |
|------|------|------|
| `TrackLink` | `TrackLink(url string, msg *CampaignMessage) string` | 生成可追踪的链接。替换原始 URL 为带追踪参数的 URL |
| `TrackView` | `TrackView(msg *CampaignMessage) template.HTML` | 生成 1x1 透明追踪像素图片 |

**TrackLink 实现** (`internal/manager/manager.go:349-360`):
```go
"TrackLink": func(url string, msg *CampaignMessage) string {
    if m.cfg.DisableTracking {
        return url  // 追踪禁用时返回原始 URL
    }
    subUUID := msg.Subscriber.UUID
    if !m.cfg.IndividualTracking {
        subUUID = dummyUUID  // 非个人追踪时使用占位 UUID
    }
    return m.trackLink(url, msg.Campaign.UUID, subUUID)
}
```

**TrackView 实现** (`internal/manager/manager.go:361-373`):
```go
"TrackView": func(msg *CampaignMessage) template.HTML {
    if m.cfg.DisableTracking {
        return template.HTML("")  // 追踪禁用时返回空
    }
    subUUID := msg.Subscriber.UUID
    if !m.cfg.IndividualTracking {
        subUUID = dummyUUID
    }
    return template.HTML(fmt.Sprintf(`<img src="%s" alt="" />`,
        fmt.Sprintf(m.cfg.ViewTrackURL, msg.Campaign.UUID, subUUID)))
}
```

#### URL 生成函数

| 函数 | 说明 | 示例输出 |
|------|------|----------|
| `UnsubscribeURL` | 退订链接 | `https://domain.com/subscription/camp-uuid/sub-uuid` |
| `ManageURL` | 订阅管理链接（退订链接 + `?manage=true`） | `https://domain.com/...?manage=true` |
| `OptinURL` | 确认订阅链接 | `https://domain.com/optin/sub-uuid` |
| `MessageURL` | 邮件在线查看链接 | `https://domain.com/campaign/camp-uuid/sub-uuid` |
| `ArchiveURL` | 归档页面 URL | `https://domain.com/archive` |
| `RootURL` | 站点根 URL | `https://domain.com` |

#### 通用工具函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `Date` | 输出当前日期时间 | `{{ Date "2006-01-02" }}` → `2026-05-05` |
| `Safe` | 输出原始 HTML（不转义） | `{{ Safe "<b>bold</b>" }}` |
| `L` | 访问国际化翻译对象 | `{{ .L.T "hello" }}` |

### 5.4 Sprig 函数库

Listmonk 集成了 Sprig 库，提供 100+ 实用函数。**注意**：出于安全考虑，以下函数已被移除：
- `env` - 环境变量访问
- `expandenv` - 环境变量替换
- `getHostByName` - DNS 查询

**常用 Sprig 函数分类**：

| 类别 | 示例函数 |
|------|----------|
| 字符串处理 | `trim`, `upper`, `lower`, `title`, `replace`, `split`, `join` |
| 列表操作 | `first`, `last`, `rest`, `reverse`, `sort`, `unique` |
| 字典操作 | `keys`, `values`, `hasKey`, `set`, `unset` |
| 类型转换 | `atoi`, `int64`, `float64`, `toString` |
| 日期时间 | `now`, `date`, `ago`, `duration` |
| 数学运算 | `add`, `sub`, `mul`, `div`, `mod`, `randInt` |
| 编码解码 | `b64enc`, `b64dec`, `sha256sum` |

**Sprig 使用示例**：
```go
// 字符串处理
{{ upper .Subscriber.Name }}
{{ trim .Subscriber.Email }}

// 条件判断
{{ if gt (len .Subscriber.Attribs) 0 }}有自定义属性{{ end }}

// 日期格式化
{{ date "2006-01-02" .Subscriber.CreatedAt }}

// 默认值
{{ default "N/A" .Subscriber.Attribs.phone }}
```

## 六、模板继承机制

### 6.1 双层模板结构

Listmonk 使用 Go 模板的 `{{ template "name" . }}` 机制实现模板继承：

```
┌─────────────────────────────────────┐
│  Base Template (基础模板)            │
│  - 固定头部、页脚、品牌元素            │
│  - {{ template "content" . }} 标记   │
├─────────────────────────────────────┤
│  Content Template (内容模板)          │
│  - 活动特定的邮件内容                  │
│  - 编译后注入到基础模板中              │
└─────────────────────────────────────┘
```

### 6.2 模板编译时的合并

**文件位置**: `models/campaigns.go:194-198`

```go
// 将 content 模板的解析树添加到 base 模板中
out, err := baseTPL.AddParseTree(ContentTpl, msgTpl.Tree)
if err != nil {
    return fmt.Errorf("error inserting child template: %v", err)
}
c.Tpl = out
```

### 6.3 模板定义规则

- **基础模板** 必须包含 `{{ template "content" . }}` 标记
- **视觉内容** (visual type) 不使用外部模板，自动使用 `{{ template "content" . }}`
- 多个模板通过 `AddParseTree` 合并为单个可执行模板

## 七、完整渲染流程示例

### 7.1 活动发送时的完整流程

```
1. 扫描活动 (scanCampaigns)
        ↓
2. 创建管道 (newPipe)
   ├── 验证 messenger
   ├── 编译模板 (CompileTemplate)
   │   ├── 预处理语法替换
   │   ├── 编译 base 模板
   │   ├── 编译 content 模板
   │   └── 合并模板
   └── 加载附件
        ↓
3. 批量获取订阅者 (NextSubscribers)
        ↓
4. 逐条渲染消息 (newMessage)
   ├── 创建 CampaignMessage
   │   ├── 绑定 Campaign
   │   └── 绑定 Subscriber
   ├── 执行渲染 (render)
   │   ├── 渲染 Subject
   │   ├── 渲染 Body (核心)
   │   ├── 渲染 AltBody
   │   └── 渲染 Headers
   └── 加入发送队列
        ↓
5. 发送消息 (worker)
```

### 7.2 预览流程

**文件位置**: `cmd/campaigns.go:137-196`

```go
func (a *App) PreviewCampaign(c echo.Context) error {
    // 1. 获取活动（用于预览）
    camp, err := a.core.GetCampaignForPreview(id, tplID)
    
    // 2. 使用虚拟数据防止追踪记录
    camp.UUID = dummySubscriber.UUID
    
    // 3. 编译模板
    if err := camp.CompileTemplate(a.manager.TemplateFuncs(&camp)); err != nil {
        // ...
    }
    
    // 4. 使用虚拟订阅者渲染
    msg, err := a.manager.NewCampaignMessage(&camp, dummySubscriber)
    
    // 5. 返回渲染结果
    return c.HTML(http.StatusOK, string(msg.Body()))
}
```

**虚拟订阅者定义** (`cmd/subscribers.go:50-55`):
```go
dummySubscriber = models.Subscriber{
    Email:   "demo@listmonk.app",
    Name:    "Demo Subscriber",
    UUID:    dummyUUID,  // "00000000-0000-0000-0000-000000000000"
    Attribs: models.JSON{"city": "Bengaluru"},
}
```

## 八、事务性消息模板

### 8.1 TxMessage 结构

**文件位置**: `models/messages.go:46-71`

```go
type TxMessage struct {
    SubscriberMode   string           // default/fallback/external
    SubscriberEmails []string         // 目标邮箱列表
    SubscriberIDs    []int            // 目标订阅者ID列表
    TemplateID       int              // 使用的模板ID
    Data             map[string]any   // 自定义数据
    FromEmail        string           // 发件人
    Subject          string           // 主题（支持模板）
    AltBody          string           // 纯文本备选
    Body             []byte           // 渲染后的主体
    Tpl              *template.Template
    SubjectTpl       *txttpl.Template
}
```

### 8.2 事务性消息渲染

**文件位置**: `models/messages.go:73-133`

```go
func (m *TxMessage) Render(sub Subscriber, tpl *Template, funcs txttpl.FuncMap) error {
    // 上下文对象
    data := struct {
        Subscriber Subscriber
        Tx         *TxMessage
    }{sub, m}

    // 渲染主体
    b := bytes.Buffer{}
    if err := tpl.Tpl.ExecuteTemplate(&b, BaseTpl, data); err != nil {
        return err
    }
    m.Body = b.Bytes()

    // 渲染主题
    if subjTpl != nil {
        if err := subjTpl.ExecuteTemplate(&b, BaseTpl, data); err != nil {
            return err
        }
        m.Subject = b.String()
    }
}
```

**事务性消息模板变量**:
| 表达式 | 说明 |
|--------|------|
| `.Subscriber` | 订阅者信息（同活动模板） |
| `.Tx.Data` | 自定义数据（通过 API 传入） |
| `.Tx.Subject` | 消息主题 |
| `.Tx.FromEmail` | 发件人邮箱 |

## 九、安全考虑

### 9.1 已移除的危险函数

在 Sprig 集成时，以下函数因安全风险被移除：

```go
// internal/manager/manager.go:650-654
sprigFuncs := sprig.GenericFuncMap()
delete(sprigFuncs, "env")         // 环境变量泄露风险
delete(sprigFuncs, "expandenv")   // 环境变量泄露风险
delete(sprigFuncs, "getHostByName") // DNS 探测风险
```

### 9.2 权限控制

文档提示 (`docs/docs/content/templating.md:7-9`):
> Sprig 模板功能强大且图灵完备，允许在模板中编程复杂行为。这意味着也可能编程出不良行为，例如通过循环连接大字符串来耗尽主机内存。确保仅将模板（活动、模板）权限授予受信任的用户。

### 9.3 预览隔离

预览功能使用虚拟 UUID 防止误追踪：
```go
// cmd/campaigns.go:173-175
// 使用虚拟活动 ID 防止 {{ TrackView }} 和 {{ TrackLink }} 在预览时注册视图和点击
camp.UUID = dummySubscriber.UUID
```

## 十、关键代码位置索引

| 功能模块 | 文件位置 | 关键行号 |
|----------|----------|----------|
| 活动模型定义 | `models/campaigns.go` | 38-83 |
| 活动模板编译 | `models/campaigns.go` | 141-242 |
| 订阅者模型 | `models/subscribers.go` | 28-37 |
| 模板语法预处理 | `models/common.go` | 35-66 |
| CampaignMessage 定义 | `internal/manager/manager.go` | 99-112 |
| 模板函数定义 | `internal/manager/manager.go` | 347-403 |
| 通用函数 + Sprig | `internal/manager/manager.go` | 632-659 |
| 消息创建入口 | `internal/manager/message.go` | 13-29 |
| 消息渲染实现 | `internal/manager/message.go` | 33-88 |
| 管道处理流程 | `internal/manager/pipe.go` | 27-70, 76-134 |
| 活动预览 API | `cmd/campaigns.go` | 137-196 |
| 虚拟订阅者 | `cmd/subscribers.go` | 50-55 |
| 事务性消息渲染 | `models/messages.go` | 73-133 |

---

**生成日期**: 2026-05-05  
**分析版本**: Listmonk (当前代码库版本)
