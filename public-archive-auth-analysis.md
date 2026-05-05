# Listmonk 公开邮件存档与订阅者自助门户访问鉴权分析报告

## 一、概述

本文档详细分析 listmonk 项目中**公开邮件存档（Public Archive）**和**订阅者自助门户（Subscriber Self-Service Portal）**的访问鉴权机制、内容裁剪策略，以及未登录访客与已订阅用户之间的内容差异。

---

## 二、公开邮件存档（Public Archive）访问鉴权机制

公开邮件存档是 listmonk 提供的一项功能，允许管理员将已发送的邮件Campaign公开到互联网上，供访客浏览。该功能采用**多层防护机制**确保安全性。

### 2.1 第一层：功能开关控制

#### 2.1.1 全局配置开关

整个公开存档功能由一个全局配置项控制：

```go
// models/settings.go:14
EnablePublicArchive bool `json:"app.enable_public_archive"`
```

#### 2.1.2 路由条件注册

只有当配置开关启用时，相关路由才会被注册：

```go
// cmd/handlers.go:263, 281-286
// API 路由
if a.cfg.EnablePublicArchive {
    g.GET("/api/public/archive", a.GetCampaignArchives)
}

// 页面路由
if a.cfg.EnablePublicArchive {
    g.GET("/archive", a.CampaignArchivesPage)
    g.GET("/archive.xml", a.GetCampaignArchivesFeed)
    g.GET("/archive/:id", a.CampaignArchivePage)
    g.GET("/archive/latest", a.CampaignArchivePageLatest)
}
```

**安全意义**：如果功能未启用，访问这些路径会直接返回 404，从根本上阻止未授权访问。

### 2.2 第二层：数据层面过滤

即使功能启用，SQL 查询层面也会进行严格的数据过滤，确保只返回符合条件的 Campaign。

#### 2.2.1 SQL 查询条件

```sql
-- queries/campaigns.sql:99-108
-- name: get-archived-campaigns
SELECT COUNT(*) OVER () AS total, campaigns.*,
    COALESCE(templates.body, (SELECT body FROM templates WHERE is_default = true LIMIT 1), '') AS template_body
    FROM campaigns
    LEFT JOIN templates ON (
        CASE WHEN $3 = 'default' THEN templates.id = campaigns.template_id
        ELSE templates.id = campaigns.archive_template_id END
    )
    WHERE campaigns.archive=true 
      AND campaigns.type='regular' 
      AND campaigns.status=ANY('{running, paused, finished}')
    ORDER by campaigns.created_at DESC OFFSET $1 LIMIT $2;
```

**过滤条件解析**：
| 条件 | 说明 | 安全意义 |
|-----|------|---------|
| `archive=true` | 只有标记为"存档"的 Campaign | 防止未准备好的内容泄露 |
| `type='regular'` | 只返回普通邮件 Campaign | 排除 optin 等内部 Campaign |
| `status IN ('running', 'paused', 'finished')` | 只返回已发送或正在发送的 | 排除草稿、已取消等状态 |

#### 2.2.2 额外业务逻辑校验

在获取单个存档 Campaign 时，还有额外的校验：

```go
// internal/core/campaigns.go:72-85
// GetArchivedCampaign retrieves a campaign with the archive template body.
func (c *Core) GetArchivedCampaign(id int, uuid, archiveSlug string) (models.Campaign, error) {
    out, err := c.getCampaign(id, uuid, archiveSlug, campaignTplArchive)
    if err != nil {
        return out, err
    }

    // 额外检查：确保 Archive 标志为 true
    if !out.Archive {
        return models.Campaign{}, echo.NewHTTPError(http.StatusBadRequest,
            c.i18n.Ts("globals.messages.notFound", "name", "{globals.terms.campaign}"))
    }

    return out, nil
}
```

**安全意义**：双重校验防止 SQL 注入或其他绕过方式。

### 2.3 第三层：内容裁剪与数据脱敏

公开存档页面使用**虚拟订阅者数据**来渲染邮件模板，确保真实订阅者数据不被泄露。

#### 2.3.1 Dummy Subscriber 机制

