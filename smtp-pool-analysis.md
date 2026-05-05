# Listmonk SMTP 池技术分析报告

## 1. 概述

本文档详细分析 listmonk 邮件投递系统中多 SMTP 服务器池的实现机制，包括配置方式、故障切换策略、配额管理和投递状态跟踪。

---

## 2. 多 SMTP 服务器配置机制

### 2.1 配置结构

SMTP 服务器配置在 `settings.go` 中定义，支持配置多个 SMTP 服务器：

```go
// models/settings.go:83-101
SMTP []struct {
    Name              string              `json:"name"`
    UUID              string              `json:"uuid"`
    Enabled           bool                `json:"enabled"`
    Host              string              `json:"host"`
    HelloHostname     string              `json:"hello_hostname"`
    Port              int                 `json:"port"`
    AuthProtocol      string              `json:"auth_protocol"`
    Username          string              `json:"username"`
    Password          string              `json:"password,omitempty"`
    EmailHeaders      []map[string]string `json:"email_headers"`
    MaxConns          int                 `json:"max_conns"`
    MaxMsgRetries     int                 `json:"max_msg_retries"`
    MsgRetryDelay     string              `json:"msg_retry_delay"`
    IdleTimeout       string              `json:"idle_timeout"`
    WaitTimeout       string              `json:"wait_timeout"`
    TLSType           string              `json:"tls_type"`
    TLSSkipVerify     bool                `json:"tls_skip_verify"`
} `json:"smtp"`
```

### 2.2 初始化机制

在 `cmd/init.go:664-712` 的 `initSMTPMessengers()` 函数中实现初始化：

**初始化流程：**
1. 遍历所有 `smtp` 配置项
2. 跳过 `enabled=false` 的服务器
3. 为每个启用的服务器创建独立的 `email.Server` 实例
4. **双重注册机制**：
   - 如果服务器有 `Name`，注册为独立 messenger（名称为 `email/{name}`）
   - 同时创建一个聚合 messenger（名称为 `email`），包含所有 SMTP 服务器

**关键代码片段：**
```go
// cmd/init.go:686-694
// If the server has a name, initialize it as a standalone e-mail messenger
// allowing campaigns to select individual SMTPs. In the UI and config, it'll appear as `email / $name`.
if s.Name != "" {
    msgr, err := email.New(s.Name, s)
    if err != nil {
        lo.Fatalf("error initializing e-mail messenger: %v", err)
    }
    out = append(out, msgr)
}

// cmd/init.go:697-701
// Initialize the 'email' messenger with all SMTP servers.
msgr, err := email.New(email.MessengerName, servers...)
if err != nil {
    lo.Fatalf("error initializing e-mail messenger: %v", err)
}
```

### 2.3 Server 结构体

每个 SMTP 服务器在 `email.go` 中表示：

```go
// internal/messenger/email/email.go:27-43
type Server struct {
    Name          string            `json:"name"`
    Username      string            `json:"username"`
    Password      string            `json:"password"`
    AuthProtocol  string            `json:"auth_protocol"`
    TLSType       string            `json:"tls_type"`
    TLSSkipVerify bool              `json:"tls_skip_verify"`
    EmailHeaders  map[string]string `json:"email_headers"`

    smtppool.Opt `json:",squash"`
    pool *smtppool.Pool
}
```

**关键点：**
- 每个 Server 拥有独立的 `smtppool.Pool` 实例
- 使用 `smtppool/v2` 库管理连接池
- 支持的认证协议：`cram` (CRAM-MD5)、`plain` (PLAIN)、`login` (LOGIN)、`none`

---

## 3. 故障切换策略（重要发现）

### 3.1 ⚠️ 核心机制：随机选择，而非故障切换

**这是本分析的最重要发现**：listmonk 的多 SMTP 服务器池**并不实现自动故障切换**，而是采用**随机选择**策略。

**关键实现代码** (`internal/messenger/email/email.go:114-126`)：

