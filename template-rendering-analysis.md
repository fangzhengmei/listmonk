# Listmonk 模板渲染机制分析报告

## 一、概述

Listmonk 使用 Go 语言标准库的 `html/template` 和 `text/template` 包实现邮件模板渲染系统。该系统支持：

- 订阅者个性化字段替换
- 活动（Campaign）元数据访问
- 事务消息自定义数据注入
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
    Attribs     JSON            // 活动自定义属性

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

### 2.4 TxMessage（事务性消息）

**文件位置**: `models/messages.go:46-71`

```go
type TxMessage struct {
    SubscriberMode   string           // default/fallback/external
    SubscriberEmails []string         // 目标邮箱列表
    SubscriberIDs    []int            // 目标订阅者ID列表
    TemplateID       int              // 使用的模板ID
    Data             map[string]any   // 自定义数据（第三类数据源）
    FromEmail        string           // 发件人
    Subject          string           // 主题（支持模板）
    AltBody          string           // 纯文本备选
    Body             []byte           // 渲染后的主体
    Tpl              *template.Template
    SubjectTpl       *txttpl.Template
}
```

## 三、多数据源内容与注入顺序

### 3.1 三类数据源概览

Listmonk 模板渲染支持三类数据源，它们通过不同的命名空间访问，**不存在覆盖关系**：

| 数据源类型 | 数据来源 | 模板访问方式 | 适用场景 |
|-----------|----------|-------------|----------|
| **订阅者自定义字段** | 数据库 `subscribers.attribs` | `.Subscriber.Attribs.xxx` | 活动邮件、事务邮件 |
| **活动元信息** | 数据库 `campaigns` 表 | `.Campaign.xxx` | 活动邮件 |
| **事务消息自定义数据** | API 请求 `data` 字段 | `.Tx.Data.xxx` | 事务邮件 |

### 3.2 活动邮件（Campaign）的数据注入

#### 渲染上下文结构

**文件位置**: `internal/manager/message.go:33-88`

活动邮件渲染时，**整个 `CampaignMessage` 对象**作为上下文（`.`）传递给模板：

```go
func (m *CampaignMessage) render() error {
    // ...
    // 核心渲染：传入 m (CampaignMessage) 作为上下文
    if err := m.Campaign.Tpl.ExecuteTemplate(&out, models.BaseTpl, m); err != nil {
        return err
    }
    // ...
}
```

#### 注入顺序

数据绑定发生在 `NewCampaignMessage` 函数中：

**文件位置**: `internal/manager/message.go:13-29`

```go
func (m *Manager) NewCampaignMessage(c *models.Campaign, s models.Subscriber) (CampaignMessage, error) {
    msg := CampaignMessage{
        Campaign:   c,  // 第一步：绑定活动元信息
        Subscriber: s,  // 第二步：绑定订阅者数据
        subject:    c.Subject,
        from:       c.FromEmail,
        to:         s.Email,
        unsubURL:   fmt.Sprintf(m.cfg.UnsubURL, c.UUID, s.UUID),
    }
    // 第三步：执行渲染
    if err := msg.render(); err != nil {
        return msg, err
    }
    return msg, nil
}
```

#### 命名空间与访问方式

```
┌─────────────────────────────────────────────────────────────┐
│                    渲染上下文: CampaignMessage                │
├─────────────────────────┬───────────────────────────────────┤
│  .Campaign              │  活动元信息（第一类）              │
│  ├── .UUID              │  活动唯一标识                      │
│  ├── .Name              │  活动内部名称                      │
│  ├── .Subject           │  邮件主题                          │
│  ├── .FromEmail         │  发件人邮箱                        │
│  ├── .Tags              │  标签列表                          │
│  ├── .Attribs           │  活动自定义属性                    │
│  ├── .CreatedAt         │  创建时间                          │
│  └── .UpdatedAt         │  更新时间                          │
├─────────────────────────┼───────────────────────────────────┤
│  .Subscriber            │  订阅者数据（第二类）              │
│  ├── .UUID              │  订阅者唯一标识                    │
│  ├── .Email             │  邮箱地址                          │
│  ├── .Name              │  姓名                              │
│  ├── .Status            │  状态                              │
│  ├── .Attribs           │  订阅者自定义属性 (JSON)          │
│  │   ├── .city          │  示例：城市                        │
│  │   ├── .age           │  示例：年龄                        │
│  │   └── ...            │  任意自定义字段                     │
│  ├── .CreatedAt         │  创建时间                          │
│  ├── .UpdatedAt         │  更新时间                          │
│  ├── .FirstName()       │  方法：提取名                      │
│  └── .LastName()        │  方法：提取姓                      │
└─────────────────────────┴───────────────────────────────────┘
```

#### 无覆盖设计

