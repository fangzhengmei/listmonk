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

两种机制都依赖于 HTTP 请求到 listmonk 服务器，从而触发统计数据的记录。本文档详细分析从追踪请求入口、事件落库、统计聚合到管理端展示的完整闭环，并特别关注**失败安全行为差异**、**campaign 不存在时的分支差异**、**统计接口完整范围**以及**unique 计数口径边界条件**。

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

        c.log.Printf("error registering campaign view: %s", err)
        return echo.NewHTTPError(http.StatusInternalServerError,
            c.i18n.Ts("globals.messages.errorUpdating", "name", "{globals.terms.campaign}", "error", pqErrMsg(err)))
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
            return c.Render(e.Code, tplMessage, makeMsgTpl(a.i18n.T("public.errorTitle"), "", e.Error()))
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
        return c.Render(e.Code, tplMessage, makeMsgTpl(a.i18n.T("public.errorTitle"), "", e.Error()))
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
            return "", echo.NewHTTPError(http.StatusBadRequest, c.i18n.Ts("public.invalidLink"))
        }

        c.log.Printf("error registering link click: %s", err)
        return "", echo.NewHTTPError(http.StatusInternalServerError, c.i18n.Ts("public.errorProcessingRequest"))
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
| `campaign_id` | INTEGER | 关联的活动 ID（**NOT NULL**，级联删除） |
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
| `campaign_id` | INTEGER | 关联的活动 ID（**可为 NULL**，级联删除） |
| `link_id` | INTEGER | 关联的链接 ID（必填） |
| `subscriber_id` | INTEGER | 关联的订阅者 ID（可为 NULL） |
| `created_at` | TIMESTAMP | 记录创建时间 |

> **关键差异**：`campaign_views.campaign_id` 是 `NOT NULL`，而 `link_clicks.campaign_id` 是 `NULL`。这导致 campaign 不存在时，两种追踪的行为完全不同。详见后续"事件落库分支差异"章节。

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
campaigns (id) ─────┬──< campaign_views (campaign_id) [CASCADE, NOT NULL]
                    │
                    └──< link_clicks (campaign_id) [CASCADE, NULL]

subscribers (id) ───┬──< campaign_views (subscriber_id) [SET NULL]
                    │
                    └──< link_clicks (subscriber_id) [SET NULL]

