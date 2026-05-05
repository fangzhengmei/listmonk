# Listmonk 邮件推广活动调度与发送链路分析报告

## 一、概述

本文档详细分析 listmonk 邮件推广活动（Campaign）从管理后台配置、调度任务设置，到后端 Worker 批量发送邮件的完整技术链路。

## 二、活动创建流程

### 2.1 前端交互层

**文件位置**: `frontend/src/views/Campaign.vue`

活动创建的前端入口提供了完整的表单界面，包含以下关键字段：

| 字段名 | 说明 | 数据流向 |
|--------|------|----------|
| `name` | 活动名称 | 提交到后端 |
| `subject` | 邮件主题 | 提交到后端 |
| `from_email` | 发件人地址 | 提交到后端 |
| `lists` | 订阅者列表ID集合 | 提交到后端 |
| `sendLater` | 是否延迟发送 | 控制调度时间 |
| `sendAtDate` | 调度发送时间 | 提交到 `send_at` 字段 |
| `contentType` | 内容格式（richtext/html/markdown/plain/visual） | 提交到后端 |
| `messenger` | 消息通道（email/自定义） | 提交到后端 |

**关键代码流程**:

```javascript
// 创建活动
createCampaign() {
  const data = {
    name: this.form.name,
    subject: this.form.subject,
    lists: this.form.lists.map((l) => l.id),
    send_at: this.form.sendLater ? this.form.sendAtDate : null,
    // ... 其他字段
  };
  this.$api.createCampaign(data).then((d) => {
    this.$router.push({ name: 'campaign', hash: '#content', params: { id: d.id } });
  });
}

// 启动/调度活动
startCampaign() {
  // 先保存活动
  this.updateCampaign().then(() => {
    let status = '';
    if (this.canStart) {
      status = 'running';  // 立即发送
    } else if (this.canSchedule) {
      status = 'scheduled'; // 调度发送
    }
    // 调用状态变更API
    this.$api.changeCampaignStatus(this.data.id, status).then(() => {
      this.$router.push({ name: 'campaigns' });
    });
  });
}
```

### 2.2 API 层

**文件位置**: `frontend/src/api/index.js`

API 层定义了与活动相关的所有 HTTP 接口：

```javascript
// 创建活动
export const createCampaign = async (data) => http.post('/api/campaigns', data, { loading: models.campaigns });

// 更新活动
export const updateCampaign = async (id, data) => http.put(`/api/campaigns/${id}`, data);

// 变更活动状态（核心调度接口）
export const changeCampaignStatus = async (id, status) => http.put(
  `/api/campaigns/${id}/status`,
  { status },
  { loading: models.campaigns }
);
```

### 2.3 后端 Handler 层

**文件位置**: `cmd/campaigns.go`

#### 2.3.1 创建活动

```go
// CreateCampaign handles campaign creation.
// Newly created campaigns are always drafts.
func (a *App) CreateCampaign(c echo.Context) error {
    var o campReq
    if err := c.Bind(&o); err != nil {
        return err
    }
    
    // 权限过滤 - 只允许用户有权限的列表
    user := auth.GetUser(c)
    o.ListIDs = user.FilterListsByPerm(auth.PermTypeGet|auth.PermTypeManage, o.ListIDs)
    
    // 验证字段
    if c, err := a.validateCampaignFields(o); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }
    
    // 调用核心层创建
    out, err := a.core.CreateCampaign(o.Campaign, o.ListIDs, o.MediaIDs)
    if err != nil {
        return err
    }
    
    return c.JSON(http.StatusOK, okResp{out})
}
```

#### 2.3.2 变更活动状态（调度核心）

```go
// UpdateCampaignStatus handles campaign status modification.
func (a *App) UpdateCampaignStatus(c echo.Context) error {
    id := getID(c)
    
    // 权限检查
    if err := a.checkCampaignPerm(auth.PermTypeManage, id, c); err != nil {
        return err
    }
    
    req := struct {
        Status string `json:"status"`
    }{}
    if err := c.Bind(&req); err != nil {
        return err
    }
    
    // 更新数据库中的状态
    out, err := a.core.UpdateCampaignStatus(id, req.Status)
    if err != nil {
        return err
    }
    
    // 如果是暂停或取消，发送信号给 Manager 停止进行中的活动
    if req.Status == models.CampaignStatusPaused || req.Status == models.CampaignStatusCancelled {
        a.manager.StopCampaign(id)
    }
    
    return c.JSON(http.StatusOK, okResp{out})
}
```

### 2.4 核心业务层

**文件位置**: `internal/core/campaigns.go`

#### 2.4.1 创建活动