活动邮件中的两类数据源通过**不同的命名空间前缀**区分，不存在覆盖问题：

| 表达式 | 数据源 | 说明 |
|--------|--------|------|
| `.Campaign.Name` | 活动元信息 | 活动的内部名称 |
| `.Subscriber.Name` | 订阅者数据 | 订阅者的姓名 |

即使字段名相同（如 `Name`），通过前缀完全区分。

### 3.3 事务邮件（Transactional）的数据注入

#### 渲染上下文结构

**文件位置**: `models/messages.go:73-133`

事务邮件使用**匿名结构体**作为上下文，包含订阅者和事务消息数据：

```go
func (m *TxMessage) Render(sub Subscriber, tpl *Template, funcs txttpl.FuncMap) error {
    // 构造上下文：匿名结构体
    data := struct {
        Subscriber Subscriber   // 订阅者数据
        Tx         *TxMessage   // 事务消息（包含自定义数据）
    }{sub, m}

    // 渲染时传入 data
    if err := tpl.Tpl.ExecuteTemplate(&b, BaseTpl, data); err != nil {
        return err
    }
    // ...
}
```

#### 注入顺序

事务消息的完整处理流程在 `cmd/tx.go:SendTxMessage` 中：

```go
func (a *App) SendTxMessage(c echo.Context) error {
    // 1. 解析请求（包含 Data 字段）
    var m models.TxMessage
    // ... 绑定请求数据到 m，包括 m.Data

    // 2. 获取模板
    tpl, err := a.manager.GetTpl(m.TemplateID)

    // 3. 遍历目标订阅者
    for n := range num {
        var sub models.Subscriber

        // 3.1 获取或创建订阅者对象
        if m.SubscriberMode == models.TxSubModeExternal {
            // external 模式：创建临时订阅者
            sub = models.Subscriber{
                Email: m.SubscriberEmails[n],
            }
        } else {
            // default/fallback 模式：从数据库查询
            sub, err = a.core.GetSubscriber(subID, "", subEmail)
            // ...
        }

        // 3.2 执行渲染：注入 sub 和 m
        if err := m.Render(sub, tpl, a.manager.GenericTemplateFuncs()); err != nil {
            // ...
        }
        // ...
    }
}
```

#### 命名空间与访问方式

```
┌─────────────────────────────────────────────────────────────┐
│              渲染上下文: struct{Subscriber, Tx}             │
├─────────────────────────┬───────────────────────────────────┤
│  .Subscriber            │  订阅者数据（同活动邮件）          │
│  ├── .UUID              │  订阅者唯一标识                    │
│  ├── .Email             │  邮箱地址                          │
│  ├── .Name              │  姓名                              │
│  ├── .Attribs           │  订阅者自定义属性                  │
│  └── ...                │                                    │
├─────────────────────────┼───────────────────────────────────┤
│  .Tx                    │  事务消息数据                      │
│  ├── .Data              │  自定义数据（第三类，来自API）     │
│  │   ├── .orderID       │  示例：订单号                      │
│  │   ├── .amount        │  示例：金额                        │
│  │   ├── .items         │  示例：商品列表                     │
│  │   └── ...            │  任意自定义数据                     │
│  ├── .Subject           │  消息主题                          │
│  ├── .FromEmail         │  发件人邮箱                        │
│  └── ...                │                                    │
└─────────────────────────┴───────────────────────────────────┘
```

### 3.4 数据注入顺序总结

#### 活动邮件（Campaign）

| 步骤 | 操作 | 代码位置 |
|------|------|----------|
| 1 | 绑定活动元信息到 `Campaign` 指针 | `internal/manager/message.go:15` |
| 2 | 绑定订阅者数据到 `Subscriber` 值 | `internal/manager/message.go:16` |
| 3 | 构造 `CampaignMessage` 上下文 | `internal/manager/message.go:14-22` |
| 4 | 执行模板渲染（传入上下文） | `internal/manager/message.go:24` |

**关键特点**：
- `Campaign` 是**指针**传递：所有订阅者共享同一个活动实例
- `Subscriber` 是**值**传递：每个订阅者有独立的副本
- 无覆盖：通过命名空间区分

#### 事务邮件（Transactional）

| 步骤 | 操作 | 代码位置 |
|------|------|----------|
| 1 | 从 API 请求解析 `TxMessage`（包含 `Data`） | `cmd/tx.go:18-37` |
| 2 | 获取或创建 `Subscriber` 对象 | `cmd/tx.go:92-128` |
| 3 | 构造匿名结构体上下文 | `models/messages.go:74-77` |
| 4 | 执行模板渲染（传入上下文） | `models/messages.go:81` |

