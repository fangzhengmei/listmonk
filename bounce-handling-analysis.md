# Listmonk 退信处理机制分析报告

## 1. 整体架构概览

Listmonk 的退信处理系统采用了**多入口、统一处理**的架构设计。退信事件可以通过以下几种方式进入系统：

1. **POP3 邮箱扫描** - 定时扫描 POP3 邮箱中的退信邮件
2. **IMAP 邮箱扫描** - 配置结构已定义，但当前实现仅支持 POP3
3. **Webhook 接口** - 支持多种邮件服务提供商的 webhook：
   - Amazon SES
   - Sendgrid
   - Postmark
   - ForwardEmail
   - Lettermint
4. **原生 API** - 自定义脚本可通过 `/webhooks/bounce` 直接推送退信事件

### 核心组件

| 组件 | 文件位置 | 职责 |
|------|----------|------|
| `bounce.Manager` | `internal/bounce/bounce.go` | 退信管理器，协调所有入口和处理流程 |
| `mailbox.POP` | `internal/bounce/mailbox/pop.go` | POP3 邮箱扫描实现 |
| `webhooks.SES` 等 | `internal/bounce/webhooks/` | 各服务商 webhook 处理器 |
| `core.RecordBounce` | `internal/core/bounces.go` | 退信记录和订阅者状态更新 |
| SQL Queries | `queries/bounces.sql` | 数据库操作，包含状态回写逻辑 |

---

## 2. POP3 退信处理流程

### 2.1 初始化与启动

POP3 邮箱扫描在 `initBounceManager` 函数中初始化：

**文件**: `cmd/init.go:819-875`

```go
func initBounceManager(cb func(models.Bounce) error, stmt *sqlx.Stmt, lo *log.Logger, ko *koanf.Koanf) *bounce.Manager {
    // 1. 读取配置：bounce.mailboxes 数组
    // 2. 目前只支持一个邮箱配置
    // 3. 根据 type 字段决定使用 POP 还是 IMAP（当前仅实现 POP）
}
```

启动流程在 `bounce.Manager.Run()` 中：

**文件**: `internal/bounce/bounce.go:118-132`

```go
func (m *Manager) Run() {
    if m.opt.MailboxEnabled {
        go m.runMailboxScanner()  // 启动后台扫描协程
    }
    
    // 主循环：从队列消费退信事件
    for b := range m.queue {
        if err := m.opt.RecordBounceCB(b); err != nil {
            continue
        }
    }
}
```

### 2.2 定时扫描机制

**文件**: `internal/bounce/bounce.go:135-144`

```go
func (m *Manager) runMailboxScanner() {
    for {
        m.log.Printf("scanning bounce mailbox %s", m.opt.Mailbox.Host)
        if err := m.mailbox.Scan(1000, m.queue); err != nil {
            m.log.Printf("error scanning bounce mailbox: %v", err)
        }
        time.Sleep(m.opt.Mailbox.ScanInterval)  // 配置的扫描间隔
    }
}
```

### 2.3 邮件扫描与解析流程

**文件**: `internal/bounce/mailbox/pop.go:79-208`

POP3 扫描的详细步骤：

1. **建立连接与认证**
   ```go
   c, err := p.client.NewConn()
   defer c.Quit()
   
   if p.opt.AuthProtocol != "none" {
       if err := c.Auth(p.opt.Username, p.opt.Password); err != nil {
           return err
       }
   }
   ```

2. **获取邮件数量**
   ```go
   count, _, err := c.Stat()
   ```

3. **下载并解析每封邮件**
   - 使用 `github.com/emersion/go-message` 解析邮件
   - 处理 multipart 邮件
   - 提取关键头部信息

4. **关键头部提取**

   **文件**: `internal/bounce/mailbox/pop.go:41-49`

   ```go
   headerLookups = []bounceHeaders{
       {models.EmailHeaderCampaignUUID, regexp.MustCompile(...)},  // 活动 UUID
       {models.EmailHeaderSubscriberUUID, regexp.MustCompile(...)}, // 订阅者 UUID
       {models.EmailHeaderDate, regexp.MustCompile(...)},           // 邮件日期
       {models.EmailHeaderFrom, regexp.MustCompile(...)},           // 发件人
       {models.EmailHeaderSubject, regexp.MustCompile(...)},        // 主题
       {models.EmailHeaderMessageId, regexp.MustCompile(...)},      // 消息ID
       {models.EmailHeaderDeliveredTo, regexp.MustCompile(...)},    // 收件人
   }
   ```