```go
// cmd/archive.go:238-277
// compileArchiveCampaigns compiles the campaign template with the subscriber data.
func (a *App) compileArchiveCampaigns(camps []models.Campaign) ([]manager.CampaignMessage, error) {
    // ...
    for _, c := range camps {
        // ...
        // Load the dummy subscriber meta.
        var sub models.Subscriber
        if err := json.Unmarshal([]byte(camp.ArchiveMeta), &sub); err != nil {
            // ...
        }

        m := manager.CampaignMessage{
            Campaign:   &camp,
            Subscriber: sub,  // 使用从 ArchiveMeta 反序列化的虚拟数据
        }
        // ...
    }
    // ...
}
```

**工作原理**：
1. 每个存档 Campaign 都有一个 `ArchiveMeta` 字段（JSON 格式）
2. 该字段存储的是一个**模板化的虚拟订阅者数据**
3. 渲染公开存档时，使用这个虚拟数据而不是真实订阅者数据

**安全意义**：
- 真实订阅者的姓名、邮箱等敏感信息不会在公开页面泄露
- 邮件中的个性化变量（如 `{{ .Subscriber.Name }}`）会被替换为预设的虚拟值

#### 2.3.2 RSS 内容控制

RSS 订阅源的内容显示有独立的配置控制：

```go
// models/settings.go:15
EnablePublicArchiveRSSContent bool `json:"app.enable_public_archive_rss_content"`

// cmd/archive.go:52-96
func (a *App) GetCampaignArchivesFeed(c echo.Context) error {
    var (
        pg              = a.pg.NewFromURL(c.Request().URL.Query())
        showFullContent = a.cfg.EnablePublicArchiveRSSContent  // 控制 RSS 是否显示完整内容
    )

    // Get archives from the DB.
    camps, _, err := a.getCampaignArchives(pg.Offset, pg.Limit, showFullContent)
    // ...
}
```

**配置选项**：
- `true`：RSS 中包含邮件完整内容
- `false`：RSS 中只包含标题和元数据，不包含正文

---

## 三、订阅者自助门户（Subscriber Self-Service Portal）权限控制

订阅者自助门户允许订阅者通过邮件中的链接访问个人订阅管理页面，进行退订、偏好管理、数据导出/删除等操作。

### 3.1 访问令牌结构与权限边界

订阅者自助门户的访问链接包含两种标识：

| 标识类型 | 参数名 | 来源 | 权限范围 |
|---------|-------|------|---------|
| 活动标识 | `campUUID` | 邮件 Campaign 的 UUID | 标识该链接来自哪封邮件 |
| 订阅者标识 | `subUUID` | 订阅者的 UUID | **核心访问令牌，决定访问权限** |

#### 3.1.1 两种标识的关联校验位置

**关键发现**：`campUUID` 和 `subUUID` 的关联校验**仅在特定操作中进行**，在大多数页面中 `campUUID` 并不作为权限校验的依据。

**路由中的标识使用情况**：

| 路由 | 标识参数 | 关联校验位置 | 校验强度 |
|-----|---------|-------------|---------|
| `GET /subscription/:campUUID/:subUUID` | campUUID + subUUID | **无关联校验** | campUUID 被忽略 |
| `POST /subscription/:campUUID/:subUUID` | campUUID + subUUID | SQL 层面（简单退订时） | 仅简单退订时使用 |
| `GET /campaign/:campUUID/:subUUID` | campUUID + subUUID | **无关联校验** | 各自独立验证存在性 |
| `GET/POST /subscription/optin/:subUUID` | 仅 subUUID | 无 campUUID | 不涉及 |
| `POST /subscription/export/:subUUID` | 仅 subUUID | 无 campUUID | 不涉及 |
| `POST /subscription/wipe/:subUUID` | 仅 subUUID | 无 campUUID | 不涉及 |

**1. 订阅管理页面（GET）- 无关联校验**

```go
// cmd/public.go:197-208
func (a *App) SubscriptionPage(c echo.Context) error {
    var (
        subUUID       = c.Param("subUUID")  // 只获取 subUUID
        showManage, _ = strconv.ParseBool(c.FormValue("manage"))
    )

    // Get the subscriber from the DB.
    s, err := a.core.GetSubscriber(0, subUUID, "")  // 只使用 subUUID
    // campUUID 完全没有被使用！
    // ...
}
```

**代码位置**：`cmd/public.go:197-248`

