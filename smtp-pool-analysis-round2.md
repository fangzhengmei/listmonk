# Listmonk SMTP 池技术分析报告 - 第二轮补充

## 本文档重点

本文档补充分析以下两个核心问题：
1. **SMTP 返回结果到 Campaign 和订阅者发送状态的持久化回流路径**
2. **多投递商配额是否按 Provider 独立管理，以及命中限额后的处理策略**

---

## 第一部分：发送状态持久化回流路径

### 1.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              发送状态回流完整链路                                    │
└─────────────────────────────────────────────────────────────────────────────────┘

  SMTP Server (投递商)
       │
       │ SMTP 返回结果 (成功/失败)
       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           internal/messenger/email/email.go                      │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │ func (e *Emailer) Push(m models.Message) error                             │  │
│  │   1. 随机选择 SMTP 服务器: srv = e.servers[rand.Intn(ln)]                 │  │
│  │   2. 调用: srv.pool.Send(em)                                               │  │
│  │   3. 返回: error (nil = 成功, 非nil = 失败)                                │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
       │
       │ 返回 error (成功=nil, 失败=error)
       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           internal/manager/manager.go                            │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │ func (m *Manager) worker()                                                  │  │
│  │                                                                              │  │
│  │   // 发送邮件                                                                │  │
│  │   err := m.messengers[msg.Campaign.Messenger].Push(out)                   │  │
│  │   if err != nil {                                                           │  │
│  │       m.log.Printf("error sending message...")                              │  │
│  │   }                                                                          │  │
│  │                                                                              │  │
│  │   // 内存状态更新                                                            │  │
│  │   if err != nil {                                                           │  │
│  │       msg.pipe.OnError()  // 错误计数+1, 超过阈值暂停Campaign               │  │
│  │   } else {                                                                   │  │
│  │       msg.pipe.sent.Add(1)        // 发送计数+1 (atomic)                   │  │
│  │       msg.pipe.rate.Incr(1)        // 速率计数+1                            │  │
│  │       msg.pipe.lastID.Store(...)   // 更新最后处理的订阅者ID                │  │
│  │   }                                                                          │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
       │
       │ 内存状态更新 (pipe.sent, pipe.errors, pipe.lastID)
       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        批量持久化到数据库 (两条路径)                               │
└─────────────────────────────────────────────────────────────────────────────────┘
       │
       ├──────────────────────────────────────┐
       │ 路径1: 周期性批量更新                   │ 路径2: Campaign 结束时最终更新
       ▼                                      ▼
┌──────────────────────────┐        ┌──────────────────────────────────┐
│ scanCampaigns 周期扫描   │        │ pipe.cleanup()                   │
│ (每5秒一次)               │        │ (所有消息处理完成后)              │
│                          │        │                                  │
│ m.getCurrentCampaigns()  │        │ UpdateCampaignCounts(           │
│   → 收集 sent.Load()     │        │   campID,                      │
│   → sent.Store(0) 重置   │        │   0,                           │
│                          │        │   int(p.sent.Load()),          │
│ NextCampaigns() 执行 SQL │        │   int(p.lastID.Load())         │
│   → sent += sent_count   │        │ )                                │
└──────────────────────────┘        └──────────────────────────────────┘
       │                                      │
       └──────────────────────────────────────┘
                      │
                      ▼
              ┌───────────────┐
              │  campaigns 表 │
              │  - sent 字段  │
              │  - last_subscriber_id │
              └───────────────┘