```go
// CreateCampaign creates a new campaign.
func (c *Core) CreateCampaign(o models.Campaign, listIDs []int, mediaIDs []int) (models.Campaign, error) {
    // 生成 UUID
    uu, err := uuid.NewV4()
    if err != nil {
        // ... error handling
    }
    
    var newID int
    // 执行 SQL 插入
    if err := c.q.CreateCampaign.Get(&newID,
        uu,           // uuid
        o.Type,       // type
        o.Name,       // name
        o.Subject,    // subject
        o.FromEmail,  // from_email
        o.Body,       // body
        o.AltBody,    // altbody
        o.ContentType, // content_type
        o.SendAt,     // send_at (调度时间)
        o.Headers,    // headers
        o.Attribs,    // attribs
        pq.StringArray(normalizeTags(o.Tags)),
        o.Messenger,  // messenger
        o.TemplateID, // template_id
        pq.Array(listIDs),
        o.Archive,
        o.ArchiveSlug,
        o.ArchiveTemplateID,
        o.ArchiveMeta,
        pq.Array(mediaIDs),
        o.BodySource,
    ); err != nil {
        // ... error handling
    }
    
    // 返回创建的活动
    out, err := c.GetCampaign(newID, "", "")
    return out, err
}
```

#### 2.4.2 状态变更验证

```go
// UpdateCampaignStatus updates a campaign's status, eg: draft to running.
func (c *Core) UpdateCampaignStatus(id int, status string) (models.Campaign, error) {
    cm, err := c.GetCampaign(id, "", "")
    if err != nil {
        return models.Campaign{}, err
    }
    
    // 状态流转验证
    errMsg := ""
    switch status {
    case models.CampaignStatusDraft:
        // 只有 scheduled 状态可以转为 draft
        if cm.Status != models.CampaignStatusScheduled {
            errMsg = c.i18n.T("campaigns.onlyScheduledAsDraft")
        }
    case models.CampaignStatusScheduled:
        // 只有 draft 或 paused 可以转为 scheduled
        if cm.Status != models.CampaignStatusDraft && cm.Status != models.CampaignStatusPaused {
            errMsg = c.i18n.T("campaigns.onlyDraftAsScheduled")
        }
        // 调度状态必须设置 send_at
        if !cm.SendAt.Valid {
            errMsg = c.i18n.T("campaigns.needsSendAt")
        }
    case models.CampaignStatusRunning:
        // 只有 paused 或 draft 可以转为 running
        if cm.Status != models.CampaignStatusPaused && cm.Status != models.CampaignStatusDraft {
            errMsg = c.i18n.T("campaigns.onlyPausedDraft")
        }
    case models.CampaignStatusPaused:
        // 只有 running 可以转为 paused
        if cm.Status != models.CampaignStatusRunning {
            errMsg = c.i18n.T("campaigns.onlyActivePause")
        }
    case models.CampaignStatusCancelled:
        // 只有 running 或 paused 可以转为 cancelled
        if cm.Status != models.CampaignStatusRunning && cm.Status != models.CampaignStatusPaused {
            errMsg = c.i18n.T("campaigns.onlyActiveCancel")
        }
    }
    
    if len(errMsg) > 0 {
        return models.Campaign{}, echo.NewHTTPError(http.StatusBadRequest, errMsg)
    }
    
    // 执行状态更新
    res, err := c.q.UpdateCampaignStatus.Exec(cm.ID, status)
    // ... error handling
    
    cm.Status = status
    return cm, nil
}
```

### 2.5 数据持久层

**文件位置**: `queries/campaigns.sql`

#### 2.5.1 创建活动 SQL

```sql
-- name: create-campaign
-- This creates the campaign and inserts campaign_lists relationships.
WITH tpl AS (
    -- 选择模板（可视模板特殊处理）
    SELECT
        (CASE WHEN type = 'campaign_visual' THEN NULL ELSE id END) AS id,
        (CASE WHEN type = 'campaign_visual' THEN body ELSE '' END) AS body,
        (CASE WHEN type = 'campaign_visual' THEN body_source ELSE NULL END) AS body_source,
        (CASE WHEN type = 'campaign_visual' THEN 'visual' ELSE 'richtext' END) AS content_type
    FROM templates
    WHERE
        CASE
            WHEN $14::INT IS NOT NULL THEN id = $14::INT
            ELSE $8 != 'visual' AND is_default = TRUE
        END
    LIMIT 1
),
camp AS (
    INSERT INTO campaigns (
        uuid, type, name, subject, from_email, body, altbody,
        content_type, send_at, headers, attribs, tags, messenger, 
        template_id, to_send, max_subscriber_id, archive, 
        archive_slug, archive_template_id, archive_meta, body_source
    )
    SELECT 
        $1, $2, $3, $4, $5,
        -- body: 优先使用传入值，其次使用模板
        COALESCE(NULLIF($6, ''), (SELECT body FROM tpl), ''),
        $7,
        $8::content_type,
        $9,  -- send_at (调度时间)
        $10, $11, $12, $13,
        (SELECT id FROM tpl),
        0, 0,  -- to_send, max_subscriber_id
        $16, $17, $18, $19,
        COALESCE($21, (SELECT body_source FROM tpl))
    RETURNING id
),
-- 插入媒体关联
med AS (
    INSERT INTO campaign_media (campaign_id, media_id, filename)
        (SELECT (SELECT id FROM camp), id, filename FROM media WHERE id=ANY($20::INT[]))
),
-- 插入列表关联
insLists AS (
    INSERT INTO campaign_lists (campaign_id, list_id, list_name)
        SELECT (SELECT id FROM camp), id, name FROM lists WHERE id=ANY($15::INT[])
)
SELECT id FROM camp;
```