**关键特点**：
- `Tx.Data` 是**每次请求**的自定义数据
- 订阅者可以来自数据库或临时创建
- 无覆盖：通过命名空间区分

### 3.5 覆盖优先级的特殊情况

虽然 Listmonk 的设计通过命名空间避免了覆盖，但在以下场景需要注意：

#### 场景1：同名字段在不同命名空间

```go
// 假设：
// .Campaign.Name = "月度促销活动"
// .Subscriber.Name = "张三"

// 模板中：
{{ .Campaign.Name }}    // 输出：月度促销活动
{{ .Subscriber.Name }}  // 输出：张三
```

**无歧义**，通过前缀完全区分。

#### 场景2：订阅者 Attribs 中的嵌套字段

```go
// .Subscriber.Attribs = map[string]any{
//     "profile": map[string]any{
//         "name": "张先生",  // 注意：这里也叫 name
//     },
// }

// 模板中：
{{ .Subscriber.Name }}               // 输出：订阅者表中的 name 字段
{{ .Subscriber.Attribs.profile.name }} // 输出：张先生
```

**层级访问**，通过路径区分。

#### 场景3：事务邮件中 Data 与 Subscriber 的关系

```go
// 假设 API 传入：
// "data": {
//     "email": "override@example.com",  // 想"覆盖"订阅者邮箱？
//     "name": "新名字"
// }

// 模板中：
{{ .Subscriber.Email }}  // 输出：订阅者实际邮箱（不受 Data 影响）
{{ .Tx.Data.email }}     // 输出：override@example.com
{{ .Subscriber.Name }}   // 输出：订阅者姓名
{{ .Tx.Data.name }}      // 输出：新名字
```

**结论**：Listmonk 的模板系统**不存在数据覆盖机制**，所有数据源通过命名空间独立访问。如果需要"覆盖"逻辑，必须在模板中显式处理：

```go
// 模板中实现覆盖逻辑
{{ if .Tx.Data.email }}
    {{ .Tx.Data.email }}
{{ else }}
    {{ .Subscriber.Email }}
{{ end }}
```

## 四、模板编译流程

### 4.1 编译触发时机

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

### 4.2 编译过程详解

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

### 4.3 模板语法预处理

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

## 五、模板渲染流程

### 5.1 渲染入口

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

### 5.2 渲染过程详解

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

