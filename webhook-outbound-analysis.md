# listmonk Webhook 外发机制分析报告

## 1. 项目概述

listmonk 是一个开源的邮件列表管理系统，使用 Go 语言开发，前端使用 Vue.js，数据库使用 PostgreSQL。本文档将分析 listmonk 的 webhook 外发机制，包括事件触发、webhook 投递机制以及与外部 CRM 系统集成的方式。

## 2. 事件系统

listmonk 实现了一个简单的事件发布-订阅机制，主要用于错误消息的广播。

### 2.1 核心实现

事件系统的核心代码位于 `internal/events/events.go`：

```go
// Event 表示系统中的单个事件
type Event struct {
    ID       string   `json:"id"`
    Type     string   `json:"type"`
    Message  string   `json:"message"`
    Data     any      `json:"data"`
    Channels []string `json:"-"`
}

// Events 管理事件的订阅和发布
type Events struct {
    subs map[string]chan Event
    sync.RWMutex
}
```

### 2.2 主要功能

- **订阅事件**：`Subscribe(id string) (chan Event, error)` - 允许订阅者注册以接收事件
- **取消订阅**：`Unsubscribe(id string)` - 允许订阅者取消订阅
- **发布事件**：`Publish(e Event) error` - 向所有订阅者发布事件
- **错误写入器**：`ErrWriter() io.Writer` - 提供一个 io.Writer 接口，用于将错误日志发布为事件

### 2.3 事件类型

目前只定义了一种事件类型：
- `TypeError = "error"` - 错误事件

### 2.4 事件流

错误日志会通过 `ErrWriter()` 方法转换为事件并发布：

```go
// 当写入包含 "error" 的消息时，发布错误事件
func (w *wri) Write(b []byte) (n int, err error) {
    if !bytes.Contains(b, []byte("error")) {
        return 0, nil
    }

    w.ev.Publish(Event{
        Type:    TypeError,
        Message: string(b),
    })

    return len(b), nil
}
```

## 3. Postback 信使（Webhook 投递机制）

listmonk 的 webhook 外发功能是通过 Postback 信使实现的，它实现了 `Messenger` 接口，用于将消息通过 HTTP POST 发送到外部 URL。

### 3.1 核心实现

Postback 信使的核心代码位于 `internal/messenger/postback/postback.go`：

```go
// Postback 表示一个 HTTP 消息服务器
type Postback struct {
    authStr string
    o       Options
    c       *http.Client
}

// Options 表示 HTTP Postback 服务器选项
type Options struct {
    Name     string        `json:"name"`
    Username string        `json:"username"`
    Password string        `json:"password"`
    RootURL  string        `json:"root_url"`
    MaxConns int           `json:"max_conns"`
    Retries  int           `json:"retries"`
    Timeout  time.Duration `json:"timeout"`
}
```

### 3.2 主要功能

- **创建 Postback 信使**：`New(o Options) (*Postback, error)` - 根据配置创建新的 Postback 信使实例
- **获取信使名称**：`Name() string` - 返回信使的名称
- **推送消息**：`Push(m models.Message) error` - 将消息推送到外部 URL
- **刷新队列**：`Flush() error` - 刷新消息队列（当前实现为空）
- **关闭连接**：`Close() error` - 关闭空闲的 HTTP 连接

### 3.3 消息结构

当推送消息时，Postback 信使会将消息序列化为以下 JSON 结构：