#### 2.5.2 更新活动状态 SQL

```sql
-- name: update-campaign-status
UPDATE campaigns SET
    status=(
        CASE
            -- 如果设置了 send_at 且要转为 running，实际设为 scheduled
            WHEN send_at IS NOT NULL AND $2 = 'running' THEN 'scheduled'
            ELSE $2::campaign_status
        END
    ),
    updated_at=NOW()
WHERE id = $1;
```

## 三、调度配置机制

### 3.1 活动状态模型

**文件位置**: `models/campaigns.go`

```go
const (
    CampaignStatusDraft         = "draft"      // 草稿
    CampaignStatusScheduled     = "scheduled"  // 已调度
    CampaignStatusRunning       = "running"    // 运行中
    CampaignStatusPaused        = "paused"     // 已暂停
    CampaignStatusFinished      = "finished"   // 已完成
    CampaignStatusCancelled     = "cancelled"  // 已取消
)

// Campaign 数据模型
type Campaign struct {
    Base
    CampaignMeta
    
    UUID        string          `db:"uuid" json:"uuid"`
    Type        string          `db:"type" json:"type"`  // regular / optin
    Name        string          `db:"name" json:"name"`
    Subject     string          `db:"subject" json:"subject"`
    FromEmail   string          `db:"from_email" json:"from_email"`
    Body        string          `db:"body" json:"body"`
    SendAt      null.Time       `db:"send_at" json:"send_at"`  // 调度时间
    Status      string          `db:"status" json:"status"`    // 当前状态
    ContentType string          `db:"content_type" json:"content_type"`
    Tags        pq.StringArray  `db:"tags" json:"tags"`
    Headers     Headers         `db:"headers" json:"headers"`
    Attribs     JSON            `db:"attribs" json:"attribs"`
    TemplateID  null.Int        `db:"template_id" json:"template_id"`
    Messenger   string          `db:"messenger" json:"messenger"`  // 消息通道
    // ... 其他字段
}
```

### 3.2 状态流转图

```
                    ┌─────────────┐
                    │   draft     │
                    │   (草稿)    │
                    └──────┬──────┘
                           │
           ┌───────────────┴───────────────┐
           │                               │
           ▼                               ▼
    ┌─────────────┐                 ┌─────────────┐
    │  scheduled  │                 │   running   │
    │  (已调度)   │                 │  (运行中)   │
    └──────┬──────┘                 └──────┬──────┘
           │                               │
           │ 时间到达                        │
           ▼                               ▼
    ┌─────────────┐                 ┌─────────────┐
    │   running   │◄────────────────│   paused    │
    │  (运行中)   │    恢复发送    │  (已暂停)   │
    └──────┬──────┘                 └──────┬──────┘
           │                               │
           │                               │
           ▼                               ▼
    ┌─────────────┐                 ┌─────────────┐
    │  finished   │                 │ cancelled   │
    │  (已完成)   │                 │  (已取消)   │
    └─────────────┘                 └─────────────┘
```

### 3.3 调度时间处理

**关键逻辑**:

1. **设置调度时间**: 用户在前端开启 `sendLater` 并选择 `sendAtDate`
2. **存储调度时间**: `SendAt` 字段存储在数据库中
3. **状态转换**: 
   - 如果设置了 `send_at` 且尝试转为 `running`，SQL 会自动转为 `scheduled`
   - `scanCampaigns` 定期检查 `scheduled` 状态且 `NOW() >= send_at` 的活动

**SQL 中的特殊处理** (`queries/campaigns.sql:444-452`):

```sql
UPDATE campaigns SET
    status=(
        CASE
            -- 关键：如果有 send_at 且要设为 running，实际设为 scheduled
            WHEN send_at IS NOT NULL AND $2 = 'running' THEN 'scheduled'
            ELSE $2::campaign_status
        END
    ),
    updated_at=NOW()
WHERE id = $1;
```

## 四、后端 Worker 批量发送链路

### 4.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        main() 启动流程                            │
├─────────────────────────────────────────────────────────────────┤
│  1. initCampaignManager()  // 初始化 Manager                     │
│  2. go mgr.Run()          // 启动 Worker 协程                    │
│     ├── scanCampaigns()   // 定期扫描数据库中的活动              │
│     ├── worker() * N      // 并发消息发送 Worker                 │
│     └── nextPipes 处理    // 处理活动管道队列                    │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Manager 初始化

