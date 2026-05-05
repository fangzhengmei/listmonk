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

### 3.1 关键约束验证

#### 3.1.1 延迟发送时间必须晚于当前时间

**验证位置**: `cmd/campaigns.go:694-699` 的 `validateCampaignFields` 函数

```go
// validateCampaignFields validates incoming campaign field values.
func (a *App) validateCampaignFields(c campReq) (campReq, error) {
    // ... 其他验证 ...

    // If there's a "send_at" date, it should be in the future.
    if c.SendAt.Valid {
        if c.SendAt.Time.Before(time.Now()) {
            return c, errors.New(a.i18n.T("campaigns.fieldInvalidSendAt"))
        }
    }

    // ... 其他验证 ...
}
```

**验证时机**:
- 创建活动时（`CreateCampaign`）
- 更新活动时（`UpdateCampaign`）

**错误消息** (国际化):
| 语言 | 消息 |
|------|------|
| 简体中文 | "预定日期应该在将来。" |
| 英文 | "Scheduled date should be in the future." |

**注意**: 这个验证只在前端提交数据时进行。如果直接操作数据库将 `send_at` 设为过去时间，`scanCampaigns` 会在下次扫描时直接触发发送。

#### 3.1.2 调度状态必须设置 send_at

**验证位置**: `internal/core/campaigns.go:262-268`

```go
case models.CampaignStatusScheduled:
    // 只有 draft 或 paused 可以转为 scheduled
    if cm.Status != models.CampaignStatusDraft && cm.Status != models.CampaignStatusPaused {
        errMsg = c.i18n.T("campaigns.onlyDraftAsScheduled")
    }
    // 关键约束：调度状态必须设置 send_at
    if !cm.SendAt.Valid {
        errMsg = c.i18n.T("campaigns.needsSendAt")
    }
```

**错误消息**: "广告系列需要安排一个日期。"

### 3.2 活动状态模型

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

### 4.9 暂停/取消后的停止机制

#### 4.9.1 停止触发路径

**场景1：用户手动暂停**

```
触发路径：
1. 用户点击"暂停"按钮
2. 前端调用 PUT /api/campaigns/:id/status {status: 'paused'}
3. 后端 Handler (UpdateCampaignStatus) 验证状态
4. 更新数据库 status = 'paused'
5. 调用 a.manager.StopCampaign(id) 发送停止信号
6. Worker 层级开始处理停止
```

**代码路径** (`cmd/campaigns.go:351-380`):

```go
// UpdateCampaignStatus handles campaign status modification.
func (a *App) UpdateCampaignStatus(c echo.Context) error {
    // ... 权限检查和参数绑定 ...
    
    // 更新数据库中的状态
    out, err := a.core.UpdateCampaignStatus(id, req.Status)
    if err != nil {
        return err
    }
    
    // 关键：如果是暂停或取消，发送信号给 Manager
    if req.Status == models.CampaignStatusPaused || req.Status == models.CampaignStatusCancelled {
        a.manager.StopCampaign(id)
    }
    
    return c.JSON(http.StatusOK, okResp{out})
}
```

**Manager 层的停止** (`internal/manager/manager.go:405-412`):

```go
// StopCampaign marks a running campaign as stopped so that all its queued messages are ignored.
func (m *Manager) StopCampaign(id int) {
    m.pipesMut.RLock()
    if p, ok := m.pipes[id]; ok {
        p.Stop(false)
    }
    m.pipesMut.RUnlock()
}
```

**Pipe 层的停止** (`internal/manager/pipe.go:153-167`):

```go
// Stop "marks" a campaign as stopped. It doesn't actually stop the processing
// of messages. That happens when every queued message in the campaign is processed,
// marking .wg, the waitgroup counter as done. That triggers cleanup().
func (p *pipe) Stop(withErrors bool) {
    // Already stopped.
    if p.stopped.Load() {
        return
    }

    if withErrors {
        p.withErrors.Store(true)
    }

    p.stopped.Store(true)
}
```

**关键设计**: `Stop()` 只是设置 `stopped` 原子标志，并不直接中断任何协程。这是一种**协作式停止**设计。

#### 4.9.2 多层停止检查机制

停止信号通过三个层级逐层检查：

| 层级 | 检查位置 | 检查时机 | 处理方式 |
|------|----------|----------|----------|
| **Layer 1: Worker 发送层** | `internal/manager/manager.go:474-479` | 消息出队后发送前 | 跳过发送，直接 wg.Done() |
| **Layer 2: NextSubscribers 层** | `internal/manager/pipe.go:83-87` | 拉取订阅者后 | 返回 false，停止继续入队 |
| **Layer 3: SQL 查询层** | `queries/campaigns.sql:318-372` | 查询订阅者时 | 状态变化导致查询返回空 |

**Layer 1: Worker 发送层检查** (`internal/manager/manager.go:463-558`):