```go
// postback 是作为 JSON 发送到 HTTP Postback 服务器的负载
type postback struct {
    Subject     string       `json:"subject"`
    FromEmail   string       `json:"from_email"`
    ContentType string       `json:"content_type"`
    Body        string       `json:"body"`
    Recipients  []recipient  `json:"recipients"`
    Campaign    *campaign    `json:"campaign"`
    Attachments []attachment `json:"attachments"`
}

type campaign struct {
    FromEmail string         `json:"from_email"`
    UUID      string         `json:"uuid"`
    Name      string         `json:"name"`
    Headers   models.Headers `json:"headers"`
    Tags      []string       `json:"tags"`
}

type recipient struct {
    UUID    string      `json:"uuid"`
    Email   string      `json:"email"`
    Name    string      `json:"name"`
    Attribs models.JSON `json:"attribs"`
    Status  string      `json:"status"`
}

type attachment struct {
    Name    string               `json:"name"`
    Header  textproto.MIMEHeader `json:"header"`
    Content []byte               `json:"content"`
}
```

### 3.4 消息推送流程

1. **构建 Postback 负载**：根据 `models.Message` 构建 `postback` 结构体
2. **序列化为 JSON**：使用 `easyjson` 将负载序列化为 JSON
3. **发送 HTTP POST 请求**：将 JSON 数据发送到配置的 `RootURL`
4. **处理响应**：检查响应状态码，非 200 状态码视为失败

### 3.5 配置选项

Postback 信使支持以下配置选项：

- **Name**：信使名称，用于识别和管理多个信使
- **Username/Password**：用于 Basic 认证的凭据
- **RootURL**：接收 POST 请求的外部 URL
- **MaxConns**：最大连接数，用于连接池管理
- **Retries**：重试次数（注意：代码中未实现重试机制）
- **Timeout**：HTTP 请求超时时间

### 3.6 连接池管理

Postback 信使使用 HTTP 连接池来优化性能：

```go
return &Postback{
    authStr: authStr,
    o:       o,
    c: &http.Client{
        Timeout: o.Timeout,
        Transport: &http.Transport{
            MaxIdleConnsPerHost:   o.MaxConns,
            MaxConnsPerHost:       o.MaxConns,
            ResponseHeaderTimeout: o.Timeout,
            IdleConnTimeout:       o.Timeout,
        },
    },
}, nil
```

## 4. 事件触发机制

listmonk 的事件触发机制主要与 campaign（活动）处理相关。当发送邮件时，消息会通过 Messenger 接口发送，Postback 信使就是其中的一种实现。

### 4.1 Campaign 处理流程

1. **扫描 Campaign**：`Manager.scanCampaigns()` 定期扫描数据库中的活动 campaign
2. **创建处理管道**：为每个活动 campaign 创建一个 `pipe`
3. **获取订阅者**：从数据库中获取下一批订阅者
4. **构建消息**：为每个订阅者构建个性化的消息
5. **发送消息**：通过配置的 Messenger 发送消息

### 4.2 消息发送流程

消息发送的核心逻辑位于 `internal/manager/manager.go` 的 `worker()` 方法：

```go
func (m *Manager) worker() {
    numMsg := 0
    for {
        select {
        // 处理 campaign 消息
        case msg, ok := <-m.campMsgQ:
            if !ok {
                return
            }

            // 检查 campaign 是否已结束或停止
            if msg.pipe != nil && msg.pipe.stopped.Load() {
                msg.pipe.wg.Done()
                continue
            }

            // 速率限制
            if numMsg >= m.cfg.MessageRate {
                time.Sleep(time.Second)
                numMsg = 0
            }
            numMsg++

            // 构建输出消息
            out := models.Message{
                From:        msg.from,
                To:          []string{msg.to},
                Subject:     msg.subject,
                ContentType: msg.Campaign.ContentType,
                Body:        msg.body,
                AltBody:     msg.altBody,
                Subscriber:  msg.Subscriber,
                Campaign:    msg.Campaign,
                Attachments: msg.Campaign.Attachments,
            }

            // 添加自定义头
            // ...

            // 通过 Messenger 推送消息
            err := m.messengers[msg.Campaign.Messenger].Push(out)
            if err != nil {
                m.log.Printf("error sending message in campaign %s: subscriber %d: %v", msg.Campaign.Name, msg.Subscriber.ID, err)
            }

            // 更新统计信息
            // ...

        // 处理任意消息
        case msg, ok := <-m.msgQ:
            if !ok {
                return
            }

            // 通过 Messenger 推送消息
            if err := m.messengers[msg.Messenger].Push(msg); err != nil {
                m.log.Printf("error sending message '%s': %v", msg.Subject, err)
            }
        }
    }
}
```

