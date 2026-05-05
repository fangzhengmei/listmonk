# Listmonk SMTP 池技术分析报告 - 第三轮：中断一致性

## 本文档重点

本文档深入分析 listmonk 在多 SMTP 发送中的**中断一致性问题**，包括：
1. **Checkpoint 前移机制** - `last_subscriber_id` 何时、如何更新
2. **队列消息处理顺序** - 消息推送与消费的时序
3. **中断场景分析** - 崩溃、暂停、重启时会发生什么
4. **补发机制验证** - 被跳过的消息是否会在后续补发
5. **语义影响** - 对 `sent` 计数和订阅者送达的实际含义

---

## 第一部分：核心执行流程

### 1.1 完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        Campaign 发送完整执行流程                                   │
└─────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ 阶段1: scanCampaigns (每5秒)                                                  │
  │ ┌──────────────────────────────────────────────────────────────────────────┐ │
  │ │ ids, counts := m.getCurrentCampaigns()  // 收集内存中的 sent 计数         │ │
  │ │ campaigns, err := m.store.NextCampaigns(ids, counts)                    │ │
  │ │   → SQL: sent = sent + $sent_count (增量更新)                            │ │
  │ │   → SQL: 更新 max_subscriber_id (所有符合条件订阅者的最大ID)              │ │
  │ └──────────────────────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ 阶段2: Run() 主循环                                                            │
  │ ┌──────────────────────────────────────────────────────────────────────────┐ │
  │ │ for p := range m.nextPipes {                                              │ │
  │ │     has, err := p.NextSubscribers()   // 关键：获取下一批订阅者           │ │
  │ │     if has {                                                              │ │
  │ │         m.nextPipes <- p  // 还有订阅者，继续处理                         │ │
  │ │     } else {                                                              │ │
  │ │         p.wg.Done()      // 没有订阅者了，标记完成                        │ │
  │ │     }                                                                     │ │
  │ │ }                                                                         │ │
  │ └──────────────────────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ 阶段3: NextSubscribers() - ⚠️ 核心问题所在                                    │
  │ ┌──────────────────────────────────────────────────────────────────────────┐ │
  │ │ // 步骤1: 查询订阅者，⚠️ 同时更新 checkpoint                                │ │
  │ │ subs, err := p.m.store.NextSubscribers(p.camp.ID, p.m.cfg.BatchSize)   │ │
  │ │                                                                           │ │
  │ │ // 对应的 SQL (queries/campaigns.sql:367-371):                         │ │
  │ │ //   WITH campLists AS (...),                                            │ │
  │ │ //        subs AS (SELECT ... WHERE s.id > $3 AND s.id <= $4),         │ │
  │ │ //        u AS (                                                          │ │
  │ │ //            UPDATE campaigns                                            │ │
  │ │ //            SET last_subscriber_id = (SELECT MAX(id) FROM subs)       │ │
  │ │ //            WHERE id=$1                                                 │ │
  │ │ //        )                                                               │ │
  │ │ //   SELECT * FROM subs;                                                  │ │
  │ │ //                                                                         │ │
  │ │ // ⚠️ 关键：在返回订阅者之前，last_subscriber_id 已经被更新到 MAX(id)     │ │
  │ │ //    这是一个原子操作，在 CTE 中完成                                      │ │
  │ │                                                                           │ │
  │ │ // 步骤2: 推送消息到队列                                                   │ │
  │ │ for _, s := range subs {                                                  │ │
  │ │     msg, _ := p.newMessage(s)  // wg.Add(1)                            │ │
  │ │     p.m.campMsgQ <- msg        // 阻塞推送                                │ │
  │ │ }                                                                         │ │
  │ └──────────────────────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ 阶段4: worker() 消费队列                                                       │
  │ ┌──────────────────────────────────────────────────────────────────────────┐ │
  │ │ for {                                                                     │ │
  │ │     select {                                                              │ │
  │ │     case msg, ok := <-m.campMsgQ:                                        │ │
  │ │         // ⚠️ 检查是否已停止                                               │ │
  │ │         if msg.pipe != nil && msg.pipe.stopped.Load() {                 │ │
  │ │             msg.pipe.wg.Done()   // 只减少计数                           │ │
  │ │             continue                  // ⚠️ 直接跳过，不发送！            │ │
  │ │         }                                                                 │ │
  │ │                                                                           │ │
  │ │         // 发送邮件                                                        │ │
  │ │         err := m.messengers[msg.Campaign.Messenger].Push(out)           │ │
  │ │                                                                           │ │
  │ │         // 更新计数                                                        │ │
  │ │         if err != nil {                                                   │ │
  │ │             msg.pipe.OnError()       // 错误计数+1                       │ │
  │ │         } else {                                                          │ │
  │ │             msg.pipe.sent.Add(1)      // ⚠️ 只有成功才 sent+1           │ │
  │ │             msg.pipe.lastID.Store(...) // 更新内存中的 lastID            │ │
  │ │         }                                                                 │ │
  │ │         msg.pipe.wg.Done()                                               │ │
  │ │     }                                                                     │ │
  │ │ }                                                                         │ │
  │ └──────────────────────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ 阶段5: cleanup() - 最终处理                                                   │
  │ ┌──────────────────────────────────────────────────────────────────────────┐ │
  │ │ // wg.Wait() 返回后触发                                                   │ │
  │ │                                                                           │ │
  │ │ // 步骤1: 更新最终计数                                                     │ │
  │ │ p.m.store.UpdateCampaignCounts(                                          │ │
  │ │     p.camp.ID,                                                           │ │
  │ │     0,                               // to_send 不更新                    │ │
  │ │     int(p.sent.Load()),              // 内存中的成功计数                  │ │
  │ │     int(p.lastID.Load())             // 内存中的 lastID                   │ │
  │ │ )                                                                         │ │
  │ │                                                                           │ │
  │ │ // 步骤2: 更新状态                                                        │ │
  │ │ if p.withErrors.Load() {                                                 │ │
  │ │     p.m.store.UpdateCampaignStatus(p.camp.ID, "paused")                 │ │
  │ │ } else if p.stopped.Load() {                                             │ │
  │ │     // 手动停止，不更新状态                                               │ │
  │ │ } else {                                                                  │ │
  │ │     p.m.store.UpdateCampaignStatus(p.camp.ID, "finished")               │ │
  │ │ }                                                                         │ │
  │ │                                                                           │ │
  │ │ // ⚠️ 没有任何补发或补偿逻辑！                                            │ │
  │ └──────────────────────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────────────────────┘