```go
func (m *Manager) worker() {
    numMsg := 0
    for {
        select {
        case msg, ok := <-m.campMsgQ:
            if !ok {
                return
            }
            
            // 关键检查：活动是否已停止
            if msg.pipe != nil && msg.pipe.stopped.Load() {
                // 如果已停止，跳过发送，直接减少等待组计数
                msg.pipe.wg.Done()
                continue
            }
            
            // ... 后续发送逻辑 ...
        }
    }
}
```

**Layer 2: NextSubscribers 层检查** (`internal/manager/pipe.go:72-134`):

```go
func (p *pipe) NextSubscribers() (bool, error) {
    // 从数据库拉取下一批订阅者
    subs, err := p.m.store.NextSubscribers(p.camp.ID, p.m.cfg.BatchSize)
    if err != nil {
        return false, fmt.Errorf("error fetching campaign subscribers (%s): %v", p.camp.Name, err)
    }
    
    // 关键检查：没有订阅者可能是因为活动已暂停/取消
    // 当活动状态从 running 变为 paused/cancelled 时，
    // next-campaign-subscribers 查询会返回空结果
    if len(subs) == 0 {
        return false, nil  // 返回 false 表示没有更多订阅者
    }
    
    // ... 继续处理订阅者 ...
}
```

**Layer 3: SQL 查询层** (`queries/campaigns.sql:318-372`):

虽然 `next-campaign-subscribers` 查询本身不直接检查活动状态，但：
1. 活动状态变化后，`scanCampaigns` 不会再选取该活动
2. 但已在处理的 pipe 会继续，直到检测到 `stopped` 标志

#### 4.9.3 已入队消息的处理

**问题**: 当调用 `StopCampaign()` 时，可能有大量消息已经入队 (`campMsgQ` 队列中)，这些消息怎么办？

**答案**: 这些消息会被 Worker 消费，但在发送前会检查 `stopped` 标志，然后直接跳过发送。

**处理流程**:

```
时间线：
T0: 活动正常运行
    - NextSubscribers() 拉取 1000 个订阅者
    - 1000 条消息推入 campMsgQ 队列
    - Worker 开始处理这些消息

T1: 用户点击暂停
    - 数据库 status = 'paused'
    - 调用 StopCampaign(id)
    - pipe.stopped.Store(true)

T2: Worker 继续消费队列中的消息
    - 消息 501: 检查 stopped=true → 跳过发送，wg.Done()
    - 消息 502: 检查 stopped=true → 跳过发送，wg.Done()
    - ...
    - 消息 1000: 检查 stopped=true → 跳过发送，wg.Done()

T3: 所有消息处理完毕
    - wg 计数器归零
    - 触发 p.cleanup()
    - 从 pipes Map 中移除
    - 记录日志 "stop processing campaign (xxx)"
```

**关键代码** (`internal/manager/pipe.go:186-239`):

```go
func (p *pipe) cleanup() {
    // ... 从 pipes Map 移除 ...
    
    // 情况1：因错误暂停
    if p.withErrors.Load() {
        // 更新数据库状态为 paused
        if err := p.m.store.UpdateCampaignStatus(p.camp.ID, models.CampaignStatusPaused); err != nil {
            // ...
        }
        _ = p.m.sendNotif(p.camp, models.CampaignStatusPaused, "Too many errors")
        return
    }
    
    // 情况2：手动停止（暂停/取消）
    if p.stopped.Load() {
        p.m.log.Printf("stop processing campaign (%s)", p.camp.Name)
        return  // 注意：这里不更新数据库状态！
    }
    
    // 情况3：自然完成
    // ...
}
```

**注意**: 手动暂停的情况下，`cleanup()` 不会更新数据库状态，因为数据库状态已经在 `UpdateCampaignStatus` Handler 中更新过了。

### 4.10 恢复机制

#### 4.10.1 可恢复状态

**可编辑/可恢复的状态** (`cmd/campaigns.go:830-834`):

```go
func canEditCampaign(status string) bool {
    return status == models.CampaignStatusDraft ||
        status == models.CampaignStatusPaused ||
        status == models.CampaignStatusScheduled
}
```

**恢复场景**:

| 当前状态 | 可恢复到 | 条件 |
|----------|----------|------|
| `paused` | `running` | 立即恢复发送 |
| `paused` | `scheduled` | 必须设置新的 send_at |
| `scheduled` | `running` | 有 send_at 时会被 SQL 拦截为 scheduled |
| `scheduled` | `draft` | 只有 scheduled 可以转回 draft |

#### 4.10.2 从 paused 恢复到 running

**触发路径**:

```
1. 用户点击"恢复"按钮
2. 前端调用 PUT /api/campaigns/:id/status {status: 'running'}
3. 验证：paused 可以转为 running（通过状态流转检查）
4. 更新数据库 status = 'running'
5. 等待 scanCampaigns 下次扫描（最多 5 秒）
6. scanCampaigns 检测到 status='running' 且不在当前处理列表中
7. 创建新的 pipe
8. 从 last_subscriber_id 继续拉取订阅者
```

**关键代码**:

**状态流转验证** (`internal/core/campaigns.go:270-273`):