### 4.3 消息类型

listmonk 支持两种类型的消息：

1. **Campaign 消息**：与特定 campaign 关联的消息，通过 `PushCampaignMessage()` 方法入队
2. **任意消息**：非 campaign 关联的消息，通过 `PushMessage()` 方法入队

## 5. 与外部 CRM 系统集成的方式

listmonk 提供了多种与外部 CRM 系统集成的方式，包括 API 集成、直接数据库操作和 Postback 信使。

### 5.1 API 集成

listmonk 提供了丰富的 RESTful API，特别是针对订阅者的操作，这是与 CRM 系统集成的主要方式。

#### 5.1.1 订阅者 API

订阅者 API 位于 `docs/docs/content/apis/subscribers.md`，主要包括：

| 方法 | 端点 | 描述 |
| ---- | ---- | ---- |
| GET | /api/subscribers | 查询和检索订阅者 |
| GET | /api/subscribers/{subscriber_id} | 检索特定订阅者 |
| GET | /api/subscribers/{subscriber_id}/export | 导出特定订阅者数据 |
| GET | /api/subscribers/{subscriber_id}/bounces | 检索订阅者的退信记录 |
| POST | /api/subscribers | 创建新订阅者 |
| POST | /api/subscribers/{subscriber_id}/optin | 发送订阅确认邮件 |
| POST | /api/public/subscription | 创建公共订阅 |
| PUT | /api/subscribers/lists | 修改订阅者列表成员关系 |
| PUT | /api/subscribers/{subscriber_id} | 更新特定订阅者 |
| PATCH | /api/subscribers/{subscriber_id} | 部分更新特定订阅者 |
| PUT | /api/subscribers/{subscriber_id}/blocklist | 拉黑特定订阅者 |
| PUT | /api/subscribers/blocklist | 拉黑一个或多个订阅者 |
| PUT | /api/subscribers/query/blocklist | 基于 SQL 表达式拉黑订阅者 |
| DELETE | /api/subscribers/{subscriber_id} | 删除特定订阅者 |
| DELETE | /api/subscribers/{subscriber_id}/bounces | 删除特定订阅者的退信记录 |
| DELETE | /api/subscribers | 删除一个或多个订阅者 |
| POST | /api/subscribers/query/delete | 基于 SQL 表达式删除订阅者 |

#### 5.1.2 API 使用示例

**创建订阅者**：
```shell
curl -u 'api_username:access_token' 'http://localhost:9000/api/subscribers' -H 'Content-Type: application/json' \
    --data '{"email":"subscriber@domain.com","name":"The Subscriber","status":"enabled","lists":[1],"attribs":{"city":"Bengaluru","projects":3,"stack":{"languages":["go","python"]}}}'
```

**查询订阅者**：
```shell
curl -u 'api_username:access_token' 'http://localhost:9000/api/subscribers?page=1&per_page=100'
```

**基于 SQL 表达式查询订阅者**：
```shell
curl -u 'api_username:access_token' -X GET 'http://localhost:9000/api/subscribers' \
    --url-query 'page=1' \
    --url-query 'per_page=100' \
    --url-query "query=subscribers.name LIKE 'Test%' AND subscribers.attribs->>'city' = 'Bengaluru'"
```

### 5.2 直接数据库操作

listmonk 使用简单的表结构来表示订阅者、列表和订阅关系，这使得直接操作数据库成为可能。

#### 5.2.1 主要表结构

根据 `schema.sql` 和文档，主要表包括：