5. **退信类型分类**

   **文件**: `internal/bounce/mailbox/pop.go:214-240`

   ```go
   func classifyBounce(b []byte) (string, string) {
       // 1. 首先检查 SMTP 状态码
       // 5.x.x = 硬退信
       // 4.x.x = 软退信
       
       // 2. 检查硬退信关键词
       // NXDOMAIN, user unknown, address not found, mailbox not found 等
       
       // 3. 默认返回软退信
   }
   ```

6. **已处理邮件删除**
   ```go
   for id := 1; id <= count; id++ {
       if err := c.Dele(id); err != nil {
           return err
       }
   }
   ```

### 2.4 退信事件入队

解析完成后，退信事件被发送到队列：

```go
ch <- models.Bounce{
    Type:           bounceType,
    CampaignUUID:   hdr[models.EmailHeaderCampaignUUID],
    SubscriberUUID: hdr[models.EmailHeaderSubscriberUUID],
    Source:         p.opt.Host,
    CreatedAt:      date,
    Meta:           meta,  // JSON 序列化的元数据
}
```

---

## 3. IMAP 退信处理流程

### 3.1 当前状态分析

**重要发现**：虽然配置结构中包含 IMAP 相关字段，但**当前代码未实现 IMAP 扫描功能**。

**配置结构定义**：`internal/bounce/mailbox/opt.go`

```go
type Opt struct {
    Host          string        `json:"host"`
    Port          int           `json:"port"`
    AuthProtocol  string        `json:"auth_protocol"`
    Username      string        `json:"username"`
    Password      string        `json:"password"`
    Folder        string        `json:"folder"`  // IMAP 专用：文件夹名称
    TLSEnabled    bool          `json:"tls_enabled"`
    TLSSkipVerify bool          `json:"tls_skip_verify"`
    ScanInterval  time.Duration `json:"scan_interval"`
}
```

### 3.2 代码实现分析

**文件**: `internal/bounce/bounce.go:76-83`

```go
if opt.MailboxEnabled {
    switch opt.MailboxType {
    case "pop":
        m.mailbox = mailbox.NewPOP(opt.Mailbox, lo)
    default:
        return nil, errors.New("unknown bounce mailbox type")
    }
}
```

**结论**：
- 只有 `"pop"` 类型有实际实现
- 任何其他类型（包括 `"imap"`）都会返回错误 `"unknown bounce mailbox type"`
- 配置中的 `Folder` 字段（用于 IMAP 文件夹）在当前实现中未被使用

### 3.3 国际化证据

从 `i18n/zh-CN.json:409` 可以看到 IMAP 相关的翻译：
```json
"settings.bounces.folderHelp": "要扫描的 IMAP 文件夹的名称。例如：收件箱。"
```

这表明：
1. 前端 UI 可能已经支持 IMAP 配置
2. 但后端实现尚未完成
3. 这是一个**计划中但未实现**的功能

---

## 4. SES Webhook 退信处理流程

### 4.1 整体架构

SES Webhook 处理涉及两个关键部分：
1. **SNS 订阅确认** - AWS 需要确认 endpoint 有效性
2. **退信/投诉通知处理** - 实际的退信事件处理

### 4.2 HTTP 入口

**文件**: `cmd/bounce.go:117-249`

```go
func (a *App) BounceWebhook(c echo.Context) error {
    // 根据 URL 参数路由到不同处理器
    service := c.Param("service")
    
    switch true {
    case service == "":
        // 原生 API，直接解析 JSON
        
    case service == "ses" && a.bounce.SES != nil:
        // SES 处理
        switch c.Request().Header.Get("X-Amz-Sns-Message-Type") {
        case "SubscriptionConfirmation", "UnsubscribeConfirmation":
            // 订阅确认
            if err := a.bounce.SES.ProcessSubscription(rawReq); err != nil {
                // 处理确认请求
            }
            
        case "Notification":
            // 实际的退信通知
            b, err := a.bounce.SES.ProcessBounce(rawReq)
            bounces = append(bounces, b)
        }
    }
    
    // 所有退信事件统一入队
    for _, b := range bounces {
        if err := a.bounce.Record(b); err != nil {
            a.log.Printf("error recording bounce: %v", err)
        }
    }
}
```

### 4.3 SNS 订阅确认流程

**文件**: `internal/bounce/webhooks/ses.go:80-105`