**文件位置**: `cmd/init.go:587-622`

```go
func initCampaignManager(msgrs []manager.Messenger, q *models.Queries, 
    u *UrlConfig, co *core.Core, md media.Store, i *i18n.I18n, ko *koanf.Koanf) *manager.Manager {
    
    if ko.Bool("passive") {
        lo.Println("running in passive mode. won't process campaigns.")
    }
    
    mgr := manager.New(manager.Config{
        BatchSize:             ko.Int("app.batch_size"),       // 每批订阅者数量
        Concurrency:           ko.Int("app.concurrency"),       // 并发 Worker 数
        MessageRate:           ko.Int("app.message_rate"),      // 每秒发送速率
        MaxSendErrors:         ko.Int("app.max_send_errors"),   // 最大错误数
        FromEmail:             ko.String("app.from_email"),
        // ... 跟踪相关配置
        SlidingWindow:         ko.Bool("app.message_sliding_window"),
        SlidingWindowDuration: ko.Duration("app.message_sliding_window_duration"),
        SlidingWindowRate:     ko.Int("app.message_sliding_window_rate"),
        ScanInterval:          time.Second * 5,                  // 扫描间隔 5 秒
        ScanCampaigns:         !ko.Bool("passive"),
    }, newManagerStore(q, co, md), i, lo)
    
    // 注册所有消息通道
    for _, m := range msgrs {
        mgr.AddMessenger(m)
    }
    
    return mgr
}
```

**配置项说明**:

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `batch_size` | 1000 | 每次从数据库拉取的订阅者数量 |
| `concurrency` | 1 | 并发 Worker 数量 |
| `message_rate` | 1 | 每秒发送速率限制 |
| `scan_interval` | 5s | 数据库扫描间隔 |
| `max_send_errors` | 100 | 最大连续错误数（超过则暂停） |

### 4.3 Manager 运行主循环

**文件位置**: `internal/manager/manager.go:266-316`

```go
// Run is a blocking function that scans the data source at regular intervals
// for pending campaigns, and queues them for processing.
func (m *Manager) Run() {
    if m.cfg.ScanCampaigns {
        // 启动定期扫描协程
        go m.scanCampaigns(m.cfg.ScanInterval)
    }
    
    // 启动 N 个消息 Worker
    for i := 0; i < m.cfg.Concurrency; i++ {
        go m.worker()
    }
    
    // 主循环：处理活动管道队列
    for p := range m.nextPipes {
        // 拉取下一批订阅者
        has, err := p.NextSubscribers()
        if err != nil {
            m.log.Printf("error processing campaign batch (%s): %v", p.camp.Name, err)
            p.Stop(false)
            p.wg.Done()
            continue
        }
        
        if has {
            // 还有更多订阅者，重新入队
            select {
            case m.nextPipes <- p:
            default:
                // 队列已满，停止后由下次扫描重新处理
                p.Stop(false)
                p.wg.Done()
            }
        } else {
            // 所有订阅者处理完毕
            p.wg.Done()
        }
    }
}
```

### 4.4 活动扫描机制

**文件位置**: `internal/manager/manager.go:423-459`

```go
// scanCampaigns is a blocking function that periodically scans the data source
// for campaigns to process and dispatches them to the manager.
func (m *Manager) scanCampaigns(tick time.Duration) {
    t := time.NewTicker(tick)
    defer t.Stop()
    
    // 定期扫描
    for range t.C {
        // 获取当前正在处理的活动ID和发送计数
        ids, counts := m.getCurrentCampaigns()
        
        // 从数据库获取下一批活动
        campaigns, err := m.store.NextCampaigns(ids, counts)
        if err != nil {
            m.log.Printf("error fetching campaigns: %v", err)
            continue
        }
        
        for _, c := range campaigns {
            // 为每个活动创建处理管道
            p, err := m.newPipe(c)
            if err != nil {
                m.log.Printf("error processing campaign (%s): %v", c.Name, err)
                continue
            }
            m.log.Printf("start processing campaign (%s)", c.Name)
            
            // 将管道加入处理队列
            select {
            case m.nextPipes <- p:
            default:
                // 队列已满，停止并等待下次扫描
                p.Stop(false)
                p.wg.Done()
            }
        }
    }
}
```

**数据库查询**: `queries/campaigns.sql:174-232`