**安全影响**：
- 只要持有有效的 `subUUID`，无论 `campUUID` 是否正确、是否存在，都可以访问订阅管理页面
- `campUUID` 在 GET 请求中只是一个"占位符"，不参与任何权限校验

**2. 订阅偏好更新（POST）- 部分关联校验**

```go
// cmd/public.go:266-280
func (a *App) SubscriptionPrefs(c echo.Context) error {
    var (
        campUUID  = c.Param("campUUID")
        subUUID   = c.Param("subUUID")
        blocklist = a.cfg.Privacy.AllowBlocklist && req.Blocklist
    )
    
    // 简单退订或加入黑名单时
    if !req.Manage || blocklist {
        // 这里使用了 campUUID
        if err := a.core.UnsubscribeByCampaign(subUUID, campUUID, blocklist); err != nil {
            // ...
        }
        // ...
    }
    
    // 偏好管理时（manage=true）
    // ... 完全不使用 campUUID ...
}
```

**SQL 层面的关联校验**（简单退订时）：

```sql
-- queries/subscribers.sql:251-267
-- name: unsubscribe-by-campaign
WITH lists AS (
    SELECT list_id FROM campaign_lists
    LEFT JOIN campaigns ON (campaign_lists.campaign_id = campaigns.id)
    WHERE campaigns.uuid = $1  -- campUUID
),
sub AS (
    UPDATE subscribers SET status = (CASE WHEN $3 IS TRUE THEN 'blocklisted' ELSE status END)
    WHERE uuid = $2 RETURNING id  -- subUUID
)
UPDATE subscriber_lists SET status = 'unsubscribed', updated_at=NOW() WHERE
    subscriber_id = (SELECT id FROM sub) AND status != 'unsubscribed' AND
    -- 关键：如果不是 blocklist，只退订该 campaign 的列表
    CASE WHEN $3 IS FALSE THEN list_id = ANY(SELECT list_id FROM lists) ELSE list_id != 0 END;
```

**校验逻辑分析**：
1. `lists` CTE：根据 `campUUID` 查找该 Campaign 关联的所有列表
2. `sub` CTE：根据 `subUUID` 查找订阅者（blocklist 时更新状态）
3. 最终 UPDATE：
   - 如果是 **blocklist=true**：退订该订阅者的**所有列表**（不限制于该 Campaign）
   - 如果是 **blocklist=false**：只退订该订阅者在**该 Campaign 列表**中的订阅

**安全影响**：
- 简单退订时：`campUUID` 用于确定要退订哪些列表
- 但即使 `campUUID` 对应的 Campaign 不存在或不属于该订阅者，SQL 仍然会执行（只是 `lists` CTE 可能返回空，退订操作不生效）
- 偏好管理时（`manage=true`）：完全不使用 `campUUID`

**3. 邮件查看页面 - 无关联校验**

```go
// cmd/public.go:148-193
func (a *App) ViewCampaignMessage(c echo.Context) error {
    // 1. 验证 campaign 存在
    campUUID := c.Param("campUUID")
    camp, err := a.core.GetCampaign(0, campUUID, "")
    if err != nil {
        // 返回 404
    }

    // 2. 验证 subscriber 存在
    subUUID := c.Param("subUUID")
    sub, err := a.core.GetSubscriber(0, subUUID, "")
    if err != nil {
        // 返回 404
    }

    // 3. 使用两者渲染邮件 - 但没有验证该订阅者是否是该 Campaign 的受众！
    msg, err := a.manager.NewCampaignMessage(&camp, sub)
    // ...
}
```

**代码位置**：`cmd/public.go:148-193`

**安全影响**：
- 如果攻击者持有一个有效的 `subUUID`，可以尝试不同的 `campUUID`
- 只要两个 UUID 都存在，就可以查看发送给该订阅者的**任意邮件**
- 没有验证该订阅者是否真的是该 Campaign 的目标受众

#### 3.1.2 关联校验缺失的风险场景

**场景 1：链接泄露后的横向访问**

假设订阅者 A 的退订链接泄露：
- 链接：`/subscription/campaign-A-uuid/subscriber-A-uuid`

