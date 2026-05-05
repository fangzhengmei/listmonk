# Listmonk 邮件打开与点击追踪机制分析报告

## 目录

1. [概述](#概述)
2. [跟踪像素 (Tracking Pixel) 实现](#跟踪像素-tracking-pixel-实现)
3. [链接点击追踪 (Link Click Tracking) 实现](#链接点击追踪-link-click-tracking-实现)
4. [数据模型与存储](#数据模型与存储)
5. [跨域回写机制与完整闭环](#跨域回写机制与完整闭环)
6. [隐私与追踪控制](#隐私与追踪控制)
7. [统计数据聚合与管理端读取](#统计数据聚合与管理端读取)
8. [实现架构图](#实现架构图)

---

## 概述

Listmonk 实现了两种核心的邮件追踪机制：

- **打开追踪 (Open Tracking)**: 通过透明像素图片实现
- **点击追踪 (Click Tracking)**: 通过中间重定向链接实现

两种机制都依赖于 HTTP 请求到 listmonk 服务器，从而触发统计数据的记录。本文档详细分析从追踪请求入口、事件落库、统计聚合到管理端展示的完整闭环。

---

## 跟踪像素 (Tracking Pixel) 实现

### 1. 工作原理

跟踪像素是一个 **14x3 像素（宽x高）**的透明 PNG 图片，嵌入在邮件 HTML 正文中。当邮件客户端加载图片时，会向 listmonk 服务器发送 HTTP 请求，从而记录这次"打开"事件。

> **注意**: 尺寸描述采用标准的 `宽 x 高` 格式。

### 2. 关键实现代码

#### 2.1 像素图片生成

**位置**: `cmd/public.go:101-102`

```go
var (
    pixelPNG = drawTransparentImage(3, 14)
)
```

**位置**: `cmd/public.go:693-701`

```go
func drawTransparentImage(h, w int) []byte {
    var (
        img = image.NewRGBA(image.Rect(0, 0, w, h))
        out = &bytes.Buffer{}
    )
    _ = png.Encode(out, img)
    return out.Bytes()
}
```

**尺寸计算分析**：
- 函数签名: `drawTransparentImage(h, w int)`
- 调用时: `drawTransparentImage(3, 14)` → `h=3`, `w=14`
- Go 标准库 `image.Rect(x0, y0, x1, y1)` 定义矩形
- `image.Rect(0, 0, w, h)` = `image.Rect(0, 0, 14, 3)`
- 宽度 = 14 - 0 = **14 像素**
- 高度 = 3 - 0 = **3 像素**
- 最终尺寸: **14 x 3 像素（宽 x 高）**

#### 2.2 模板函数 - TrackView

**位置**: `internal/manager/manager.go:361-373`

```go
"TrackView": func(msg *CampaignMessage) template.HTML {
    if m.cfg.DisableTracking {
        return template.HTML("")
    }

    subUUID := msg.Subscriber.UUID
    if !m.cfg.IndividualTracking {
        subUUID = dummyUUID
    }

    return template.HTML(fmt.Sprintf(`<img src="%s" alt="" />`,
        fmt.Sprintf(m.cfg.ViewTrackURL, msg.Campaign.UUID, subUUID)))
}
```

**URL 格式**: `{root}/campaign/{campUUID}/{subUUID}/px.png`

#### 2.3 HTTP 处理 - RegisterCampaignView

**位置**: `cmd/public.go:569-592`

```go
func (a *App) RegisterCampaignView(c echo.Context) error {
    // 如果全局追踪禁用，直接返回像素不记录
    if a.cfg.Privacy.DisableTracking {
        c.Response().Header().Set("Cache-Control", "no-cache")
        return c.Blob(http.StatusOK, "image/png", pixelPNG)
    }

    // 如果个体追踪禁用，不记录订阅者ID
    subUUID := c.Param("subUUID")
    if !a.cfg.Privacy.IndividualTracking {
        subUUID = ""
    }

    // 排除模板预览时的 dummy 请求
    campUUID := c.Param("campUUID")
    if campUUID != dummyUUID && subUUID != dummyUUID {
        if err := a.core.RegisterCampaignView(campUUID, subUUID); err != nil {
            a.log.Printf("error registering campaign view: %s", err)
        }
    }

    // 始终返回像素图片（即使记录失败）
    c.Response().Header().Set("Cache-Control", "no-cache")
    return c.Blob(http.StatusOK, "image/png", pixelPNG)
}
```

#### 2.4 核心层 - 记录视图

**位置**: `internal/core/campaigns.go:424-436`

```go
func (c *Core) RegisterCampaignView(campUUID, subUUID string) error {
    if _, err := c.q.RegisterCampaignView.Exec(campUUID, subUUID); err != nil {
        // 如果 campaign_id 不存在（外键约束），静默忽略
        if pqErr, ok := err.(*pq.Error); ok && pqErr.Column == "campaign_id" {
            return nil
        }
        // ... 错误处理
    }
    return nil
}
```

#### 2.5 SQL 查询

**位置**: `queries/campaigns.sql:481-487`

```sql
WITH view AS (
    SELECT campaigns.id as campaign_id, subscribers.id AS subscriber_id 
    FROM campaigns
    LEFT JOIN subscribers ON (CASE WHEN $2::TEXT != '' THEN subscribers.uuid = $2::UUID ELSE FALSE END)
    WHERE campaigns.uuid = $1
)
INSERT INTO campaign_views (campaign_id, subscriber_id)
    VALUES((SELECT campaign_id FROM view), (SELECT subscriber_id FROM view));
```

### 3. 路由注册与中间件

**位置**: `cmd/handlers.go:279`

```go
g.GET("/campaign/:campUUID/:subUUID/px.png", 
    noIndex(a.hasUUID(a.RegisterCampaignView, "campUUID", "subUUID")))
```

**中间件链路**（从外到内执行）：

| 中间件 | 功能 | 代码位置 |
|--------|------|----------|
| `noIndex` | 添加 `X-Robots-Tag: noindex` 响应头，防止搜索引擎索引 | `cmd/handlers.go:405-411` |
| `hasUUID` | 验证 URL 参数中的 UUID 格式（正则校验） | `cmd/handlers.go:358-369` |

> **重要事实修正**：追踪端点 `/campaign/:campUUID/:subUUID/px.png` **没有 `hasSub` 中间件**。`hasSub` 只用于订阅相关页面（如 `/subscription/:campUUID/:subUUID`），用于验证订阅者是否存在。

`hasSub` 中间件定义（但追踪端点不使用）：

**位置**: `cmd/handlers.go:384-403`

```go
// hasSub middleware checks if a subscriber exists given the UUID
// param in a request.
func (a *App) hasSub(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        subUUID := c.Param("subUUID")

        if _, err := a.core.GetSubscriber(0, subUUID, ""); err != nil {
            // ... 订阅者不存在时返回 404
        }
        return next(c)
    }
}
```

---

## 链接点击追踪 (Link Click Tracking) 实现

### 1. 工作原理

链接点击追踪通过**中间重定向**机制实现：

1. 发送邮件时，原始链接被替换为 listmonk 的重定向 URL
2. 当用户点击链接时，浏览器首先请求 listmonk 服务器
3. listmonk 记录点击事件后，返回 307 临时重定向到原始 URL

### 2. 关键实现代码

#### 2.1 URL 配置初始化

**位置**: `cmd/init.go:456-484`

```go
func initUrlConfig(ko *koanf.Koanf) *UrlConfig {
    root := strings.TrimSuffix(ko.String("app.root_url"), "/")

    return &UrlConfig{
        // ...
        // 链接追踪 URL: {root}/link/{linkUUID}/{campUUID}/{subUUID}
        LinkTrackURL: fmt.Sprintf("%s/link/%%s/%%s/%%s", root),
        
        // 视图追踪 URL: {root}/campaign/{campUUID}/{subUUID}/px.png
        ViewTrackURL: fmt.Sprintf("%s/campaign/%%s/%%s/px.png", root),
    }
}
```

#### 2.2 模板函数 - TrackLink

**位置**: `internal/manager/manager.go:349-360`

```go
"TrackLink": func(url string, msg *CampaignMessage) string {
    if m.cfg.DisableTracking {
        return url
    }

    subUUID := msg.Subscriber.UUID
    if !m.cfg.IndividualTracking {
        subUUID = dummyUUID
    }

    return m.trackLink(url, msg.Campaign.UUID, subUUID)
}
```

#### 2.3 链接注册与缓存

**位置**: `internal/manager/manager.go:583-613`

```go
func (m *Manager) trackLink(url, campUUID, subUUID string) string {
    if m.cfg.DisableTracking {
        return url
    }

    // 统一处理 HTML 实体转义
    url = strings.ReplaceAll(url, "&amp;", "&")

    // 首先检查缓存
    m.linksMut.RLock()
    if uu, ok := m.links[url]; ok {
        m.linksMut.RUnlock()
        return fmt.Sprintf(m.cfg.LinkTrackURL, uu, campUUID, subUUID)
    }
    m.linksMut.RUnlock()

    // 缓存未命中，注册新链接到数据库
    uu, err := m.store.CreateLink(url)
    if err != nil {
        m.log.Printf("error registering tracking for link '%s': %v", url, err)
        // 失败时降级：使用原始 URL
        return url
    }

    // 存入缓存
    m.linksMut.Lock()
    m.links[url] = uu
    m.linksMut.Unlock()

    return fmt.Sprintf(m.cfg.LinkTrackURL, uu, campUUID, subUUID)
}
```

#### 2.4 创建链接 SQL

**位置**: `queries/links.sql:2-3`

```sql
INSERT INTO links (uuid, url) VALUES($1, $2) 
ON CONFLICT (url) DO UPDATE SET url=EXCLUDED.url 
RETURNING uuid;
```

#### 2.5 HTTP 重定向处理 - LinkRedirect

**位置**: `cmd/public.go:534-563`

```go
func (a *App) LinkRedirect(c echo.Context) error {
    var (
        linkUUID = c.Param("linkUUID")
        campUUID = c.Param("campUUID")
    )

    // 如果全局追踪禁用，直接解析 URL 不记录点击
    if a.cfg.Privacy.DisableTracking {
        url, err := a.core.GetLinkURL(linkUUID)
        if err != nil {
            e := err.(*echo.HTTPError)
            return c.Render(e.Code, tplMessage, 
                makeMsgTpl(a.i18n.T("public.errorTitle"), "", e.Error()))
        }
        return c.Redirect(http.StatusTemporaryRedirect, url)
    }

    // 如果个体追踪禁用，不记录订阅者ID
    subUUID := c.Param("subUUID")
    if !a.cfg.Privacy.IndividualTracking {
        subUUID = ""
    }

    // 记录点击并获取原始 URL
    url, err := a.core.RegisterCampaignLinkClick(linkUUID, campUUID, subUUID)
    if err != nil {
        e := err.(*echo.HTTPError)
        return c.Render(e.Code, tplMessage, 
            makeMsgTpl(a.i18n.T("public.errorTitle"), "", e.Error()))
    }

    // 307 临时重定向（保持原始请求方法）
    return c.Redirect(http.StatusTemporaryRedirect, url)
}
```

#### 2.6 核心层 - 记录点击

**位置**: `internal/core/campaigns.go:448-461`

```go
func (c *Core) RegisterCampaignLinkClick(linkUUID, campUUID, subUUID string) (string, error) {
    var url string
    if err := c.q.RegisterLinkClick.Get(&url, linkUUID, campUUID, subUUID); err != nil {
        // 如果 link_id 不存在
        if pqErr, ok := err.(*pq.Error); ok && pqErr.Column == "link_id" {
            return "", echo.NewHTTPError(http.StatusBadRequest, 
                c.i18n.Ts("public.invalidLink"))
        }
        // ... 错误处理
    }
    return url, nil
}
```

#### 2.7 记录点击 SQL

**位置**: `queries/links.sql:8-18`

```sql
WITH link AS(
    SELECT id, url FROM links WHERE uuid = $1
)
INSERT INTO link_clicks (campaign_id, subscriber_id, link_id) VALUES(
    (SELECT id FROM campaigns WHERE uuid = $2),
    (SELECT id FROM subscribers WHERE
        (CASE WHEN $3::TEXT != '' THEN subscribers.uuid = $3::UUID ELSE FALSE END)
    ),
    (SELECT id FROM link)
) RETURNING (SELECT url FROM link);
```

### 3. 路由注册与中间件

**位置**: `cmd/handlers.go:277`

```go
g.GET("/link/:linkUUID/:campUUID/:subUUID", 
    noIndex(a.hasUUID(a.LinkRedirect, "linkUUID", "campUUID", "subUUID")))
```

**中间件链路**（与像素追踪相同）：

| 中间件 | 功能 |
|--------|------|
| `noIndex` | 添加 `X-Robots-Tag: noindex` |
| `hasUUID` | 验证 URL 参数中的 UUID 格式 |

> **同样重要**：链接追踪端点 `/link/:linkUUID/:campUUID/:subUUID` **也没有 `hasSub` 中间件**。这意味着即使订阅者 UUID 无效或对应订阅者已删除，请求仍然会被处理（但 `subscriber_id` 会被设置为 NULL）。

---

## 数据模型与存储

### 1. 数据库表结构

#### 1.1 链接表 (links)

**位置**: `schema.sql:198-204`

```sql
CREATE TABLE links (
    id          SERIAL PRIMARY KEY,
    uuid        uuid NOT NULL UNIQUE,
    url         TEXT NOT NULL UNIQUE,
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | SERIAL | 自增主键 |
| `uuid` | uuid | 公开的链接 UUID（用于 URL） |
| `url` | TEXT | 原始 URL，唯一约束 |
| `created_at` | TIMESTAMP | 创建时间 |

#### 1.2 打开记录表 (campaign_views)

**位置**: `schema.sql:155-167`

```sql
CREATE TABLE campaign_views (
    id               BIGSERIAL PRIMARY KEY,
    campaign_id      INTEGER NOT NULL REFERENCES campaigns(id) 
                     ON DELETE CASCADE ON UPDATE CASCADE,
    subscriber_id    INTEGER NULL REFERENCES subscribers(id) 
                     ON DELETE SET NULL ON UPDATE CASCADE,
    created_at       TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | BIGSERIAL | 自增主键 |
| `campaign_id` | INTEGER | 关联的活动 ID（级联删除） |
| `subscriber_id` | INTEGER | 关联的订阅者 ID（可为 NULL，订阅者删除时置 NULL） |
| `created_at` | TIMESTAMP | 记录创建时间 |

#### 1.3 点击记录表 (link_clicks)

**位置**: `schema.sql:206-219`

```sql
CREATE TABLE link_clicks (
    id               BIGSERIAL PRIMARY KEY,
    campaign_id      INTEGER NULL REFERENCES campaigns(id) 
                     ON DELETE CASCADE ON UPDATE CASCADE,
    link_id          INTEGER NOT NULL REFERENCES links(id) 
                     ON DELETE CASCADE ON UPDATE CASCADE,
    subscriber_id    INTEGER NULL REFERENCES subscribers(id) 
                     ON DELETE SET NULL ON UPDATE CASCADE,
    created_at       TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | BIGSERIAL | 自增主键 |
| `campaign_id` | INTEGER | 关联的活动 ID（可为 NULL） |
| `link_id` | INTEGER | 关联的链接 ID（必填） |
| `subscriber_id` | INTEGER | 关联的订阅者 ID（可为 NULL） |
| `created_at` | TIMESTAMP | 记录创建时间 |

### 2. 索引设计

**位置**: `schema.sql`

```sql
-- campaign_views 索引
CREATE INDEX idx_views_camp_id ON campaign_views(campaign_id);
CREATE INDEX idx_views_subscriber_id ON campaign_views(subscriber_id);
CREATE INDEX idx_views_date ON campaign_views(created_at);

-- link_clicks 索引
CREATE INDEX idx_clicks_camp_id ON link_clicks(campaign_id);
CREATE INDEX idx_clicks_link_id ON link_clicks(link_id);
CREATE INDEX idx_clicks_sub_id ON link_clicks(subscriber_id);
CREATE INDEX idx_clicks_date ON link_clicks(created_at);
```

### 3. 数据完整性设计

```
campaigns (id) ─────┬──< campaign_views (campaign_id) [CASCADE]
                    │
                    └──< link_clicks (campaign_id) [CASCADE]

subscribers (id) ───┬──< campaign_views (subscriber_id) [SET NULL]
                    │
                    └──< link_clicks (subscriber_id) [SET NULL]

links (id) ────────────< link_clicks (link_id) [CASCADE]
```

**关键设计点**：
- 活动删除时，相关的打开/点击记录**级联删除**
- 订阅者删除时，相关记录中的 `subscriber_id` **置 NULL**（保留统计数据但匿名化）
- 链接删除时，相关点击记录**级联删除**

---

## 跨域回写机制与完整闭环

### 1. CORS 配置

**位置**: `cmd/handlers.go:43-49`

```go
// 如果配置了 CORS 域名，启用 CORS 中间件
if len(a.cfg.Security.CorsOrigins) > 0 {
    e.Use(middleware.CORSWithConfig(middleware.CORSConfig{
        AllowOrigins: a.cfg.Security.CorsOrigins,
        AllowHeaders: []string{echo.HeaderOrigin, echo.HeaderContentType, echo.HeaderAccept},
    }))
}
```

**配置项**：

**位置**: `schema.sql:264`

```sql
('security.cors_origins', '[]'),
```

### 2. 追踪请求的跨域处理

#### 2.1 跟踪像素请求

跟踪像素的请求是通过 `<img>` 标签发起的，属于**简单请求**，不受 CORS 限制：

```html
<img src="https://listmonk.example.com/campaign/xxx/yyy/px.png" alt="" />
```

#### 2.2 链接重定向请求

链接点击通过 `GET /link/:uuid` 发起，重定向使用 **307 Temporary Redirect**：

```
GET /link/link-uuid/camp-uuid/sub-uuid HTTP/1.1
Host: listmonk.example.com

HTTP/1.1 307 Temporary Redirect
Location: https://original-url.com/path
```

**注意**：307 重定向会保持原始请求方法，但对于链接点击通常是 GET 请求，不影响。

### 3. 完整回写闭环分析

#### 3.1 闭环流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           完整追踪回写闭环                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐                                                        │
│  │  邮件客户端      │                                                        │
│  │  ┌───────────┐  │                                                        │
│  │  │ <img src= │  │                                                        │
│  │  │  px.png> │  │                                                        │
│  │  │ <a href=  │  │                                                        │
│  │  │ /link/...>│  │                                                        │
│  │  └─────┬─────┘  │                                                        │
│  └────────┼─────────┘                                                        │
│           │                                                                    │
│           │ HTTP GET 请求                                                      │
│           v                                                                    │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                        阶段 1: 请求入口与中间件                      │     │
│  ├────────────────────────────────────────────────────────────────────┤     │
│  │  中间件执行顺序（从外到内）：                                          │     │
│  │                                                                      │     │
│  │  1. noIndex()                                                        │     │
│  │     └──> 添加 X-Robots-Tag: noindex 响应头                         │     │
│  │                                                                      │     │
│  │  2. hasUUID()                                                        │     │
│  │     └──> 正则校验 UUID 格式: ^[0-9a-fA-F]{8}-...$                 │     │
│  │           - 校验失败: 返回 400 "invalid UUID"                       │     │
│  │           - 校验成功: 继续执行                                        │     │
│  │                                                                      │     │
│  │  注意: 追踪端点 NO hasSub()！                                        │     │
│  │        订阅者 ID 验证只在订阅相关页面执行                              │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│           │                                                                    │
│           v                                                                    │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                        阶段 2: 隐私控制分支判断                       │     │
│  ├────────────────────────────────────────────────────────────────────┤     │
│  │                                                                      │     │
│  │  ┌────────────────────────────────────────────────────────────┐   │     │
│  │  │ 分支 A: 全局禁用追踪 (DisableTracking = true)               │   │     │
│  │  ├────────────────────────────────────────────────────────────┤   │     │
│  │  │  像素追踪:                                                    │   │     │
│  │  │    └──> 直接返回 14x3 PNG，不插入 campaign_views           │   │     │
│  │  │                                                                 │   │     │
│  │  │  链接追踪:                                                    │   │     │
│  │  │    └──> 只查询 links 表获取原始 URL                           │   │     │
│  │  │    └──> 307 重定向，不插入 link_clicks                       │   │     │
│  │  └────────────────────────────────────────────────────────────┘   │     │
│  │                           │                                          │     │
│  │                           v (DisableTracking = false)               │     │
│  │  ┌────────────────────────────────────────────────────────────┐   │     │
│  │  │ 分支 B: 匿名追踪 (IndividualTracking = false)               │   │     │
│  │  ├────────────────────────────────────────────────────────────┤   │     │
│  │  │  subUUID 处理:                                               │   │     │
│  │  │    └──> 将 subUUID 设为空字符串 ""                          │   │     │
│  │  │                                                                 │   │     │
│  │  │  SQL 层面:                                                    │   │     │
│  │  │    LEFT JOIN subscribers ON                                  │   │     │
│  │  │      CASE WHEN $2::TEXT != '' THEN ... ELSE FALSE END      │   │     │
│  │  │    └──> subUUID 为空时，subscriber_id = NULL               │   │     │
│  │  │                                                                 │   │     │
│  │  │  记录效果:                                                    │   │     │
│  │  │    └──> campaign_views / link_clicks 表记录事件             │   │     │
│  │  │    └──> subscriber_id = NULL (匿名化)                        │   │     │
│  │  └────────────────────────────────────────────────────────────┘   │     │
│  │                           │                                          │     │
│  │                           v (IndividualTracking = true)            │     │
│  │  ┌────────────────────────────────────────────────────────────┐   │     │
│  │  │ 分支 C: 个体追踪 (IndividualTracking = true)                │   │     │
│  │  ├────────────────────────────────────────────────────────────┤   │     │
│  │  │  subUUID 处理:                                               │   │     │
│  │  │    └──> 保持 URL 中的原始 subUUID                           │   │     │
│  │  │    └──> 但需要排除 dummyUUID (全零)                         │   │     │
│  │  │                                                                 │   │     │
│  │  │  Dummy UUID 排除逻辑:                                         │   │     │
│  │  │    if campUUID != dummyUUID && subUUID != dummyUUID {      │   │     │
│  │  │        // 执行记录操作                                        │   │     │
│  │  │    }                                                         │   │     │
│  │  │    └──> 模板预览使用全零 UUID，不记录统计                    │   │     │
│  │  │                                                                 │   │     │
│  │  │  记录效果:                                                    │   │     │
│  │  │    └──> subscriber_id = 实际订阅者 ID                       │   │     │
│  │  │    └──> 可追踪每个订阅者的具体行为                            │   │     │
│  │  └────────────────────────────────────────────────────────────┘   │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│           │                                                                    │
│           v                                                                    │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                        阶段 3: 事件落库                              │     │
│  ├────────────────────────────────────────────────────────────────────┤     │
│  │                                                                      │     │
│  │  像素追踪:                                                           │     │
│  │  ┌────────────────────────────────────────────────────────────┐   │     │
│  │  │ INSERT INTO campaign_views                                   │   │     │
│  │  │   (campaign_id, subscriber_id)                               │   │     │
│  │  │ VALUES                                                        │   │     │
│  │  │   (SELECT campaigns.id WHERE uuid = $1,                     │   │     │
│  │  │    SELECT subscribers.id WHERE uuid = $2)                   │   │     │
│  │  └────────────────────────────────────────────────────────────┘   │     │
│  │                                                                      │     │
│  │  链接追踪:                                                           │     │
│  │  ┌────────────────────────────────────────────────────────────┐   │     │
│  │  │ INSERT INTO link_clicks                                      │   │     │
│  │  │   (campaign_id, subscriber_id, link_id)                     │   │     │
│  │  │ VALUES                                                        │   │     │
│  │  │   (SELECT campaigns.id WHERE uuid = $2,                     │   │     │
│  │  │    SELECT subscribers.id WHERE uuid = $3,                   │   │     │
│  │  │    SELECT links.id WHERE uuid = $1)                          │   │     │
│  │  │ RETURNING (SELECT url FROM link)                             │   │     │
│  │  └────────────────────────────────────────────────────────────┘   │     │
│  │                                                                      │     │
│  │  外键异常处理:                                                       │     │
│  │    └──> campaign_id 不存在 → 静默忽略 (返回 nil)                   │     │
│  │    └──> link_id 不存在 → 返回错误 "invalid link"                   │     │
│  │    └──> subscriber_id 不存在 → subscriber_id = NULL               │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│           │                                                                    │
│           v                                                                    │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                        阶段 4: 响应返回                              │     │
│  ├────────────────────────────────────────────────────────────────────┤     │
│  │                                                                      │     │
│  │  像素追踪:                                                           │     │
│  │    └──> Content-Type: image/png                                     │     │
│  │    └──> Cache-Control: no-cache                                     │     │
│  │    └──> 14x3 透明 PNG 二进制数据                                   │     │
│  │                                                                      │     │
│  │  链接追踪:                                                           │     │
│  │    └──> Status: 307 Temporary Redirect                             │     │
│  │    └──> Location: {原始 URL}                                        │     │
│  │                                                                      │     │
│  │  失败安全设计 (Fail-Safe):                                          │     │
│  │    └──> 即使数据库插入失败，仍然返回响应                            │     │
│  │    └──> 不影响用户体验                                              │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│           │                                                                    │
│           v                                                                    │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                        阶段 5: 统计聚合与管理端展示                  │     │
│  ├────────────────────────────────────────────────────────────────────┤     │
│  │                                                                      │     │
│  │  启动时动态查询准备 (prepareQueries):                               │     │
│  │    ┌──────────────────────────────────────────────────────────┐   │     │
│  │    │ IndividualTracking = true                                  │   │     │
│  │    │   └──> 使用去重计数查询 (DISTINCT ON subscriber_id)       │   │     │
│  │    │   └──> 每个订阅者只计一次                                   │   │     │
│  │    │                                                             │   │     │
│  │    │ IndividualTracking = false                                 │   │     │
│  │    │   └──> 使用普通计数查询 (COUNT(*))                         │   │     │
│  │    │   └──> 每次访问都计数                                       │   │     │
│  │    └──────────────────────────────────────────────────────────┘   │     │
│  │                                                                      │     │
│  │  管理端 API 端点:                                                    │     │
│  │    └──> GET /api/campaigns/analytics/:type                        │     │
│  │    └──> 需要权限: campaigns:get_analytics                          │     │
│  │                                                                      │     │
│  │  统计类型:                                                           │     │
│  │    ┌──────────────┬────────────────────────────────────────────┐  │     │
│  │    │ type = views │ 按小时/天聚合的打开时间序列                  │  │     │
│  │    ├──────────────┼────────────────────────────────────────────┤  │     │
│  │    │ type = clicks│ 按小时/天聚合的点击时间序列                  │  │     │
│  │    ├──────────────┼────────────────────────────────────────────┤  │     │
│  │    │ type = links │ 各链接的点击次数排行榜 (TOP 50)             │  │     │
│  │    └──────────────┴────────────────────────────────────────────┘  │     │
│  │                                                                      │     │
│  │  时间粒度自动选择:                                                   │     │
│  │    └──> 时间间隔 < 7 天: 按小时聚合                                │     │
│  │    └──> 时间间隔 >= 7 天: 按天聚合                                 │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.2 隐私分支的详细对比

| 配置组合 | DisableTracking | IndividualTracking | 模板渲染 | 请求处理 | 数据库记录 |
|---------|-----------------|--------------------|----------|----------|------------|
| **全局禁用** | `true` | - | 不嵌入像素/不重写链接 | 返回响应但跳过记录 | 无记录 |
| **匿名追踪** | `false` | `false` | 嵌入像素/重写链接，但 subUUID = dummyUUID | subUUID 置空字符串 | 记录事件，subscriber_id = NULL |
| **个体追踪** | `false` | `true` | 嵌入像素/重写链接，使用真实 subUUID | 保留原始 subUUID | 记录事件，subscriber_id = 实际 ID |

#### 3.3 CORS 与追踪请求的关系

**重要结论**：邮件追踪本身**不需要**配置 CORS。

原因分析：

| 追踪类型 | 请求方式 | 浏览器行为 | CORS 限制 |
|---------|----------|------------|----------|
| 像素追踪 | `<img src="...">` | 简单资源请求 | 无限制 |
| 链接追踪 | `<a href="...">` 点击 | 页面导航/重定向 | 无限制 |

**需要 CORS 的场景**：

1. **内嵌订阅表单**：外部网站嵌入订阅表单，通过 AJAX 调用 `POST /api/public/subscription`
2. **自定义前端**：独立部署的管理前端，通过 API 访问 listmonk
3. **管理端 SPA**：前端 JavaScript 调用管理 API

---

## 隐私与追踪控制

### 1. 三层控制机制

Listmonk 提供三层隐私控制：

| 层级 | 配置项 | 效果 |
|------|--------|------|
| 全局禁用 | `privacy.disable_tracking` | 完全不记录任何追踪数据 |
| 匿名追踪 | `privacy.individual_tracking = false` | 记录事件但不关联具体订阅者 |
| 个体追踪 | `privacy.individual_tracking = true` | 记录事件并关联具体订阅者 |

### 2. 全局禁用追踪

**位置**: `cmd/public.go:541-548` (LinkRedirect)

```go
// 如果全局追踪禁用，直接解析 URL 不记录点击
if a.cfg.Privacy.DisableTracking {
    url, err := a.core.GetLinkURL(linkUUID)
    if err != nil {
        // ... 错误处理
    }
    return c.Redirect(http.StatusTemporaryRedirect, url)
}
```

**位置**: `cmd/public.go:571-574` (RegisterCampaignView)

```go
if a.cfg.Privacy.DisableTracking {
    c.Response().Header().Set("Cache-Control", "no-cache")
    return c.Blob(http.StatusOK, "image/png", pixelPNG)
}
```

### 3. 匿名追踪模式

当 `privacy.individual_tracking = false` 时：

#### 3.1 视图追踪

**位置**: `cmd/public.go:576-580`

```go
subUUID := c.Param("subUUID")
if !a.cfg.Privacy.IndividualTracking {
    subUUID = ""  // 清空订阅者 UUID
}
```

#### 3.2 链接追踪

**位置**: `cmd/public.go:550-554`

```go
subUUID := c.Param("subUUID")
if !a.cfg.Privacy.IndividualTracking {
    subUUID = ""  // 清空订阅者 UUID
}
```

#### 3.3 SQL 层面处理

**位置**: `queries/campaigns.sql:484`

```sql
LEFT JOIN subscribers ON (CASE WHEN $2::TEXT != '' THEN subscribers.uuid = $2::UUID ELSE FALSE END)
```

当 `$2` 为空字符串时，`subscriber_id` 为 NULL。

#### 3.4 模板渲染时

**位置**: `internal/manager/manager.go:355-357` (TrackLink)

```go
if !m.cfg.IndividualTracking {
    subUUID = dummyUUID  // 使用 "00000000-0000-0000-0000-000000000000"
}
```

**位置**: `internal/manager/manager.go:366-369` (TrackView)

```go
if !m.cfg.IndividualTracking {
    subUUID = dummyUUID
}
```

### 4. Dummy UUID 排除

在处理请求时，会排除预览时的 dummy UUID：

**位置**: `cmd/public.go:583-588`

```go
campUUID := c.Param("campUUID")
if campUUID != dummyUUID && subUUID != dummyUUID {
    if err := a.core.RegisterCampaignView(campUUID, subUUID); err != nil {
        a.log.Printf("error registering campaign view: %s", err)
    }
}
```

### 5. 隐私控制流程图

```
邮件发送时
    │
    ├─── Check DisableTracking?
    │       ├─── Yes ──> 不插入追踪像素 / 不重写链接
    │       │
    │       └─── No ──> Check IndividualTracking?
    │                       ├─── Yes ──> 使用真实 subUUID
    │                       │
    │                       └─── No ──> 使用 dummyUUID
    │
请求处理时
    │
    ├─── Check DisableTracking?
    │       ├─── Yes ──> 返回像素/重定向，但不记录
    │       │
    │       └─── No ──> Check IndividualTracking?
    │                       ├─── Yes ──> 记录 subUUID
    │                       │
    │                       └─── No ──> subUUID 设为空，不关联订阅者
```

---

## 统计数据聚合与管理端读取

### 1. 活动统计查询

**位置**: `queries/campaigns.sql:110-150`

```sql
WITH lists AS (
    SELECT campaign_id, JSON_AGG(JSON_BUILD_OBJECT('id', list_id, 'name', list_name)) AS lists 
    FROM campaign_lists WHERE campaign_id = ANY($1) GROUP BY campaign_id
),
media AS (...),
views AS (
    SELECT campaign_id, COUNT(campaign_id) as num FROM campaign_views
    WHERE campaign_id = ANY($1) GROUP BY campaign_id
),
clicks AS (
    SELECT campaign_id, COUNT(campaign_id) as num FROM link_clicks
    WHERE campaign_id = ANY($1) GROUP BY campaign_id
),
bounces AS (...)
SELECT ...
```

### 2. 时间序列统计

支持两种聚合模式：

| 模式 | 条件 | 时间粒度 |
|------|------|----------|
| 细粒度 | 时间间隔 < 7 天 | 按小时聚合 |
| 粗粒度 | 时间间隔 >= 7 天 | 按天聚合 |

**位置**: `queries/campaigns.sql:234-257`

```sql
WITH intval AS (
    SELECT CASE WHEN (EXTRACT (EPOCH FROM ($3::TIMESTAMP - $2::TIMESTAMP)) / 86400) >= 7 
           THEN 'day' ELSE 'hour' END
)
SELECT campaign_id, COUNT(*) AS "count", 
       DATE_TRUNC((SELECT * FROM intval), created_at) AS "timestamp"
FROM %s
WHERE campaign_id=ANY($1) AND created_at >= $2 AND created_at <= $3
GROUP BY campaign_id, "timestamp" ORDER BY "timestamp" ASC;
```

### 3. 去重统计 (Unique Counts)

当 `privacy.individual_tracking = true` 时，使用去重统计：

**位置**: `queries/campaigns.sql:234-246`

```sql
WITH uniqIDs AS (
    SELECT DISTINCT ON(subscriber_id) subscriber_id, campaign_id, 
           DATE_TRUNC((SELECT * FROM intval), created_at) AS "timestamp"
    FROM %s
    WHERE campaign_id=ANY($1) AND created_at >= $2 AND created_at <= $3
    ORDER BY subscriber_id, "timestamp"
)
SELECT COUNT(*) AS "count", campaign_id, "timestamp"
FROM uniqIDs GROUP BY campaign_id, "timestamp" ORDER BY "timestamp" ASC;
```

### 4. 链接点击排行榜

**位置**: `queries/campaigns.sql:269-276`

```sql
SELECT COUNT(%s) AS "count", url
FROM link_clicks
LEFT JOIN links ON (link_clicks.link_id = links.id)
WHERE campaign_id=ANY($1) AND link_clicks.created_at >= $2 AND link_clicks.created_at <= $3
GROUP BY links.url ORDER BY "count" DESC LIMIT 50;
```

### 5. 启动时的查询准备

**位置**: `cmd/init.go:400-429`

```go
func prepareQueries(qMap goyesql.Queries, db *sqlx.DB, ko *koanf.Koanf) *models.Queries {
    var (
        countQuery = "get-campaign-analytics-counts"
        linkSel    = "*"
    )
    if ko.Bool("privacy.individual_tracking") {
        countQuery = "get-campaign-analytics-unique-counts"  // 去重计数
        linkSel = "DISTINCT subscriber_id"                   // 去重链接计数
    }

    // 动态组装视图和点击统计查询
    qMap["get-campaign-view-counts"] = &goyesql.Query{
        Query: fmt.Sprintf(qMap[countQuery].Query, "campaign_views"),
        Tags:  map[string]string{"name": "get-campaign-view-counts"},
    }
    qMap["get-campaign-click-counts"] = &goyesql.Query{
        Query: fmt.Sprintf(qMap[countQuery].Query, "link_clicks"),
        Tags:  map[string]string{"name": "get-campaign-click-counts"},
    }
    qMap["get-campaign-link-counts"].Query = fmt.Sprintf(qMap["get-campaign-link-counts"].Query, linkSel)
    // ...
}
```

### 6. 管理端 API 处理

**位置**: `cmd/campaigns.go:603-642`

```go
func (a *App) GetCampaignViewAnalytics(c echo.Context) error {
    ids, err := parseStringIDs(c.Request().URL.Query()["id"])
    // ... 参数验证

    var (
        typ  = c.Param("type")  // views | clicks | links
        from = c.QueryParams().Get("from")
        to   = c.QueryParams().Get("to")
    )

    // 链接统计
    if typ == "links" {
        out, err := a.core.GetCampaignAnalyticsLinks(ids, typ, from, to)
        if err != nil {
            return err
        }
        return c.JSON(http.StatusOK, okResp{out})
    }

    // 视图/点击统计（时间序列）
    out, err := a.core.GetCampaignAnalyticsCounts(ids, typ, from, to)
    if err != nil {
        return err
    }

    return c.JSON(http.StatusOK, okResp{out})
}
```

**路由注册**：

**位置**: `cmd/handlers.go:164`

```go
g.GET("/api/campaigns/analytics/:type", pm(a.GetCampaignViewAnalytics, "campaigns:get_analytics"))
```

---

## 实现架构图

### 1. 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         邮件发送流程                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  模板渲染阶段                                                         │
│  ┌──────────────┐    ┌─────────────────────────────────────────┐   │
│  │  Campaign    │───>│  TemplateFuncs (TrackView / TrackLink)  │   │
│  │  Template    │    └─────────────────────────────────────────┘   │
│  └──────────────┘                    │                               │
│                                        v                               │
│                              ┌─────────────────┐                     │
│                              │ trackLink()     │                     │
│                              │  - 检查缓存      │                     │
│                              │  - 注册链接      │                     │
│                              │  - 返回追踪URL   │                     │
│                              └─────────────────┘                     │
│                                        │                               │
│                                        v                               │
│                              ┌─────────────────┐                     │
│                              │  最终邮件内容   │                     │
│                              │  - 嵌入像素     │                     │
│                              │  - 重写链接     │                     │
│                              └─────────────────┘                     │
│                                        │                               │
│                                        v                               │
│                              ┌─────────────────┐                     │
│                              │  SMTP / 发送    │                     │
│                              └─────────────────┘                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                         邮件接收与追踪流程                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  邮件客户端                                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  <img src="https://listmonk/campaign/xxx/yyy/px.png">      │   │
│  │  <a href="https://listmonk/link/aaa/bbb/ccc">点击这里</a>   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                         │                    │                        │
│                         v                    v                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    HTTP 请求到 listmonk                         │   │
│  │  GET /campaign/:campUUID/:subUUID/px.png                      │   │
│  │  GET /link/:linkUUID/:campUUID/:subUUID                        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                         │                    │                        │
│                         v                    v                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Echo 路由中间件（修正后）                    │   │
│  │                                                                  │   │
│  │  像素追踪端点:                                                   │   │
│  │    GET /campaign/:campUUID/:subUUID/px.png                    │   │
│  │    ┌─────────────┐                                              │   │
│  │    │ noIndex()   │ ──> X-Robots-Tag: noindex                  │   │
│  │    └──────┬──────┘                                              │   │
│  │           v                                                       │   │
│  │    ┌─────────────┐                                              │   │
│  │    │ hasUUID()   │ ──> 正则校验 UUID 格式                       │   │
│  │    └──────┬──────┘                                              │   │
│  │           v                                                       │   │
│  │    ┌─────────────────────────────────┐                         │   │
│  │    │ RegisterCampaignView()          │                         │   │
│  │    │ - 没有 hasSub！                 │                         │   │
│  │    │ - 不验证订阅者是否存在          │                         │   │
│  │    └─────────────────────────────────┘                         │   │
│  │                                                                  │   │
│  │  链接追踪端点:                                                   │   │
│  │    GET /link/:linkUUID/:campUUID/:subUUID                     │   │
│  │    ┌─────────────┐                                              │   │
│  │    │ noIndex()   │ ──> X-Robots-Tag: noindex                  │   │
│  │    └──────┬──────┘                                              │   │
│  │           v                                                       │   │
│  │    ┌─────────────┐                                              │   │
│  │    │ hasUUID()   │ ──> 正则校验 UUID 格式                       │   │
│  │    └──────┬──────┘                                              │   │
│  │           v                                                       │   │
│  │    ┌─────────────────────────────────┐                         │   │
│  │    │ LinkRedirect()                  │                         │   │
│  │    │ - 没有 hasSub！                 │                         │   │
│  │    │ - 不验证订阅者是否存在          │                         │   │
│  │    └─────────────────────────────────┘                         │   │
│  │                                                                  │   │
│  │  注意: hasSub 只用于订阅相关页面，如:                           │   │
│  │       /subscription/:campUUID/:subUUID                          │   │
│  │       /subscription/optin/:subUUID                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                         │                    │                        │
│                         v                    v                        │
│  ┌──────────────────────────┐    ┌──────────────────────────────┐   │
│  │ RegisterCampaignView()   │    │ LinkRedirect()               │   │
│  │ - 检查追踪配置            │    │ - 检查追踪配置                │   │
│  │ - 检查个体追踪            │    │ - 检查个体追踪                │   │
│  │ - 排除 dummy UUID        │    │ - 记录点击到 link_clicks      │   │
│  │ - 插入 campaign_views    │    │ - 307 重定向到原始 URL       │   │
│  │ - 返回 14x3 透明 PNG     │    └──────────────────────────────┘   │
│  └──────────────────────────┘                                         │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. 数据流向图

```
┌──────────────────────────────────────────────────────────────────────┐
│                            数据存储层                                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────────┐   │
│  │  campaigns   │      │   links      │      │   subscribers    │   │
│  │  (活动表)    │      │  (链接表)    │      │   (订阅者表)     │   │
│  └──────┬───────┘      └──────┬───────┘      └────────┬─────────┘   │
│         │                     │                        │              │
│         │                     │                        │              │
│         v                     v                        v              │
│  ┌──────────────┐      ┌──────────────┐               │              │
│  │campaign_views│      │ link_clicks  │               │              │
│  │ (打开记录表)  │      │ (点击记录表)  │               │              │
│  ├──────────────┤      ├──────────────┤               │              │
│  │ campaign_id  │      │ campaign_id  │◄──────────────┘              │
│  │ subscriber_id│◄─────│ subscriber_id│                              │
│  │ created_at   │      │ link_id      │◄───────────────────────────┐ │
│  └──────────────┘      │ created_at   │                            │ │
│                         └──────────────┘                            │ │
│                                                                       │ │
│  外键约束:                                                            │ │
│  - campaign_id:   ON DELETE CASCADE  (活动删除时级联删除记录)        │ │
│  - subscriber_id: ON DELETE SET NULL (订阅者删除时置空，保留统计)    │ │
│  - link_id:       ON DELETE CASCADE  (链接删除时级联删除点击记录)    │ │
│                                                                       │ │
└──────────────────────────────────────────────────────────────────────┘
```

### 3. URL 格式汇总

| 类型 | URL 格式 | 处理函数 | 中间件 |
|------|----------|----------|--------|
| 打开追踪像素 | `{root}/campaign/{campUUID}/{subUUID}/px.png` | `RegisterCampaignView` | `noIndex` → `hasUUID` |
| 链接追踪重定向 | `{root}/link/{linkUUID}/{campUUID}/{subUUID}` | `LinkRedirect` | `noIndex` → `hasUUID` |
| 邮件在线查看 | `{root}/campaign/{campUUID}/{subUUID}` | `ViewCampaignMessage` | `noIndex` → `hasUUID` |
| 订阅管理页面 | `{root}/subscription/:campUUID/:subUUID` | `SubscriptionPage` | `noIndex` → `hasUUID` → `hasSub` |
| 统计分析 API | `{root}/api/campaigns/analytics/:type` | `GetCampaignViewAnalytics` | `pm (权限校验)` |

---

## 关键文件位置汇总

| 功能 | 文件路径 |
|------|----------|
| HTTP 处理器 | `cmd/public.go` |
| 路由注册与中间件 | `cmd/handlers.go` |
| 追踪模板函数 | `internal/manager/manager.go` |
| 核心业务逻辑 | `internal/core/campaigns.go` |
| 数据库 Schema | `schema.sql` |
| SQL 查询 | `queries/campaigns.sql`, `queries/links.sql` |
| URL 配置 | `cmd/init.go` |
| 管理端 API | `cmd/campaigns.go` |

---

## 总结

### 1. 修正的事实错误

#### 1.1 像素尺寸

| 错误描述 | 正确事实 |
|---------|---------|
| "3x14 像素"（含糊） | **14x3 像素（宽 x 高）** |
| - | `drawTransparentImage(3, 14)` → h=3, w=14 |
| - | `image.Rect(0, 0, 14, 3)` → 宽 14, 高 3 |

#### 1.2 中间件链路

| 错误描述 | 正确事实 |
|---------|---------|
| 追踪端点有 `hasSub` 中间件 | **追踪端点 NO `hasSub`** |
| - | 像素追踪: `noIndex` → `hasUUID` → `RegisterCampaignView` |
| - | 链接追踪: `noIndex` → `hasUUID` → `LinkRedirect` |
| - | `hasSub` 只用于 `/subscription/...` 等订阅页面 |

### 2. 设计亮点

1. **失败安全 (Fail-Safe)**: 即使记录失败，也会返回像素图片或执行重定向，不影响用户体验
2. **多层隐私控制**: 从全局禁用到匿名追踪再到个体追踪，灵活满足不同隐私需求
3. **数据完整性**: 订阅者删除时保留匿名化统计数据（SET NULL），活动删除时清理相关记录（CASCADE）
4. **缓存优化**: 链接注册使用内存缓存，避免重复数据库查询
5. **智能统计**: 根据时间间隔自动选择聚合粒度（小时/天），根据追踪模式选择计数方式（去重/普通）

### 3. 隐私设计要点

1. **Dummy UUID**: 模板预览时使用 `00000000-0000-0000-0000-000000000000`，避免污染统计数据
2. **SET NULL 策略**: 订阅者删除后，记录中的 `subscriber_id` 置 NULL，保留历史统计但无法关联到具体个人
3. **可选个体追踪**: 管理员可选择是否记录具体订阅者的行为
4. **无 `hasSub` 中间件**: 追踪端点不验证订阅者存在，即使订阅者已删除，事件仍可匿名记录

### 4. 技术要点

1. **307 重定向**: 保持原始 HTTP 方法，虽然链接点击通常是 GET
2. **透明 PNG**: 14x3 像素（宽x高），Go 标准库动态生成，无需外部文件
3. **Cache-Control: no-cache**: 防止浏览器缓存像素请求，确保每次打开都被记录
4. **X-Robots-Tag: noindex**: 防止搜索引擎索引追踪端点
5. **动态查询准备**: 启动时根据 `individual_tracking` 配置选择去重/普通计数查询

---

*分析基于 listmonk 代码库，修订时间: 2026-05-05*