```go
func (e *Emailer) Push(m models.Message) error {
    // If there are more than one SMTP servers, send to a random
    // one from the list.
    var (
        ln  = len(e.servers)
        srv *Server
    )
    if ln > 1 {
        srv = e.servers[rand.Intn(ln)]  // 随机选择！
    } else {
        srv = e.servers[0]
    }
    // ... 构建邮件
    return srv.pool.Send(em)  // 直接返回错误，不重试其他服务器
}
```

### 3.2 策略分析

| 特性 | 实际行为 |
|------|----------|
| 服务器选择 | 随机选择 (`rand.Intn()`) |
| 失败重试 | **无** - 错误直接返回 |
| 故障检测 | **无** - 没有健康检查机制 |
| 自动切换 | **无** - 不会尝试其他服务器 |

### 3.3 错误处理流程

当邮件发送失败时，错误处理流程如下（`internal/manager/manager.go:521-544`）：

```go
// Push the message to the messenger.
err := m.messengers[msg.Campaign.Messenger].Push(out)
if err != nil {
    m.log.Printf("error sending message in campaign %s: subscriber %d: %v", 
                 msg.Campaign.Name, msg.Subscriber.ID, err)
}

// Increment the send rate or the error counter if there was an error.
if msg.pipe != nil {
    msg.pipe.wg.Done()
    if err != nil {
        // Call the error callback, which keeps track of the error count
        // and stops the campaign if the error count exceeds the threshold.
        msg.pipe.OnError()  // 仅记录错误，不切换 SMTP
    } else {
        // 成功处理
        msg.pipe.sent.Add(1)
    }
}
```

### 3.4 OnError 机制

`OnError()` 函数的作用是**暂停 Campaign**，而非切换 SMTP 服务器：

```go
// internal/manager/pipe.go:136-151
func (p *pipe) OnError() {
    if p.m.cfg.MaxSendErrors < 1 {
        return
    }

    // 如果错误阈值达到，暂停 campaign
    count := p.errors.Add(1)
    if int(count) < p.m.cfg.MaxSendErrors {
        return
    }

    p.Stop(true)  // 停止整个 campaign！
    p.m.log.Printf("error count exceeded %d. pausing campaign %s", 
                   p.m.cfg.MaxSendErrors, p.camp.Name)
}
```

**行为总结：**
1. 每次发送失败，错误计数器 `errors` 加 1
2. 当错误数达到 `MaxSendErrors` 阈值时
3. **暂停整个 Campaign**，而不是切换 SMTP 服务器
4. 发送通知邮件给管理员

### 3.5 配置项：MaxSendErrors

`MaxSendErrors` 是控制错误容忍度的关键配置：

```go
// internal/manager/manager.go:597
MaxSendErrors: ko.Int("app.max_send_errors"),
```

**作用：**
- 当连续（或累计）错误数达到此值时，Campaign 被暂停
- 设置为 0 或负数时，禁用此保护机制

---

## 4. 配额管理机制

### 4.1 发送速率控制

listmonk 实现了**两层**发送速率控制：

#### 层1：固定速率限制 (`MessageRate`)

```go
// internal/manager/manager.go:464-485
func (m *Manager) worker() {
    numMsg := 0
    for {
        select {
        case msg, ok := <-m.campMsgQ:
            // ... 处理消息
            
            // Pause on hitting the message rate.
            if numMsg >= m.cfg.MessageRate {
                time.Sleep(time.Second)  // 每秒限制
                numMsg = 0
            }
            numMsg++
            // ...
        }
    }
}
```

**配置项：** `app.message_rate` - 每秒最大发送消息数

#### 层2：滑动窗口限制 (`SlidingWindow`)