```go
case models.CampaignStatusRunning:
    // 只有 paused 或 draft 可以转为 running
    if cm.Status != models.CampaignStatusPaused && cm.Status != models.CampaignStatusDraft {
        errMsg = c.i18n.T("campaigns.onlyPausedDraft")
    }
```

**scanCampaigns 重新处理** (`internal/manager/manager.go:423-459`):

```go
func (m *Manager) scanCampaigns(tick time.Duration) {
    // ...
    for range t.C {
        // 获取当前正在处理的活动 ID
        ids, counts := m.getCurrentCampaigns()
        
        // 查询条件：
        // WHERE (status='running' OR (status='scheduled' AND NOW()>=send_at))
        // AND NOT(campaigns.id = ANY($1::INT[]))  // 排除当前正在处理的
        campaigns, err := m.store.NextCampaigns(ids, counts)
        // ...
        
        for _, c := range campaigns {
            // 创建新的 pipe
            p, err := m.newPipe(c)
            // ...
            
            // 加入处理队列
            select {
            case m.nextPipes <- p:
            default:
                // 队列满则稍后重试
                p.Stop(false)
                p.wg.Done()
            }
        }
    }
}
```

#### 4.10.3 断点续传机制（游标设计）

**核心设计**: 使用 `last_subscriber_id` 作为游标，而非 OFFSET 分页。

**数据表中的字段**:

```sql
campaigns 表:
- last_subscriber_id: 上次处理的最后一个订阅者 ID
- max_subscriber_id: 本次活动的订阅者 ID 上限（首次扫描时计算）
```

**SQL 查询** (`queries/campaigns.sql:318-372`):

```sql
-- name: next-campaign-subscribers
WITH campLists AS (
    -- ...
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
            -- 关键：游标位置
            AND s.id > $3   -- last_subscriber_id
            -- 关键：上限 ID（优化查询）
            AND s.id <= $4  -- max_subscriber_id
            -- 排除黑名单
            AND s.status != 'blocklisted'
            -- ... 订阅状态检查 ...
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

**恢复时的数据流**:

```
场景：活动发送了 2500 个订阅者后被暂停，然后恢复

暂停前：
- last_subscriber_id = 2500
- max_subscriber_id = 10000
- sent = 2500

恢复时：
1. scanCampaigns 创建新的 pipe
2. NextSubscribers() 调用 SQL 查询
3. SQL: WHERE s.id > 2500 AND s.id <= 10000
4. 返回订阅者 2501-3500（假设 batch_size=1000）
5. 继续发送，sent 从 2500 开始递增
6. 直到 s.id > 10000，查询返回空
7. 活动完成，sent = 10000
```

**关键代码更新游标** (`internal/manager/pipe.go:219-239` cleanup):

```go
func (p *pipe) cleanup() {
    // ...
    
    // 更新发送计数和最后 ID 到数据库
    // 这确保即使活动被暂停，进度也会被保存
    if err := p.m.store.UpdateCampaignCounts(
        p.camp.ID, 
        0, 
        int(p.sent.Load()), 
        int(p.lastID.Load())
    ); err != nil {
        p.m.log.Printf("error updating campaign counts (%s): %v", p.camp.Name, err)
    }
    
    // ...
}
```

**注意**: `lastID` 是在 Worker 成功发送后更新的原子变量 (`internal/manager/manager.go:537-540`):

```go
} else {
    // 只有发送成功才更新 lastID
    id := uint64(msg.Subscriber.ID)
    if id > msg.pipe.lastID.Load() {
        msg.pipe.lastID.Store(uint64(msg.Subscriber.ID))
    }
    msg.pipe.rate.Incr(1)
    msg.pipe.sent.Add(1)
}
```

#### 4.10.4 恢复场景总结

| 场景 | 数据库状态 | 恢复方式 | 进度保存 |
|------|-----------|----------|----------|
| 暂停后恢复 | `status='paused'` | 设为 `running` | `last_subscriber_id` 保持 |
| 暂停后重新调度 | `status='paused'` | 设为 `scheduled`（需设置 `send_at`） | `last_subscriber_id` 保持 |
| 调度后改草稿 | `status='scheduled'` | 设为 `draft` | `last_subscriber_id` 重置为 0 |
| 取消后 | `status='cancelled'` | 不可恢复 | 进度保留但无法继续 |

**注意**: `cancelled` 状态是**终态**，无法恢复。如果需要恢复，只能创建新的活动。

---

## 五、高级边界分析

### 5.1 恢复后发送进度如何衔接（断点续传深度解析）

#### 5.1.1 两个关键进度字段

在断点续传机制中，有两个字段协同工作来保存和恢复发送进度：

| 字段名 | 数据类型 | 更新时机 | 作用 |
|--------|----------|----------|------|
| `sent` | INT | 每批清理时累加 | 统计已发送总数（用于显示进度） |
| `last_subscriber_id` | INT | 两种方式更新 | 游标位置（用于继续拉取） |

**SQL 更新逻辑** (`queries/campaigns.sql:435-441`):

```sql
-- name: update-campaign-counts
UPDATE campaigns SET
    -- to_send: 只有传入非0值才更新
    to_send=(CASE WHEN $2 != 0 THEN $2 ELSE to_send END),
    -- sent: 累加更新！关键！
    sent=sent+$3,
    -- last_subscriber_id: 只有传入>0时才更新
    last_subscriber_id=(CASE WHEN $4 > 0 THEN $4 ELSE last_subscriber_id END),
    updated_at=NOW()