```go
func (s *SES) ProcessSubscription(b []byte) error {
    var n sesNotif
    if err := json.Unmarshal(b, &n); err != nil {
        return err
    }
    
    // 1. 验证签名
    if err := s.verifyNotif(n); err != nil {
        return err
    }
    
    // 2. 访问 SubscribeURL 或 UnsubscribeURL 完成确认
    u := n.SubscribeURL
    if n.Type == "UnsubscriptionConfirmation" {
        u = n.UnsubscribeURL
    }
    
    resp, err := http.Get(u)
    // ...
}
```

### 4.4 关键风险：类型判断前后不一致

**问题定位**：

| 位置 | 代码 | 使用的字符串 |
|------|------|-------------|
| `cmd/bounce.go:159` (HTTP 路由层) | `case "SubscriptionConfirmation", "UnsubscribeConfirmation"` | `"UnsubscribeConfirmation"` |
| `internal/bounce/webhooks/ses.go:91` (业务逻辑层) | `if n.Type == "UnsubscriptionConfirmation"` | `"UnsubscriptionConfirmation"` |

**差异**：
- 路由层：`"UnsubscribeConfirmation"` (无额外的 'n')
- 业务层：`"UnsubscriptionConfirmation"` (有额外的 'n' → Un**s**cript**ion**)

**AWS SNS 实际行为**：

根据 AWS SNS 官方文档，消息类型值为：
- `SubscriptionConfirmation` - 订阅确认
- `UnsubscribeConfirmation` - 退订确认
- `Notification` - 实际通知

注意：AWS 使用的是 `"UnsubscribeConfirmation"`（与路由层一致）。

**触发条件**：

当以下情况发生时，bug 会被触发：

1. 用户在 AWS 控制台删除 SNS 订阅
2. AWS 向 listmonk endpoint 发送 `UnsubscribeConfirmation` 通知
3. HTTP Header `X-Amz-Sns-Message-Type` = `"UnsubscribeConfirmation"`
4. JSON Body 中的 `Type` 字段 = `"UnsubscribeConfirmation"`

**执行流程分析**：

```
请求到达：
┌─────────────────────────────────────────────────────────────┐
│ Header: X-Amz-Sns-Message-Type = "UnsubscribeConfirmation"  │
│ Body: {"Type": "UnsubscribeConfirmation", ...}              │
└─────────────────────────────────────────────────────────────┘
                           ↓
              cmd/bounce.go:159 路由判断
                           ↓
              case "SubscriptionConfirmation", "UnsubscribeConfirmation":
                           ↓
              ✅ 路由匹配，调用 ProcessSubscription()
                           ↓
              internal/bounce/webhooks/ses.go:89-93
                           ↓
              u := n.SubscribeURL  // 默认使用 SubscribeURL
              if n.Type == "UnsubscriptionConfirmation" {  // ❌ 条件判断
                  u = n.UnsubscribeURL  // 期望：访问 UnsubscribeURL
              }
                           ↓
              ❌ 条件不满足（n.Type = "UnsubscribeConfirmation"）
                           ↓
              ❌ 实际访问 n.SubscribeURL 而不是 n.UnsubscribeURL
```

**影响评估**：

| 维度 | 影响 |
|------|------|
| **功能正确性** | 退订确认请求会访问错误的 URL (`SubscribeURL` 而非 `UnsubscribeURL`) |
| **AWS 状态同步** | AWS 可能认为退订确认未完成，订阅状态可能不一致 |
| **安全风险** | 较低 - 因为这是退订确认，而非订阅确认 |
| **可利用性** | 需要 AWS 发送请求，非外部可主动触发 |

**证据对比**：

**文件**：`internal/bounce/webhooks/ses.go:41-44` (数据结构定义)

```go
type sesNotif struct {
    // ...
    Type             string `json:"Type"`
    SubscribeURL     string `json:"SubscribeURL"`
    UnsubscribeURL   string `json:"UnsubscribeURL"`  // 注意：字段名是 "UnsubscribeURL"
}
```

数据结构中的 `UnsubscribeURL` 也印证了 AWS 使用 `"Unsubscribe"` 而非 `"Unsubscription"`。

**修复建议**：

将 `ses.go:91` 中的：
```go
if n.Type == "UnsubscriptionConfirmation" {
```
修改为：
```go
if n.Type == "UnsubscribeConfirmation" {
```

### 4.5 签名验证机制

**文件**: `internal/bounce/webhooks/ses.go:198-258`

SES 的签名验证涉及以下步骤：

1. **构建签名字符串**
   ```go
   func (s *SES) buildSignature(n sesNotif) []byte {
       var b bytes.Buffer
       b.WriteString("Message" + "\n" + n.Message + "\n")
       b.WriteString("MessageId" + "\n" + n.MessageId + "\n")
       // ... 按特定顺序拼接所有字段
       return b.Bytes()
   }
   ```