攻击者可以：
1. 访问 `/subscription/any-valid-campaign-uuid/subscriber-A-uuid` → **成功访问 A 的管理页面**
2. 访问 `/campaign/campaign-B-uuid/subscriber-A-uuid` → **成功查看发送给 A 的 Campaign B 邮件**（如果 B 存在）
3. 访问 `/subscription/optin/subscriber-A-uuid` → **查看 A 的未确认订阅**
4. POST 到 `/subscription/export/subscriber-A-uuid` → **导出 A 的所有数据**
5. POST 到 `/subscription/wipe/subscriber-A-uuid` → **删除 A 的所有数据**

**场景 2：campUUID 伪造**

攻击者持有 `subUUID: 11111111-1111-1111-1111-111111111111`

可以尝试：
```
GET /subscription/00000000-0000-0000-0000-000000000000/11111111-1111-1111-1111-111111111111
```

- 如果 `campUUID` 格式正确但不存在：
  - `hasUUID` 中间件通过（格式正确）
  - `hasSub` 中间件通过（subUUID 存在）
  - `SubscriptionPage` 成功执行（不使用 campUUID）
  - **结果：成功访问订阅管理页面**

- 如果 `campUUID` 格式错误：
  - `hasUUID` 中间件返回 400 Bad Request
  - **结果：拒绝访问**

### 3.2 访问鉴权机制

订阅者门户采用**中间件链（Middleware Chain）**的方式进行访问验证。

#### 3.2.1 路由定义与中间件链

```go
// cmd/handlers.go:271-279
// 订阅管理页面
g.GET("/subscription/:campUUID/:subUUID", noIndex(a.hasUUID(a.hasSub(a.SubscriptionPage), "campUUID", "subUUID")))
g.POST("/subscription/:campUUID/:subUUID", a.hasUUID(a.hasSub(a.SubscriptionPrefs), "campUUID", "subUUID"))

// 双重确认订阅
g.GET("/subscription/optin/:subUUID", noIndex(a.hasUUID(a.hasSub(a.OptinPage), "subUUID")))
g.POST("/subscription/optin/:subUUID", a.hasUUID(a.hasSub(a.OptinPage), "subUUID"))

// 数据操作
g.POST("/subscription/export/:subUUID", a.hasUUID(a.hasSub(a.SelfExportSubscriberData), "subUUID"))
g.POST("/subscription/wipe/:subUUID", a.hasUUID(a.hasSub(a.WipeSubscriberData), "subUUID"))

// 邮件查看
g.GET("/campaign/:campUUID/:subUUID", noIndex(a.hasUUID(a.ViewCampaignMessage, "campUUID", "subUUID")))
g.GET("/campaign/:campUUID/:subUUID/px.png", noIndex(a.hasUUID(a.RegisterCampaignView, "campUUID", "subUUID")))
```

#### 3.1.2 中间件详解

##### hasSub 中间件 - 订阅者存在性验证

```go
// cmd/handlers.go:386-403
// hasSub middleware checks if a subscriber exists given the UUID param in a request.
func (a *App) hasSub(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        subUUID := c.Param("subUUID")

        if _, err := a.core.GetSubscriber(0, subUUID, ""); err != nil {
            if er, ok := err.(*echo.HTTPError); ok && er.Code == http.StatusBadRequest {
                // 订阅者不存在，返回 404
                return c.Render(http.StatusNotFound, tplMessage,
                    makeMsgTpl(a.i18n.T("public.notFoundTitle"), "", er.Message.(string)))
            }
            // 其他错误，返回 500
            return c.Render(http.StatusInternalServerError, tplMessage,
                makeMsgTpl(a.i18n.T("public.errorTitle"), "", a.i18n.T("public.errorProcessingRequest")))
        }

        return next(c)
    }
}
```

**验证逻辑**：
1. 从 URL 参数获取 `subUUID`
2. 调用 `GetSubscriber` 查询数据库
3. 如果返回 `BadRequest`（表示不存在），返回友好的 404 页面
4. 如果存在，继续执行后续处理器

**安全意义**：
- 只有持有有效订阅者 UUID 的用户才能访问这些页面
- UUID 是随机生成的 GUID，暴力破解几乎不可能
- 错误信息模糊，不泄露"订阅者是否存在"的信息（都返回 404）

##### hasUUID 中间件 - UUID 格式验证