WHERE id=$1;
```

**关键发现**: `sent` 字段是**累加更新**的（`sent = sent + $3`），这意味着即使活动被暂停多次，每次恢复时 sent 计数都会从之前的基础上继续累加。

#### 5.1.2 两种更新游标的方式

**方式1：每批拉取后自动更新** (`queries/campaigns.sql:318-372`)

在 `next-campaign-subscribers` 查询中，每次成功拉取一批订阅者后，会自动更新 `last_subscriber_id`：

```sql
WITH campLists AS (
    -- ... 关联列表 ...
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
            -- 游标条件：大于上次的 last_subscriber_id
            AND s.id > $3   -- last_subscriber_id
            -- 上限条件：不超过 max_subscriber_id
            AND s.id <= $4  -- max_subscriber_id
            AND s.status != 'blocklisted'
            -- ... 订阅状态检查 ...
        ORDER BY s.id LIMIT $6  -- batch_size
    ) subIDs JOIN subscribers s ON (s.id = subIDs.id) ORDER BY s.id
),
u AS (
    -- 关键：每批拉取后自动更新游标
    UPDATE campaigns
    SET last_subscriber_id = (SELECT MAX(id) FROM subs), updated_at = NOW()
    WHERE (SELECT COUNT(id) FROM subs) > 0 AND id=$1
)
SELECT * FROM subs;
```

**方式2：活动停止/完成时更新** (`internal/manager/pipe.go:186-239`)

在 `cleanup()` 函数中，会调用 `UpdateCampaignCounts` 来保存最终进度：

```go
func (p *pipe) cleanup() {
    // ... 从 pipes Map 移除 ...

    // 关键：更新发送计数和最后 ID 到数据库
    // 这确保即使活动被暂停，进度也会被保存
    if err := p.m.store.UpdateCampaignCounts(
        p.camp.ID, 
        0,                  // to_send: 0 表示不更新
        int(p.sent.Load()), // sent: 本次 pipe 发送的数量（累加）
        int(p.lastID.Load()) // last_subscriber_id: 最后处理的 ID
    ); err != nil {
        p.m.log.Printf("error updating campaign counts (%s): %v", p.camp.Name, err)
    }

    // ... 后续处理 ...
}
```

#### 5.1.3 内存中的进度跟踪

每个 `pipe` 都有两个原子变量来跟踪内存中的进度：

```go
type pipe struct {
    // ... 其他字段 ...
    sent       atomic.Int64    // 本次 pipe 发送成功的计数
    lastID     atomic.Uint64   // 最后一个发送成功的订阅者 ID
    // ...
}
```

**lastID 的更新时机** (`internal/manager/manager.go:537-543`):

```go
} else {
    // 关键：只有发送成功才更新 lastID
    id := uint64(msg.Subscriber.ID)
    if id > msg.pipe.lastID.Load() {
        msg.pipe.lastID.Store(uint64(msg.Subscriber.ID))
    }
    msg.pipe.rate.Incr(1)
    msg.pipe.sent.Add(1)  // sent 也只在成功时增加
}
```

**重要**: `lastID` 和 `sent` 都只在**发送成功**时更新。这意味着：
- 如果发送失败，`lastID` 不会更新，`sent` 也不会增加
- 恢复时会从上次成功发送的 ID 开始继续

#### 5.1.4 恢复时的完整数据流

让我们通过一个详细的时间线来理解断点续传的完整流程：

```
场景：活动有 10000 个订阅者，发送 2500 个后被暂停，然后恢复

================================================================================
阶段1：首次运行
================================================================================

T0: scanCampaigns 首次扫描
    - 检测到 status='running' 且不在当前处理列表
    - 创建新的 pipe
    - 查询 next-campaigns SQL 计算：
        * to_send = 10000
        * max_subscriber_id = 10000
    - 更新数据库：to_send=10000, max_subscriber_id=10000
    - 此时数据库状态：
        sent = 0
        last_subscriber_id = 0
        max_subscriber_id = 10000

T1: NextSubscribers() 第1次调用
    - SQL: s.id > 0 AND s.id <= 10000
    - 返回订阅者 1-1000（假设 batch_size=1000）
    - SQL 自动更新 last_subscriber_id = 1000
    - 此时数据库状态：
        last_subscriber_id = 1000

T2: Worker 处理消息 1-1000
    - 每个消息发送成功后：
        pipe.sent++
        pipe.lastID 更新为该订阅者 ID（如果更大）
    - 全部发送成功后：
        pipe.sent = 1000
        pipe.lastID = 1000

T3: NextSubscribers() 第2次调用
    - SQL: s.id > 1000 AND s.id <= 10000
    - 返回订阅者 1001-2000
    - SQL 自动更新 last_subscriber_id = 2000