```sql
-- name: next-campaigns
-- 获取需要处理的活动（running 或 scheduled 且时间已到）
WITH camps AS (
    -- 获取所有 running 活动，或 scheduled 且时间已到的活动
    SELECT campaigns.*, COALESCE(templates.body, ...) AS template_body
    FROM campaigns
    LEFT JOIN templates ON (templates.id = campaigns.template_id)
    WHERE (status='running' OR (status='scheduled' AND NOW() >= campaigns.send_at))
    AND NOT(campaigns.id = ANY($1::INT[]))  -- 排除当前正在处理的
),
campLists AS (
    -- 获取活动关联的列表
    SELECT lists.id AS list_id, campaign_id, optin FROM lists
    INNER JOIN campaign_lists ON (campaign_lists.list_id = lists.id)
    WHERE campaign_lists.campaign_id = ANY(SELECT id FROM camps)
),
counts AS (
    -- 计算每个活动需要发送的订阅者总数
    SELECT camps.id AS campaign_id, 
           COUNT(DISTINCT sl.subscriber_id) AS to_send,
           COALESCE(MAX(sl.subscriber_id), 0) AS max_subscriber_id
    FROM camps
    JOIN campLists cl ON cl.campaign_id = camps.id
    JOIN subscriber_lists sl ON sl.list_id = cl.list_id
        AND (
            CASE
                WHEN camps.type = 'optin' THEN sl.status = 'unconfirmed' AND cl.optin = 'double'
                WHEN cl.optin = 'double' THEN sl.status = 'confirmed'
                ELSE sl.status != 'unsubscribed'
            END
        )
    JOIN subscribers s ON (s.id = sl.subscriber_id AND s.status != 'blocklisted')
    GROUP BY camps.id
),
u AS (
    -- 更新活动的发送统计，并将 scheduled 转为 running
    UPDATE campaigns AS ca
    SET to_send = co.to_send,
        status = (CASE WHEN status != 'running' THEN 'running' ELSE status END),
        max_subscriber_id = co.max_subscriber_id,
        started_at=(CASE WHEN ca.started_at IS NULL THEN NOW() ELSE ca.started_at END)
    FROM (SELECT * FROM counts) co
    WHERE ca.id = co.campaign_id
)
SELECT camps.*, campMedia.media_id FROM camps LEFT JOIN campMedia ON (campMedia.campaign_id = camps.id);
```

### 4.5 Pipe 管道机制

**文件位置**: `internal/manager/pipe.go`

```go
type pipe struct {
    camp       *models.Campaign      // 关联的活动
    rate       *ratecounter.RateCounter // 发送速率统计
    wg         *sync.WaitGroup       // 等待组（跟踪消息处理）
    sent       atomic.Int64          // 已发送计数
    lastID     atomic.Uint64         // 最后处理的订阅者ID
    errors     atomic.Uint64         // 错误计数
    stopped    atomic.Bool           // 是否已停止
    withErrors atomic.Bool           // 是否因错误停止
    m          *Manager              // 父 Manager
}

// newPipe adds a campaign to the process queue.
func (m *Manager) newPipe(c *models.Campaign) (*pipe, error) {
    // 验证消息通道
    if _, ok := m.messengers[c.Messenger]; !ok {
        m.store.UpdateCampaignStatus(c.ID, models.CampaignStatusCancelled)
        return nil, fmt.Errorf("unknown messenger %s on campaign %s", c.Messenger, c.Name)
    }
    
    // 编译模板
    if err := c.CompileTemplate(m.TemplateFuncs(c)); err != nil {
        return nil, err
    }
    
    // 加载附件
    if err := m.attachMedia(c); err != nil {
        return nil, err
    }
    
    // 创建管道
    p := &pipe{
        camp: c,
        rate: ratecounter.NewRateCounter(time.Minute),
        wg:   &sync.WaitGroup{},
        m:    m,
    }
    
    // 初始 +1，确保 Wait() 阻塞直到有消息处理
    p.wg.Add(1)
    
    // 启动清理协程
    go func() {
        p.wg.Wait()  // 等待所有消息处理完成
        p.cleanup()  // 清理
    }()
    
    // 注册到活动管道 Map
    m.pipesMut.Lock()
    m.pipes[c.ID] = p
    m.pipesMut.Unlock()
    
    return p, nil
}
```

### 4.6 批量拉取订阅者

**文件位置**: `internal/manager/pipe.go:72-134`