2. **获取并验证证书**
   ```go
   func (s *SES) getCert(certURL string) (*x509.Certificate, error) {
       // 验证证书 URL 必须是 Amazon 的域名
       if !sesRegCertURL.MatchString(certURL) {
           return nil, fmt.Errorf("invalid SNS certificate URL: %v", u.Host)
       }
       
       // 下载证书并缓存
       // ...
   }
   ```

3. **验证签名**
   ```go
   cert.CheckSignature(x509.SHA1WithRSA, s.buildSignature(n), sign)
   ```

### 4.5 退信通知处理

**文件**: `internal/bounce/webhooks/ses.go:107-173`

```go
func (s *SES) ProcessBounce(b []byte) (models.Bounce, error) {
    // 1. 解析外层 SNS 通知
    var n sesNotif
    json.Unmarshal(b, &n)
    
    // 2. 验证签名
    s.verifyNotif(n)
    
    // 3. 解析内层 Message（SES 实际退信数据）
    var m sesMail
    json.Unmarshal([]byte(n.Message), &m)
    
    // 4. 验证事件类型
    if (m.EventType != "" && m.EventType != "Bounce") ||
       (m.NotifType != "" && (m.NotifType != "Bounce" && m.NotifType != "Complaint")) {
        return bounce, errors.New("notification type is not bounce")
    }
    
    // 5. 确定退信类型
    typ := models.BounceTypeSoft
    if m.Bounce.BounceType == "Permanent" {
        typ = models.BounceTypeHard
    }
    if m.NotifType == "Complaint" {
        typ = models.BounceTypeComplaint
    }
    // 特殊处理：Transient 中的 5.4.4 (无效域名) 视为硬退信
    if m.Bounce.BounceType == "Transient" && 
       len(m.Bounce.BouncedRecipients) > 0 &&
       m.Bounce.BouncedRecipients[0].Status == "5.4.4" {
        typ = models.BounceTypeHard
    }
    
    // 6. 从 headers 提取 campaign_uuid
    campUUID := ""
    for _, h := range m.Mail.Headers {
        if h["name"] == models.EmailHeaderCampaignUUID {
            campUUID = h["value"]
            break
        }
    }
    
    return models.Bounce{
        Email:        strings.ToLower(m.Mail.Destination[0]),
        CampaignUUID: campUUID,
        Type:         typ,
        Source:       "ses",
        Meta:         json.RawMessage(n.Message),
        CreatedAt:    time.Time(m.Mail.Timestamp),
    }, nil
}
```

### 4.6 SES 数据结构

```go
type sesMail struct {
    EventType string  // "Bounce"
    NotifType string  // "Bounce" 或 "Complaint"
    Bounce    struct {
        BounceType        string  // "Permanent" | "Transient" | "Undetermined"
        BouncedRecipients []struct {
            Status string  // SMTP 状态码，如 "5.1.1"
        }
    }
    Mail struct {
        Timestamp   sesTimestamp
        Destination []string  // 收件人邮箱列表
        Headers     []map[string]string  // 包含 listmonk 注入的自定义头部
    }
}
```

---

## 5. 其他 Webhook 支持

Listmonk 还支持以下邮件服务商的 webhook：

| 服务商 | 端点路径 | 验证方式 | 文件 |
|--------|----------|----------|------|
| Sendgrid | `/webhooks/service/sendgrid` | `X-Twilio-Email-Event-Webhook-Signature` + 时间戳 | `webhooks/sendgrid.go` |
| Postmark | `/webhooks/service/postmark` | Basic Auth | `webhooks/postmark.go` |
| ForwardEmail | `/webhooks/service/forwardemail` | `X-Webhook-Signature` | `webhooks/forwardemail.go` |
| Lettermint | `/webhooks/service/lettermint` | `X-Lettermint-Signature` | `webhooks/lettermint.go` |

---

## 6. 订阅者状态回写流程

### 6.1 整体流程

```
退信事件入队
    ↓
Manager.Run() 消费队列
    ↓
调用 RecordBounceCB (即 core.RecordBounce)
    ↓
执行 SQL 查询 (record-bounce)
    ↓
根据退信类型和计数决定动作
    ↓
更新 subscribers 表状态
    ↓
更新 subscriber_lists 表状态 (可选)
    ↓
插入 bounces 记录
```

### 6.2 配置驱动的动作机制