links (id) ────────────< link_clicks (link_id) [CASCADE, NOT NULL]
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
│  │               阶段 3: 事件落库（含失败安全与分支差异）              │     │
│  ├────────────────────────────────────────────────────────────────────┤     │
│  │                                                                      │     │
│  │  ┌────────────────────────────────────────────────────────────┐   │     │
│  │  │ 子阶段 3.1: campaign 不存在时的分支差异                      │   │     │
│  │  ├────────────────────────────────────────────────────────────┤   │     │
│  │  │                                                                 │   │     │
│  │  │  【像素追踪 - campaign_views】                                │   │     │
│  │  │  - campaign_id 字段: NOT NULL REFERENCES campaigns(id)       │   │     │
│  │  │  - SQL 逻辑:                                                  │   │     │
│  │  │    INSERT INTO campaign_views (campaign_id, ...)             │   │     │
│  │  │    VALUES((SELECT id FROM campaigns WHERE uuid = $1), ...)   │   │     │
│  │  │  - 如果 campaign 不存在:                                      │   │     │
│  │  │    SELECT 返回 NULL → 插入失败（NOT NULL 约束）               │   │     │
│  │  │  - core 层处理:                                                │   │     │
│  │  │    if pqErr.Column == "campaign_id" { return nil }           │   │     │
│  │  │    → 静默忽略，不返回错误                                      │   │     │
│  │  │  - 最终效果:                                                   │   │     │
│  │  │    不插入记录，但 handler 继续返回像素                         │   │     │
│  │  │                                                                 │   │     │
│  │  │  【链接追踪 - link_clicks】                                    │   │     │
│  │  │  - campaign_id 字段: NULL REFERENCES campaigns(id)            │   │     │
│  │  │  - SQL 逻辑:                                                  │   │     │
│  │  │    INSERT INTO link_clicks (campaign_id, ...) VALUES(        │   │     │
│  │  │      (SELECT id FROM campaigns WHERE uuid = $2), ...          │   │     │
│  │  │    )                                                           │   │     │
│  │  │  - 如果 campaign 不存在:                                      │   │     │
│  │  │    SELECT 返回 NULL → 插入成功（允许 NULL）                    │   │     │
│  │  │  - core 层处理:                                                │   │     │
│  │  │    只检查 pqErr.Column == "link_id"                          │   │     │
│  │  │    不检查 campaign_id！                                        │   │     │
│  │  │  - 最终效果:                                                   │   │     │
│  │  │    插入成功，campaign_id = NULL                                │   │     │
│  │  │    返回原始 URL，继续重定向                                    │   │     │
│  │  └────────────────────────────────────────────────────────────┘   │     │
│  │                                                                      │     │
│  │  ┌────────────────────────────────────────────────────────────┐   │     │
│  │  │ 子阶段 3.2: 数据库报错时的失败安全行为差异                  │   │     │
│  │  ├────────────────────────────────────────────────────────────┤   │     │
│  │  │                                                                 │   │     │
│  │  │  【像素追踪 - 真正的失败安全】                                │   │     │
│  │  │                                                                 │   │     │
│  │  │  Handler 层代码 (cmd/public.go:583-591):                    │   │     │
│  │  │    if err := a.core.RegisterCampaignView(...); err != nil {  │   │     │
│  │  │        a.log.Printf("error registering campaign view: %s", err) │   │
│  │  │    }                                                         │   │     │
│  │  │    // 无论如何都返回像素                                      │   │     │
│  │  │    c.Response().Header().Set("Cache-Control", "no-cache")   │   │     │
│  │  │    return c.Blob(http.StatusOK, "image/png", pixelPNG)       │   │     │
│  │  │                                                                 │   │     │
│  │  │  行为分析:                                                     │   │     │
│  │  │  - 数据库错误只被 log，不返回给客户端                          │   │     │
│  │  │  - 始终返回 200 OK + PNG 二进制                               │   │     │
│  │  │  - 客户端完全感知不到失败                                     │   │     │
│  │  │  - 这才是真正的"失败安全"                                     │   │     │
│  │  │                                                                 │   │     │
│  │  │  【链接追踪 - 非失败安全】                                    │   │     │
│  │  │                                                                 │   │     │
│  │  │  Handler 层代码 (cmd/public.go:556-562):                    │   │     │
│  │  │    url, err := a.core.RegisterCampaignLinkClick(...)          │   │     │
│  │  │    if err != nil {                                            │   │     │
│  │  │        e := err.(*echo.HTTPError)                             │   │     │
│  │  │        return c.Render(e.Code, tplMessage, ...)              │   │     │
│  │  │    }                                                         │   │     │
│  │  │    return c.Redirect(http.StatusTemporaryRedirect, url)       │   │     │
│  │  │                                                                 │   │     │
│  │  │  行为分析:                                                     │   │     │
│  │  │  - 数据库错误直接返回给客户端                                  │   │     │
│  │  │  - 渲染错误页面，而非继续重定向                                │   │     │
│  │  │  - 用户会看到错误提示                                          │   │     │
│  │  │  - 这**不是**失败安全！                                       │   │     │
│  │  │                                                                 │   │     │
│  │  │  唯一例外：link_id 不存在时                                   │   │     │
│  │  │    core 层返回 400 "invalid link"                            │   │     │
│  │  │    这是预期行为，而非系统错误                                  │   │     │
│  │  └────────────────────────────────────────────────────────────┘   │     │
│  │                                                                      │     │
│  │  外键异常处理总览:                                                   │     │
│  │    ┌──────────────────────────────────────────────────────────┐   │     │
│  │    │ 异常类型          │ 像素追踪          │ 链接追踪          │   │     │
│  │    ├──────────────────────────────────────────────────────────┤   │     │
│  │    │ campaign_id 不存在 │ 静默忽略(200)    │ 插入成功(NULL)   │   │     │
│  │    │ link_id 不存在     │ 不适用            │ 400 invalid link │   │     │
│  │    │ subscriber_id 不存在 │ 插入成功(NULL)  │ 插入成功(NULL)   │   │     │
│  │    │ 其他数据库错误     │ 静默忽略(200)    │ 返回错误页面     │   │     │
│  │    └──────────────────────────────────────────────────────────┘   │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│           │                                                                    │
│           v                                                                    │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                        阶段 4: 响应返回                              │     │
│  ├────────────────────────────────────────────────────────────────────┤     │
│  │                                                                      │     │
│  │  像素追踪:                                                           │     │
│  │    ┌────────────────────────────────────────────────────────────┐   │     │
│  │    │ 成功或失败:                                                  │   │     │
│  │    │   Content-Type: image/png                                    │   │     │
│  │    │   Cache-Control: no-cache                                    │   │     │
│  │    │   Status: 200 OK                                             │   │     │
│  │    │   Body: 14x3 透明 PNG 二进制数据                              │   │     │
│  │    └────────────────────────────────────────────────────────────┘   │     │
│  │                                                                      │     │
│  │  链接追踪:                                                           │     │
│  │    ┌────────────────────────────────────────────────────────────┐   │     │
│  │    │ 成功路径:                                                    │   │     │
│  │    │   Status: 307 Temporary Redirect                            │   │     │
│  │    │   Location: {原始 URL}                                      │   │     │
│  │    │                                                              │   │     │
│  │    │ 失败路径:                                                    │   │     │
│  │    │   Status: 400 / 500                                         │   │     │
│  │    │   Content-Type: text/html                                    │   │     │
│  │    │   Body: 错误页面渲染                                         │   │     │
│  │    └────────────────────────────────────────────────────────────┘   │     │
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
│  │  统计类型 (type 参数完整范围):                                       │     │
│  │    ┌──────────────┬────────────────────────────────────────────┐  │     │
│  │    │ type = views │ 按小时/天聚合的打开时间序列                  │  │     │
│  │    ├──────────────┼────────────────────────────────────────────┤  │     │
│  │    │ type = clicks│ 按小时/天聚合的点击时间序列                  │  │     │
│  │    ├──────────────┼────────────────────────────────────────────┤  │     │
│  │    │ type = bounces│ 按小时/天聚合的退信时间序列                 │  │     │
│  │    ├──────────────┼────────────────────────────────────────────┤  │     │
│  │    │ type = links │ 各链接的点击次数排行榜 (TOP 50)             │  │     │
│  │    └──────────────┴────────────────────────────────────────────┘  │     │
│  │                                                                      │     │
│  │  时间粒度自动选择:                                                   │     │
│  │    └──> 时间间隔 < 7 天: 按小时聚合                                │     │
│  │    └──> 时间间隔 >= 7 天: 按天聚合                                 │     │
│  │                                                                      │     │
│  │  Unique 计数口径边界说明:                                           │     │
│  │    ┌──────────────────────────────────────────────────────────┐   │     │
│  │    │ PostgreSQL DISTINCT ON(subscriber_id) 行为:               │   │     │
│  │    │                                                              │   │     │
│  │    │  真实订阅者 (subscriber_id != NULL):                        │   │     │
│  │    │    └──> 同一订阅者的多条记录只保留第一行                    │   │     │
│  │    │    └──> 每个订阅者只计一次                                   │   │     │
│  │    │                                                              │   │     │
│  │    │  匿名事件 (subscriber_id = NULL):                           │   │     │
│  │    │    └──> PostgreSQL 中 NULL != NULL                          │   │     │
│  │    │    └──> 每个 NULL 值都被视为不同                            │   │     │
│  │    │    └──> 每条匿名记录都计一次                                 │   │     │
│  │    │                                                              │   │     │
│  │    │  边界情况:                                                    │   │     │
│  │    │    └──> 匿名追踪模式下，unique 计数 = 普通计数              │   │     │
│  │    │    └──> 因为所有记录的 subscriber_id 都是 NULL             │   │     │
│  │    │    └──> 每条记录都独立计数                                   │   │     │
│  │    └──────────────────────────────────────────────────────────┘   │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.2 失败安全行为对比表