```go
// internal/manager/pipe.go:89-130
func (p *pipe) NextSubscribers() (bool, error) {
    // Is there a sliding window limit configured?
    hasSliding := p.m.cfg.SlidingWindow &&
        p.m.cfg.SlidingWindowRate > 0 &&
        p.m.cfg.SlidingWindowDuration.Seconds() > 1

    for _, s := range subs {
        // ... 推送消息到队列
        
        // Check if the sliding window is active.
        if hasSliding {
            diff := time.Since(p.m.slidingStart)
            
            // Window has expired. Reset the clock.
            if diff >= p.m.cfg.SlidingWindowDuration {
                p.m.slidingStart = time.Now()
                p.m.slidingCount = 0
            }
            
            // Have the messages exceeded the limit?
            p.m.slidingCount++
            if p.m.slidingCount >= p.m.cfg.SlidingWindowRate {
                wait := p.m.cfg.SlidingWindowDuration - diff
                p.m.log.Printf("messages exceeded (%d) for the window (%v). Sleeping for %s.",
                    p.m.slidingCount, p.m.cfg.SlidingWindowDuration, wait)
                p.m.slidingCount = 0
                time.Sleep(wait)  // 睡眠等待窗口重置
            }
        }
    }
}
```

**滑动窗口配置项：**
| 配置项 | 说明 |
|--------|------|
| `app.message_sliding_window` | 是否启用滑动窗口 |
| `app.message_sliding_window_duration` | 窗口时长（如 "1h"） |
| `app.message_sliding_window_rate` | 窗口内最大发送数 |

### 4.2 SMTP 连接池配置

每个 SMTP 服务器有独立的连接池配置（通过 `smtppool.Opt`）：

| 配置项 | 说明 |
|--------|------|
| `max_conns` | 最大连接数 |
| `max_msg_retries` | 单条消息最大重试次数 |
| `msg_retry_delay` | 重试延迟 |
| `idle_timeout` | 空闲连接超时 |
| `wait_timeout` | 获取连接等待超时 |

---

## 5. 投递状态跟踪

### 5.1 Pipe 状态追踪

每个运行中的 Campaign 对应一个 `pipe` 实例，跟踪投递状态：

```go
// internal/manager/pipe.go:13-24
type pipe struct {
    camp       *models.Campaign
    rate       *ratecounter.RateCounter  // 发送速率计数器（每分钟）
    wg         *sync.WaitGroup            // 等待所有消息处理完成
    sent       atomic.Int64                // 已发送消息数
    lastID     atomic.Uint64               // 最后处理的订阅者 ID
    errors     atomic.Uint64               // 错误计数
    stopped    atomic.Bool                  // 是否已停止
    withErrors atomic.Bool                  // 是否因错误停止
    m *Manager
}
```

### 5.2 状态更新流程

**发送成功时：**
```go
// internal/manager/manager.go:536-542
if err != nil {
    msg.pipe.OnError()
} else {
    id := uint64(msg.Subscriber.ID)
    if id > msg.pipe.lastID.Load() {
        msg.pipe.lastID.Store(uint64(msg.Subscriber.ID))
    }
    msg.pipe.rate.Incr(1)      // 速率计数+1
    msg.pipe.sent.Add(1)        // 发送计数+1
}
```

**发送失败时：**
```go
// internal/manager/pipe.go:138-150
func (p *pipe) OnError() {
    count := p.errors.Add(1)  // 错误计数+1
    if int(count) < p.m.cfg.MaxSendErrors {
        return
    }
    p.Stop(true)  // 超过阈值，停止 Campaign
}
```

### 5.3 Campaign 状态流转

| 状态 | 触发条件 |
|------|----------|
| `running` | Campaign 开始运行 |
| `paused` | 手动暂停 或 错误数超过 `MaxSendErrors` |
| `cancelled` | 手动取消 |
| `finished` | 所有订阅者处理完成 |

### 5.4 数据库持久化

Campaign 状态通过以下函数持久化到数据库：

```go
// internal/manager/manager.go:36-45
type Store interface {
    UpdateCampaignStatus(campID int, status string) error
    UpdateCampaignCounts(campID int, toSend int, sent int, lastSubID int) error
    // ...
}
```

---

## 6. 架构总结

### 6.1 组件关系图