```

### 1.2 发送成功时的回流路径

#### 阶段1：内存计数更新

**代码位置**: `internal/manager/manager.go:528-544`

```go
if msg.pipe != nil {
    msg.pipe.wg.Done()
    
    if err != nil {
        msg.pipe.OnError()
    } else {
        // 成功 - 更新内存状态
        id := uint64(msg.Subscriber.ID)
        if id > msg.pipe.lastID.Load() {
            msg.pipe.lastID.Store(uint64(msg.Subscriber.ID))  // 最后处理的订阅者ID
        }
        msg.pipe.rate.Incr(1)      // 速率计数器 (用于统计)
        msg.pipe.sent.Add(1)        // 发送计数 +1 (atomic.Int64)
    }
}
```

**关键点**：
- `pipe.sent` 是 `atomic.Int64`，保证并发安全
- 发送成功后**立即更新内存**，但不立即写入数据库
- 这是一个**高性能设计** - 避免每条消息都写数据库

#### 阶段2：周期性批量持久化

**代码位置**: `internal/manager/manager.go:560-579`

```go
// getCurrentCampaigns returns the IDs of campaigns currently being processed
// and their sent counts.
func (m *Manager) getCurrentCampaigns() ([]int64, []int64) {
    m.pipesMut.RLock()
    defer m.pipesMut.RUnlock()
    
    var (
        ids    = make([]int64, 0, len(m.pipes))
        counts = make([]int64, 0, len(m.pipes))
    )
    for _, p := range m.pipes {
        ids = append(ids, int64(p.camp.ID))
        // 获取累计发送数，并重置为0
        counts = append(counts, p.sent.Load())
        p.sent.Store(0)  // 重置，下次增量更新
    }
    return ids, counts
}
```

**触发时机**: `scanCampaigns` 每 5 秒调用一次 (`internal/manager/manager.go:422`):
```go
func (m *Manager) scanCampaigns(tick time.Duration) {
    t := time.NewTicker(tick)  // tick = 5秒
    // ...
    for range t.C {
        ids, counts := m.getCurrentCampaigns()  // 收集增量
        campaigns, err := m.store.NextCampaigns(ids, counts)  // SQL 批量更新
        // ...
    }
}
```

#### 阶段3：SQL 增量更新

**代码位置**: `queries/campaigns.sql:216-221`

```sql
WITH uc (campaign_id, sent_count) AS (SELECT * FROM unnest($1::INT[], $2::INT[]))
UPDATE campaigns
SET sent = sent + uc.sent_count  -- 增量累加
FROM uc WHERE campaigns.id = uc.campaign_id
```

**关键点**：
- 使用 `unnest` 将数组展开为表
- `sent = sent + uc.sent_count` - 增量更新，不是覆盖
- 这保证了**高性能**和**数据一致性**

#### 阶段4：Campaign 结束时最终更新

**代码位置**: `internal/manager/pipe.go:187-238`

```go
func (p *pipe) cleanup() {
    // ...
    // Update campaign's 'sent count.
    if err := p.m.store.UpdateCampaignCounts(
        p.camp.ID, 
        0,                    // to_send (不更新)
        int(p.sent.Load()),   // 最后的增量
        int(p.lastID.Load())  // 最后的订阅者ID
    ); err != nil {
        p.m.log.Printf("error updating campaign counts (%s): %v", p.camp.Name, err)
    }
    // ...
}
```

**调用时机**:
- 所有消息处理完成后 (`wg.Wait()` 返回后)
- 或 Campaign 被暂停/取消时

#### 阶段5：UpdateCampaignCounts SQL

**代码位置**: `queries/campaigns.sql:435-441`

```sql
UPDATE campaigns SET
    to_send=(CASE WHEN $2 != 0 THEN $2 ELSE to_send END),
    sent=sent+$3,                              -- 增量更新
    last_subscriber_id=(CASE WHEN $4 > 0 THEN $4 ELSE last_subscriber_id END),
    updated_at=NOW()