```go
// cmd/handlers.go:359-369
// hasUUID middleware validates the UUID string format for a given set of params.
func (a *App) hasUUID(next echo.HandlerFunc, params ...string) echo.HandlerFunc {
    return func(c echo.Context) error {
        for _, p := range params {
            if !reUUID.MatchString(c.Param(p)) {
                return c.Render(http.StatusBadRequest, tplMessage, makeMsgTpl(a.i18n.T("public.errorTitle"), "",
                    a.i18n.T("globals.messages.invalidUUID")))
            }
        }
        return next(c)
    }
}
```

**正则表达式**（定义在 `cmd/handlers.go:29`）：
```go
reUUID = regexp.MustCompile("^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$")
```

**安全意义**：
- 防止 SQL 注入和路径遍历攻击
- 在查询数据库之前就过滤掉无效格式的请求
- 减少数据库无效查询压力

##### noIndex 中间件 - 搜索引擎保护

```go
// cmd/handlers.go:406-411
// noIndex adds the HTTP header requesting robots to not crawl the page.
func noIndex(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        c.Response().Header().Set("X-Robots-Tag", "noindex")
        return next(c)
    }
}
```

**安全意义**：
- 防止订阅者管理页面被搜索引擎索引
- 保护订阅者隐私，避免个人链接被公开搜索到

### 3.2 隐私配置细粒度控制

订阅者自助门户的功能可用性由一系列隐私配置项精确控制。

#### 3.2.1 配置项一览

```go
// models/settings.go:31-41
PrivacyIndividualTracking bool     `json:"privacy.individual_tracking"`  // 个人追踪
PrivacyDisableTracking    bool     `json:"privacy.disable_tracking"`     // 完全禁用追踪
PrivacyUnsubHeader        bool     `json:"privacy.unsubscribe_header"`   // 退订邮件头
PrivacyAllowBlocklist     bool     `json:"privacy.allow_blocklist"`      // 允许加入黑名单
PrivacyAllowPreferences   bool     `json:"privacy.allow_preferences"`    // 允许管理偏好
PrivacyAllowExport        bool     `json:"privacy.allow_export"`         // 允许导出数据
PrivacyAllowWipe          bool     `json:"privacy.allow_wipe"`           // 允许删除数据
PrivacyExportable         []string `json:"privacy.exportable"`           // 可导出字段
PrivacyRecordOptinIP      bool     `json:"privacy.record_optin_ip"`      // 记录确认订阅 IP
```

#### 3.2.2 配置项如何影响功能

##### 偏好管理控制

```go
// cmd/public.go:227-245
// SubscriptionPage - 订阅管理页面
func (a *App) SubscriptionPage(c echo.Context) error {
    // ...
    out := unsubTpl{
        // ...
        AllowPreferences: a.cfg.Privacy.AllowPreferences,
        // ...
    }

    // 只有启用偏好管理时才显示管理界面
    if a.cfg.Privacy.AllowPreferences {
        out.ShowManage = showManage

        // 获取订阅者的列表
        subs, err := a.core.GetSubscriptions(0, subUUID, false)
        // ...
    }
    // ...
}
```

**模板中的条件渲染**（`static/public/templates/subscription.html:20-22`）：
```html
{{ if .Data.AllowPreferences }}
    <a href="?manage=true">{{ L.T "public.managePrefs" }}</a>
{{ end }}
```

##### 黑名单控制

```go
// cmd/public.go:266-280
// SubscriptionPrefs - 处理订阅偏好更新
func (a *App) SubscriptionPrefs(c echo.Context) error {
    var (
        // ...
        blocklist = a.cfg.Privacy.AllowBlocklist && req.Blocklist
    )
    if !req.Manage || blocklist {
        // 执行退订或加入黑名单
        if err := a.core.UnsubscribeByCampaign(subUUID, campUUID, blocklist); err != nil {
            // ...
        }
        // ...
    }
    // ...
}
```

**模板渲染**（`static/public/templates/subscription.html:8-14`）：
```html
{{ if .Data.AllowBlocklist }}
    <p>{{ L.T "public.unsubHelp" }}</p>
    <p>
        <input id="privacy-blocklist" type="checkbox" name="blocklist" value="true" />
        <label for="privacy-blocklist">{{ L.T "public.unsubFull" }}</label>
    </p>
{{ end }}
```

##### 数据导出/删除控制