```

---

## 第二部分：Checkpoint 前移机制深度分析

### 2.1 SQL 执行时序

**代码位置**: `queries/campaigns.sql:318-372`

```sql
-- name: next-campaign-subscribers
-- 注释说明：
-- Returns a batch of subscribers in a given campaign starting from the last checkpoint
-- (last_subscriber_id). Every fetch updates the checkpoint and the sent count...
-- 
-- ⚠️ 关键注释："Every fetch updates the checkpoint"
-- 意思是：每次获取都会更新 checkpoint

WITH campLists AS (
    SELECT lists.id AS list_id, optin FROM lists
    LEFT JOIN campaign_lists ON campaign_lists.list_id = lists.id
    WHERE campaign_lists.campaign_id = $1
),
subs AS (
    -- 步骤1: 查询符合条件的订阅者
    -- 条件: s.id > $3 (last_subscriber_id)
    --       s.id <= $4 (max_subscriber_id)
    SELECT s.*
    FROM (
        SELECT DISTINCT s.id
        FROM subscriber_lists sl
        JOIN campLists ON sl.list_id = campLists.list_id
        JOIN subscribers s ON s.id = sl.subscriber_id
        WHERE
            sl.list_id = ANY($5::INT[])
            AND s.id > $3           -- $3 = last_subscriber_id (上一个 checkpoint)
            AND s.id <= $4          -- $4 = max_subscriber_id (所有符合条件的最大ID)
            AND s.status != 'blocklisted'
            -- ... 其他条件
        ORDER BY s.id LIMIT $6      -- $6 = batch_size
    ) subIDs JOIN subscribers s ON (s.id = subIDs.id) ORDER BY s.id
),
u AS (
    -- 步骤2: ⚠️ 更新 checkpoint！
    -- 这是在 CTE 中，与上面的查询是同一个原子操作
    UPDATE campaigns
    SET 
        last_subscriber_id = (SELECT MAX(id) FROM subs),  -- 前移到这批的最大ID
        updated_at = NOW()
    WHERE (SELECT COUNT(id) FROM subs) > 0 AND id=$1
)
SELECT * FROM subs;  -- 步骤3: 返回订阅者
```

### 2.2 执行顺序的关键问题

```
时序图 (单批次处理):