WHERE id=$1;
```

### 1.3 发送失败时的回流路径

#### 阶段1：OnError 错误计数

**代码位置**: `internal/manager/pipe.go:136-151`

```go
func (p *pipe) OnError() {
    if p.m.cfg.MaxSendErrors < 1 {
        return  // 禁用错误保护
    }
    
    // 错误计数 +1
    count := p.errors.Add(1)
    if int(count) < p.m.cfg.MaxSendErrors {
        return  // 未达到阈值，继续
    }
    
    // 达到阈值 - 暂停 Campaign
    p.Stop(true)
    p.m.log.Printf("error count exceeded %d. pausing campaign %s", 
                   p.m.cfg.MaxSendErrors, p.camp.Name)
}
```

**关键点**：
- `pipe.errors` 是 `atomic.Uint64`，并发安全
- 错误计数**不持久化到数据库**，只在内存中
- 超过阈值后 `p.Stop(true)` 设置 `withErrors = true`

#### 阶段2：Stop 设置停止标记

**代码位置**: `internal/manager/pipe.go:156-167`

```go
func (p *pipe) Stop(withErrors bool) {
    if p.stopped.Load() {
        return  // 已停止
    }
    
    if withErrors {
        p.withErrors.Store(true)  // 标记为因错误停止
    }
    
    p.stopped.Store(true)  // 标记为已停止
}
```

#### 阶段3：cleanup 持久化状态

**代码位置**: `internal/manager/pipe.go:199-209`

```go
func (p *pipe) cleanup() {
    // ...
    
    // The campaign was auto-paused due to errors.
    if p.withErrors.Load() {
        // 更新状态为 'paused'
        if err := p.m.store.UpdateCampaignStatus(
            p.camp.ID, 
            models.CampaignStatusPaused
        ); err != nil {
            p.m.log.Printf("error updating campaign (%s) status: %v", p.camp.Name, err)
        }
        
        // 发送通知给管理员
        _ = p.m.sendNotif(p.camp, models.CampaignStatusPaused, "Too many errors")
        return
    }
    // ...
}
```

### 1.4 ⚠️ 重要发现：没有逐订阅者发送状态表

#### 当前数据库表分析

| 表名 | 用途 | 是否记录发送结果 |
|------|------|------------------|
| `campaigns` | Campaign 元数据 | `sent` 是**聚合计数**，不是逐订阅者 |
| `subscribers` | 订阅者信息 | 无发送状态字段 |
| `campaign_views` | 邮件打开追踪 | **仅记录打开**，不记录发送结果 |
| `link_clicks` | 链接点击追踪 | **仅记录点击**，不记录发送结果 |
| `bounces` | 退信记录 | **事后记录**，不是实时 SMTP 结果 |

#### 结论

**listmonk 不持久化每条消息的发送结果**。它只追踪：

1. **聚合计数**: `campaigns.sent` - 成功发送的总数量
2. **错误阈值**: 内存中的 `pipe.errors` - 用于判断是否暂停 Campaign
3. **事后追踪**:
   - `campaign_views` - 用户打开邮件时记录（通过 tracking pixel）
   - `link_clicks` - 用户点击链接时记录
   - `bounces` - 退信邮箱扫描或 Webhook 回调时记录

#### 数据回流的不对称性

```
发送路径:
内存计数 (pipe.sent) → 批量 SQL (sent += count) → campaigns.sent (聚合)

缺少的路径:
SMTP 返回结果 → ❌ 无逐订阅者发送状态表
```

**这意味着**：
- 你无法知道 "订阅者 A 的邮件发送成功了吗？"
- 你只能知道 "这个 Campaign 总共发送成功了 N 封"
- 失败的邮件只是被记录到日志，没有持久化到数据库供后续分析

### 1.5 Bounce 处理回流路径

Bounce 是**异步**处理的，不是实时从 SMTP 返回获取的。

#### Bounce 来源

| 来源 | 实现方式 |
|------|----------|
| **邮箱扫描** | POP/IMAP 定期扫描退信邮箱 (`bounce/mailbox/`) |
| **Webhook** | SES, Sendgrid, Postmark, ForwardEmail, Lettermint |

#### Bounce 记录流程

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  投递商退信     │────▶│  bounce.Manager  │────▶│  bounces 表     │
│  (邮箱/Webhook) │     │  .Record()       │     │                 │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                                                     │
                                                     ▼
                                              ┌─────────────────┐
                                              │  配置的动作:    │
                                              │  - none         │
                                              │  - blocklist    │
                                              └─────────────────┘
```