### 5.3 渲染上下文对象

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
.Campaign.Attribs    // JSON - 活动自定义属性
.Campaign.CreatedAt  // time.Time - 创建时间
.Campaign.UpdatedAt  // time.Time - 更新时间
```

## 六、模板函数系统

### 6.1 函数注册

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

### 6.2 通用模板函数

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

### 6.3 模板函数详细说明

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

### 6.4 Sprig 函数库

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

## 七、模板继承机制

### 7.1 双层模板结构

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

### 7.2 模板编译时的合并

**文件位置**: `models/campaigns.go:194-198`

```go
// 将 content 模板的解析树添加到 base 模板中
out, err := baseTPL.AddParseTree(ContentTpl, msgTpl.Tree)
if err != nil {
    return fmt.Errorf("error inserting child template: %v", err)
}
c.Tpl = out
```

### 7.3 模板定义规则

- **基础模板** 必须包含 `{{ template "content" . }}` 标记
- **视觉内容** (visual type) 不使用外部模板，自动使用 `{{ template "content" . }}`
- 多个模板通过 `AddParseTree` 合并为单个可执行模板

## 八、变量缺失与错误处理

### 8.1 Go 模板的默认行为

Listmonk 使用标准 `html/template` 包，**未设置 `missingkey=error` 选项**，因此对缺失变量的默认行为是：

| 缺失情况 | 行为 | 输出示例 |
|----------|------|----------|
| 结构体字段不存在 | 输出 `<no value>` 或空字符串 | `{{ .Subscriber.NonExistent }}` → `` |
| Map 键不存在 | 输出零值（空字符串） | `{{ .Subscriber.Attribs.unknown }}` → `` |
| 方法不存在 | 编译错误 | - |
| 函数不存在 | 编译错误 | - |

**注意**：Listmonk 没有使用 `template.Option("missingkey=error")`，这意味着：
- 模板不会因访问不存在的字段而崩溃
- 缺失字段会静默输出空值
- 这种设计有利于模板的灵活性，但可能隐藏错误

### 8.2 编译阶段错误处理

编译阶段发生在模板解析时，检测到以下错误会立即返回：

#### 错误类型

| 错误类型 | 触发条件 | 错误示例 |
|----------|----------|----------|
| 语法错误 | 模板语法不合法 | `{{ .Subscriber.Name`（缺少 `}}`） |
| 函数不存在 | 调用未注册的函数 | `{{ NonExistentFunc }}` |
| 管道错误 | 管道操作符使用错误 | `{{ .Name | NonExistentFunc }}` |

#### 错误处理路径

**1. 活动开始时的编译错误**

**文件位置**: `internal/manager/pipe.go:27-42`

```go
func (m *Manager) newPipe(c *models.Campaign) (*pipe, error) {
    // ...
    // 编译模板
    if err := c.CompileTemplate(m.TemplateFuncs(c)); err != nil {
        return nil, err  // 直接返回错误，活动不会开始
    }
    // ...
}
```

**处理结果**：
- 活动无法启动
- 错误被记录到日志
- 活动状态可能被标记为 `cancelled`

**2. 预览时的编译错误**

**文件位置**: `cmd/campaigns.go:176-180`

```go
if err := camp.CompileTemplate(a.manager.TemplateFuncs(&camp)); err != nil {
    a.log.Printf("error compiling template: %v", err)
    return echo.NewHTTPError(http.StatusBadRequest,
        a.i18n.Ts("templates.errorCompiling", "error", err.Error()))
}
```

**处理结果**：
- 返回 HTTP 400 错误
- 错误信息包含详细描述（如 `templates.errorCompiling: parse error: ...`）
- 前端可显示具体错误

**3. 事务模板编译错误**

事务模板在缓存时编译：

**文件位置**: `internal/manager/manager.go:319-323`

```go
func (m *Manager) CacheTpl(id int, tpl *models.Template) {
    // 模板在传入前已编译
    m.tplsMut.Lock()
    m.tpls[id] = tpl
    m.tplsMut.Unlock()
}
```

实际编译发生在 `cmd/templates.go` 中：

```go
// 模板保存时编译
if err := tpl.Compile(a.manager.GenericTemplateFuncs()); err != nil {
    return echo.NewHTTPError(http.StatusBadRequest,
        a.i18n.Ts("templates.errorCompiling", "error", err.Error()))
}
```

### 8.3 渲染阶段错误处理

渲染阶段发生在模板执行时，将数据上下文应用到已编译的模板。

#### 错误类型

| 错误类型 | 触发条件 | 示例 |
|----------|----------|------|
| 函数执行错误 | 模板函数返回错误 | 自定义函数 panic 或返回 error |
| 类型断言错误 | 数据类型不匹配 | `{{ add .Subscriber.Name 1 }}`（Name 是字符串） |
| nil 指针引用 | 访问 nil 指数字段 | 极少发生，Listmonk 确保上下文有效 |

#### 错误处理路径

**1. 活动发送时的渲染错误**

**文件位置**: `internal/manager/pipe.go:95-100`

```go
for _, s := range subs {
    msg, err := p.newMessage(s)
    if err != nil {
        // 记录错误日志
        p.m.log.Printf("error rendering message (%s) (%s): %v", p.camp.Name, s.Email, err)
        // 跳过该订阅者，继续处理下一个
        continue
    }
    // ... 发送消息
}
```

**关键行为**：
- **单个订阅者失败不影响其他订阅者**
- 错误被记录（包含活动名、订阅者邮箱、错误详情）
- **不会累加到 `MaxSendErrors` 计数器**（那是发送错误）
- 活动继续运行

**2. 预览时的渲染错误**

**文件位置**: `cmd/campaigns.go:183-188`

```go
msg, err := a.manager.NewCampaignMessage(&camp, dummySubscriber)
if err != nil {
    a.log.Printf("error rendering message: %v", err)
    return echo.NewHTTPError(http.StatusBadRequest,
        a.i18n.Ts("templates.errorRendering", "error", err.Error()))
}
```

**处理结果**：
- 返回 HTTP 400 错误
- 错误信息显示给用户

**3. 事务消息渲染错误**

**文件位置**: `cmd/tx.go:132-135`

```go
if err := m.Render(sub, tpl, a.manager.GenericTemplateFuncs()); err != nil {
    return echo.NewHTTPError(http.StatusBadRequest,
        a.i18n.Ts("globals.messages.errorFetching", "name"))
}
```

**处理结果**：
- 整个事务邮件请求失败
- 返回 HTTP 400 错误
- **注意**：错误消息较模糊（`errorFetching`），不显示详细错误

### 8.4 完整错误处理流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        模板生命周期                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                               │
│  │  模板编译阶段  │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ├── 语法错误？ ──→ 返回错误（活动不启动 / API 返回400）│
│         │                                                        │
│         ├── 函数不存在？ ──→ 返回错误                          │
│         │                                                        │
│         └── 成功 ──→ 继续                                       │
│                                                                 │
│  ┌──────────────┐                                               │
│  │  模板渲染阶段  │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ├── 场景1：活动发送                                      │
│         │   │                                                    │
│         │   └── 错误？ ──→ 记录日志 + continue（跳过当前订阅者）│
│         │                                                        │
│         ├── 场景2：活动预览                                      │
│         │   │                                                    │
│         │   └── 错误？ ──→ HTTP 400 + 详细错误信息            │
│         │                                                        │
│         └── 场景3：事务邮件                                      │
│             │                                                    │
│             └── 错误？ ──→ HTTP 400 + 模糊错误信息            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.5 错误处理的关键设计决策

| 决策点 | 设计选择 | 原因 |
|--------|----------|------|
| 编译错误在活动启动时检测 | 阻止活动启动 | 避免发送大量错误邮件 |
| 渲染错误在活动发送时跳过 | 单个订阅者失败不影响全局 | 容错性设计 |
| 渲染错误不计入 `MaxSendErrors` | 与发送错误区分 | 渲染错误是模板问题，不是网络问题 |
| 预览时返回详细错误 | 帮助用户调试模板 | 预览场景需要快速反馈 |
| 事务邮件返回模糊错误 | 安全性考虑 | 避免暴露内部实现细节 |

### 8.6 变量缺失的处理建议

由于 Listmonk 对缺失变量静默处理，建议在模板中使用以下模式：

#### 使用 `default` 函数（Sprig）

```go
// 提供默认值
{{ default "未知城市" .Subscriber.Attribs.city }}