| 维度 | 像素追踪 (RegisterCampaignView) | 链接追踪 (LinkRedirect) |
|------|---------------------------------|-------------------------|
| **Handler 层错误处理** | 只 log，不返回 | 直接返回错误页面 |
| **数据库错误时的响应** | 始终返回 200 + PNG | 返回 400/500 错误页面 |
| **用户感知** | 完全无感知 | 会看到错误提示 |
| **是否失败安全** | **是** | **否** |
| **代码位置** | `cmd/public.go:583-591` | `cmd/public.go:556-562` |

#### 3.3 Campaign 不存在时的分支差异

| 维度 | 像素追踪 (campaign_views) | 链接追踪 (link_clicks) |
|------|---------------------------|------------------------|
| **campaign_id 约束** | `NOT NULL REFERENCES` | `NULL REFERENCES` |
| **campaign 不存在时** | SELECT 返回 NULL → 插入失败 | SELECT 返回 NULL → 插入成功 |
| **core 层处理** | 检查 `pqErr.Column == "campaign_id"`，返回 nil | 只检查 `link_id`，不检查 `campaign_id` |
| **最终效果** | 不插入记录，返回像素 | 插入成功，`campaign_id = NULL`，继续重定向 |
| **统计影响** | 无记录，不影响统计 | 有记录，但 campaign_id 为 NULL，无法关联到具体活动 |