**代码位置**: `internal/core/bounces.go:60-87`

```go
func (c *Core) RecordBounce(b models.Bounce) error {
    action, ok := c.consts.BounceActions[b.Type]
    // ...
    
    _, err := c.q.RecordBounce.Exec(
        b.SubscriberUUID,
        b.Email,
        b.CampaignUUID,
        b.Type,       // soft / hard / complaint
        b.Source,     // 来源: ses, sendgrid, pop3 等
        b.Meta,       // 额外元数据
        b.CreatedAt,
        action.Count, // 触发动作的次数阈值
        action.Action // 动作: none / blocklist
    )
    // ...
}
```

**Bounce 配置** (`schema.sql:286`):
```sql
('bounce.actions', '{"soft": {"count": 2, "action": "none"}, 
                     "hard": {"count": 1, "action": "blocklist"}, 
                     "complaint" : {"count": 1, "action": "blocklist"}}')
```

---

## 第二部分：多投递商配额管理策略

### 2.1 配额管理架构

#### 关键发现：全局配额，不是按 Provider 独立管理

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           当前配额管理架构                                      │
└──────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  Manager (全局)                                                             │
  │  ┌────────────────────────────────────────────────────────────────────┐  │
  │  │  全局配额配置 (所有 Provider 共享)                                    │  │
  │  │                                                                      │  │
  │  │  cfg.MessageRate          = 10    // 每秒 10 条 (全局)              │  │
  │  │  cfg.SlidingWindow        = true  // 启用滑动窗口                    │  │
  │  │  cfg.SlidingWindowRate    = 10000 // 窗口内 10000 条 (全局)        │  │
  │  │  cfg.SlidingWindowDuration = 1h   // 窗口时长 1 小时                 │  │
  │  │                                                                      │  │
  │  │  // 全局状态                                                          │  │
  │  │  slidingCount  int           // 当前窗口已发送数 (所有 Provider)     │  │
  │  │  slidingStart  time.Time     // 窗口开始时间                          │  │
  │  └────────────────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            ▼                       ▼                       ▼
    ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
    │   SMTP #1    │        │   SMTP #2    │        │   SMTP #N    │
    │  (Provider A)│        │  (Provider B)│        │  (Provider C)│
    │              │        │              │        │              │
    │  无独立配额   │        │  无独立配额   │        │  无独立配额   │
    │  仅连接池配置 │        │  仅连接池配置 │        │  仅连接池配置 │
    └──────────────┘        └──────────────┘        └──────────────┘