```go
// NextSubscribers processes the next batch of subscribers in a given campaign.
func (p *pipe) NextSubscribers() (bool, error) {
    // 从数据库拉取下一批订阅者
    subs, err := p.m.store.NextSubscribers(p.camp.ID, p.m.cfg.BatchSize)
    if err != nil {
        return false, fmt.Errorf("error fetching campaign subscribers (%s): %v", p.camp.Name, err)
    }
    
    // 没有更多订阅者
    if len(subs) == 0 {
        return false, nil
    }
    
    // 检查滑动窗口限流
    hasSliding := p.m.cfg.SlidingWindow &&
        p.m.cfg.SlidingWindowRate > 0 &&
        p.m.cfg.SlidingWindowDuration.Seconds() > 1
    
    // 为每个订阅者创建消息并推入队列
    for _, s := range subs {
        msg, err := p.newMessage(s)
        if err != nil {
            p.m.log.Printf("error rendering message (%s) (%s): %v", p.camp.Name, s.Email, err)
            continue
        }
        
        // 推入消息队列（阻塞等待）
        p.m.campMsgQ <- msg
        
        // 滑动窗口限流逻辑
        if hasSliding {
            diff := time.Since(p.m.slidingStart)
            
            // 窗口过期，重置
            if diff >= p.m.cfg.SlidingWindowDuration {
                p.m.slidingStart = time.Now()
                p.m.slidingCount = 0
            }
            
            p.m.slidingCount++
            // 达到限流阈值，休眠等待
            if p.m.slidingCount >= p.m.cfg.SlidingWindowRate {
                wait := p.m.cfg.SlidingWindowDuration - diff
                p.m.log.Printf("messages exceeded (%d) for the window (%v). Sleeping for %s.",
                    p.m.slidingCount, p.m.cfg.SlidingWindowDuration, wait)
                p.m.slidingCount = 0
                time.Sleep(wait)
            }
        }
    }
    
    return true, nil
}
```

**订阅者查询 SQL**: `queries/campaigns.sql:318-372`

```sql
-- name: next-campaign-subscribers
-- 拉取活动的下一批订阅者（基于 last_subscriber_id 游标）
WITH campLists AS (
    SELECT lists.id AS list_id, optin FROM lists
    LEFT JOIN campaign_lists ON campaign_lists.list_id = lists.id
    WHERE campaign_lists.campaign_id = $1
),
subs AS (
    SELECT s.*
    FROM (
        SELECT DISTINCT s.id
        FROM subscriber_lists sl
        JOIN campLists ON sl.list_id = campLists.list_id
        JOIN subscribers s ON s.id = sl.subscriber_id
        WHERE
            sl.list_id = ANY($5::INT[])
            -- 游标：上次处理的最后一个ID
            AND s.id > $3   -- last_subscriber_id
            -- 上限ID（优化查询）
            AND s.id <= $4  -- max_subscriber_id
            -- 排除黑名单
            AND s.status != 'blocklisted'
            AND (
                -- 确认订阅状态逻辑
                ($2 = 'optin' AND sl.status = 'unconfirmed' AND campLists.optin = 'double')
                OR (
                    $2 != 'optin' AND (
                        (campLists.optin = 'double' AND sl.status = 'confirmed') OR
                        (campLists.optin != 'double' AND sl.status != 'unsubscribed')
                    )
                )
            )
        ORDER BY s.id LIMIT $6  -- batch_size
    ) subIDs JOIN subscribers s ON (s.id = subIDs.id) ORDER BY s.id
),
u AS (
    -- 更新游标位置
    UPDATE campaigns
    SET last_subscriber_id = (SELECT MAX(id) FROM subs), updated_at = NOW()
    WHERE (SELECT COUNT(id) FROM subs) > 0 AND id=$1
)
SELECT * FROM subs;
```

### 4.7 消息发送 Worker

**文件位置**: `internal/manager/manager.go:463-558`

```go
func (m *Manager) worker() {
    numMsg := 0  // 速率限制计数器
    for {
        select {
        // 处理活动消息
        case msg, ok := <-m.campMsgQ:
            if !ok {
                return
            }
            
            // 检查活动是否已停止
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
            
            // 构建外发消息
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
            
            // 添加邮件头
            h := textproto.MIMEHeader{}
            h.Set(models.EmailHeaderCampaignUUID, msg.Campaign.UUID)
            h.Set(models.EmailHeaderSubscriberUUID, msg.Subscriber.UUID)
            
            // List-Unsubscribe 头
            if m.cfg.UnsubHeader {
                h.Set("List-Unsubscribe-Post", "List-Unsubscribe=One-Click")
                h.Set("List-Unsubscribe", `<`+msg.unsubURL+`>`)
            }
            
            // 自定义头
            for _, set := range msg.headers {
                for hdr, val := range set {
                    h.Add(hdr, val)
                }
            }
            out.Headers = h
            
            // 调用 Messenger 发送
            err := m.messengers[msg.Campaign.Messenger].Push(out)
            if err != nil {
                m.log.Printf("error sending message in campaign %s: subscriber %d: %v", 
                    msg.Campaign.Name, msg.Subscriber.ID, err)
            }
            
            // 更新管道统计
            if msg.pipe != nil {
                msg.pipe.wg.Done()
                
                if err != nil {
                    // 错误处理：计数并可能暂停
                    msg.pipe.OnError()
                } else {
                    // 记录最后处理的订阅者ID
                    id := uint64(msg.Subscriber.ID)
                    if id > msg.pipe.lastID.Load() {
                        msg.pipe.lastID.Store(uint64(msg.Subscriber.ID))
                    }
                    msg.pipe.rate.Incr(1)
                    msg.pipe.sent.Add(1)
                }
            }
            
        // 处理任意消息（非活动消息，如测试邮件）
        case msg, ok := <-m.msgQ:
            if !ok {
                return
            }
            if err := m.messengers[msg.Messenger].Push(msg); err != nil {
                m.log.Printf("error sending message '%s': %v", msg.Subject, err)
            }
        }
    }
}
```