#### 3.4 CORS 与追踪请求的关系

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
        e := err.(*echo.HTTPError)
        return c.Render(e.Code, tplMessage, 
            makeMsgTpl(a.i18n.T("public.errorTitle"), "", e.Error()))
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

### 3. 统计接口 type 的完整范围

**位置**: `internal/core/campaigns.go:384-409`

```go
func (c *Core) GetCampaignAnalyticsCounts(campIDs []int, typ, fromDate, toDate string) ([]models.CampaignAnalyticsCount, error) {
    var stmt *sqlx.Stmt
    switch typ {
    case "views":
        stmt = c.q.GetCampaignViewCounts
    case "clicks":
        stmt = c.q.GetCampaignClickCounts
    case "bounces":  // 新增：退信统计
        stmt = c.q.GetCampaignBounceCounts
    default:
        // ...
    }
    // ...
}
```

**完整的 type 参数范围**：

| type 值 | 统计类型 | 数据源 | 时间聚合 |
|---------|----------|--------|----------|
| `views` | 打开统计 | `campaign_views` | 按小时/天聚合 |
| `clicks` | 点击统计 | `link_clicks` | 按小时/天聚合 |
| `bounces` | 退信统计 | `bounces` | 按小时/天聚合 |
| `links` | 链接排行榜 | `link_clicks` + `links` | TOP 50，按 URL 分组 |

**退信统计 SQL**：

**位置**: `queries/campaigns.sql:259-267`

```sql
-- name: get-campaign-bounce-counts
WITH intval AS (
    SELECT CASE WHEN (EXTRACT (EPOCH FROM ($3::TIMESTAMP - $2::TIMESTAMP)) / 86400) >= 7 THEN 'day' ELSE 'hour' END
)
SELECT campaign_id, COUNT(*) AS "count", DATE_TRUNC((SELECT * FROM intval), created_at) AS "timestamp"
    FROM bounces
    WHERE campaign_id=ANY($1) AND created_at >= $2 AND created_at <= $3
    GROUP BY campaign_id, "timestamp" ORDER BY "timestamp" ASC;
```

### 4. Unique 计数口径与边界说明

#### 4.1 Unique 计数查询

**位置**: `queries/campaigns.sql:234-246`

```sql
-- name: get-campaign-analytics-unique-counts
WITH intval AS (
    SELECT CASE WHEN (EXTRACT (EPOCH FROM ($3::TIMESTAMP - $2::TIMESTAMP)) / 86400) >= 7 THEN 'day' ELSE 'hour' END
),
uniqIDs AS (
    SELECT DISTINCT ON(subscriber_id) subscriber_id, campaign_id, 
           DATE_TRUNC((SELECT * FROM intval), created_at) AS "timestamp"
    FROM %s
    WHERE campaign_id=ANY($1) AND created_at >= $2 AND created_at <= $3
    ORDER BY subscriber_id, "timestamp"
)
SELECT COUNT(*) AS "count", campaign_id, "timestamp"
FROM uniqIDs GROUP BY campaign_id, "timestamp" ORDER BY "timestamp" ASC;
```