```

### 2.2 配额配置详解

#### 配置项位置

| 配置项 | TOML 路径 | 类型 | 作用范围 |
|--------|-----------|------|----------|
| `message_rate` | `app.message_rate` | int | **全局** - 每秒发送数 |
| `concurrency` | `app.concurrency` | int | **全局** - 并发 Worker 数 |
| `max_send_errors` | `app.max_send_errors` | int | **全局** - 错误阈值 |
| `message_sliding_window` | `app.message_sliding_window` | bool | **全局** - 是否启用滑动窗口 |
| `message_sliding_window_rate` | `app.message_sliding_window_rate` | int | **全局** - 窗口内最大发送数 |
| `message_sliding_window_duration` | `app.message_sliding_window_duration` | duration | **全局** - 窗口时长 |

#### SMTP 服务器级别的配置（非配额）

每个 SMTP 服务器有以下配置，但这些是**连接池配置**，不是发送配额：

**代码位置**: `models/settings.go:83-101`

```go
SMTP []struct {
    // ...
    MaxConns          int    `json:"max_conns"`      // 最大连接数
    MaxMsgRetries     int    `json:"max_msg_retries"` // 单条消息最大重试次数
    MsgRetryDelay     string `json:"msg_retry_delay"` // 重试延迟
    IdleTimeout       string `json:"idle_timeout"`    // 空闲连接超时
    WaitTimeout       string `json:"wait_timeout"`    // 获取连接等待超时
    // ...
}
```

**关键点**：
- `MaxConns` 限制的是**并发连接数**，不是发送速率
- `MaxMsgRetries` 是单条消息的重试次数，不是配额
- **没有任何按 SMTP 服务器的发送速率限制**

### 2.3 固定速率限制实现

**代码位置**: `internal/manager/manager.go:464-485`

```go
func (m *Manager) worker() {
    numMsg := 0  // 每个 Worker 独立计数
    for {
        select {
        case msg, ok := <-m.campMsgQ:
            // ... 处理消息
            
            // Pause on hitting the message rate.
            if numMsg >= m.cfg.MessageRate {
                time.Sleep(time.Second)  // 休眠 1 秒
                numMsg = 0
            }
            numMsg++
            // ...
        }
    }
}
```

**关键点**：
- `numMsg` 是**每个 Worker** 的局部变量
- 如果 `concurrency=10`，实际速率是 `10 * MessageRate`
- **全局共享速率限制**，不区分是哪个 SMTP Provider

### 2.4 滑动窗口限制实现

**代码位置**: `internal/manager/pipe.go:89-130`

```go
func (p *pipe) NextSubscribers() (bool, error) {
    // Is there a sliding window limit configured?
    hasSliding := p.m.cfg.SlidingWindow &&
        p.m.cfg.SlidingWindowRate > 0 &&
        p.m.cfg.SlidingWindowDuration.Seconds() > 1

    for _, s := range subs {
        // ... 推送消息到队列
        
        // 检查滑动窗口
        if hasSliding {
            diff := time.Since(p.m.slidingStart)  // m 是 Manager，全局
            
            // Window has expired. Reset the clock.
            if diff >= p.m.cfg.SlidingWindowDuration {
                p.m.slidingStart = time.Now()    // 全局状态
                p.m.slidingCount = 0              // 全局状态
            }
            
            // Have the messages exceeded the limit?
            p.m.slidingCount++                    // 全局计数++
            if p.m.slidingCount >= p.m.cfg.SlidingWindowRate {
                wait := p.m.cfg.SlidingWindowDuration - diff
                
                p.m.log.Printf("messages exceeded (%d) for the window (%v). Sleeping for %s.",
                    p.m.slidingCount,
                    p.m.cfg.SlidingWindowDuration,
                    wait)
                
                p.m.slidingCount = 0
                time.Sleep(wait)  // 休眠等待窗口重置
            }
        }
    }
}
```

**滑动窗口状态定义** (`internal/manager/manager.go:91-92`):

```go
type Manager struct {
    // ...
    slidingCount int           // 全局 - 当前窗口已发送数
    slidingStart time.Time     // 全局 - 窗口开始时间
    // ...
}
```

**关键点**：
- `slidingCount` 和 `slidingStart` 是 `Manager` 结构体的字段
- 这意味着**所有 Campaign、所有 SMTP Provider 共享同一个滑动窗口**
- 即使配置了多个 SMTP 服务器，它们的发送量会被累加计算

### 2.5 命中限额后的处理策略

#### 策略1：固定速率命中 - Sleep 1 秒

**代码位置**: `internal/manager/manager.go:482-484`

```go
if numMsg >= m.cfg.MessageRate {
    time.Sleep(time.Second)  // 简单休眠
    numMsg = 0
}
```

**行为**：
- 当 Worker 在 1 秒内发送了 `MessageRate` 条消息
- 休眠 1 秒，然后继续
- **不会切换到其他 Provider**
- **不会降级处理**

#### 策略2：滑动窗口命中 - Sleep 直到窗口重置

**代码位置**: `internal/manager/pipe.go:118-127`

```go
if p.m.slidingCount >= p.m.cfg.SlidingWindowRate {
    wait := p.m.cfg.SlidingWindowDuration - diff
    
    p.m.log.Printf("messages exceeded (%d) for the window (%v since %s). Sleeping for %s.",
        p.m.slidingCount,
        p.m.cfg.SlidingWindowDuration,
        p.m.slidingStart.Format(time.RFC822Z),
        wait)
    
    p.m.slidingCount = 0
    time.Sleep(wait)  // 可能休眠很长时间！
}
```

**行为**：
- 计算需要等待的时间：`窗口时长 - 已过时间`
- 例如：窗口 1 小时，已过 10 分钟，达到速率限制 → **休眠 50 分钟**
- **不会尝试使用其他 Provider**
- **整个 Campaign 处理被阻塞**

### 2.6 ⚠️ 配额管理的严重局限性

#### 问题1：全局配额不考虑多 Provider 能力

```
场景示例：
- 配置 2 个 SMTP Provider：
  - Provider A: SendGrid，配额 100,000/小时
  - Provider B: Mailgun，配额 100,000/小时