### 4.8 错误处理与暂停

**文件位置**: `internal/manager/pipe.go:136-151`

```go
// OnError keeps track of the number of errors that occur while sending messages
// and pauses the campaign if the error threshold is met.
func (p *pipe) OnError() {
    if p.m.cfg.MaxSendErrors < 1 {
        return
    }
    
    // 增加错误计数
    count := p.errors.Add(1)
    if int(count) < p.m.cfg.MaxSendErrors {
        return
    }
    
    // 达到阈值，暂停活动
    p.Stop(true)
    p.m.log.Printf("error count exceeded %d. pausing campaign %s", p.m.cfg.MaxSendErrors, p.camp.Name)
}
```

### 4.9 活动清理与完成

**文件位置**: `internal/manager/pipe.go:186-239`

```go
func (p *pipe) cleanup() {
    defer func() {
        // 从活动 Map 中移除
        p.m.pipesMut.Lock()
        delete(p.m.pipes, p.camp.ID)
        p.m.pipesMut.Unlock()
    }()
    
    // 更新发送计数到数据库
    if err := p.m.store.UpdateCampaignCounts(p.camp.ID, 0, 
        int(p.sent.Load()), int(p.lastID.Load())); err != nil {
        p.m.log.Printf("error updating campaign counts (%s): %v", p.camp.Name, err)
    }
    
    // 情况1：因错误暂停
    if p.withErrors.Load() {
        if err := p.m.store.UpdateCampaignStatus(p.camp.ID, models.CampaignStatusPaused); err != nil {
            p.m.log.Printf("error updating campaign (%s) status to %s: %v", 
                p.camp.Name, models.CampaignStatusPaused, err)
        }
        _ = p.m.sendNotif(p.camp, models.CampaignStatusPaused, "Too many errors")
        return
    }
    
    // 情况2：手动停止（暂停/取消）
    if p.stopped.Load() {
        p.m.log.Printf("stop processing campaign (%s)", p.camp.Name)
        return
    }
    
    // 情况3：自然完成（所有订阅者处理完毕）
    c, err := p.m.store.GetCampaign(p.camp.ID)
    if err != nil {
        p.m.log.Printf("error fetching campaign (%s) for ending: %v", p.camp.Name, err)
        return
    }
    
    // 标记为完成
    if c.Status == models.CampaignStatusRunning || c.Status == models.CampaignStatusScheduled {
        c.Status = models.CampaignStatusFinished
        if err := p.m.store.UpdateCampaignStatus(p.camp.ID, models.CampaignStatusFinished); err != nil {
            p.m.log.Printf("error finishing campaign (%s): %v", p.camp.Name, err)
        }
    }
    
    // 发送通知给管理员
    _ = p.m.sendNotif(c, c.Status, "")
}
```