T4: Worker 处理消息 1001-2000
    - 全部发送成功后：
        pipe.sent = 2000
        pipe.lastID = 2000

T5: NextSubscribers() 第3次调用
    - SQL: s.id > 2000 AND s.id <= 10000
    - 返回订阅者 2001-3000
    - SQL 自动更新 last_subscriber_id = 3000

T6: Worker 开始处理消息 2001-3000
    - 处理到 2500 个时，用户点击暂停

================================================================================
阶段2：用户暂停操作
================================================================================

T7: 前端调用 PUT /api/campaigns/:id/status {status: 'paused'}
    - 后端更新数据库 status = 'paused'
    - 调用 a.manager.StopCampaign(id)
    - pipe.stopped.Store(true)

T8: Worker 继续处理队列中的消息
    - 消息 2501: 检查 stopped=true → 跳过发送，wg.Done()
    - 消息 2502: 检查 stopped=true → 跳过发送，wg.Done()
    - ...
    - 消息 3000: 检查 stopped=true → 跳过发送，wg.Done()

T9: wg 计数器归零，触发 cleanup()
    - 调用 UpdateCampaignCounts(
        campID, 
        0,                  // to_send 不更新
        2500,               // sent = 2500（累加）
        2500                // last_subscriber_id = 2500
      )
    - 数据库更新：
        sent = 0 + 2500 = 2500
        last_subscriber_id = 2500 （覆盖之前 SQL 设置的 3000）
    - 关键：cleanup 中的 lastID 以实际发送成功的 2500 为准！
    - 从 pipes Map 中移除 pipe

T10: 暂停后的数据库状态
    status = 'paused'
    sent = 2500
    last_subscriber_id = 2500
    max_subscriber_id = 10000

================================================================================
阶段3：用户恢复操作
================================================================================

T11: 用户点击"恢复"
    - 前端调用 PUT /api/campaigns/:id/status {status: 'running'}
    - 后端验证：paused 可以转为 running
    - 更新数据库 status = 'running'
    - 注意：不调用 StopCampaign，因为没有停止

T12: 等待 scanCampaigns 下次扫描（最多 5 秒）

T13: scanCampaigns 扫描
    - 检测到 status='running' 且不在当前处理列表
    - 创建新的 pipe（新的实例！）
    - 新 pipe 的初始状态：
        pipe.sent = 0
        pipe.lastID = 0
    - 调用 next-campaigns SQL
    - 此时数据库的：
        last_subscriber_id = 2500
        max_subscriber_id = 10000

T14: NextSubscribers() 第1次调用（新 pipe）
    - SQL: s.id > 2500 AND s.id <= 10000  ← 关键：从 2501 开始！
    - 返回订阅者 2501-3500
    - SQL 自动更新 last_subscriber_id = 3500

T15: Worker 处理消息 2501-3500
    - 每个消息发送成功后：
        pipe.sent++ （新的计数，从 0 开始）
        pipe.lastID 更新
    - 全部发送成功后：
        pipe.sent = 1000
        pipe.lastID = 3500

T16: 继续这个循环...

================================================================================
阶段4：活动完成
================================================================================

T17: 所有订阅者处理完毕（10000 个）
    - NextSubscribers() 返回空
    - 触发 cleanup()
    - 调用 UpdateCampaignCounts(
        campID, 
        0, 
        7500,  // 本次 pipe 发送 7500 个
        10000
      )
    - 数据库更新：
        sent = 2500 + 7500 = 10000  ← 累加！
        last_subscriber_id = 10000
    - 更新 status = 'finished'