// 嵌套属性的默认值
{{ default "N/A" .Subscriber.Attribs.profile.address }}
```

#### 使用条件判断

```go
{{ if .Subscriber.Attribs.vip }}
    <p>尊敬的 VIP 用户</p>
{{ else }}
    <p>尊敬的用户</p>
{{ end }}
```

#### 使用 `with` 管道

```go
{{ with .Subscriber.Attribs.profile }}
    <p>城市: {{ .city }}</p>
    <p>电话: {{ default "未提供" .phone }}</p>
{{ else }}
    <p>暂无个人资料</p>
{{ end }}
```

## 九、完整渲染流程示例

### 9.1 活动发送时的完整流程

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

### 9.2 预览流程

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

### 9.3 端到端完整示例

让我们通过一个完整的示例，从输入数据到最终邮件正文，展示整个渲染过程。

#### 场景描述

发送一封**订单确认邮件**（事务邮件），包含：
- 订阅者信息
- 订单数据（通过 API 传入的自定义数据）
- 动态计算的内容

#### 步骤1：准备输入数据

##### API 请求（事务邮件）

```json
POST /api/tx
Content-Type: application/json

{
    "template_id": 1,
    "subscriber_email": "zhangwei@example.com",
    "subscriber_mode": "default",
    "from_email": "no-reply@shop.example.com",
    "subject": "您的订单 {{ .Tx.Data.orderNo }} 已确认",
    "data": {
        "orderNo": "ORD-2026-005432",
        "orderDate": "2026-05-05",
        "totalAmount": 1299.00,
        "items": [
            {"name": "无线蓝牙耳机", "quantity": 1, "price": 599},
            {"name": "手机保护壳", "quantity": 2, "price": 150}
        ],
        "shipping": {
            "address": "北京市朝阳区xxx街道xxx号",
            "phone": "138****8888"
        },
        "discountCode": "VIP2026"
    }
}
```

##### 数据库中的订阅者记录

```go
// subscribers 表中的记录
Subscriber{
    UUID:    "sub-uuid-12345",
    Email:   "zhangwei@example.com",
    Name:    "张伟",
    Status:  "enabled",
    Attribs: JSON{
        "city":      "北京",
        "memberLevel": "VIP",
        "preferences": map[string]any{
            "theme": "dark",
        },
    },
    CreatedAt: time.Date(2023, 1, 15, 10, 0, 0, 0, time.UTC),
}
```

##### 使用的模板（Template ID: 1）

**模板主体**：
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>订单确认</title>
    <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
        .container { max-width: 600px; margin: 0 auto; padding: 20px; }
        .header { background: #4a90d9; color: white; padding: 20px; text-align: center; }
        .content { padding: 20px; background: #f9f9f9; }
        .footer { text-align: center; padding: 20px; font-size: 12px; color: #666; }
        .item { border-bottom: 1px solid #eee; padding: 10px 0; }
        .total { font-size: 18px; font-weight: bold; margin-top: 20px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>订单确认通知</h1>
        </div>
        
        <div class="content">
            <p>尊敬的 <strong>{{ .Subscriber.Name }}</strong>，您好！</p>
            
            {{ if eq .Subscriber.Attribs.memberLevel "VIP" }}
            <p style="color: #d9534f; font-weight: bold;">
                🎉 VIP 会员专享：您的订单已享受优先处理！
            </p>
            {{ end }}
            
            <h2>订单信息</h2>
            <p><strong>订单号：</strong>{{ .Tx.Data.orderNo }}</p>
            <p><strong>下单日期：</strong>{{ .Tx.Data.orderDate }}</p>
            
            <h2>商品清单</h2>
            {{ range .Tx.Data.items }}
            <div class="item">
                <p><strong>{{ .name }}</strong></p>
                <p>数量：{{ .quantity }} × ¥{{ .price }} = ¥{{ mul .quantity .price }}</p>
            </div>
            {{ end }}
            
            <div class="total">
                <p>订单总额：<span style="color: #d9534f;">¥{{ .Tx.Data.totalAmount }}</span></p>
            </div>
            
            {{ if .Tx.Data.discountCode }}
            <p><strong>使用优惠码：</strong>{{ .Tx.Data.discountCode }}</p>
            {{ end }}
            
            <h2>配送信息</h2>
            {{ with .Tx.Data.shipping }}
            <p><strong>收货地址：</strong>{{ .address }}</p>
            <p><strong>联系电话：</strong>{{ .phone }}</p>
            {{ end }}
            
            <p style="margin-top: 30px; padding-top: 20px; border-top: 1px solid #eee;">
                如有疑问，请联系客服。<br>
                感谢您的订购！
            </p>
        </div>
        
        <div class="footer">
            <p>此邮件由系统自动发送，请勿回复。</p>
            <p>© 2026 Example Shop. All rights reserved.</p>
            <p>
                如需退订营销邮件，请 
                <a href="{{ UnsubscribeURL }}">点击这里</a>
            </p>
            {{ TrackView }}
        </div>
    </div>
</body>
</html>
```