时间轴 →
│
│  T0: 调用 p.m.store.NextSubscribers(campID, batchSize)
│       │
│       ▼
│  T1: SQL 开始执行 (原子操作)
│       ├── subs CTE: 查询 id > last_subscriber_id (假设=100) 的订阅者
│       │           找到 id=101, 102, 103, ..., 200 (共100个)
│       │
│       ├── u CTE: UPDATE campaigns 
│       │           SET last_subscriber_id = MAX(id) = 200
│       │           ⚠️ 这里 checkpoint 已经从 100 前移到 200！
│       │
│       └── SELECT * FROM subs → 返回 100 个订阅者
│       │
│  T2: SQL 返回，subs 变量包含 100 个订阅者
│       ⚠️ 注意：此时数据库中 last_subscriber_id 已经是 200 了！
│       │
│       ▼
│  T3: Go 代码开始推送消息到队列
│       for _, s := range subs {
│           msg, _ := p.newMessage(s)   // wg.Add(1)
│           p.m.campMsgQ <- msg         // 阻塞推送
│       }
│       │
│       ▼ 可能的中断点
│
│  T4: worker 消费消息
│       for msg := range campMsgQ {
│           // 发送邮件...
│       }
│
```

### 2.3 数据不一致的窗口

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                        ⚠️ 危险时间窗口 ⚠️                                        │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  时间点: T2 (SQL 返回后) ~ T4 (所有消息处理完成)                               │
│                                                                                │
│  在此窗口内:                                                                   │
│  ✅ 数据库中 last_subscriber_id = 200 (已前移)                                │
│  ⚠️ 内存中可能还有消息未处理                                                    │
│  ⚠️ campMsgQ 队列中可能还有消息未消费                                          │
│                                                                                │
│  如果此时发生以下任意事件:                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ 1. 进程崩溃 (OOM, 被 kill, 硬件故障)                                      │ │
│  │ 2. 用户手动暂停 Campaign                                                   │ │
│  │ 3. 系统重启 (SIGHUP 触发 respawn)                                         │ │
│  │ 4. 容器重启/调度器迁移                                                     │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
│                                                                                │
│  后果:                                                                         │
│  ❌ 队列中的消息丢失                                                           │
│  ❌ checkpoint 已前移，下次恢复从 201 开始                                    │
│  ❌ id=101~200 之间未处理的订阅者永远丢失！                                    │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 第三部分：中断场景详细分析

### 3.1 场景 A：进程崩溃

#### 条件假设
- Campaign: ID=1, 总订阅者 10000 人
- 已发送: 0-100 (last_subscriber_id=100)
- 批次大小: 100
- 队列容量: `cfg.Concurrency * cfg.MessageRate * 2` (假设=200)

#### 时间线

```
初始状态:
  campaigns 表:
    - last_subscriber_id = 100
    - sent = 0
  内存状态:
    - pipe.sent = 0

时间 T0:
  p.NextSubscribers() 被调用
  
  SQL 执行:
    1. subs CTE: 查询 id > 100, 找到 101-200
    2. u CTE: UPDATE campaigns SET last_subscriber_id = 200
       ⚠️ 数据库中 last_subscriber_id 变为 200！
    3. 返回 100 个订阅者

时间 T1:
  Go 代码拿到 subs (100 个订阅者: 101-200)
  
  开始推送到队列:
    for i, s := range subs {
        msg, _ := p.newMessage(s)  // wg.Add(1)
        p.m.campMsgQ <- msg
        
        // 假设推送了 50 个后 (id=101-150)
        if i == 49 {
            // ⚠️ 进程崩溃！ (OOM / 被 kill -9 / 硬件故障)
        }
    }

时间 T2: 崩溃后
  内存状态全部丢失:
    ❌ pipe.sent = 0 (丢失)
    ❌ pipe.wg 计数 (丢失)
    ❌ campMsgQ 队列中的 50 条消息 (丢失)
  
  数据库状态:
    ✅ last_subscriber_id = 200 (已保存)
    ✅ sent = 0 (未更新，因为还没到批次结束)

时间 T3: 系统重启
  scanCampaigns 运行:
    读取 campaigns 表:
      - last_subscriber_id = 200
      - max_subscriber_id = 10000 (所有符合条件的最大ID)
  
  next-campaign-subscribers 下次查询:
    WHERE s.id > $3  →  s.id > 200
    
  ⚠️ 问题:
    - id=101-150: 已推送到队列，但崩溃时丢失，从未发送
    - id=151-200: 从未推送到队列
    - 下次从 id=201 开始
    - ❌ id=101-200 这 100 个订阅者永远丢失！