#### 4.2 PostgreSQL DISTINCT ON 行为分析

**关键知识点**：PostgreSQL 中 `NULL != NULL`，即两个 NULL 值不相等。

**场景对比**：

| 场景 | subscriber_id 值 | DISTINCT ON 行为 | 计数结果 |
|------|------------------|------------------|----------|
| 真实订阅者 A | 100, 100, 100 | 只保留第一行 | 计 1 次 |
| 真实订阅者 B | 200, 200 | 只保留第一行 | 计 1 次 |
| 匿名事件 | NULL, NULL, NULL | 每个 NULL 视为不同 | 计 3 次 |

#### 4.3 边界情况详解

**边界情况 1：匿名追踪模式 (IndividualTracking = false)**

- 所有记录的 `subscriber_id` 都是 NULL
- `DISTINCT ON(subscriber_id)` 对 NULL 无效
- **结果**：unique 计数 = 普通计数

**边界情况 2：混合模式（部分订阅者已删除）**

- 真实订阅者：每个只计一次
- 已删除订阅者（subscriber_id = NULL）：每条都计一次
- **结果**：真实订阅者去重，匿名事件累加

**边界情况 3：同一订阅者多次打开**

- `subscriber_id = 100` 的多条记录
- `DISTINCT ON(subscriber_id)` 只保留第一行
- **结果**：每个订阅者每天/每小时只计一次

#### 4.4 Unique 计数的启动时准备

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

**关键设计**：
- Unique/普通计数是**启动时决定**的，不是运行时
- `privacy.individual_tracking` 配置决定使用哪种查询
- 匿名追踪模式下，unique 计数查询实际效果 = 普通计数

### 5. 链接点击排行榜

**位置**: `queries/campaigns.sql:269-276`

```sql
-- name: get-campaign-link-counts
-- raw: true
-- %s = * or DISTINCT subscriber_id (prepared based on based on individual tracking=on/off). Prepared on boot.
SELECT COUNT(%s) AS "count", url
FROM link_clicks
LEFT JOIN links ON (link_clicks.link_id = links.id)
WHERE campaign_id=ANY($1) AND link_clicks.created_at >= $2 AND link_clicks.created_at <= $3
GROUP BY links.url ORDER BY "count" DESC LIMIT 50;
```

**排行榜的 unique 计数**：
- `IndividualTracking = true`: `COUNT(DISTINCT subscriber_id)`
- `IndividualTracking = false`: `COUNT(*)`

### 6. 管理端 API 处理

**位置**: `cmd/campaigns.go:603-642`

```go
func (a *App) GetCampaignViewAnalytics(c echo.Context) error {
    ids, err := parseStringIDs(c.Request().URL.Query()["id"])
    // ... 参数验证

    var (
        typ  = c.Param("type")  // views | clicks | bounces | links
        from = c.QueryParams().Get("from")
        to   = c.QueryParams().Get("to")
    )

    // 链接统计（排行榜）
    if typ == "links" {
        out, err := a.core.GetCampaignAnalyticsLinks(ids, typ, from, to)
        if err != nil {
            return err
        }
        return c.JSON(http.StatusOK, okResp{out})
    }

    // 视图/点击/退信统计（时间序列）
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
│  │    │ - 真正的失败安全（始终返回PNG） │                         │   │
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
│  │    │ - 非失败安全（出错返回错误页）  │                         │   │
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
│  │ - 真正的失败安全          │    │ - 非失败安全                  │   │
│  │ - 始终返回 200 + PNG      │    │ - 出错返回错误页面            │   │
│  │ - campaign 不存在时静默忽略│    │ - campaign 不存在时插入成功   │   │
│  └──────────────────────────┘    └──────────────────────────────┘   │
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
│  │ campaign_id  │ NOT  │ campaign_id  │ NULL ◄────────┘              │
│  │ subscriber_id│◄─────│ subscriber_id│                              │
│  │ created_at   │      │ link_id      │◄───────────────────────────┐ │
│  └──────────────┘      │ created_at   │                            │ │
│                         └──────────────┘                            │ │
│                                                                       │ │
│  关键差异:                                                            │ │
│  -