#### 步骤2：数据注入与上下文构造

当 API 请求到达后，系统执行以下操作：

##### 1. 解析请求参数

```go
// cmd/tx.go 中
var m models.TxMessage
// 解析 JSON 后：
m = models.TxMessage{
    TemplateID:       1,
    SubscriberEmails: []string{"zhangwei@example.com"},
    SubscriberMode:   "default",
    FromEmail:        "no-reply@shop.example.com",
    Subject:          "您的订单 {{ .Tx.Data.orderNo }} 已确认",
    Data: map[string]any{
        "orderNo":     "ORD-2026-005432",
        "orderDate":   "2026-05-05",
        "totalAmount": 1299.00,
        "items": []any{
            map[string]any{"name": "无线蓝牙耳机", "quantity": 1, "price": 599},
            map[string]any{"name": "手机保护壳", "quantity": 2, "price": 150},
        },
        "shipping": map[string]any{
            "address": "北京市朝阳区xxx街道xxx号",
            "phone":   "138****8888",
        },
        "discountCode": "VIP2026",
    },
}
```

##### 2. 查询订阅者

```go
// 从数据库查询
sub, err := a.core.GetSubscriber(0, "", "zhangwei@example.com")
// 结果：
sub = models.Subscriber{
    UUID:    "sub-uuid-12345",
    Email:   "zhangwei@example.com",
    Name:    "张伟",
    Status:  "enabled",
    Attribs: JSON{
        "city":        "北京",
        "memberLevel": "VIP",
        "preferences": map[string]any{"theme": "dark"},
    },
    CreatedAt: time.Date(2023, 1, 15, 10, 0, 0, 0, time.UTC),
}
```

##### 3. 构造渲染上下文

**文件位置**: `models/messages.go:74-77`

```go
// 构造匿名结构体作为模板上下文
data := struct {
    Subscriber models.Subscriber
    Tx         *models.TxMessage
}{
    Subscriber: sub,  // 订阅者数据
    Tx:         &m,   // 事务消息（包含 Data）
}
```

**内存中的上下文结构**：
```
data (匿名结构体)
├── Subscriber
│   ├── UUID:        "sub-uuid-12345"
│   ├── Email:       "zhangwei@example.com"
│   ├── Name:        "张伟"
│   ├── Status:      "enabled"
│   ├── Attribs:
│   │   ├── city:        "北京"
│   │   ├── memberLevel: "VIP"
│   │   └── preferences: map["theme": "dark"]
│   └── CreatedAt:   2023-01-15 10:00:00
│
└── Tx (*TxMessage)
    ├── Data:
    │   ├── orderNo:     "ORD-2026-005432"
    │   ├── orderDate:   "2026-05-05"
    │   ├── totalAmount: 1299.00
    │   ├── items:       [item1, item2]
    │   ├── shipping:
    │   │   ├── address: "北京市朝阳区xxx街道xxx号"
    │   │   └── phone:   "138****8888"
    │   └── discountCode: "VIP2026"
    │
    ├── Subject:   "您的订单 {{ .Tx.Data.orderNo }} 已确认"
    └── FromEmail: "no-reply@shop.example.com"
```