```

#### 影响总结

| 项目 | 实际情况 |
|------|----------|
| 数据库 checkpoint | 200 (已前移) |
| 实际发送 | 0 (崩溃前未完成) |
| 下次恢复起点 | 201 |
| 丢失订阅者 | 101-200 (共 100 人) |
| `sent` 计数 | 0 (未更新) |
| 用户感知 | 显示 sent=0，但实际有 100 人永远收不到 |

### 3.2 场景 B：手动暂停 Campaign

#### 条件假设
- Campaign 正在运行
- 一批订阅者 101-200 已获取，checkpoint 已前移到 200
- campMsgQ 队列中有 50 条消息等待处理
- 用户通过 UI 点击"暂停"按钮

#### 时间线

```
时间 T0:
  NextSubscribers 已完成:
    - last_subscriber_id = 200 (数据库)
    - campMsgQ 中有 50 条消息 (id=101-150)
    - wg 计数 = 50 (50 条消息待处理)

时间 T1: 用户点击暂停
  API 调用 → p.Stop(false)
  
  p.Stop 执行:
    p.stopped.Store(true)  // 设置停止标记
    // ⚠️ 注意：只是设置标记，不处理队列中的消息！

时间 T2: worker 继续消费队列
  worker 代码 (manager.go:474-479):
  
  for msg := range m.campMsgQ {
      // ⚠️ 检查停止标记
      if msg.pipe != nil && msg.pipe.stopped.Load() {
          // 管道已停止
          msg.pipe.wg.Done()   // 只减少 wg 计数
          continue              // ⚠️ 直接跳过，不发送！
      }
      
      // 发送邮件... (不会执行到这里)
  }

时间 T3: 所有消息被"处理"
  campMsgQ 中的 50 条消息:
    - 每条都被 worker 取出
    - 每条都检查到 stopped=true
    - 每条都执行 wg.Done() + continue
    - ❌ 没有一条被实际发送！

时间 T4: wg.Wait() 返回，cleanup() 触发
  cleanup() 执行:
    1. UpdateCampaignCounts(campID, 0, int(p.sent.Load()), ...)
       - p.sent = 0 (因为没有成功发送)
       
    2. 检查状态:
       if p.stopped.Load() {
           // 手动停止
           log.Printf("stop processing campaign (%s)", p.camp.Name)
           return  // ⚠️ 直接返回，没有任何补偿！
       }

时间 T5: Campaign 状态
  数据库:
    - status = 'paused' (由 API 调用更新)
    - last_subscriber_id = 200
    - sent = 0
  
  ⚠️ 问题:
    - id=101-200 的订阅者从未收到邮件
    - checkpoint 已前移到 200
    - 用户点击"继续"后，从 id=201 开始
    - ❌ id=101-200 永远丢失！
```

#### 关键代码证据

**代码位置**: `internal/manager/manager.go:474-479`

```go
// If the campaign has ended or stopped, ignore the message.
if msg.pipe != nil && msg.pipe.stopped.Load() {
    // Reduce the message counter on the pipe.
    msg.pipe.wg.Done()
    continue  // ⚠️ 直接跳过，没有重试、没有记录、没有补偿！
}
```

**代码位置**: `internal/manager/pipe.go:211-215`

```go
// The campaign was manually stopped (pause, cancel).
if p.stopped.Load() {
    p.m.log.Printf("stop processing campaign (%s)", p.camp.Name)
    return  // ⚠️ 直接返回，没有检查哪些消息实际被发送了！
}
```

### 3.3 场景 C：优雅重启 (SIGHUP)

#### 代码位置: `cmd/init.go:1039-1069`

```go
func awaitReload(sigChan chan os.Signal, closerWait chan bool, closer func()) chan bool {
    out := make(chan bool)
    
    respawn := func() {
        syscall.Exec(os.Args[0], os.Args, os.Environ())
        os.Exit(0)
    }
    
    go func() {
        for range sigChan {
            lo.Println("reloading on signal ...")
            
            go closer()  // 触发关闭逻辑
            
            select {
            case <-closerWait:
                // 优雅关闭完成
                respawn()
            case <-time.After(time.Second * 3):
                // ⚠️ 超时 3 秒后强制重启！
                respawn()
            }
        }
    }()
    
    return out
}
```

#### 分析

**优雅关闭的问题**：
1. `closer()` 应该会等待所有 worker 完成
2. 但只有 **3 秒超时**
3. 如果队列中有很多消息，3 秒内处理不完
4. 强制重启后，队列中的消息丢失
5. checkpoint 已前移，同样造成订阅者丢失

---

## 第四部分：不会补发的证据

### 4.1 核心证据 1：SQL 查询条件

**代码位置**: `queries/campaigns.sql:344-347`

```sql
WHERE
    sl.list_id = ANY($5::INT[])
    -- last_subscriber_id
    AND s.id > $3        -- ⚠️ 关键：只查询大于 checkpoint 的 ID
     -- max_subscriber_id
    AND s.id <= $4       -- 上限