================================================================================
最终状态
================================================================================
status = 'finished'
sent = 10000  ← 正确的总数
last_subscriber_id = 10000
max_subscriber_id = 10000
```

#### 5.1.5 关键设计要点总结

| 设计决策 | 实现方式 | 优势 |
|----------|----------|------|
| **游标而非分页** | `s.id > last_subscriber_id` | 支持千万级表，避免 OFFSET 性能问题 |
| **双层游标更新** | SQL 每批更新 + cleanup 最终更新 | 即使中途崩溃，SQL 更新的游标也能作为起点 |
| **实际发送优先** | cleanup 的 lastID 覆盖 SQL 的 last_subscriber_id | 确保从实际发送成功的位置恢复 |
| **累加计数** | `sent = sent + $3` | 多次暂停/恢复后，sent 统计正确 |
| **内存独立** | 每次恢复创建新的 pipe | 无历史状态干扰，干净恢复 |

### 5.2 可恢复与不可恢复状态的边界

#### 5.2.1 前端状态判断（计算属性）

**文件位置**: `frontend/src/views/Campaign.vue:700-720`

```javascript
computed: {
    // 哪些状态可以编辑活动属性？
    canEdit() {
        return this.isNew
            || this.data.status === 'draft' 
            || this.data.status === 'scheduled' 
            || this.data.status === 'paused';
    },

    // 哪些状态可以立即发送？
    canStart() {
        return (this.data.status === 'draft' || this.data.status === 'paused') 
            && !this.form.sendLater;
    },

    // 哪些状态可以调度？
    canSchedule() {
        return (this.data.status === 'draft' || this.data.status === 'paused') 
            && (this.form.sendLater && this.form.sendAtDate);
    },

    // 哪些状态可以取消调度？
    canUnSchedule() {
        return this.data.status === 'scheduled';
    },

    // 哪些状态可以归档？
    canArchive() {
        return this.data.status !== 'cancelled' && this.data.type !== 'optin';
    },
}
```

#### 5.2.2 后端状态流转验证

**文件位置**: `internal/core/campaigns.go:250-303`

```go
func (c *Core) UpdateCampaignStatus(id int, status string) (models.Campaign, error) {
    cm, err := c.GetCampaign(id, "", "")
    if err != nil {
        return models.Campaign{}, err
    }
    
    errMsg := ""
    switch status {
    case models.CampaignStatusDraft:
        // 只能从 scheduled 转回 draft
        if cm.Status != models.CampaignStatusScheduled {
            errMsg = c.i18n.T("campaigns.onlyScheduledAsDraft")
        }
    case models.CampaignStatusScheduled:
        // 只能从 draft 或 paused 转为 scheduled
        if cm.Status != models.CampaignStatusDraft && cm.Status != models.CampaignStatusPaused {
            errMsg = c.i18n.T("campaigns.onlyDraftAsScheduled")
        }
        // 必须有 send_at
        if !cm.SendAt.Valid {
            errMsg = c.i18n.T("campaigns.needsSendAt")
        }
    case models.CampaignStatusRunning:
        // 只能从 draft 或 paused 转为 running
        if cm.Status != models.CampaignStatusPaused && cm.Status != models.CampaignStatusDraft {
            errMsg = c.i18n.T("campaigns.onlyPausedDraft")
        }
    case models.CampaignStatusPaused:
        // 只能从 running 转为 paused
        if cm.Status != models.CampaignStatusRunning {
            errMsg = c.i18n.T("campaigns.onlyActivePause")
        }
    case models.CampaignStatusCancelled:
        // 只能从 running 或 paused 转为 cancelled
        if cm.Status != models.CampaignStatusRunning && cm.Status != models.CampaignStatusPaused {
            errMsg = c.i18n.T("campaigns.onlyActiveCancel")
        }
    }
    
    if len(errMsg) > 0 {
        return models.Campaign{}, echo.NewHTTPError(http.StatusBadRequest, errMsg)
    }
    
    // 执行状态更新
    res, err := c.q.UpdateCampaignStatus.Exec(cm.ID, status)
    // ...
}
```

#### 5.2.3 完整状态流转图（带边界）

```
                    ┌──────────────────────────────────────────────────────────────┐
                    │                    状态流转边界图                              │
                    └──────────────────────────────────────────────────────────────┘
                    
                    ┌──────────────────────────────────────────────────────────────┐
                    │  可编辑状态（canEdit=true）                                   │
                    │  ┌─────────┐     ┌───────────┐     ┌───────────┐            │
                    │  │  draft  │────►│ scheduled │     │  paused   │            │
                    │  │ (草稿)  │     │ (已调度)  │◄────│ (已暂停)  │            │
                    │  └────┬────┘     └─────┬─────┘     └─────┬─────┘            │
                    │       │                │                 │                   │
                    │       │                │                 │                   │
                    └───────┼────────────────┼─────────────────┼───────────────────┘
                            │                │                 │
                            │                │                 │
                            ▼                ▼                 ▼
                    ┌──────────────────────────────────────────────────────────────┐
                    │  运行中状态（canEdit=false）                                 │
                    │  ┌─────────────────────────────────────────────────────────┐ │
                    │  │                      running                             │ │
                    │  │                     (运行中)                            │ │
                    │  └─────────────────────────────────────────────────────────┘ │
                    │                              │                               │
                    │                              │                               │
                    └──────────────────────────────┼───────────────────────────────┘
                                                   │
                                                   │
                           ┌───────────────────────┼───────────────────────┐
                           │                       │                       │
                           ▼                       ▼                       ▼
                    ┌───────────┐         ┌───────────┐         ┌───────────┐
                    │  paused   │         │ cancelled │         │ finished  │
                    │ (可恢复)  │         │ (不可恢复)│         │ (不可恢复)│
                    └───────────┘         └───────────┘         └───────────┘
                           │                       │                       │
                           │                       │                       │
                           │                   终态（无法转出）              │
                           └───────────────────────────────────────────────┘

状态流转规则：
──────────────────────────────────────────────────────────────────────────────
可编辑状态（canEdit=true）：
  - draft, scheduled, paused
  - 可以修改活动属性、列表、模板等

运行中状态（canEdit=false）：
  - running
  - 禁止修改任何属性
  - 只能转为 paused 或 cancelled

终态（不可恢复）：
  - finished: 活动正常完成
  - cancelled: 活动被取消
  - 无法转出到任何其他状态