```go
// cmd/public.go:598-649
// SelfExportSubscriberData - 导出订阅者数据
func (a *App) SelfExportSubscriberData(c echo.Context) error {
    // 检查是否允许导出
    if !a.cfg.Privacy.AllowExport {
        return c.Render(http.StatusBadRequest, tplMessage,
            makeMsgTpl(a.i18n.T("public.errorTitle"), "", a.i18n.Ts("public.invalidFeature")))
    }
    // ...
}

// cmd/public.go:654-670
// WipeSubscriberData - 删除订阅者数据
func (a *App) WipeSubscriberData(c echo.Context) error {
    // 检查是否允许删除
    if !a.cfg.Privacy.AllowWipe {
        return c.Render(http.StatusBadRequest, tplMessage,
            makeMsgTpl(a.i18n.T("public.errorTitle"), "", a.i18n.Ts("public.invalidFeature")))
    }
    // ...
}
```

**模板渲染**（`static/public/templates/subscription.html:64-89`）：
```html
{{ if or .Data.AllowExport .Data.AllowWipe }}
<form id="data-form" class="data-form" method="post" action="" onsubmit="return handleData()">
    <section>
        <h2>{{ L.T "public.privacyTitle" }}</h2>
        {{ if .Data.AllowExport }}
        <div class="row">
            <input id="privacy-export" type="radio" name="data-action" value="export" required />
            <label for="privacy-export"><strong>{{ L.T "public.privacyExport" }}</strong></label>
            <!-- ... -->
        </div>
        {{ end }}
        
        {{ if .Data.AllowWipe }}
        <div class="row">
            <input id="privacy-wipe" type="radio" name="data-action" value="wipe" required />
            <label for="privacy-wipe"><strong>{{ L.T "public.privacyWipe" }}</strong></label>
            <!-- ... -->
        </div>
        {{ end }}
    </section>
</form>
{{ end }}
```

### 3.3 私有列表过滤（内容裁剪）

订阅者管理页面会自动过滤掉**私有列表（Private Lists）**，订阅者只能看到和管理公开列表。

#### 3.3.1 显示层面过滤

```go
// cmd/public.go:236-244
// SubscriptionPage 中获取订阅列表时过滤私有列表
out.Subscriptions = make([]models.Subscription, 0, len(subs))
for _, s := range subs {
    // Private lists shouldn't be rendered in the template.
    if s.Type == models.ListTypePrivate {
        continue  // 跳过私有列表
    }
    out.Subscriptions = append(out.Subscriptions, s)
}
```

**常量定义**（`models/lists.go` 中）：
```go
const (
    ListTypePublic  = "public"   // 公开列表
    ListTypePrivate = "private"  // 私有列表
)
```

#### 3.3.2 更新层面过滤

在处理订阅偏好更新时，同样会忽略私有列表：

```go
// cmd/public.go:324-332
// SubscriptionPrefs 中处理列表更新
for _, s := range subs {
    if s.Type == models.ListTypePrivate {
        continue  // 私有列表不允许用户操作
    }
    if _, ok := reqUUIDs[s.UUID]; !ok {
        unsubUUIDs = append(unsubUUIDs, s.UUID)
    }
}
```

**安全意义**：
- 私有列表是管理员用于内部管理的，订阅者不应该知道其存在
- 即使订阅者通过某种方式获取了私有列表的 UUID，也无法通过前端操作
- 保护内部列表的隐私性

---

## 四、未登录访客 vs 已订阅用户：内容差异对比

### 4.1 访问权限矩阵

| 功能/页面 | 未登录访客 | 已订阅用户（有效 UUID） | 说明 |
|----------|-----------|----------------------|-----|
| `/archive` 存档列表 | ✅ 可访问（如启用） | ✅ 可访问 | 功能开关控制 |
| `/archive/:id` 存档详情 | ✅ 可访问（如启用） | ✅ 可访问 | 使用虚拟订阅者数据 |
| `/archive.xml` RSS | ✅ 可访问（如启用） | ✅ 可访问 | 内容显示可配置 |
| `/subscription/form` 订阅表单 | ✅ 可访问（如启用） | ✅ 可访问 | 功能开关控制 |
| `/subscription/:campUUID/:subUUID` 订阅管理 | ❌ 404 错误 | ✅ 可访问 | 需要有效 UUID |
| `/campaign/:campUUID/:subUUID` 邮件查看 | ❌ 404 错误 | ✅ 可访问 | 需要有效 UUID |
| `/subscription/export/:subUUID` 数据导出 | ❌ 404 错误 | ✅ 可访问（如配置允许） | 需要 UUID + 配置 |
| `/subscription/wipe/:subUUID` 数据删除 | ❌ 404 错误 | ✅ 可访问（如配置允许） | 需要 UUID + 配置 |