```

**分析**：
- 查询条件是 `s.id > last_subscriber_id`
- 一旦 `last_subscriber_id` 被更新到 200
- id ≤ 200 的订阅者**永远不会再被查询**
- 没有任何"补发"或"重试"机制会重新访问这些 ID

### 4.2 核心证据 2：暂停时直接跳过

**代码位置**: `internal/manager/manager.go:474-479` (重复引用，强调重要性)

```go
if msg.pipe != nil && msg.pipe.stopped.Load() {
    msg.pipe.wg.Done()
    continue  // ⚠️ 没有任何记录、没有任何补偿、直接丢弃
}
```

**分析**：
- 当 Campaign 被暂停时
- 队列中的消息被取出
- 检查到 `stopped=true`
- 执行 `wg.Done()` 只是为了让 `cleanup()` 能执行
- **消息本身被完全忽略，没有任何重试机制**

### 4.3 核心证据 3：cleanup 没有补偿逻辑

**代码位置**: `internal/manager/pipe.go:187-239`

```go
func (p *pipe) cleanup() {
    // ...
    
    // Update campaign's 'sent count.
    if err := p.m.store.UpdateCampaignCounts(
        p.camp.ID, 
        0, 
        int(p.sent.Load()),   // 只更新成功发送的计数
        int(p.lastID.Load())   // 只更新内存中的 lastID
    ); err != nil {
        p.m.log.Printf("error updating campaign counts (%s): %v", p.camp.Name, err)
    }
    
    // The campaign was auto-paused due to errors.
    if p.withErrors.Load() {
        // 更新状态为 paused，发送通知
        // ⚠️ 没有补发逻辑
        return
    }
    
    // The campaign was manually stopped (pause, cancel).
    if p.stopped.Load() {
        p.m.log.Printf("stop processing campaign (%s)", p.camp.Name)
        return  // ⚠️ 直接返回，什么都不做
    }
    
    // Campaign 自然结束
    // ⚠️ 同样没有补偿逻辑
    // ...
}
```

**分析**：
- `cleanup()` 只负责：
  1. 更新 `sent` 计数（内存中的成功计数）
  2. 更新 `last_subscriber_id`（内存中的 lastID）
  3. 更新状态
- **完全没有**：
  - 检查哪些订阅者实际收到了邮件
  - 对比 checkpoint 和实际发送的差异
  - 任何形式的补发或重试

### 4.4 核心证据 4：内存状态不持久化

让我回顾一下状态存储的位置：

| 状态 | 存储位置 | 崩溃后是否保留 |
|------|----------|----------------|
| `last_subscriber_id` | 数据库 `campaigns` 表 | ✅ 保留 |
| `sent` (聚合计数) | 数据库 `campaigns` 表 | ✅ 保留 |
| `pipe.sent` | 内存 `atomic.Int64` | ❌ 丢失 |
| `pipe.lastID` | 内存 `atomic.Uint64` | ❌ 丢失 |
| `pipe.errors` | 内存 `atomic.Uint64` | ❌ 丢失 |
| `pipe.wg` | 内存 `sync.WaitGroup` | ❌ 丢失 |
| `campMsgQ` | 内存 channel | ❌ 丢失 |

**关键问题**：
- `last_subscriber_id` 在 SQL 查询时就被更新到数据库
- 但实际发送状态（哪些成功、哪些失败）只在内存中
- 崩溃后：
  - ✅ checkpoint 已保留（前移了）
  - ❌ 实际发送状态丢失
  - ❌ 队列中的消息丢失
- **没有机制**来对比 checkpoint 和实际发送的差异

---

## 第五部分：对 Sent 计数和送达语义的影响

### 5.1 Sent 计数的实际含义

**代码位置**: `internal/manager/manager.go:532-543`

```go
if err != nil {
    msg.pipe.OnError()
} else {
    id := uint64(msg.Subscriber.ID)
    if id > msg.pipe.lastID.Load() {
        msg.pipe.lastID.Store(uint64(msg.Subscriber.ID))
    }
    msg.pipe.rate.Incr(1)
    msg.pipe.sent.Add(1)  // ⚠️ 只有成功发送时才 +1
}
```

**代码位置**: `queries/campaigns.sql:435-441`

```sql
UPDATE campaigns SET
    to_send=(CASE WHEN $2 != 0 THEN $2 ELSE to_send END),
    sent=sent+$3,                              -- ⚠️ 增量累加
    last_subscriber_id=(CASE WHEN $4 > 0 THEN $4 ELSE last_subscriber_id END),
    updated_at=NOW()