恢复路径：
  - paused → running (立即恢复)
  - paused → scheduled (重新调度)
  - scheduled → draft (转为草稿)
  - scheduled → running (时间到达后自动)
```

#### 5.2.4 为什么 cancelled 是终态？

**代码层面的原因**：

1. **后端状态流转验证不允许转出** (`internal/core/campaigns.go`):

```go
// 没有任何 case 允许从 cancelled 转出
// UpdateCampaignStatus 只验证"目标状态"的前置条件
// 但如果当前状态是 cancelled，任何目标状态的验证都会失败

// 例如：要转为 running，当前状态必须是 draft 或 paused
case models.CampaignStatusRunning:
    if cm.Status != models.CampaignStatusPaused && cm.Status != models.CampaignStatusDraft {
        errMsg = c.i18n.T("campaigns.onlyPausedDraft")
    }
// 如果 cm.Status 是 cancelled，这里就会报错
```

2. **前端计算属性不允许编辑** (`frontend/src/views/Campaign.vue`):

```javascript
canEdit() {
    return this.isNew
        || this.data.status === 'draft' 
        || this.data.status === 'scheduled' 
        || this.data.status === 'paused';
    // 没有 'cancelled'！
}
```

3. **scanCampaigns 不会选取 cancelled** (`queries/campaigns.sql:186`):

```sql
WHERE (status='running' OR (status='scheduled' AND NOW() >= campaigns.send_at))
```

**业务层面的原因**：

- `cancelled` 表示用户明确放弃这个活动
- `paused` 表示暂时停止，可能还会继续
- `finished` 表示正常完成
- 如果 `cancelled` 可以恢复，语义上就和 `paused` 没有区别了

### 5.3 错误阈值触发暂停与人工暂停的差异

#### 5.3.1 两种暂停的触发方式对比

| 对比项 | 错误阈值触发暂停 | 人工暂停 |
|--------|------------------|----------|
| **触发者** | 系统自动 (`OnError()`) | 用户手动操作 |
| **触发条件** | 连续错误达到 `max_send_errors` | 用户点击"暂停"按钮 |
| **代码入口** | `internal/manager/pipe.go:136-151` | `cmd/campaigns.go:351-380` |
| **数据库更新时机** | `cleanup()` 中更新 | Handler 中立即更新 |
| **通知发送** | 自动发送"Too many errors"通知 | 不发送通知 |

#### 5.3.2 代码实现差异

**错误阈值触发暂停** (`internal/manager/pipe.go:136-151`):

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
    
    // 关键：Stop(true) - withErrors = true
    p.Stop(true)
    p.m.log.Printf("error count exceeded %d. pausing campaign %s", p.m.cfg.MaxSendErrors, p.camp.Name)
}
```

**人工暂停** (`cmd/campaigns.go:351-380`):

```go
func (a *App) UpdateCampaignStatus(c echo.Context) error {
    // ... 权限检查和参数绑定 ...
    
    // 更新数据库中的状态
    out, err := a.core.UpdateCampaignStatus(id, req.Status)
    if err != nil {
        return err
    }
    
    // 如果是暂停或取消，发送信号给 Manager
    if req.Status == models.CampaignStatusPaused || req.Status == models.CampaignStatusCancelled {
        // 关键：StopCampaign 最终调用 p.Stop(false) - withErrors = false
        a.manager.StopCampaign(id)
    }
    
    return c.JSON(http.StatusOK, okResp{out})
}
```

**Manager 层的 StopCampaign** (`internal/manager/manager.go:405-412`):

```go
func (m *Manager) StopCampaign(id int) {
    m.pipesMut.RLock()
    if p, ok := m.pipes[id]; ok {
        // 关键：Stop(false) - withErrors = false
        p.Stop(false)
    }
    m.pipesMut.RUnlock()
}
```

**Pipe 层的 Stop** (`internal/manager/pipe.go:153-167`):

```go
func (p *pipe) Stop(withErrors bool) {
    // Already stopped.
    if p.stopped.Load() {
        return
    }

    // 关键差异：withErrors 标志
    if withErrors {
        p.withErrors.Store(true)
    }

    p.stopped.Store(true)
}
```

#### 5.3.3 cleanup() 中的处理差异

**文件位置**: `internal/manager/pipe.go:186-239`