## 五、关键数据流图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           完整数据流                                           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────┐      POST /api/campaigns        ┌──────────────────┐          │
│  │  Frontend │ ───────────────────────────────►│  cmd/campaigns  │          │
│  │ (Vue.js)  │    {name, subject, lists,      │  (CreateCampaign)│          │
│  └──────────┘     send_at, messenger}         └────────┬─────────┘          │
│                                                          │                    │
│                                                          ▼                    │
│                                                ┌──────────────────┐          │
│                                                │ internal/core    │          │
│                                                │  (CreateCampaign)│          │
│                                                └────────┬─────────┘          │
│                                                          │                    │
│                                                          ▼                    │
│                                                ┌──────────────────┐          │
│                                                │   PostgreSQL     │          │
│                                                │  campaigns 表    │          │
│                                                │  status='draft'  │          │
│                                                │  send_at=...     │          │
│                                                └──────────────────┘          │
│                                                                              │
│  ┌──────────┐   PUT /api/campaigns/:id/status  ┌──────────────────┐          │
│  │  Frontend │ ───────────────────────────────►│  cmd/campaigns   │          │
│  │          │    {status: 'scheduled'}         │ (UpdateCampaign  │          │
│  └──────────┘    或 {status: 'running'}        │     Status)      │          │
│                                                └────────┬─────────┘          │
│                                                          │                    │
│                                                          ▼                    │
│                                                ┌──────────────────┐          │
│                                                │   PostgreSQL     │          │
│                                                │  status 更新为   │          │
│                                                │  'scheduled' 或  │          │
│                                                │  'running'       │          │
│                                                └──────────────────┘          │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         Manager.Run() 后台循环                         │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │                                                                      │   │
│  │  scanCampaigns() ◄──────────────────────────────────────────────────│   │
│  │       │                    每 5 秒                                   │   │
│  │       ▼                                                               │   │
│  │  SELECT ... FROM campaigns                                            │   │
│  │  WHERE (status='running' OR (status='scheduled' AND NOW()>=send_at))│   │
│  │       │                                                               │   │
│  │       ▼                                                               │   │
│  │  newPipe() ──► 编译模板、加载附件                                      │   │
│  │       │                                                               │   │
│  │       ▼                                                               │   │
│  │  加入 nextPipes 队列                                                  │   │
│  │       │                                                               │   │
│  │       ▼                                                               │   │
│  │  NextSubscribers()                                                   │   │
│  │       │                                                               │   │
│  │       ▼                                                               │   │
│  │  SELECT ... FROM subscribers                                          │   │
│  │  WHERE id > last_subscriber_id                                        │   │
│  │  LIMIT batch_size                                                     │   │
│  │       │                                                               │   │
│  │       ▼                                                               │   │
│  │  为每个订阅者 newMessage()                                             │   │
│  │       │                                                               │   │
│  │       ▼                                                               │   │
│  │  推入 campMsgQ 队列                                                   │   │
│  │       │                                                               │   │
│  │       ▼                                                               │   │
│  │  worker() 协程消费                                                     │   │
│  │       │                                                               │   │
│  │       ▼                                                               │   │
│  │  messenger.Push() 发送邮件（SMTP/Postback）                          │   │
│  │       │                                                               │   │
│  │       ▼                                                               │   │
│  │  更新 sent、lastID、rate 统计                                          │   │
│  │                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  所有订阅者处理完毕？                                                         │
│       │                                                                      │
│       ▼ (是)                                                                 │
│  cleanup()                                                                   │
│       │                                                                      │
│       ▼                                                                      │
│  UPDATE campaigns SET status='finished'                                      │
│  发送通知邮件给管理员                                                          │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

## 六、关键文件索引

| 文件路径 | 职责说明 |
|----------|----------|
| `frontend/src/views/Campaign.vue` | 活动创建/编辑前端界面 |
| `frontend/src/api/index.js` | 前端 API 调用封装 |
| `cmd/campaigns.go` | 活动相关 HTTP Handler |
| `cmd/main.go` | 主入口，启动 Manager |
| `cmd/init.go` | Manager 初始化配置 |
| `internal/core/campaigns.go` | 活动核心业务逻辑 |
| `internal/manager/manager.go` | 调度管理器主逻辑 |
| `internal/manager/pipe.go` | 单个活动处理管道 |
| `internal/manager/message.go` | 邮件消息构建 |
| `models/campaigns.go` | 活动数据模型定义 |
| `queries/campaigns.sql` | 活动相关 SQL 查询 |

## 七、配置调优建议

### 7.1 并发与吞吐量配置

```toml
[app]
# 每批从数据库拉取的订阅者数量（建议设为 Worker 数 * 100-1000）
batch_size = 1000

# 并发 Worker 数量（建议设为 CPU 核心数）
concurrency = 4

# 每个 Worker 每秒发送消息数（需根据 SMTP 服务商限制调整）
message_rate = 10

# 最大连续错误数（达到后暂停活动）
max_send_errors = 100
```

### 7.2 滑动窗口限流（应对服务商限流）

```toml
[app]
# 启用滑动窗口限流
message_sliding_window = true

# 窗口时长
message_sliding_window_duration = "1h"

# 窗口内最大发送量
message_sliding_window_rate = 10000
```

### 7.3 多实例部署

配置 `passive = true` 可让某些实例只处理 Web 请求而不处理发送任务：

```toml
[app]
# 被动模式：不处理活动发送
passive = true
```

## 八、总结

listmonk 的活动调度与发送体系设计精巧，主要特点：

1. **状态驱动**: 通过数据库 `status` 字段驱动整个生命周期，支持从 draft → scheduled → running → finished 的完整流转

2. **轮询调度**: 使用 5 秒间隔的数据库轮询（`scanCampaigns`）检查待处理活动，实现调度时间触发

3. **游标分页**: 使用 `last_subscriber_id` 游标而非 OFFSET 进行订阅者分页，支持千万级订阅者表高效查询

4. **管道隔离**: 每个活动有独立的 `pipe` 管道，使用 `WaitGroup` 跟踪消息处理，支持优雅暂停/恢复

5. **多级限流**: 支持每秒速率（`message_rate`）和滑动窗口（`SlidingWindow`）两级限流，应对不同服务商限制

6. **错误自愈**: 连续错误超过阈值自动暂停，避免因 SMTP 故障导致大量失败

这种设计既保证了批量发送的高效性，又通过数据库状态持久化保证了可靠性，是邮件群发系统的典型优秀实践。