WHERE id=$1;
```

### 5.2 语义不一致的场景

#### 场景：批次部分成功，然后崩溃

```
假设:
- 批次: id=101-200 (100 个订阅者)
- worker 发送了 60 个成功 (id=101-160)
- worker 发送了 10 个失败 (id=161-170)
- 还有 30 个在队列中 (id=171-200)

此时:
  内存状态:
    pipe.sent = 60 (成功计数)
    pipe.lastID = 160 (最后一个成功的 ID)
    pipe.errors = 10 (错误计数)
  
  数据库状态:
    last_subscriber_id = 200 (已前移)
    sent = 之前的值 (还没更新)

⚠️ 此时进程崩溃

重启后:
  数据库状态:
    last_subscriber_id = 200 (不变)
    sent = 之前的值 (内存中的 60 丢失了)
  
  下次查询:
    s.id > 200 → 从 201 开始

实际结果:
  ✅ id=101-160: 发送成功 (60 人)
  ⚠️ id=161-170: 发送失败，不会重试 (10 人)
  ❌ id=171-200: 从未发送，也不会补发 (30 人)
  
  数据库 sent 计数:
    如果崩溃前没有调用 getCurrentCampaigns
    → sent 不会增加 60
    → 显示 sent 比实际少 60
```

### 5.3 语义总结

| 指标 | 实际含义 | 问题 |
|------|----------|------|
| `campaigns.sent` | **成功发送**的数量 | 可能与实际送达不一致（崩溃时内存计数丢失） |
| `campaigns.last_subscriber_id` | **已处理批次**的最大 ID | 不代表这些 ID 都实际收到了邮件 |
| `to_send` | 总订阅者数 |  |
| `sent / to_send` | 表面发送率 | **不是实际送达率** |

### 5.4 关键问题：无法确定谁收到了邮件

由于 listmonk 没有逐订阅者的发送记录表：

```
你无法回答以下问题：
├── 订阅者 A (ID=123) 收到邮件了吗？
│   └── ❌ 无法回答，没有记录表
│
├── 这个 Campaign 实际送达率是多少？
│   └── ⚠️ 只能看 sent 计数，但：
│       - sent 可能在崩溃时丢失更新
│       - 不包括失败的重试情况
│       - 不包括中断时丢失的消息
│
├── 上次中断时丢失了哪些订阅者？
│   └── ❌ 完全无法追踪
│
└── 如何补发那些未发送的订阅者？
    └── ❌ 没有机制，只能重新发送整个 Campaign
        → 会重复发送给已收到的用户
```

---

## 第六部分：与多 SMTP 发送的关联

### 6.1 多 SMTP 场景下的问题放大

回顾 Round1 的发现：多 SMTP 是**随机选择**，不是故障切换。

结合本轮发现：

```
场景：配置 2 个 SMTP Provider
  - Provider A: 正常工作
  - Provider B: 已故障（或配额耗尽）

执行流程：
1. NextSubscribers 获取 id=101-200，checkpoint 前移到 200
2. 推送 100 条消息到队列
3. worker 消费，随机选择 SMTP：
   - 消息 1-50: 随机选到 A，发送成功 ✓
   - 消息 51-100: 随机选到 B，发送失败 ✗
4. 错误计数累积到 MaxSendErrors
5. OnError() 调用 → p.Stop(true)
6. 队列中如果还有消息，会被跳过（参考场景 B）

结果：
  - checkpoint 已前移到 200
  - 50 人成功收到
  - 50 人失败，且不会重试
  - 不会切换到其他 Provider
  - 下次恢复从 201 开始
  - ❌ 50 人永远丢失！