```go
func (p *pipe) cleanup() {
    // ... 从 pipes Map 移除 ...
    
    // 更新发送计数到数据库
    if err := p.m.store.UpdateCampaignCounts(
        p.camp.ID, 
        0, 
        int(p.sent.Load()), 
        int(p.lastID.Load())
    ); err != nil {
        p.m.log.Printf("error updating campaign counts (%s): %v", p.camp.Name, err)
    }

    // ============================================================
    // 分支1：因错误暂停（withErrors = true）
    // ============================================================
    if p.withErrors.Load() {
        // 关键：需要在这里更新数据库状态！
        // 因为 OnError() 只是设置了标志，没有更新数据库
        if err := p.m.store.UpdateCampaignStatus(
            p.camp.ID, 
            models.CampaignStatusPaused
        ); err != nil {
            p.m.log.Printf(
                "error updating campaign (%s) status to %s: %v", 
                p.camp.Name, 
                models.CampaignStatusPaused, 
                err
            )
        } else {
            p.m.log.Printf("set campaign (%s) to %s", p.camp.Name, models.CampaignStatusPaused)
        }
        
        // 关键：发送通知给管理员
        _ = p.m.sendNotif(
            p.camp, 
            models.CampaignStatusPaused, 
            "Too many errors"
        )
        return
    }

    // ============================================================
    // 分支2：手动停止（paused 或 cancelled）
    // ============================================================
    if p.stopped.Load() {
        // 关键：不需要更新数据库状态！
        // 因为 UpdateCampaignStatus Handler 已经更新过了
        p.m.log.Printf("stop processing campaign (%s)", p.camp.Name)
        return
    }

    // ============================================================
    // 分支3：自然完成
    // ============================================================
    // ... 完成处理 ...
}
```

#### 5.3.4 完整流程图对比

**场景A：错误阈值触发暂停**

```
时间线：
──────────────────────────────────────────────────────────────────────────────

T0: 活动正常运行
    - status = 'running'
    - 正在发送邮件

T1: SMTP 服务器故障，连续发送失败
    - Worker 检测到错误
    - 调用 msg.pipe.OnError()
    - pipe.errors 计数增加

T2: 连续错误达到 max_send_errors（假设 100）
    - OnError() 中：count >= MaxSendErrors
    - 调用 p.Stop(true)  ← withErrors = true
    - 设置 pipe.stopped = true
    - 设置 pipe.withErrors = true
    - 日志："error count exceeded 100. pausing campaign XXX"

T3: Worker 继续处理队列中的消息
    - 每个消息检查 pipe.stopped = true
    - 跳过发送，直接 wg.Done()

T4: wg 归零，触发 cleanup()
    - 分支1：p.withErrors.Load() = true
    - 调用 UpdateCampaignStatus(status='paused')  ← 数据库状态更新
    - 日志："set campaign (XXX) to paused"
    - 调用 sendNotif(camp, 'paused', 'Too many errors')  ← 发送通知
    - 从 pipes Map 移除

T5: 最终状态
    - status = 'paused'
    - 用户看到"暂停"状态
    - 用户收到"Too many errors"通知
    - 用户可以选择恢复或取消
```

**场景B：人工暂停**

```
时间线：
──────────────────────────────────────────────────────────────────────────────

T0: 活动正常运行
    - status = 'running'
    - 正在发送邮件

T1: 用户点击"暂停"按钮
    - 前端调用 PUT /api/campaigns/:id/status {status: 'paused'}

T2: 后端 Handler 处理 (UpdateCampaignStatus)
    - 验证：当前状态是 running，可以转为 paused
    - 调用 core.UpdateCampaignStatus()
    - 执行 SQL：UPDATE campaigns SET status='paused' WHERE id=?  ← 立即更新！
    - 调用 a.manager.StopCampaign(id)
    - 返回成功响应

T3: Manager 层处理 (StopCampaign)
    - 从 pipes Map 找到 pipe
    - 调用 p.Stop(false)  ← withErrors = false
    - 设置 pipe.stopped = true
    - 不设置 pipe.withErrors

T4: Worker 继续处理队列中的消息
    - 每个消息检查 pipe.stopped = true
    - 跳过发送，直接 wg.Done()

T5: wg 归零，触发 cleanup()
    - 分支1：p.withErrors.Load() = false  ← 不进入
    - 分支2：p.stopped.Load() = true  ← 进入
    - 日志："stop processing campaign (XXX)"
    - 关键：不更新数据库状态！（Handler 已经更新过了）
    - 关键：不发送通知！
    - 从 pipes Map 移除

T6: 最终状态
    - status = 'paused'
    - 用户看到"暂停"状态
    - 用户没有收到通知
    - 用户可以选择恢复或取消
```

#### 5.3.5 差异总结表

| 对比维度 | 错误阈值触发暂停 | 人工暂停 |
|----------|------------------|----------|
| **触发条件** | 连续错误达到 `max_send_errors` | 用户手动点击 |
| **Stop() 参数** | `Stop(true)` - withErrors=true | `Stop(false)` - withErrors=false |
| **数据库更新时机** | `cleanup()` 中更新 | Handler 中立即更新 |
| **数据库更新位置** | `pipe.go:cleanup()` | `campaigns.go:UpdateCampaignStatus` |
| **通知发送** | 发送"Too many errors"通知 | 不发送通知 |
| **恢复方式** | 与人工暂停相同 | 与错误暂停相同 |
| **用户感知** | 收到错误通知，知道是系统问题 | 自己操作，不需要通知 |

**关键差异点**：

1. **错误暂停**：系统检测到问题，需要通知管理员
2. **人工暂停**：用户主动操作，不需要通知
3. **两种暂停的恢复方式完全相同** - 都是从 `paused` 转为 `running` 或 `scheduled`

### 4.11 活动清理与完成

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