### 4.2 内容差异详细对比

#### 4.2.1 邮件内容渲染差异

**未登录访客（公开存档）**：
```go
// cmd/archive.go:252-262
// Load the dummy subscriber meta.
var sub models.Subscriber
if err := json.Unmarshal([]byte(camp.ArchiveMeta), &sub); err != nil {
    // ...
}

m := manager.CampaignMessage{
    Campaign:   &camp,
    Subscriber: sub,  // 虚拟数据
}
```

**已订阅用户（个人邮件查看）**：
```go
// cmd/public.go:146-192
// ViewCampaignMessage - 查看个人邮件
func (a *App) ViewCampaignMessage(c echo.Context) error {
    // ...
    // 获取真实订阅者数据
    subUUID := c.Param("subUUID")
    sub, err := a.core.GetSubscriber(0, subUUID, "")
    // ...
    
    // 使用真实订阅者数据渲染
    msg, err := a.manager.NewCampaignMessage(&camp, sub)
    // ...
}
```

**差异示例**：

假设邮件模板中有以下内容：
```html
<p>亲爱的 {{ .Subscriber.Name }}，</p>
<p>您的邮箱是：{{ .Subscriber.Email }}</p>
<p>点击查看详情：{{ TrackLink "https://example.com" }}</p>
```

**未登录访客看到的**（使用 ArchiveMeta 中的虚拟数据）：
```html
<p>亲爱的 Valued Subscriber，</p>
<p>您的邮箱是：subscriber@example.com</p>
<p>点击查看详情：<a href="/link/xxx/yyy/dummy-uuid">https://example.com</a></p>
```

**已订阅用户看到的**（真实数据）：
```html
<p>亲爱的 张三，</p>
<p>您的邮箱是：zhangsan@example.com</p>
<p>点击查看详情：<a href="/link/xxx/yyy/zhangsan-uuid">https://example.com</a></p>
```

#### 4.2.2 订阅管理页面差异

**未登录访客**：
- 访问 `/subscription/xxx/yyy` 会直接返回 404
- 无法看到任何订阅相关内容

**已订阅用户**：
- 可以看到个人退订选项
- 可以看到并管理订阅列表（仅公开列表）
- 可以更新个人姓名
- 可以看到数据导出/删除选项（如配置允许）

**页面状态示例**：

1. **默认视图（退订）**：
   ```
   退订
   
   您确定要退订吗？
   
   [ ] 完全停止接收所有邮件（加入黑名单）
   
   [确认退订]
   
   [管理订阅偏好]  ← 仅当 AllowPreferences=true 时显示
   ```

2. **偏好管理视图（`?manage=true`）**：
   ```
   管理订阅偏好
   
   姓名：[张三________]
   
   订阅列表：
   [x] 产品更新公告
   [x] 技术博客
   [x] 促销活动
   （注：私有列表不会出现在这里）
   
   [保存]
   ```

3. **隐私选项**：
   ```
   隐私选项
   
   ( ) 导出我的数据
       导出您的个人资料、订阅历史、邮件打开和点击记录
   
   ( ) 删除我的数据
       永久删除您的个人资料和所有相关数据
   
   [继续]
   ```

---

## 五、关键代码位置汇总