- 理论总能力：200,000/小时

当前行为：
- 滑动窗口配置 100,000/小时
- 实际上只使用了总能力的 50%
- 因为两个 Provider 共享同一个全局计数器
```

#### 问题2：配额命中后不会切换 Provider

```
场景示例：
- 配置 2 个 SMTP Provider
- Provider A 暂时不可用（或达到自身配额）
- SMTP 返回错误

当前行为：
1. 随机选择 Provider A
2. 发送失败，错误计数 +1
3. 下次发送仍然是**随机选择**
   - 可能继续选到失败的 Provider A
4. 错误累积到 MaxSendErrors
5. **暂停整个 Campaign** ❌
6. 不会尝试切换到 Provider B
```

#### 问题3：连接池配置不影响发送决策

```go
// 每个 SMTP 服务器可以配置独立的 MaxConns
Server struct {
    // ...
    smtppool.Opt `json:",squash"`  // 包含 MaxConns
    // ...
}
```

但 `MaxConns` 只影响：
- 同时建立多少个 TCP 连接
- 不会影响**发送速率**的决策
- 不会因为某个 Provider 连接池满了就切换到其他 Provider

### 2.7 与 Round1 发现的关联

回顾 Round1 的发现：

| 发现 | 与配额管理的关系 |
|------|------------------|
| 随机选择，不故障切换 | 配额命中后也是同样的策略 - 不切换 |
| 错误超过阈值暂停 Campaign | 配额命中也是暂停/休眠，不是切换 |
| 无健康检查 | 也不会检查 Provider 是否达到自身配额 |

**设计哲学一致**：
- listmonk 的多 SMTP 设计目标是**负载均衡**（随机选择），不是**高可用**（故障切换）
- 配额管理是**全局流控**，不是**多 Provider 协同**

---

## 第三部分：架构总结与改进建议

### 3.1 当前架构完整视图

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                        Listmonk 发送架构完整视图                                 │
└────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │                              Campaign Manager                                   │
  │                                                                                │
  │  ┌────────────────────────────────────────────────────────────────────────┐  │
  │  │                        全局配额 (所有 Provider 共享)                      │  │
  │  │  ┌─────────────────┐    ┌──────────────────────────────────────────┐   │  │
  │  │  │ MessageRate     │    │ SlidingWindow (slidingCount, slidingStart)│   │  │
  │  │  │ (每秒发送数)     │    │ (窗口内发送数 + 窗口开始时间)              │   │  │
  │  │  └─────────────────┘    └──────────────────────────────────────────┘   │  │
  │  │                                                                          │  │
  │  │  命中后行为: time.Sleep() - 不会切换 Provider                            │  │
  │  └────────────────────────────────────────────────────────────────────────┘  │
  │                                                                                │
  │  ┌────────────────────────────────────────────────────────────────────────┐  │
  │  │                            Worker 池                                      │  │
  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                  │  │
  │  │  │ Worker 1 │ │ Worker 2 │ │ Worker 3 │ │ Worker N │                  │  │
  │  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘                  │  │
  │  └───────┼─────────────┼─────────────┼─────────────┼────────────────────────┘  │
  └──────────┼─────────────┼─────────────┼─────────────┼───────────────────────────┘
             │             │             │             │
             │    随机选择   │             │             │
             ▼             ▼             ▼             ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │                            Emailer (Messenger)                                │
  │                                                                                │
  │  srv = e.servers[rand.Intn(ln)]  // 随机选择，无故障切换                      │
  │                                                                                │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                          │
  │  │  Server 1   │  │  Server 2   │  │  Server N   │                          │
  │  │             │  │             │  │             │                          │
  │  │ smtppool    │  │ smtppool    │  │ smtppool    │                          │
  │  │ - MaxConns  │  │ - MaxConns  │  │ - MaxConns  │                          │
  │  │ (连接池配置) │  │ (连接池配置) │  │ (连接池配置) │                          │
  │  │             │  │             │  │             │                          │
  │  │ 无独立配额   │  │ 无独立配额   │  │ 无独立配额   │                          │
  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                          │
  └─────────┼────────────────┼────────────────┼───────────────────────────────────┘
            │                │                │
            ▼                ▼                ▼
  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
  │ SMTP Provider│  │ SMTP Provider│  │ SMTP Provider│
  │    (A)      │  │    (B)      │  │    (C)      │
  └─────────────┘  └─────────────┘  └─────────────┘
```