- **subscribers**：订阅者表
- **lists**：列表表
- **subscriber_lists**：订阅者-列表关系表
- **campaigns**：活动表
- **campaign_views**：活动浏览记录
- **link_clicks**：链接点击记录
- **bounces**：退信记录

#### 5.2.2 适用场景

直接数据库操作适用于以下场景：
- 批量数据同步
- 复杂的数据转换
- 与现有数据库系统的深度集成

### 5.3 Postback 信使集成

Postback 信使是 listmonk 内置的 webhook 外发机制，可以将邮件内容和订阅者信息发送到外部系统。

#### 5.3.1 配置方式

Postback 信使可以通过以下方式配置：

1. **前端配置**：在管理后台的 Settings -> Messengers 页面配置
2. **配置文件**：在 `config.toml` 中配置

前端配置界面位于 `frontend/src/views/settings/messengers.vue`，支持配置以下参数：
- 启用/禁用
- 名称
- URL（接收 POST 请求的外部 URL）
- 用户名（用于 Basic 认证）
- 密码（用于 Basic 认证）
- 最大连接数
- 重试次数
- 超时时间

#### 5.3.2 使用场景

Postback 信使适用于以下场景：
- 实时同步邮件发送事件到外部系统
- 触发外部系统的工作流
- 集成需要实时数据的 CRM 系统

#### 5.3.3 数据格式

当使用 Postback 信使发送消息时，外部系统将收到以下格式的 JSON 数据：

```json
{
  "subject": "邮件主题",
  "from_email": "sender@example.com",
  "content_type": "text/html",
  "body": "<html><body>邮件内容</body></html>",
  "recipients": [
    {
      "uuid": "subscriber-uuid",
      "email": "subscriber@example.com",
      "name": "订阅者姓名",
      "attribs": {
        "city": "城市",
        "custom_field": "自定义字段值"
      },
      "status": "enabled"
    }
  ],
  "campaign": {
    "from_email": "sender@example.com",
    "uuid": "campaign-uuid",
    "name": "活动名称",
    "headers": {
      "自定义头": "值"
    },
    "tags": ["标签1", "标签2"]
  },
  "attachments": [
    {
      "name": "附件名称.pdf",
      "header": {
        "Content-Type": ["application/pdf"],
        "Content-Disposition": ["attachment; filename=附件名称.pdf"]
      },
      "content": "base64编码的附件内容"
    }
  ]
}
```

## 6. 初始化流程

### 6.1 Postback 信使初始化

Postback 信使在应用启动时通过 `initPostbackMessengers()` 函数初始化，该函数位于 `cmd/init.go`：

```go
// initPostbackMessengers 初始化并返回所有启用的 HTTP postback 信使后端
func initPostbackMessengers(ko *koanf.Koanf) []manager.Messenger {
    items := ko.Slices("messengers")
    if len(items) == 0 {
        return nil
    }

    var out []manager.Messenger
    for _, item := range items {
        if !item.Bool("enabled") {
            continue
        }

        // 读取 Postback 服务器配置
        var (
            name = item.String("name")
            o    postback.Options
        )
        if err := item.UnmarshalWithConf("", &o, koanf.UnmarshalConf{Tag: "json"}); err != nil {
            lo.Fatalf("error reading Postback config: %v", err)
        }

        // 初始化 Messenger
        p, err := postback.New(o)
        if err != nil {
            lo.Fatalf("error initializing Postback messenger %s: %v", name, err)
        }
        out = append(out, p)

        lo.Printf("loaded Postback messenger: %s", name)
    }

    return out
}
```

### 6.2 信使注册

初始化后的信使会被添加到 `Manager` 中，用于发送消息：

```go
// 初始化所有信使，SMTP 和 postback
msgrs = append(initSMTPMessengers(), initPostbackMessengers(ko)...)

// Campaign 管理器
mgr = initCampaignManager(msgrs, queries, urlCfg, core, media, i18n, ko)
```