**配置结构**：`models/settings.go:117-120`

```go
BounceActions map[string]struct {
    Count  int    `json:"count"`   // 触发动作的退信次数阈值
    Action string `json:"action"`  // 达到阈值后的动作
} `json:"bounce.actions"`
```

**典型配置示例**（来自文档）：
```toml
[bounce.actions]
[bounce.actions.soft]
count = 2
action = "none"

[bounce.actions.hard]
count = 1
action = "blocklist"

[bounce.actions.complaint]
count = 1
action = "blocklist"
```

### 6.3 核心处理函数

**文件**: `internal/core/bounces.go:59-87`

```go
func (c *Core) RecordBounce(b models.Bounce) error {
    // 1. 获取该退信类型对应的配置
    action, ok := c.consts.BounceActions[b.Type]
    if !ok {
        return echo.NewHTTPError(http.StatusBadRequest, ...)
    }
    
    // 2. 执行数据库查询，传入配置的 count 和 action
    _, err := c.q.RecordBounce.Exec(
        b.SubscriberUUID,
        b.Email,
        b.CampaignUUID,
        b.Type,
        b.Source,
        b.Meta,
        b.CreatedAt,
        action.Count,   // $8: 阈值次数
        action.Action,  // $9: 动作类型
    )
    
    // 3. 忽略"订阅者不存在"的错误
    if pqErr, ok := err.(*pq.Error); ok && pqErr.Column == "subscriber_id" {
        c.log.Printf("bounced subscriber (%s / %s) not found", b.SubscriberUUID, b.Email)
        return nil
    }
    
    return err
}
```

### 6.4 数据库层面的状态回写

**文件**: `queries/bounces.sql:1-30`

这是整个退信处理的核心逻辑，使用 PostgreSQL 的 CTE (Common Table Expression) 实现原子操作：

```sql
-- name: record-bounce
WITH sub AS (
    -- 步骤1: 查找订阅者（优先用 uuid，其次用 email）
    SELECT id, status FROM subscribers 
    WHERE CASE WHEN $1 != '' THEN uuid = $1::UUID ELSE email = $2 END
),
camp AS (
    -- 步骤2: 查找关联的活动
    SELECT id FROM campaigns WHERE $3 != '' AND uuid = $3::UUID
),
num AS (
    -- 步骤3: 计算该订阅者此类型退信的累计次数（包含当前这次）
    SELECT COUNT(*) + 1 AS num 
    FROM bounces 
    WHERE subscriber_id = (SELECT id FROM sub) AND type = $4
),
-- 步骤4: 根据动作类型执行状态更新

-- 动作: blocklist
block1 AS (
    UPDATE subscribers SET status='blocklisted'
    WHERE $9 = 'blocklist' 
      AND (SELECT num FROM num) >= $8 
      AND id = (SELECT id FROM sub) 
      AND (SELECT status FROM sub) != 'blocklisted'
),

-- 动作: unsubscribe（从所有列表退订）
block2 AS (
    UPDATE subscriber_lists SET status='unsubscribed'
    WHERE $9 = 'unsubscribe' 
      AND (SELECT num FROM num) >= $8 
      AND subscriber_id = (SELECT id FROM sub) 
      AND (SELECT status FROM sub) != 'blocklisted'
),

-- 步骤5: 记录退信（有条件）
bounce AS (
    INSERT INTO bounces (subscriber_id, campaign_id, type, source, meta, created_at)
    SELECT (SELECT id FROM sub), (SELECT id FROM camp), $4, $5, $6, $7
    -- 只有在以下情况才记录：
    -- 1. 订阅者未被拉黑
    -- 2. 累计次数未超过阈值（防止重复记录）
    WHERE NOT EXISTS (
        SELECT 1 WHERE (SELECT status FROM sub) = 'blocklisted' 
                     OR (SELECT num FROM num) > $8
    )
)

-- 步骤6: 动作: delete（删除订阅者）
DELETE FROM subscribers
WHERE $9 = 'delete' 
  AND (SELECT num FROM num) >= $8 
  AND id = (SELECT id FROM sub);
```

**重要澄清：自动退信处理 vs 手动拉黑 API**

`record-bounce` 是**自动退信处理**的查询，其动作是互斥的。但 listmonk 还提供了一个**独立的手动 API**：

| 查询名称 | 触发方式 | 行为 |
|----------|----------|------|
| `record-bounce` | 自动（POP3 扫描、Webhook） | 动作互斥：`blocklist` **或** `unsubscribe` **或** `delete` |
| `blocklist-bounced-subscribers` | 手动调用 `PUT /api/bounces/blocklist` | **同时**：拉黑订阅者 **并** 从所有列表退订 |