#### 步骤3：模板编译与语法预处理

在模板保存时已编译，但主题 `Subject` 包含模板表达式，需要在渲染时处理：

##### 主题预处理

原始主题：
```
您的订单 {{ .Tx.Data.orderNo }} 已确认
```

由于主题使用 `text/template`，直接编译，不需要 `regTplFuncs` 预处理（那些是针对 `TrackLink` 等函数的）。

#### 步骤4：执行渲染

##### 渲染主题

**文件位置**: `models/messages.go:121-130`

```go
// 主题模板执行
// 上下文: data (上面的匿名结构体)
// 模板内容: "您的订单 {{ .Tx.Data.orderNo }} 已确认"

if subjTpl != nil {
    b.Reset()
    if err := subjTpl.ExecuteTemplate(&b, BaseTpl, data); err != nil {
        return err
    }
    m.Subject = b.String()
}
```

**主题渲染过程**：
```
模板: "您的订单 {{ .Tx.Data.orderNo }} 已确认"
      ↓
访问 .Tx → 得到 *TxMessage
      ↓
访问 .Data → 得到 map[string]any
      ↓
访问 .orderNo → 得到 "ORD-2026-005432"
      ↓
结果: "您的订单 ORD-2026-005432 已确认"
```

##### 渲染主体（核心步骤）

**文件位置**: `models/messages.go:80-86`

```go
// 主体模板执行
b := bytes.Buffer{}
if err := tpl.Tpl.ExecuteTemplate(&b, BaseTpl, data); err != nil {
    return err
}
m.Body = b.Bytes()
```

让我们追踪几个关键模板表达式的渲染：

###### 表达式1：`{{ .Subscriber.Name }}`

```
模板: {{ .Subscriber.Name }}
      ↓
访问 .Subscriber → 得到 Subscriber 结构体
      ↓
访问 .Name → 得到 "张伟"
      ↓
输出: 张伟
```

###### 表达式2：VIP 条件判断

```
模板:
{{ if eq .Subscriber.Attribs.memberLevel "VIP" }}
    <p style="color: #d9534f; font-weight: bold;">
        🎉 VIP 会员专享：您的订单已享受优先处理！
    </p>
{{ end }}

渲染过程:
1. 访问 .Subscriber.Attribs → 得到 map[string]any
2. 访问 .memberLevel → 得到 "VIP"
3. 比较 eq "VIP" "VIP" → true
4. 渲染条件块内的内容

输出:
<p style="color: #d9534f; font-weight: bold;">
    🎉 VIP 会员专享：您的订单已享受优先处理！
</p>
```

###### 表达式3：订单号

```
模板: {{ .Tx.Data.orderNo }}
      ↓
访问 .Tx → 得到 *TxMessage
      ↓
访问 .Data → 得到 map[string]any
      ↓
访问 .orderNo → 得到 "ORD-2026-005432"
      ↓
输出: ORD-2026-005432
```

###### 表达式4：循环渲染商品列表

```
模板:
{{ range .Tx.Data.items }}
<div class="item">
    <p><strong>{{ .name }}</strong></p>
    <p>数量：{{ .quantity }} × ¥{{ .price }} = ¥{{ mul .quantity .price }}</p>
</div>
{{ end }}

渲染过程:
1. .Tx.Data.items → 得到包含2个商品的数组

第一次迭代 (当前项 = {"name": "无线蓝牙耳机", "quantity": 1, "price": 599}):
   - .name → "无线蓝牙耳机"
   - .quantity → 1
   - .price → 599
   - mul 1 599 → 599 (Sprig 函数)

第二次迭代 (当前项 = {"name": "手机保护壳", "quantity": 2, "price": 150}):
   - .name → "手机保护壳"
   - .quantity → 2
   - .price → 150
   - mul 2 150 → 300

输出:
<div class="item">
    <p><strong>无线蓝牙耳机</strong></p>
    <p>数量：1 × ¥599 = ¥599</p>
</div>
<div class="item">
    <p><strong>手机保护壳</strong></p>
    <p>数量：2 × ¥150 = ¥300</p>
</div>
```

###### 表达式5：with 管道（配送信息）

```
模板:
{{ with .Tx.Data.shipping }}
<p><strong>收货地址：</strong>{{ .address }}</p>
<p><strong>联系电话：</strong>{{ .phone }}</p>
{{ end }}

渲染过程:
1. .Tx.Data.shipping → 得到 {"address": "...", "phone": "..."}
2. 由于 shipping 不为空/nil，进入 with 块
3. 在 with 块内，`.` 被重定义为 shipping 的值
4. .address → "北京市朝阳区xxx街道xxx号"
5. .phone → "138****8888"

输出:
<p><strong>收货地址：</strong>北京市朝阳区xxx街道xxx号</p>
<p><strong>联系电话：</strong>138****8888</p>
```