```
┌─────────────────────────────────────────────────────────────┐
│                      Campaign Manager                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │   Worker 1   │    │   Worker 2   │    │   Worker N   │ │
│  │  (并发发送)   │    │  (并发发送)   │    │  (并发发送)   │ │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘ │
└─────────┼───────────────────┼───────────────────┼──────────┘
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                    Emailer (Messenger)                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              服务器选择：随机选择 (rand.Intn)         │   │
│  └─────────────────────────────────────────────────────┘   │
│         │              │              │                      │
│         ▼              ▼              ▼                      │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│  │ Server 1 │   │ Server 2 │   │ Server N │               │
│  │ smtppool │   │ smtppool │   │ smtppool │               │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘               │
└───────┼───────────────┼───────────────┼─────────────────────┘
        │               │               │
        ▼               ▼               ▼
   ┌─────────┐     ┌─────────┐     ┌─────────┐
   │ SMTP #1 │     │ SMTP #2 │     │ SMTP #N │
   │ (投递商) │     │ (投递商) │     │ (投递商) │
   └─────────┘     └─────────┘     └─────────┘
```

### 6.2 关键设计决策

| 决策点 | 设计选择 | 影响 |
|--------|----------|------|
| 多 SMTP 选择策略 | 随机选择 | 简单但无故障转移 |
| 错误处理 | 计数 + 暂停 Campaign | 保守策略，防止资源浪费 |
| 速率控制 | 固定速率 + 滑动窗口 | 灵活的配额管理 |
| 状态跟踪 | 内存原子操作 + 数据库持久化 | 高性能 + 可恢复 |

---

## 7. 局限性与改进建议

### 7.1 当前实现的局限性

1. **无自动故障切换**：当某个 SMTP 服务器故障时，不会自动切换到其他服务器
2. **无健康检查**：没有机制检测 SMTP 服务器的可用性
3. **随机选择可能导致问题**：故障服务器可能被持续随机选中，增加错误率
4. **错误处理过于激进**：错误超过阈值直接暂停整个 Campaign，而不是降级处理

### 7.2 改进建议

#### 建议1：实现故障切换机制

```go
// 伪代码示例
func (e *Emailer) PushWithFailover(m models.Message) error {
    servers := make([]*Server, len(e.servers))
    copy(servers, e.servers)
    
    // 随机打乱顺序，实现负载均衡
    rand.Shuffle(len(servers), func(i, j int) {
        servers[i], servers[j] = servers[j], servers[i]
    })
    
    var lastErr error
    for _, srv := range servers {
        err := srv.pool.Send(em)
        if err == nil {
            return nil
        }
        lastErr = err
        // 记录服务器故障，可用于健康状态跟踪
    }
    return lastErr
}
```

#### 建议2：实现 SMTP 服务器健康检查

- 维护每个服务器的故障计数器
- 故障超过阈值时，暂时将服务器移出可用池
- 定期尝试恢复健康状态

#### 建议3：更精细的错误分类

区分不同类型的错误：
- **临时错误**（如网络超时）：可重试同一服务器
- **永久错误**（如认证失败）：需要切换服务器或人工干预
- **配额超限**：切换到其他投递商

---

## 8. 附录：关键代码位置速查

| 功能 | 文件路径 | 行号范围 |
|------|----------|----------|
| SMTP 配置结构体 | `models/settings.go` | 83-101 |
| SMTP 初始化 | `cmd/init.go` | 664-712 |
| 邮件推送（随机选择） | `internal/messenger/email/email.go` | 114-207 |
| Worker 错误处理 | `internal/manager/manager.go` | 521-544 |
| OnError 暂停逻辑 | `internal/manager/pipe.go` | 136-151 |
| 滑动窗口限流 | `internal/manager/pipe.go` | 89-130 |
| Pipe 状态结构体 | `internal/manager/pipe.go` | 13-24 |

---

**分析日期**：2026-05-05  
**分析版本**：基于当前代码库  
**结论**：listmonk 支持多 SMTP 服务器配置，但采用**随机选择**策略，**不支持自动故障切换**。投递失败时错误累积到阈值会暂停 Campaign，需要人工干预恢复。