### 3.2 状态持久化总结

| 数据类型 | 持久化方式 | 存储位置 | 实时性 |
|----------|------------|----------|--------|
| 发送成功计数 | 内存计数 + 批量 SQL | `campaigns.sent` | 延迟 (5秒/批量) |
| 最后处理的订阅者ID | 内存 + 最终更新 | `campaigns.last_subscriber_id` | 延迟 |
| 错误计数 | 仅内存 | ❌ 不持久化 | N/A |
| Campaign 状态 | 状态变更时 SQL | `campaigns.status` | 状态变更时 |
| 逐订阅者发送结果 | ❌ 不记录 | 无 | N/A |
| 邮件打开 (追踪) | 点击时 SQL | `campaign_views` | 实时 (用户打开时) |
| 链接点击 (追踪) | 点击时 SQL | `link_clicks` | 实时 (用户点击时) |
| 退信 (Bounce) | 邮箱扫描/Webhook | `bounces` | 异步 |

### 3.3 配额管理总结

| 维度 | 当前实现 | 期望的高可用实现 |
|------|----------|------------------|
| 配额范围 | 全局共享 | 按 Provider 独立 |
| 速率限制 | Manager 全局 | 每个 SMTP Server 独立 |
| 滑动窗口 | Manager 全局 | 每个 SMTP Server 独立 |
| 配额命中后 | Sleep 等待 | 切换到其他 Provider |
| 错误处理 | 暂停 Campaign | 隔离故障 Provider |
| 健康检查 | 无 | 检查 Provider 可用性 |

### 3.4 改进建议

#### 建议1：按 Provider 独立配额管理

```go
type Server struct {
    Name          string
    // ... 现有配置
    
    // 新增：独立配额配置
    RateLimit     int           // 每秒发送限制
    DailyQuota    int           // 日配额
    HourlyQuota   int           // 小时配额
    
    // 新增：运行时状态
    rateCount     atomic.Int64   // 当前秒计数
    windowCount   atomic.Int64   // 当前窗口计数
    windowStart   atomic.Value    // time.Time
    lastError     atomic.Value    // 最后错误时间
    healthStatus  atomic.Bool     // 健康状态
}
```

#### 建议2：配额/错误命中后切换 Provider

```go
func (e *Emailer) PushWithFailover(m models.Message) error {
    servers := e.getHealthyServers()  // 只选择健康的
    if len(servers) == 0 {
        return errors.New("no healthy SMTP servers available")
    }
    
    // 按配额剩余量排序，优先选择配额充足的
    sort.Slice(servers, func(i, j int) bool {
        return servers[i].remainingQuota() > servers[j].remainingQuota()
    })
    
    var lastErr error
    for _, srv := range servers {
        // 检查是否有可用配额
        if !srv.hasQuota() {
            continue
        }
        
        // 尝试发送
        err := srv.pool.Send(em)
        if err == nil {
            srv.consumeQuota()  // 消耗配额
            return nil
        }
        
        lastErr = err
        srv.recordError(err)  // 记录错误，可能标记为不健康
    }
    
    return lastErr
}
```