| 功能模块 | 文件路径 | 关键函数/行号 |
|---------|---------|--------------|
| **公开存档** | | |
| 功能开关定义 | `models/settings.go` | L14-15 |
| 路由条件注册 | `cmd/handlers.go` | L263, L281-286 |
| 存档列表 API | `cmd/archive.go` | L27-50 |
| 存档页面渲染 | `cmd/archive.go` | L99-116 |
| 单篇存档详情 | `cmd/archive.go` | L119-174 |
| 虚拟订阅者数据 | `cmd/archive.go` | L238-277 |
| 获取存档 Campaign | `internal/core/campaigns.go` | L72-85, L145-150 |
| SQL 查询 | `queries/campaigns.sql` | L99-108 |
| **订阅者门户** | | |
| 路由与中间件 | `cmd/handlers.go` | L271-279 |
| hasSub 中间件 | `cmd/handlers.go` | L386-403 |
| hasUUID 中间件 | `cmd/handlers.go` | L359-369 |
| noIndex 中间件 | `cmd/handlers.go` | L406-411 |
| 订阅管理页面 | `cmd/public.go` | L195-248 |
| 偏好更新处理 | `cmd/public.go` | L253-343 |
| 数据导出 | `cmd/public.go` | L594-649 |
| 数据删除 | `cmd/public.go` | L651-670 |
| 私有列表过滤 | `cmd/public.go` | L236-244, L324-332 |
| 隐私配置定义 | `models/settings.go` | L31-41 |
| **模板文件** | | |
| 存档页面模板 | `static/public/templates/archive.html` | 全文 |
| 订阅管理模板 | `static/public/templates/subscription.html` | 全文 |
| 订阅表单模板 | `static/public/templates/subscription-form.html` | 全文 |

---

## 六、安全设计总结

listmonk 的公开邮件存档和订阅者自助门户采用了**多层、纵深防御**的安全设计：

### 6.1 公开存档安全机制

1. **功能开关**：全局配置控制整个功能是否启用
2. **路由保护**：未启用时路由不注册，直接返回 404
3. **SQL 过滤**：查询条件严格限制（archive=true, type=regular, status 有效）
4. **业务校验**：获取单个存档时双重检查 Archive 标志
5. **数据脱敏**：使用虚拟订阅者数据渲染，保护真实用户隐私
6. **RSS 控制**：独立配置控制 RSS 内容显示粒度

### 6.2 订阅者门户安全机制

1. **UUID 验证**：基于 GUID 的访问令牌，暴力破解不可行
2. **中间件链**：格式验证 → 存在性验证 → 业务处理
3. **错误模糊**：无效 UUID 和不存在订阅者都返回 404，不泄露信息
4. **隐私配置**：细粒度控制每个功能的可用性
5. **内容裁剪**：自动过滤私有列表，保护内部管理信息
6. **搜索引擎保护**：noIndex 头防止个人页面被索引

### 6.3 最佳实践体现

- **最小权限原则**：每个功能都有独立的开关控制
- **纵深防御**：多层验证，一层被绕过还有下一层
- **隐私 by design**：从设计上考虑数据保护（虚拟订阅者、私有列表过滤）
- **错误处理安全**：不向攻击者泄露内部状态信息
- **配置驱动**：安全性可通过配置调整，无需代码修改

---

## 七、附录：配置项参考

### 7.1 公开存档相关配置

| 配置项 | 类型 | 默认值 | 说明 |
|-------|------|-------|-----|
| `app.enable_public_archive` | bool | false | 是否启用公开邮件存档功能 |
| `app.enable_public_archive_rss_content` | bool | false | RSS 中是否显示邮件完整内容 |

### 7.2 订阅者门户相关配置

| 配置项 | 类型 | 默认值 | 说明 |
|-------|------|-------|-----|
| `app.enable_public_subscription_page` | bool | false | 是否启用公开订阅表单 |
| `privacy.allow_blocklist` | bool | true | 是否允许订阅者加入黑名单 |
| `privacy.allow_preferences` | bool | true | 是否允许管理订阅偏好 |
| `privacy.allow_export` | bool | true | 是否允许导出个人数据 |
| `privacy.allow_wipe` | bool | true | 是否允许删除个人数据 |
| `privacy.exportable` | array | ["profile", "subscriptions", "campaign_views", "link_clicks"] | 可导出的数据类型 |
| `privacy.individual_tracking` | bool | true | 是否启用个人级别的追踪 |
| `privacy.disable_tracking` | bool | false | 是否完全禁用追踪 |

### 7.3 列表类型

| 类型 | 常量值 | 说明 |
|-----|-------|-----|
| 公开列表 | `"public"` | 订阅者可见、可管理 |
| 私有列表 | `"private"` | 订阅者不可见，仅管理员使用 |