**手动 API 的实现**：`queries/bounces.sql:66-75`

```sql
-- name: blocklist-bounced-subscribers
WITH subs AS (
    SELECT subscriber_id FROM bounces
),
b AS (
    UPDATE subscribers SET status='blocklisted', updated_at=NOW()
    WHERE id = ANY(SELECT subscriber_id FROM subs)
)
UPDATE subscriber_lists SET status='unsubscribed', updated_at=NOW()
    WHERE subscriber_id = ANY(SELECT subscriber_id FROM subs);
```

这个手动 API 会：
1. 找出**所有**有退信记录的订阅者
2. 将他们设为 `blocklisted`
3. **同时**将他们从所有列表退订

**这与自动退信处理的 `action = 'blocklist'` 行为不同**，后者只更新 `subscribers.status`，不影响 `subscriber_lists`。

### 6.5 动作类型详解

**核心证据**：`queries/bounces.sql:14-20`

```sql
-- block1: 仅当 $9 = 'blocklist' 时执行
block1 AS (
    UPDATE subscribers SET status='blocklisted'
    WHERE $9 = 'blocklist' ...
),
-- block2: 仅当 $9 = 'unsubscribe' 时执行
block2 AS (
    UPDATE subscriber_lists SET status='unsubscribed'
    WHERE $9 = 'unsubscribe' ...
),
```

**关键发现**：`block1` 和 `block2` 的执行条件是**互斥**的，取决于 `$9`（即配置的 `action` 值）。

| 动作 | 实际行为 | 证据位置 | 适用场景 |
|------|----------|----------|----------|
| `none` | 仅记录退信到 `bounces` 表，**不做任何状态更新** | 两个 CTE 条件都不满足 | 软退信，可重试 |
| `blocklist` | **仅**将 `subscribers.status` 设为 `'blocklisted'`，**不会**更新 `subscriber_lists` | `block1` 条件 `$9 = 'blocklist'` | 硬退信、投诉 |
| `unsubscribe` | **仅**将 `subscriber_lists.status` 设为 `'unsubscribed'`，**不会**更新 `subscribers.status` | `block2` 条件 `$9 = 'unsubscribe'` | 需要用户主动重新订阅 |
| `delete` | 从 `subscribers` 表删除订阅者记录 | 最后的 `DELETE` 语句 | GDPR 等合规要求 |

**重要澄清**：
- `blocklist` **不会**同步退订列表，这与 `blocklist-bounced-subscribers` 手动 API 不同
- `unsubscribe` **不会**改变订阅者的全局状态，只是从列表中退订

### 6.6 状态流转

**核心证据**：
1. `queries/bounces.sql:15` - 仅设置 `status='blocklisted'`
2. `models/subscribers.go:14-22` - 状态常量定义
3. `internal/migrations/v0.7.0.go:40` - ENUM 类型定义

```go
const (
    SubscriberStatusEnabled     = "enabled"
    SubscriberStatusDisabled    = "disabled"    // 退信处理中未使用
    SubscriberStatusBlockListed = "blocklisted"
)
```

**实际状态流转**（退信处理范围内）：

```
                    ┌──────────────┐
                    │  enabled     │ ← 正常状态
                    └──────┬───────┘
                           │
                           │ 退信计数达到阈值
                           │ 动作: blocklist
                           ▼
                    ┌──────────────┐
                    │ blocklisted  │ ← 拉黑状态（终止状态）
                    └──────────────┘
```

**关于 `disabled` 状态的澄清**：

`disabled` 状态**不在退信处理流程中使用**。它的实际用途：

1. **ENUM 类型定义**：`internal/migrations/v0.7.0.go:40` 定义了 `subscriber_status AS ENUM ('enabled', 'disabled', 'blocklisted')`
2. **退信关键词匹配**：`internal/bounce/mailbox/pop.go:59` 中 `account.*disabled` 是用来**匹配邮件内容**的关键词，用于识别硬退信
3. **可能的手动操作**：管理员可能通过其他方式将订阅者设为 `disabled`，但这不是退信自动处理的一部分

**注意**：
- 一旦进入 `blocklisted` 状态，后续的退信将**不会**被记录（见 SQL 中的 `WHERE NOT EXISTS` 条件：`(SELECT status FROM sub) = 'blocklisted'`）
- `blocklisted` 是退信处理中的**终止状态**，没有自动流转到其他状态的逻辑