```

### 6.2 问题叠加效应

| 问题 | 单独影响 | 与多 SMTP 叠加 |
|------|----------|-----------------|
| Checkpoint 前移 | 崩溃时丢失 | 加上随机选择，失败的消息也被计入 checkpoint |
| 无故障切换 | 错误累积 | 错误累积更快触发暂停 |
| 暂停时跳过消息 | 队列消息丢失 | 暂停后不会尝试其他 Provider |
| 无逐订阅者记录 | 无法追踪 | 无法知道谁通过哪个 Provider 成功/失败 |

---

## 第七部分：改进建议

### 7.1 短期修复：Checkpoint 后移策略

**问题**：当前是"获取前更新 checkpoint"，应该改为"成功发送后更新 checkpoint"

**建议方案**：两阶段提交

```
┌────────────────────────────────────────────────────────────────────────────────┐
│ 方案 A: 批次确认模式                                                            │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  执行流程:                                                                     │
│  1. 查询订阅者 (不更新 checkpoint)                                             │
│     SELECT * FROM subscribers WHERE id > last_subscriber_id ...              │
│                                                                                │
│  2. 推送消息到队列，发送                                                        │
│                                                                                │
│  3. 批次全部完成后，更新 checkpoint:                                           │
│     UPDATE campaigns SET last_subscriber_id = batch_max_id                   │
│     WHERE id = ?                                                              │
│                                                                                │
│  问题: 批次中部分成功部分失败怎么办？                                           │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────────┐
│ 方案 B: 逐订阅者确认 (需要新增表)                                                │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  新增表: campaign_sends                                                         │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ id              BIGSERIAL PRIMARY KEY                                      │ │
│  │ campaign_id     INTEGER NOT NULL REFERENCES campaigns(id)                  │ │
│  │ subscriber_id   INTEGER NOT NULL REFERENCES subscribers(id)                │ │
│  │ smtp_server     TEXT                      -- 使用的 Provider                │ │
│  │ status          TEXT NOT NULL             -- 'pending'/'sent'/'failed'   │ │
│  │ error_message   TEXT                      -- 失败原因                      │ │
│  │ attempts        INTEGER NOT NULL DEFAULT 1                                │ │
│  │ sent_at         TIMESTAMP WITH TIME ZONE                                   │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
│                                                                                │
│  执行流程:                                                                     │
│  1. 查询订阅者时，同时检查 campaign_sends 表                                   │
│     只选择尚未发送或发送失败的订阅者                                           │
│                                                                                │
│  2. 发送前插入 status='pending'                                                │
│                                                                                │
│  3. 发送成功更新 status='sent'                                                 │
│     发送失败更新 status='failed'，记录 error_message                          │
│                                                                                │
│  4. 恢复时可以重新查询 'pending'/'failed' 的记录进行补发                      │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 中期修复：暂停时不丢弃消息

**当前问题代码** (`manager.go:474-479`):
```go
if msg.pipe != nil && msg.pipe.stopped.Load() {
    msg.pipe.wg.Done()
    continue  // ⚠️ 直接丢弃
}
```

**建议修改**:
```go
// 方案1: 暂停时重新入队
if msg.pipe != nil && msg.pipe.stopped.Load() {
    // 尝试重新入队，供下次恢复时处理
    select {
    case m.campMsgQ <- msg:
        // 重新入队成功
    default:
        // 队列满了，记录日志
        m.log.Printf("warning: cannot requeue message for subscriber %d", msg.Subscriber.ID)
    }
    msg.pipe.wg.Done()
    continue
}

// 方案2: 更好的设计 - 把待处理消息记录到数据库
// 需要配合 campaign_sends 表使用
```

### 7.3 长期改进：事务性发送

**设计目标**：
1. **Exactly-once 语义**：每个订阅者确保收到一次（或明确失败）
2. **可追踪**：知道谁收到了、谁失败了、使用哪个 Provider
3. **可恢复**：中断后能精确恢复，不丢不重
4. **Provider 切换**：失败时切换到其他 Provider

**架构建议**：

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                        事务性发送架构                                            │
└────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ 1. 准备阶段                                                                    │
  │ ┌──────────────────────────────────────────────────────────────────────────┐ │
  │ │ INSERT INTO campaign_sends (campaign_id, subscriber_id, status)         │ │
  │ │ SELECT $1, subscriber_id, 'pending'                                       │ │
  │ │ FROM subscriber_lists WHERE ...                                            │ │
  │ │ ON CONFLICT DO NOTHING;                                                    │ │
  │ └──────────────────────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ 2. 发送阶段                                                                    │
  │ ┌──────────────────────────────────────────────────────────────────────────┐ │
  │ │ FOR each pending send:                                                    │ │
  │ │   1. SELECT * FROM campaign_sends WHERE status='pending' FOR UPDATE SKIP LOCKED │ │
  │ │                                                                           │ │
  │ │   2. 尝试发送:                                                            │ │
  │ │      - 随机选择 Provider (或按健康度/配额排序)                            │ │
  │ │      - 发送邮件                                                           │ │
  │ │                                                                           │ │
  │ │   3. 更新状态:                                                            │ │
  │ │      IF success:                                                          │ │
  │ │          UPDATE campaign_sends SET status='sent', sent_at=NOW()         │ │
  │ │          WHERE id = ?                                                     │ │
  │ │                                                                           │ │
  │ │      IF failed:                                                           │ │
  │ │          IF temporary error AND attempts < max_attempts:                 │ │
  │ │              UPDATE campaign_sends SET attempts=attempts+1               │ │
  │ │              -- 下次会重试                                                │ │
  │ │          ELSE:                                                            │ │
  │ │              UPDATE campaign_sends SET status='failed', error_message=?  │ │
  │ └──────────────────────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ 3. 恢复阶段 (启动/中断后)                                                      │
  │ ┌──────────────────────────────────────────────────────────────────────────┐ │
  │ │ // 查询所有未完成的发送                                                    │ │
  │ │ SELECT * FROM campaign_sends                                              │ │
  │ │ WHERE status IN ('pending', 'failed')                                    │ │
  │ │ AND campaign_id = ?                                                       │ │
  │ │ ORDER BY subscriber_id                                                    │ │
  │ │                                                                           │ │
  │ │ // 重新尝试发送                                                           │ │
  │ │ // 'failed' 且 attempts < max_attempts 的也会重试                        │ │
  │ └──────────────────────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────────────────────┘