###### 表达式6：优惠码条件判断

```
模板:
{{ if .Tx.Data.discountCode }}
<p><strong>使用优惠码：</strong>{{ .Tx.Data.discountCode }}</p>
{{ end }}

渲染过程:
1. .Tx.Data.discountCode → "VIP2026" (非空字符串)
2. if 条件为 true
3. 渲染条件块

输出:
<p><strong>使用优惠码：</strong>VIP2026</p>
```

###### 表达式7：UnsubscribeURL 函数

```
模板: <a href="{{ UnsubscribeURL }}">点击这里</a>

注意：UnsubscribeURL 是活动特定函数，事务邮件中需要使用
      订阅者管理链接或其他方式。

在事务邮件中，访问 URL 的方式：
- 使用 RootURL 手动构造
- 或在 .Tx.Data 中传入自定义链接
```

###### 表达式8：TrackView 函数

```
模板: {{ TrackView }}

注意：TrackView 也是活动特定函数，用于生成追踪像素。
      事务邮件中可能不适用，或需要单独处理。
```

#### 步骤5：最终输出结果

##### 渲染后的主题

```
您的订单 ORD-2026-005432 已确认
```

##### 渲染后的邮件正文

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>订单确认</title>
    <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
        .container { max-width: 600px; margin: 0 auto; padding: 20px; }
        .header { background: #4a90d9; color: white; padding: 20px; text-align: center; }
        .content { padding: 20px; background: #f9f9f9; }
        .footer { text-align: center; padding: 20px; font-size: 12px; color: #666; }
        .item { border-bottom: 1px solid #eee; padding: 10px 0; }
        .total { font-size: 18px; font-weight: bold; margin-top: 20px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>订单确认通知</h1>
        </div>
        
        <div class="content">
            <p>尊敬的 <strong>张伟</strong>，您好！</p>
            
            <p style="color: #d9534f; font-weight: bold;">
                🎉 VIP 会员专享：您的订单已享受优先处理！
            </p>
            
            <h2>订单信息</h2>
            <p><strong>订单号：</strong>ORD-2026-005432</p>
            <p><strong>下单日期：</strong>2026-05-05</p>
            
            <h2>商品清单</h2>
            <div class="item">
                <p><strong>无线蓝牙耳机</strong></p>
                <p>数量：1 × ¥599 = ¥599</p>
            </div>
            <div class="item">
                <p><strong>手机保护壳</strong></p>
                <p>数量：2 × ¥150 = ¥300</p>
            </div>
            
            <div class="total">
                <p>订单总额：<span style="color: #d9534f;">¥1299</span></p>
            </div>
            
            <p><strong>使用优惠码：</strong>VIP2026</p>
            
            <h2>配送信息</h2>
            <p><strong>收货地址：</strong>北京市朝阳区xxx街道xxx号</p>
            <p><strong>联系电话：</strong>138****8888</p>
            
            <p style="margin-top: 30px; padding-top: 20px; border-top: 1px solid #eee;">
                如有疑问，请联系客服。<br>
                感谢您的订购！
            </p>
        </div>
        
        <div class="footer">
            <p>此邮件由系统自动发送，请勿回复。</p>
            <p>© 2026 Example Shop. All rights reserved.</p>
        </div>
    </div>
</body>
</html>
```

#### 步骤6：端到端流程图总结

```
┌────────────────────────────────────────────────────────────────────┐
│                    端到端渲染流程（事务邮件）                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. API 请求                                                        │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ POST /api/tx                                                   │ │
│  │ {                                                              │ │
│  │   "subscriber_email": "zhangwei@example.com",                │ │
│  │   "data": {                                                    │ │
│  │     "orderNo": "ORD-2026-005432",                            │ │
│  │     "items": [...],                                            │ │
│  │     "shipping": {...}                                          │ │
│  │   }                                                            │ │
│  │ }                                                              │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              ↓                                     │
│  2. 数据收集                                                        │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  a. 从数据库查询订阅者                                          │ │
│  │     Subscriber{                                                │ │
│  │       Name: "张伟",                                            │ │
│  │       Attribs: {"memberLevel": "VIP", "city": "北京"}        │ │
│  │     }                                                          │ │
│  │                                                               │ │
│  │  b. 从请求获取自定义数据                                        │ │
│  │     Tx.Data = {                                                │ │
│  │       "orderNo": "ORD-2026-005432",                          │ │
│  │       "totalAmount": 1299.00                                   │ │
│  │     }                                                          │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              ↓                                     │
│  3. 构造上下文                                                      │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  data = struct {                                               │ │
│  │      Subscriber Subscriber    ←