### 6.7 阈值计数逻辑

**关键点**：
1. 计数是**按退信类型**分别统计的（`type = $4`）
2. 软退信和硬退信的计数器是独立的
3. `COUNT(*) + 1` 包含了当前正在处理的这次退信
4. 当 `num > threshold` 时，不再记录退信（防止无限累加）

---

## 7. 关键数据结构

### 7.1 Bounce 模型

**文件**: `models/bounces.go`

```go
type Bounce struct {
    ID        int             `db:"id" json:"id"`
    Type      string          `db:"type" json:"type"`           // "hard" | "soft" | "complaint"
    Source    string          `db:"source" json:"source"`       // 来源："ses", "sendgrid", POP3 host 等
    Meta      json.RawMessage `db:"meta" json:"meta"`           // 原始元数据 JSON
    CreatedAt time.Time       `db:"created_at" json:"created_at"`
    
    // 订阅者标识（至少提供一个）
    Email            string `db:"email" json:"email,omitempty"`
    SubscriberUUID   string `db:"subscriber_uuid" json:"subscriber_uuid,omitempty"`
    SubscriberID     int    `db:"subscriber_id" json:"subscriber_id,omitempty"`
    SubscriberStatus string `db:"subscriber_status" json:"subscriber_status"`  // 查询时返回
    
    // 活动关联
    CampaignUUID string           `db:"campaign_uuid" json:"campaign_uuid,omitempty"`
    Campaign     *json.RawMessage `db:"campaign" json:"campaign"`  // 查询时关联返回
    
    // 分页用
    Total int `db:"total" json:"-"`
}
```

### 7.2 退信类型常量

```go
const (
    BounceTypeHard      = "hard"      // 永久性失败
    BounceTypeSoft      = "soft"      // 临时性失败
    BounceTypeComplaint = "complaint" // 用户投诉（标记为垃圾邮件）
)
```

---

## 8. 配置项汇总

### 8.1 主要配置

```toml
[bounce]
enabled = true           # 全局开关
webhooks_enabled = true  # 启用 webhook 端点
ses_enabled = true       # 启用 SES webhook
sendgrid_enabled = false
sendgrid_key = ""

[bounce.postmark]
enabled = false
username = ""
password = ""

[bounce.forwardemail]
enabled = false
key = ""

[bounce.lettermint]
enabled = false
key = ""

# 退信动作配置
[bounce.actions.soft]
count = 2
action = "none"

[bounce.actions.hard]
count = 1
action = "blocklist"

[bounce.actions.complaint]
count = 1
action = "blocklist"

# POP3 邮箱配置（数组，当前只支持一个）
[[bounce.mailboxes]]
uuid = "pop3-001"
enabled = true
type = "pop"           # 目前仅支持 "pop"
host = "pop.example.com"
port = 995
auth_protocol = "user" # "user" 或 "none"
username = "bounces@example.com"
password = "secret"
tls_enabled = true
tls_skip_verify = false
scan_interval = "5m"   # 扫描间隔
return_path = ""        # Return-Path 地址
```

### 8.2 邮件头部注入

Listmonk 在发送邮件时会注入自定义头部，用于退信追踪：

```go
models.EmailHeaderCampaignUUID   // "X-Listmonk-Campaign"
models.EmailHeaderSubscriberUUID // "X-Listmonk-Subscriber"
```

这些头部在退信邮件中被保留，用于关联回原始的活动和订阅者。

---