```

### 7.4 配置建议

对于当前版本（无法修改代码），用户可以采取以下策略降低风险：

| 风险 | 缓解策略 |
|------|----------|
| 崩溃时丢失消息 | 使用较小的 `batch_size`，减少每个批次的窗口 |
| 暂停时丢失消息 | 避免在 Campaign 运行时手动暂停，等待自然结束 |
| sent 计数不准确 | 不要完全依赖 `sent` 字段，结合 bounce 和 open/click 追踪 |
| 无法补发 | 发送前导出订阅者列表，发送后对比 |

---

## 第八部分：关键代码位置速查

| 功能 | 文件路径 | 行号范围 |
|------|----------|----------|
| next-campaign-subscribers SQL | `queries/campaigns.sql` | 318-372 |
| Checkpoint 更新 (CTE) | `queries/campaigns.sql` | 367-371 |
| NextSubscribers 方法 | `internal/manager/pipe.go` | 76-134 |
| Worker 消费队列 | `internal/manager/manager.go` | 463-558 |
| 暂停时跳过消息 | `internal/manager/manager.go` | 474-479 |
| Cleanup 逻辑 | `internal/manager/pipe.go` | 187-239 |
| 成功时 sent+1 | `internal/manager/manager.go` | 541-542 |
| UpdateCampaignCounts SQL | `queries/campaigns.sql` | 435-441 |
| 优雅重启超时 | `cmd/init.go` | 1039-1069 |

---

## 总结

### 核心发现

| 发现 | 影响等级 | 说明 |
|------|----------|------|
| **Checkpoint 提前更新** | 🔴 严重 | SQL 返回前就更新 `last_subscriber_id`，创建危险时间窗口 |
| **暂停时消息被丢弃** | 🔴 严重 | `stopped=true` 时，队列中的消息被直接 `continue` 跳过 |
| **没有补发机制** | 🔴 严重 | checkpoint 前移后，之前的 ID 永远不会被查询 |
| **内存状态不持久化** | 🟠 中等 | `pipe.sent`, `pipe.wg`, `campMsgQ` 都在内存中，崩溃丢失 |
| **无逐订阅者记录** | 🟠 中等 | 无法追踪谁收到了、谁丢失了 |

### 语义澄清

| 用户可能认为 | 实际情况 |
|--------------|----------|
| `sent=1000` 表示 1000 人收到了邮件 | `sent` 是成功发送计数，但崩溃时可能丢失更新，且不包括中断时丢失的消息 |
| `last_subscriber_id=1000` 表示前 1000 人都处理了 | 只表示 checkpoint 前移到 1000，不代表这些人都实际收到了 |
| 暂停后继续，会从中断处恢复 | checkpoint 已前移，会跳过中间未发送的订阅者 |
| 配置多 SMTP 是为了高可用 | 多 SMTP 是随机负载均衡，没有故障切换，失败会暂停 Campaign |

### 实际风险

在当前实现下：

1. **任何崩溃都可能导致订阅者丢失** - 只要发生在"SQL 返回"和"所有消息处理完成"之间
2. **手动暂停几乎必然导致丢失** - 如果队列中还有消息
3. **无法恢复** - checkpoint 已前移，没有补发机制
4. **无法审计** - 没有逐订阅者记录，无法知道谁丢失了

---

**分析日期**：2026-05-05  
**分析版本**：基于当前代码库  
**关联报告**：smtp-pool-analysis.md (Round1), smtp-pool-analysis-round2.md (Round2)