#### 建议3：逐订阅者发送状态表

```sql
-- 建议新增的表
CREATE TABLE campaign_sends (
    id               BIGSERIAL PRIMARY KEY,
    campaign_id      INTEGER NOT NULL REFERENCES campaigns(id),
    subscriber_id    INTEGER NOT NULL REFERENCES subscribers(id),
    smtp_server      TEXT NOT NULL,              -- 使用的 SMTP Provider
    status           TEXT NOT NULL,              -- 'sent' / 'failed' / 'bounced'
    error_message    TEXT,                       -- 失败时的错误信息
    attempts         INTEGER NOT NULL DEFAULT 1, -- 尝试次数
    sent_at          TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_at       TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_sends_camp_id ON campaign_sends(campaign_id);
CREATE INDEX idx_sends_sub_id ON campaign_sends(subscriber_id);
CREATE INDEX idx_sends_status ON campaign_sends(status);
```

#### 建议4：更细粒度的错误分类处理

```go
type SendError struct {
    Type       ErrorType  // Temporary / Permanent / QuotaExceeded / AuthFailed
    Provider   string     // 哪个 Provider
    Message    string
    Retryable  bool
}

type ErrorType string

const (
    ErrorTemporary    ErrorType = "temporary"    // 网络超时等，可重试同一 Provider
    ErrorPermanent    ErrorType = "permanent"    // 无效地址等，不应重试
    ErrorQuotaExceeded ErrorType = "quota"        // 配额超限，切换 Provider
    ErrorAuthFailed   ErrorType = "auth"          // 认证失败，标记为不健康
    ErrorConnection   ErrorType = "connection"     // 连接失败，标记为不健康
)
```

---

## 附录：关键代码位置速查

| 功能 | 文件路径 | 行号范围 |
|------|----------|----------|
| 发送成功后内存计数 | `internal/manager/manager.go` | 536-544 |
| 发送失败 OnError | `internal/manager/pipe.go` | 136-151 |
| 周期性批量更新计数 | `internal/manager/manager.go` | 560-579 |
| Campaign 结束更新计数 | `internal/manager/pipe.go` | 194-197 |
| 固定速率限制 | `internal/manager/manager.go` | 482-484 |
| 滑动窗口限制 | `internal/manager/pipe.go` | 89-130 |
| 滑动窗口全局状态 | `internal/manager/manager.go` | 91-92 |
| UpdateCampaignCounts SQL | `queries/campaigns.sql` | 435-441 |
| NextCampaigns 增量更新 | `queries/campaigns.sql` | 216-221 |
| Bounce 记录 | `internal/core/bounces.go` | 60-87 |

---

**分析日期**：2026-05-05  
**分析版本**：基于当前代码库  
**与 Round1 的关系**：补充 Round1 未覆盖的状态回流路径和配额管理细节

## 关键结论汇总

1. **状态回流**：
   - ✅ 发送成功计数通过「内存原子计数 + 批量 SQL 增量更新」实现高性能持久化
   - ❌ **没有逐订阅者的发送状态表**，只记录聚合计数
   - ❌ 错误计数只在内存中，不持久化
   - ✅ Bounce 通过邮箱扫描/Webhook 异步记录

2. **配额管理**：
   - ❌ **全局配额**，不是按 Provider 独立管理
   - ❌ 固定速率和滑动窗口都是 Manager 级别的全局状态
   - ❌ 命中配额后 `time.Sleep()`，**不会切换到其他 Provider**
   - ⚠️ 配置多个 SMTP 但共享配额，实际上限制了总吞吐量

3. **设计哲学**：
   - 多 SMTP = 负载均衡（随机选择），**不是** 高可用（故障切换）
   - 配额 = 全局流控，**不是** 多 Provider 协同
   - 错误处理 = 暂停 Campaign，**不是** 隔离/降级