## 9. 流程图总结

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        退信入口层                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌─────────────────────────────┐ │
│  │  POP3 扫描   │    │ IMAP (待实现) │    │       Webhook 端点          │ │
│  │  (定时轮询)  │    │              │    │                             │ │
│  └──────┬───────┘    └──────────────┘    └──────────────┬──────────────┘ │
│         │                                                  │                 │
│         │                    ┌─────────────────────────────┼──────────────┐ │
│         │                    │                             │              │ │
│         ▼                    ▼                             ▼              ▼ │
│  ┌──────────────┐    ┌──────────────┐             ┌──────────────┐   ┌───┐│
│  │ 解析邮件内容  │    │ SES SNS 确认 │             │Sendgrid等    │   │API││
│  │ 提取头部信息  │    │ 签名验证      │             │服务商验证     │   │   ││
│  │ 分类退信类型  │    │ 解析退信数据  │             │解析退信数据   │   │   ││
│  └──────┬───────┘    └──────┬───────┘             └──────┬───────┘   └─┬─┘│
│         │                    │                             │               │  │
│         └────────────────────┴─────────────────────────────┴───────────────┘  │
│                                              │                                    │
│                                              ▼                                    │
│                                   ┌──────────────────────┐                         │
│                                   │   统一转换为          │                         │
│                                   │   models.Bounce      │                         │
│                                   └──────────┬───────────┘                         │
└──────────────────────────────────────────────┼──────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        处理队列层                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                         ┌──────────────────────┐                            │
│                         │  bounce.queue        │                            │
│                         │  (带缓冲的 channel)   │                            │
│                         └──────────┬───────────┘                            │
│                                    │                                         │
│                                    ▼                                         │
│                         ┌──────────────────────┐                            │
│                         │  Manager.Run()       │                            │
│                         │  循环消费队列         │                            │
│                         └──────────┬───────────┘                            │
│                                    │                                         │
└────────────────────────────────────┼──────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        状态回写层                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                    ┌─────────────────────────────────┐                       │
│                    │   core.RecordBounce()           │                       │
│                    │   读取配置: BounceActions        │                       │
│                    └───────────────┬─────────────────┘                       │
│                                    │                                         │
│                                    ▼                                         │
│                    ┌─────────────────────────────────┐                       │
│                    │   SQL: record-bounce            │                       │
│                    │   (PostgreSQL CTE)              │                       │
│                    │                                 │                       │
│                    │   1. 查找订阅者                  │                       │
│                    │   2. 计算累计退信次数            │                       │
│                    │   3. 根据配置执行动作            │                       │
│                    │      - blocklist                │                       │
│                    │      - unsubscribe              │                       │
│                    │      - delete                   │                       │
│                    │      - none                     │                       │
│                    │   4. 记录退信到 bounces 表      │                       │
│                    └───────────────┬─────────────────┘                       │
│                                    │                                         │
│                                    ▼                                         │
│                    ┌─────────────────────────────────┐                       │
│                    │   数据库表更新                   │                       │
│                    │                                 │                       │
│                    │   subscribers.status            │                       │
│                    │     → 'blocklisted' (可选)      │                       │
│                    │                                 │                       │
│                    │   subscriber_lists.status       │                       │
│                    │     → 'unsubscribed' (可选)     │                       │
│                    │                                 │                       │
│                    │   bounces 表插入记录            │                       │
│                    └─────────────────────────────────┘                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. 发现与建议

### 10.1 重要发现

1. **IMAP 未实现**：虽然配置结构和前端翻译都已准备好，但后端只实现了 POP3。

2. **单邮箱限制**：`initBounceManager` 中使用了 `break`，意味着即使配置了多个邮箱，也只会使用第一个。

3. **退信计数按类型独立**：软退信和硬退信的计数器是分开的，这是合理的设计。

4. **已处理邮件删除**：POP3 扫描后会从服务器删除邮件，这意味着退信邮件**不会保留**在邮箱中。

5. **签名验证安全**：SES 和 Sendgrid 等 webhook 都有严格的签名验证机制，防止伪造请求。

### 10.2 潜在问题

1. **队列缓冲有限**：`make(chan models.Bounce, 1000)`，在大规模退信时可能阻塞。

2. **错误处理**：`RecordBounceCB` 中的错误被直接忽略（`continue`），没有重试机制。

3. **POP3 无重试**：单次扫描失败后，只是记录日志，等待下一个扫描周期。

### 10.3 扩展建议

1. **实现 IMAP 支持**：利用配置中已有的 `Folder` 字段，添加 IMAP 扫描实现。

2. **多邮箱支持**：修改 `initBounceManager` 支持同时配置多个邮箱。

3. **死信队列**：为处理失败的退信添加死信队列（DLQ）机制。

4. **指标监控**：添加 Prometheus 指标，监控退信数量、处理延迟等。

---

## 附录：相关文件速查

| 功能模块 | 文件路径 |
|----------|----------|
| 退信管理器 | `internal/bounce/bounce.go` |
| POP3 实现 | `internal/bounce/mailbox/pop.go` |
| 邮箱配置结构 | `internal/bounce/mailbox/opt.go` |
| SES Webhook | `internal/bounce/webhooks/ses.go` |
| 核心业务逻辑 | `internal/core/bounces.go` |
| 数据模型 | `models/bounces.go` |
| SQL 查询 | `queries/bounces.sql` |
| HTTP 处理器 | `cmd/bounce.go` |
| 初始化 | `cmd/init.go` |
| 配置结构 | `models/settings.go` |
| 用户文档 | `docs/docs/content/bounces.md` |