## 7. 优缺点分析

### 7.1 优点

1. **灵活的信使架构**：通过 `Messenger` 接口实现了可插拔的消息发送机制
2. **连接池管理**：Postback 信使使用 HTTP 连接池，提高了性能
3. **丰富的 API**：提供了完整的订阅者管理 API，便于与外部系统集成
4. **多种集成方式**：支持 API 集成、直接数据库操作和 Postback 信使三种集成方式
5. **事件系统**：内置简单的事件发布-订阅机制，便于扩展

### 7.2 缺点

1. **事件类型有限**：目前只支持错误事件类型，缺乏更丰富的业务事件
2. **重试机制缺失**：配置中有 `Retries` 选项，但代码中未实现重试逻辑
3. **Postback 信使功能单一**：主要用于发送邮件内容，缺乏对订阅者生命周期事件的支持
4. **文档不够详细**：关于 webhook 外发机制的文档不够详细，需要深入代码才能理解

## 8. 与外部 CRM 系统集成的最佳实践

### 8.1 选择合适的集成方式

- **实时集成**：使用 Postback 信使，将邮件发送事件实时推送到 CRM 系统
- **批量同步**：使用 API 或直接数据库操作，定期同步订阅者数据
- **复杂集成**：结合使用多种方式，如 API 用于实时操作，直接数据库操作用于批量同步

### 8.2 安全考虑

- **API 认证**：使用 API 密钥或 Basic 认证保护 API 端点
- **Postback 认证**：为 Postback 信使配置 Basic 认证，确保只有授权系统能接收数据
- **数据加密**：在传输敏感数据时使用 HTTPS

### 8.3 错误处理

- **API 调用**：实现重试机制和错误处理，处理 API 调用失败的情况
- **Postback 投递**：由于 listmonk 未实现重试机制，外部系统应实现自己的重试逻辑或使用消息队列
- **日志记录**：记录所有集成操作的日志，便于问题排查

### 8.4 性能优化

- **批量操作**：使用 API 的批量操作功能，减少 API 调用次数
- **连接池**：确保外部系统使用连接池处理 Postback 请求
- **异步处理**：对于耗时的集成操作，使用异步处理方式，避免阻塞 listmonk 的正常运行

## 9. 代码参考

### 9.1 核心文件

- **事件系统**：`internal/events/events.go`
- **Postback 信使**：`internal/messenger/postback/postback.go`
- **消息管理器**：`internal/manager/manager.go`
- **订阅者核心逻辑**：`internal/core/subscribers.go`
- **初始化逻辑**：`cmd/init.go`、`cmd/main.go`
- **前端配置界面**：`frontend/src/views/settings/messengers.vue`

### 9.2 关键接口

#### Messenger 接口

```go
// Messenger 是通用消息后端的接口，例如电子邮件、SMS 等
type Messenger interface {
    Name() string
    Push(models.Message) error
    Flush() error
    Close() error
}
```

#### Event 结构体

```go
// Event 表示系统中的单个事件
type Event struct {
    ID       string   `json:"id"`
    Type     string   `json:"type"`
    Message  string   `json:"message"`
    Data     any      `json:"data"`
    Channels []string `json:"-"`
}
```

## 10. 总结

listmonk 的 webhook 外发机制主要通过 Postback 信使实现，它提供了一种将邮件内容和订阅者信息发送到外部系统的方式。虽然目前的实现相对简单，主要用于发送邮件内容，但结合 listmonk 丰富的 API 和直接数据库操作能力，可以实现与外部 CRM 系统的多种集成方式。

对于需要更复杂事件触发机制的场景，可能需要扩展 listmonk 的事件系统，添加更多的业务事件类型，或者使用中间件来捕获和转换事件。

总体而言，listmonk 提供了一个灵活的架构，可以根据具体需求选择合适的集成方式，实现与外部 CRM 系统的有效集成。
