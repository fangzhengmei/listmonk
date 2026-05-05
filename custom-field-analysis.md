# Listmonk 自定义字段（Attributes）数据流分析报告

## 概述

本报告详细分析 listmonk 中订阅者自定义字段（Attributes）的存储结构、数据传输流程，以及与订阅表单的交互机制。特别关注了 Forms.vue 页面的表单生成链路、公共订阅页面的服务端渲染链路，并对比说明为什么它们都不属于真正的"自定义字段 schema 驱动动态渲染"。

---

## 核心发现

### ⚠️ 重要澄清

经过深入代码分析，发现 **listmonk 目前没有实现"独立的自定义字段 Schema 系统"**：

1. **不存在独立的字段定义存储** - 没有专门的表或配置来存储字段元数据（如字段类型、标签、验证规则、是否必填等）
2. **订阅表单不支持动态渲染自定义字段** - 公共订阅表单和管理后台表单都没有基于 schema 动态生成字段的机制
3. **`attribs` 是自由格式的 JSON 存储** - 订阅者属性以无 schema 约束的 JSONB 格式存储

---

## 一、后端存储结构

### 1.1 数据库 Schema

**文件**: `schema.sql:20-30`

```sql
CREATE TABLE subscribers (
    id              SERIAL PRIMARY KEY,
    uuid uuid       NOT NULL UNIQUE,
    email           TEXT NOT NULL UNIQUE,
    name            TEXT NOT NULL,
    attribs         JSONB NOT NULL DEFAULT '{}',  -- 自定义属性存储
    status          subscriber_status NOT NULL DEFAULT 'enabled',
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

**关键特性**:
- `attribs` 字段类型为 `JSONB`，支持 PostgreSQL 的 JSON 操作符
- 默认值为空对象 `'{}'`
- 可以通过 `->>` 操作符进行查询（如 `subscribers.attribs->>'city' = 'Bengaluru'`）

### 1.2 Go 数据模型

**文件**: `models/subscribers.go:28-37`

```go
type Subscriber struct {
    Base
    UUID    string         `db:"uuid" json:"uuid"`
    Email   string         `db:"email" json:"email" form:"email"`
    Name    string         `db:"name" json:"name" form:"name"`
    Attribs JSON           `db:"attribs" json:"attribs"`  -- 自定义属性
    Status  string         `db:"status" json:"status"`
    Lists   types.JSONText `db:"lists" json:"lists"`
}
```

**文件**: `models/common.go:112-134`

```go
-- JSON 是 map[string]any 的类型别名，用于处理数据库的 JSONB 字段
type JSON map[string]any

-- Value 实现 driver.Valuer 接口，将 JSON 序列化存入数据库
func (s JSON) Value() (driver.Value, error) {
    return json.Marshal(s)
}

-- Scan 实现 sql.Scanner 接口，从数据库反序列化 JSON
func (s JSON) Scan(b any) error {
    if b == nil {
        s = make(JSON)
        return nil
    }
    if data, ok := b.([]byte); ok {
        return json.Unmarshal(data, &s)
    }
    return fmt.Errorf("could not not decode type %T -> %T", b, s)
}
```

### 1.3 典型属性数据结构

**文件**: `docs/docs/content/concepts.md:11-23`

```json
{
  "city": "Bengaluru",
  "likes_tea": true,
  "spoken_languages": ["English", "Malayalam"],
  "projects": 3,
  "stack": {
    "frameworks": ["echo", "go"],
    "languages": ["go", "python"],
    "preferred_language": "go"
  }
}
```

**支持的数据类型**:
- 字符串 (`string`)
- 布尔值 (`boolean`)
- 数字 (`number`)
- 数组 (`array`)
- 嵌套对象 (`nested object`)

---

## 二、配置系统详解：从数据库到表单字段的完整映射

### 2.1 配置键的完整映射链

#### 2.1.1 数据库层：settings 表

配置存储在 `settings` 表中，使用点分隔的键名：

```sql
-- 示例配置键
app.root_url
app.enable_public_subscription_page
app.lang
security.captcha.altcha.enabled
security.captcha.altcha.complexity
security.captcha.hcaptcha.enabled
security.captcha.hcaptcha.key
privacy.allow_preferences
-- ... 更多配置
```

#### 2.1.2 后端加载：initSettings() → initConstConfig()

**文件**: `cmd/init.go:431-454` - `initSettings()`

```go
func initSettings(query string, db *sqlx.DB, ko *koanf.Koanf) {
    var s types.JSONText
    -- 从数据库 settings 表读取所有配置（JSONB 格式）
    if err := db.Get(&s, query); err != nil {
        lo.Fatalf("error reading settings from DB: %s", msg)
    }

    -- 将点分隔的键解嵌套为 map
    var out map[string]any
    if err := json.Unmarshal(s, &out); err != nil {
        lo.Fatalf("error unmarshalling settings from DB: %v", err)
    }
    
    -- 加载到 koanf 配置系统
    if err := ko.Load(confmap.Provider(out, "."), nil); err != nil {
        lo.Fatalf("error parsing settings from DB: %v", err)
    }
}
```

**文件**: `cmd/init.go:486-545` - `initConstConfig()`

```go
func initConstConfig(ko *koanf.Koanf) *Config {
    var c Config
    
    -- 从 koanf 解析配置到 Config 结构
    if err := ko.Unmarshal("app", &c); err != nil {
        lo.Fatalf("error loading app config: %v", err)
    }
    if err := ko.Unmarshal("privacy", &c.Privacy); err != nil {
        lo.Fatalf("error loading app.privacy config: %v", err)
    }
    if err := ko.Unmarshal("security", &c.Security); err != nil {
        lo.Fatalf("error loading app.security config: %v", err)
    }
    
    -- ... 其他配置解析
    
    return &c
}
```

#### 2.1.3 Config 结构定义

**文件**: `cmd/init.go:80-154`

```go
type Config struct {
    SiteName                      string   `koanf:"site_name"`
    FromEmail                     string   `koanf:"from_email"`
    NotifyEmails                  []string `koanf:"notify_emails"`
    EnablePublicSubPage           bool     `koanf:"enable_public_subscription_page"`
    EnablePublicArchive           bool     `koanf:"enable_public_archive"`
    EnablePublicArchiveRSSContent bool     `koanf:"enable_public_archive_rss_content"`
    Lang                          string   `koanf:"lang"`
    
    Privacy struct {
        IndividualTracking bool            `koanf:"individual_tracking"`
        DisableTracking    bool            `koanf:"disable_tracking"`
        AllowPreferences   bool            `koanf:"allow_preferences"`
        AllowBlocklist     bool            `koanf:"allow_blocklist"`
        AllowExport        bool            `koanf:"allow_export"`
        AllowWipe          bool            `koanf:"allow_wipe"`
        RecordOptinIP      bool            `koanf:"record_optin_ip"`
        UnsubHeader        bool            `koanf:"unsubscribe_header"`
        Exportable         map[string]bool `koanf:"-"`
    } `koanf:"privacy"`
    
    Security struct {
        OIDC struct {
            Enabled           bool   `koanf:"enabled"`
            ProviderURL       string `koanf:"provider_url"`
            ProviderName      string `koanf:"provider_name"`
            ClientID          string `koanf:"client_id"`
            ClientSecret      string `koanf:"client_secret"`
            AutoCreateUsers   bool   `koanf:"auto_create_users"`
            DefaultUserRoleID int    `koanf:"default_user_role_id"`
            DefaultListRoleID int    `koanf:"default_list_role_id"`
        } `koanf:"oidc"`

        Captcha struct {
            Altcha struct {
                Enabled    bool `koanf:"enabled"`
                Complexity int  `koanf:"complexity"`
            } `koanf:"altcha"`
            HCaptcha struct {
                Enabled bool   `koanf:"enabled"`
                Key     string `koanf:"key"`
                Secret  string `koanf:"secret"`
            } `koanf:"hcaptcha"`
        } `koanf:"captcha"`
        
        CorsOrigins []string `koanf:"cors_origins"`
    } `koanf:"security"`
    
    -- ... 其他配置
}
```

#### 2.1.4 配置键映射表

| 数据库 settings 表键名 | koanf 路径 | Config 结构字段 |
|------------------------|-----------|-----------------|
| `app.enable_public_subscription_page` | `app.enable_public_subscription_page` | `Config.EnablePublicSubPage` |
| `app.root_url` | `app.root_url` | `UrlConfig.RootURL` |
| `security.captcha.altcha.enabled` | `security.captcha.altcha.enabled` | `Config.Security.Captcha.Altcha.Enabled` |
| `security.captcha.altcha.complexity` | `security.captcha.altcha.complexity` | `Config.Security.Captcha.Altcha.Complexity` |
| `security.captcha.hcaptcha.enabled` | `security.captcha.hcaptcha.enabled` | `Config.Security.Captcha.HCaptcha.Enabled` |
| `security.captcha.hcaptcha.key` | `security.captcha.hcaptcha.key` | `Config.Security.Captcha.HCaptcha.Key` |
| `privacy.allow_preferences` | `privacy.allow_preferences` | `Config.Privacy.AllowPreferences` |

---

## 三、公共订阅页面的服务端渲染链路

### 3.1 路由配置

**文件**: `cmd/handlers.go:269-270`

```go
g.GET("/subscription/form", a.SubscriptionFormPage)   -- GET：渲染表单页面
g.POST("/subscription/form", a.SubscriptionForm)        -- POST：处理表单提交
```

### 3.2 GET 请求：SubscriptionFormPage - 渲染订阅表单

**文件**: `cmd/public.go:411-448`

```go
func (a *App) SubscriptionFormPage(c echo.Context) error {
    -- 检查公共订阅页面是否启用
    if !a.cfg.EnablePublicSubPage {
        return c.Render(http.StatusNotFound, tplMessage,
            makeMsgTpl(a.i18n.T("public.errorTitle"), "", a.i18n.Ts("public.invalidFeature")))
    }

    -- 获取所有公共列表（用于在表单中显示）
    lists, err := a.core.GetLists(models.ListTypePublic, models.ListStatusActive, true, nil)
    if err != nil {
        return c.Render(http.StatusInternalServerError, tplMessage,
            makeMsgTpl(a.i18n.T("public.errorTitle"), "", a.i18n.Ts("public.errorFetchingLists")))
    }

    -- 检查是否有可用的公共列表
    if len(lists) == 0 {
        return c.Render(http.StatusInternalServerError, tplMessage,
            makeMsgTpl(a.i18n.T("public.errorTitle"), "", a.i18n.Ts("public.noListsAvailable")))
    }

    -- 准备模板数据
    out := subFormTpl{}
    out.Title = a.i18n.T("public.sub")
    out.Lists = lists

    -- 验证码配置（用于模板渲染）
    if a.cfg.Security.Captcha.Altcha.Enabled {
        out.Captcha.Enabled = true
        out.Captcha.Provider = "altcha"
        out.Captcha.Complexity = a.cfg.Security.Captcha.Altcha.Complexity
    } else if a.cfg.Security.Captcha.HCaptcha.Enabled {
        out.Captcha.Enabled = true
        out.Captcha.Provider = "hcaptcha"
        out.Captcha.Key = a.cfg.Security.Captcha.HCaptcha.Key
    }

    -- 渲染 subscription-form 模板
    return c.Render(http.StatusOK, "subscription-form", out)
}
```

### 3.3 模板数据结构

**文件**: `cmd/public.go:90-99`

```go
type subFormTpl struct {
    publicTpl
    Lists   []models.List
    Captcha struct {
        Enabled    bool
        Provider   string
        Key        string
        Complexity int
    }
}
```

### 3.4 公共订阅表单模板

**文件**: `static/public/templates/subscription-form.html`

```html
<form method="post" action="" class="form">
  <div>
    <p>
      <label for="email">{{ L.T "subscribers.email" }}</label>
      <input id="email" name="email" required="true" type="email"
             placeholder='{{ L.T "subscribers.email" }}'
             value="" />
    </p>
    <p>
      <label for="name">{{ L.T "public.subName" }}</label>
      <input id="name" name="name" type="text"
             placeholder='{{ L.T "public.subName" }}'
             value="" />
    </p>

    -- 邮件列表选择（动态来自 Lists 数据）
    <ul class="lists">
      {{ range $i, $l := .Data.Lists }}
        <li>
          <input checked="true" id="l-{{ $l.UUID}}" type="checkbox" name="l" value="{{ $l.UUID }}" >
          <label for="l-{{ $l.UUID}}">{{ $l.Name }}</label>
        </li>
      {{ end }}
    </ul>

    -- 验证码（条件性渲染，基于 Captcha.Enabled）
    {{ if .Data.Captcha.Enabled }}
      <div class="captcha">
        {{ if eq .Data.Captcha.Provider "altcha" }}
          <altcha-widget challengeurl="{{ RootURL }}/api/public/captcha/altcha"
                         complexity="{{ .Data.Captcha.Complexity }}"></altcha-widget>
          <script type="module" src="{{ RootURL }}/public/static/altcha.umd.js" async defer></script>
        {{ else }}
          <div class="h-captcha" data-sitekey="{{ .Data.Captcha.Key }}"></div>
          <script src="https://js.hcaptcha.com/1/api.js" async defer></script>
        {{ end }}
      </div>
    {{ end }}

    <p>
      <button type="submit" class="button">{{ L.T "public.sub" }}</button>
    </p>
  </div>
</form>
```

### 3.5 服务端渲染数据流图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│              公共订阅页面服务端渲染数据流                                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  GET /subscription/form                                                          │
│         │                                                                        │
│         ▼                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ SubscriptionFormPage(c echo.Context)                                    │  │
│  │ ┌─────────────────────────────────────────────────────────────────────┐ │  │
│  │ │ 1. 检查 a.cfg.EnablePublicSubPage 是否为 true                        │ │  │
│  │ │    └─→ 从 Config 结构读取（从数据库 settings 表加载）                  │ │  │
│  │ │                                                                       │ │  │
│  │ │ 2. 获取公共列表：a.core.GetLists(ListTypePublic, ...)                │ │  │
│  │ │    └─→ 查询 lists 表，条件 type = 'public'                           │ │  │
│  │ │                                                                       │ │  │
│  │ │ 3. 准备模板数据 subFormTpl{}                                          │ │  │
│  │ │    ├─→ out.Lists = 公共列表数据                                        │ │  │
│  │ │    └─→ out.Captcha = 验证码配置（从 a.cfg.Security.Captcha 读取）     │ │  │
│  │ │                                                                       │ │  │
│  │ │ 4. 渲染模板：c.Render(http.StatusOK, "subscription-form", out)        │ │  │
│  │ └─────────────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│         │                                                                        │
│         ▼                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ 模板渲染：subscription-form.html                                          │  │
│  │ ┌─────────────────────────────────────────────────────────────────────┐ │  │
│  │ │ 硬编码字段：                                                          │ │  │
│  │ │ - email: <input type="email" name="email">                          │ │  │
│  │ │ - name:  <input type="text" name="name">                            │ │  │
│  │ │                                                                       │ │  │
│  │ │ 动态字段（基于数据）：                                                 │ │  │
│  │ │ - 列表 checkbox：循环 .Data.Lists                                     │ │  │
│  │ │ - 验证码：条件性渲染（基于 .Data.Captcha.Enabled）                     │ │  │
│  │ │                                                                       │ │  │
│  │ │ ⚠️ 没有自定义字段！                                                     │ │  │
│  │ └─────────────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 四、订阅表单的两个入口：HTML 表单 vs 公共 API

### 4.1 入口概述

listmonk 提供了两个不同的订阅入口，它们在校验步骤上有差异，但最终汇合到同一个处理函数。

| 入口 | 路由 | 处理函数 | 适用场景 |
|------|------|---------|---------|
| **HTML 表单入口** | `POST /subscription/form` | `SubscriptionForm()` | 用户通过浏览器填写表单提交 |
| **公共 API 入口** | `POST /api/public/subscription` | `PublicSubscription()` | 第三方系统通过 API 调用 |

### 4.2 入口 1：HTML 表单提交 - SubscriptionForm

**文件**: `cmd/public.go:450-512`

```go
func (a *App) SubscriptionForm(c echo.Context) error {
    -- 检查公共订阅页面是否启用
    if !a.cfg.EnablePublicSubPage {
        return echo.NewHTTPError(http.StatusNotFound, a.i18n.T("public.invalidFeature"))
    }

    -- ⚠️ 非空值检查：反机器人措施
    -- hidden 字段 nonce 应该为空（正常用户不会填写）
    -- 如果有值，说明是机器人自动填充
    if c.FormValue("nonce") != "" {
        return echo.NewHTTPError(http.StatusBadGateway, a.i18n.T("public.invalidFeature"))
    }

    -- ⚠️ 验证码验证（仅 HTML 表单入口需要）
    if a.captcha.IsEnabled() {
        var val string

        -- 根据验证码提供商获取相应的响应字段
        switch a.captcha.GetProvider() {
        case captcha.ProviderHCaptcha:
            val = c.FormValue("h-captcha-response")
        case captcha.ProviderAltcha:
            val = c.FormValue("altcha")
        default:
            return c.Render(http.StatusBadRequest, tplMessage,
                makeMsgTpl(a.i18n.T("public.errorTitle"), "", a.i18n.T("public.invalidCaptcha")))
        }

        -- 检查验证码响应是否为空
        if val == "" {
            return c.Render(http.StatusBadRequest, tplMessage,
                makeMsgTpl(a.i18n.T("public.errorTitle"), "", a.i18n.T("public.invalidCaptcha")))
        }

        -- 验证验证码
        err, ok := a.captcha.Verify(val)
        if err != nil {
            a.log.Printf("captcha request failed: %v", err)
        }

        if !ok {
            return c.Render(http.StatusBadRequest, tplMessage,
                makeMsgTpl(a.i18n.T("public.errorTitle"), "", a.i18n.T("public.invalidCaptcha")))
        }
    }

    -- 汇合点：调用公共处理函数
    hasOptin, err := a.processSubForm(c)
    if err != nil {
        e, ok := err.(*echo.HTTPError)
        if !ok {
            return err
        }
        return c.Render(e.Code, tplMessage, makeMsgTpl(a.i18n.T("public.errorTitle"), "", fmt.Sprintf("%s", e.Message)))
    }

    -- 渲染 HTML 响应页面
    msg := "public.subConfirmed"
    if hasOptin {
        msg = "public.subOptinPending"
    }
    return c.Render(http.StatusOK, tplMessage, makeMsgTpl(a.i18n.T("public.subTitle"), "", a.i18n.Ts(msg)))
}
```

### 4.3 入口 2：公共 API - PublicSubscription

**文件**: `cmd/public.go:514-529`

```go
func (a *App) PublicSubscription(c echo.Context) error {
    -- 检查公共订阅页面是否启用
    if !a.cfg.EnablePublicSubPage {
        return echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("public.invalidFeature"))
    }

    -- ⚠️ 没有验证码验证
    -- ⚠️ 没有 nonce 反机器人检查
    
    -- 直接调用公共处理函数
    hasOptin, err := a.processSubForm(c)
    if err != nil {
        return err
    }

    -- 返回 JSON 响应
    return c.JSON(http.StatusOK, okResp{struct {
        HasOptin bool `json:"has_optin"`
    }{hasOptin}})
}
```

### 4.4 汇合点：processSubForm

**文件**: `cmd/public.go:703-788`

```go
func (a *App) processSubForm(c echo.Context) (bool, error) {
    -- 绑定请求参数（支持 form 和 JSON）
    var req struct {
        Name          string   `form:"name" json:"name"`
        Email         string   `form:"email" json:"email"`
        FormListUUIDs []string `form:"l" json:"list_uuids"`
    }
    if err := c.Bind(&req); err != nil {
        return false, err
    }

    -- 验证：至少选择一个列表
    if len(req.FormListUUIDs) == 0 {
        return false, echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("public.noListsSelected"))
    }

    -- 验证：邮箱长度
    if len(req.Email) > 1000 {
        return false, echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("subscribers.invalidEmail"))
    }

    -- 验证：邮箱格式
    em, err := a.importer.SanitizeEmail(req.Email)
    if err != nil {
        return false, echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }
    req.Email = em

    -- 验证：姓名格式
    req.Name = strings.TrimSpace(req.Name)
    if len(req.Name) == 0 {
        -- 如果没有姓名，使用邮箱前缀
        req.Name = strings.Split(req.Email, "@")[0]
    } else if len(req.Name) > stdInputMaxLen {
        return false, echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("subscribers.invalidName"))
    }

    listUUIDs := pq.StringArray(req.FormListUUIDs)

    -- 验证：列表类型（确保不是私有列表）
    listTypes, err := a.core.GetListTypes(nil, req.FormListUUIDs)
    if err != nil {
        return false, echo.NewHTTPError(http.StatusInternalServerError, fmt.Sprintf("%s", err.(*echo.HTTPError).Message))
    }

    for _, t := range listTypes {
        if t == models.ListTypePrivate {
            return false, echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("globals.messages.invalidUUID"))
        }
    }

    -- ⚠️ 插入订阅者（注意：没有 attribs 字段！）
    _, hasOptin, err := a.core.InsertSubscriber(models.Subscriber{
        Name:   req.Name,
        Email:  req.Email,
        Status: models.SubscriberStatusEnabled,
        -- 没有设置 Attribs！
    }, nil, listUUIDs, false, true)
    
    -- ... 后续处理（更新已有订阅者等）
    
    return hasOptin, nil
}
```

### 4.5 两个入口的差异对比表

| 校验步骤 | HTML 表单入口 (`SubscriptionForm`) | 公共 API 入口 (`PublicSubscription`) |
|---------|------------------------------------|--------------------------------------|
| **启用检查** | ✅ 检查 `EnablePublicSubPage` | ✅ 检查 `EnablePublicSubPage` |
| **Nonce 反机器人** | ✅ 检查 `nonce` 字段是否为空 | ❌ 无此检查 |
| **验证码验证** | ✅ 验证 hcaptcha 或 altcha | ❌ 无此验证 |
| **请求格式** | 仅 `application/x-www-form-urlencoded` | 支持 `application/json` 和 `form` |
| **响应格式** | HTML 渲染页面 | JSON 响应 |
| **最终处理** | `processSubForm()` | `processSubForm()` |

### 4.6 完整数据流与汇合点图示

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    订阅表单两个入口的数据流与汇合点                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                        入口 1：HTML 表单提交                              │  │
│  │  POST /subscription/form                                                 │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐ │  │
│  │  │ SubscriptionForm(c echo.Context)                                     │ │  │
│  │  │                                                                       │ │  │
│  │  │ 1. 检查 EnablePublicSubPage                                          │ │  │
│  │  │ 2. ⚠️ Nonce 反机器人检查：c.FormValue("nonce") != "" → 拒绝           │ │  │
│  │  │ 3. ⚠️ 验证码验证：                                                    │ │  │
│  │  │    - a.captcha.IsEnabled() → true                                    │ │  │
│  │  │    - 获取 h-captcha-response 或 altcha                               │ │  │
│  │  │    - a.captcha.Verify(val) → false → 拒绝                            │ │  │
│  │  │ 4. 调用 processSubForm(c) → 汇合点                                    │ │  │
│  │  │ 5. 渲染 HTML 响应页面                                                  │ │  │
│  │  └─────────────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                         入口 2：公共 API 调用                              │  │
│  │  POST /api/public/subscription                                          │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐ │  │
│  │  │ PublicSubscription(c echo.Context)                                   │ │  │
│  │  │                                                                       │ │  │
│  │  │ 1. 检查 EnablePublicSubPage                                          │ │  │
│  │  │ 2. ⚠️ 没有 Nonce 检查                                                 │ │  │
│  │  │ 3. ⚠️ 没有验证码验证                                                  │ │  │
│  │  │ 4. 直接调用 processSubForm(c) → 汇合点                                │ │  │
│  │  │ 5. 返回 JSON 响应：{"has_optin": bool}                               │ │  │
│  │  └─────────────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│                                    │                                              │
│                                    ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │                         汇合点：processSubForm                                │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐ │  │
│  │  │ 1. 绑定请求参数：name, email, list_uuids (支持 form 和 JSON)            │ │  │
│  │  │ 2. 验证：至少选择一个列表                                                 │ │  │
│  │  │ 3. 验证：邮箱长度和格式                                                  │ │  │
│  │  │ 4. 验证：姓名长度                                                        │ │  │
│  │  │ 5. 验证：列表类型（确保不是私有列表）                                     │ │  │
│  │  │ 6. ⚠️ 调用 core.InsertSubscriber()                                        │ │  │
│  │  │    ┌─→ 创建 models.Subscriber{Name, Email, Status}                      │ │  │
│  │  │    └─→ ⚠️ 没有设置 Attribs！                                            │ │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │                              最终存储：数据库                                  │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐ │  │
│  │  │ subscribers 表：                                                         │ │  │
│  │  │ - email, name, status → 正确设置                                        │ │  │
│  │  │ - attribs → 默认值 '{}'（空对象）                                        │ │  │
│  │  │                                                                         │ │  │
│  │  │ ⚠️ 公共订阅无法设置自定义属性！                                           │ │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 五、Forms.vue 表单生成链路分析

### 5.1 功能概述

**Forms.vue** (`frontend/src/views/Forms.vue`) 是管理后台的一个页面，用于生成**可嵌入的 HTML 订阅表单代码**。它的主要功能是：

1. 显示所有 `type === 'public'` 的邮件列表
2. 让用户勾选需要包含在表单中的列表
3. 根据选择动态生成 HTML 表单代码
4. 用户可以复制这些代码嵌入到外部网站

### 5.2 数据流详解

#### 步骤 1：配置从数据库加载到内存

**文件**: `cmd/init.go:431-454`

```go
func initSettings(query string, db *sqlx.DB, ko *koanf.Koanf) {
    var s types.JSONText
    -- 从数据库 settings 表读取所有配置
    if err := db.Get(&s, query); err != nil {
        lo.Fatalf("error reading settings from DB: %s", msg)
    }

    -- 将点分隔的键解嵌套为 map
    var out map[string]any
    if err := json.Unmarshal(s, &out); err != nil {
        lo.Fatalf("error unmarshalling settings from DB: %v", err)
    }
    
    -- 加载到 koanf 配置系统
    if err := ko.Load(confmap.Provider(out, "."), nil); err != nil {
        lo.Fatalf("error parsing settings from DB: %v", err)
    }
}
```

#### 步骤 2：配置通过 API 下发到前端

**文件**: `cmd/admin.go:41-91`

```go
type serverConfig struct {
    RootURL            string `json:"root_url"`
    FromEmail          string `json:"from_email"`
    PublicSubscription struct {
        Enabled          bool        `json:"enabled"`
        CaptchaEnabled   bool        `json:"captcha_enabled"`
        CaptchaProvider  null.String `json:"captcha_provider"`
        CaptchaKey       null.String `json:"captcha_key"`
        AltchaComplexity int         `json:"altcha_complexity"`
    } `json:"public_subscription"`
    -- ... 其他配置字段
}

-- GET /api/config 端点
func (a *App) GetServerConfig(c echo.Context) error {
    out := serverConfig{
        RootURL:       a.urlCfg.RootURL,
        FromEmail:     a.cfg.FromEmail,
        -- ...
    }
    out.PublicSubscription.Enabled = a.cfg.EnablePublicSubPage

    -- 验证码配置
    if a.cfg.Security.Captcha.Altcha.Enabled {
        out.PublicSubscription.CaptchaEnabled = true
        out.PublicSubscription.CaptchaProvider = null.StringFrom(captcha.ProviderAltcha)
        out.PublicSubscription.AltchaComplexity = a.cfg.Security.Captcha.Altcha.Complexity
    } else if a.cfg.Security.Captcha.HCaptcha.Enabled {
        out.PublicSubscription.CaptchaEnabled = true
        out.PublicSubscription.CaptchaProvider = null.StringFrom(captcha.ProviderHCaptcha)
        out.PublicSubscription.CaptchaKey = null.StringFrom(a.cfg.Security.Captcha.HCaptcha.Key)
    }

    -- ... 其他配置处理

    return c.JSON(http.StatusOK, okResp{out})
}
```

#### 步骤 3：前端获取并存储配置

**文件**: `frontend/src/api/index.js:418-421`

```javascript
-- ⚠️ 正确的函数名：getServerConfig，不是 getConfig
export const getServerConfig = async () => http.get(
  '/api/config',
  { loading: models.serverConfig, store: models.serverConfig, camelCase: false },
);
```

**文件**: `frontend/src/store/index.js`

```javascript
export default new Vuex.Store({
  state: {
    -- 所有模型数据，包括 serverConfig
    ...Object.keys(models).reduce((obj, cur) => ({ ...obj, [cur]: [] }), {}),
    
    loading: Object.keys(models).reduce((obj, cur) => ({ ...obj, [cur]: false }), {}),
  },

  mutations: {
    setModelResponse(state, { model, data }) {
      state[model] = data;
    },
    -- ...
  },
  -- ...
});
```

#### 步骤 4：Forms.vue 渲染表单 HTML

**文件**: `frontend/src/views/Forms.vue:69-111`

```javascript
methods: {
  renderHTML() {
    let h = `<form method="post" action="${this.serverConfig.root_url}/subscription/form" class="listmonk-form">\n`
      + '  <div>\n'
      + `    <h3>${this.$t('public.sub')}</h3>\n`
      + '    <input type="hidden" name="nonce" />\n\n'
      + `    <p><input type="email" name="email" required placeholder="${this.$t('subscribers.email')}" /></p>\n`
      + `    <p><input type="text" name="name" placeholder="${this.$t('public.subName')}" /></p>\n\n`;

    -- 动态生成列表 checkbox（基于用户勾选的 publicLists）
    this.checked.forEach((i) => {
      const l = this.publicLists[parseInt(i, 10)];

      h += '    <p>\n'
        + `      <input id="${l.uuid.substr(0, 5)}" type="checkbox" name="l" checked value="${l.uuid}" />\n`
        + `      <label for="${l.uuid.substr(0, 5)}">${l.name}</label>\n`;

      if (l.description) {
        h += '      <br />\n'
          + `      <span>${l.description}</span>\n`;
      }

      h += '    </p>\n';
    });

    -- 条件性添加验证码（基于 serverConfig 配置）
    if (this.serverConfig.public_subscription.captcha_enabled) {
      if (this.serverConfig.public_subscription.captcha_provider === 'altcha') {
        h += '\n'
          + `    <altcha-widget challengeurl="${this.serverConfig.root_url}/api/public/captcha/altcha"></altcha-widget>\n`
          + `    <${'script'} type="module" src="${this.serverConfig.root_url}/public/static/altcha.umd.js" async defer></${'script'}>\n`;
      } else if (this.serverConfig.public_subscription.captcha_provider === 'hcaptcha') {
        h += '\n'
          + `    <div class="h-captcha" data-sitekey="${this.serverConfig.public_subscription.captcha_key}"></div>\n`
          + `    <${'script'} src="https://js.hcaptcha.com/1/api.js" async defer></${'script'}>\n`;
      }
    }

    h += '\n'
      + `    <input type="submit" value="${this.$t('public.sub')} " />\n`
      + '  </div>\n'
      + '</form>';

    this.html = h;
  },
},
```

### 5.3 配置键从数据库到前端的完整映射

| 数据库 settings 表 | koanf 路径 | Config 结构 | serverConfig 响应 | 前端 Vuex |
|-------------------|-----------|------------|------------------|----------|
| `app.root_url` | `app.root_url` | `UrlConfig.RootURL` | `root_url` | `serverConfig.root_url` |
| `app.enable_public_subscription_page` | `app.enable_public_subscription_page` | `Config.EnablePublicSubPage` | `public_subscription.enabled` | `serverConfig.public_subscription.enabled` |
| `security.captcha.altcha.enabled` | `security.captcha.altcha.enabled` | `Config.Security.Captcha.Altcha.Enabled` | `public_subscription.captcha_enabled` | `serverConfig.public_subscription.captcha_enabled` |
| `security.captcha.altcha.complexity` | `security.captcha.altcha.complexity` | `Config.Security.Captcha.Altcha.Complexity` | `public_subscription.altcha_complexity` | `serverConfig.public_subscription.altcha_complexity` |
| `security.captcha.hcaptcha.enabled` | `security.captcha.hcaptcha.enabled` | `Config.Security.Captcha.HCaptcha.Enabled` | `public_subscription.captcha_enabled` | `serverConfig.public_subscription.captcha_enabled` |
| `security.captcha.hcaptcha.key` | `security.captcha.hcaptcha.key` | `Config.Security.Captcha.HCaptcha.Key` | `public_subscription.captcha_key` | `serverConfig.public_subscription.captcha_key` |

### 5.4 Forms.vue 完整数据流向图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    Forms.vue 表单生成数据流                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                      存储层：数据库                                         │  │
│  │  ┌──────────────┐              ┌──────────────┐                          │  │
│  │  │  settings 表  │              │  lists 表    │                          │  │
│  │  │ (JSONB存储)   │              │ (邮件列表)    │                          │  │
│  │  │ - root_url   │              │ - id         │                          │  │
│  │  │ - enable_... │              │ - uuid       │                          │  │
│  │  │ - captcha_*  │              │ - name       │                          │  │
│  │  └──────┬───────┘              │ - type       │                          │  │
│  │         │                        │ - description│                          │  │
│  │         │                        └──────┬───────┘                          │  │
│  │         │                               │                                  │  │
│  └─────────┼───────────────────────────────┼──────────────────────────────────┘  │
│            │                               │                                     │
│            ▼                               ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                      后端：配置加载与 API                                   │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │ initSettings() - 从数据库加载配置到 koanf                              │  │  │
│  │  │ initConstConfig() - 解析配置到 Config 结构                            │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  │                                    │                                         │  │
│  │                                    ▼                                         │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │ GetServerConfig() - GET /api/config                                    │  │  │
│  │  │  返回:                                                                  │  │  │
│  │  │  - root_url                                                            │  │  │
│  │  │  - public_subscription.enabled                                         │  │  │
│  │  │  - public_subscription.captcha_enabled                                 │  │  │
│  │  │  - public_subscription.captcha_provider                                │  │  │
│  │  │  - public_subscription.captcha_key                                     │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                    │                                                 │
│                                    ▼                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │                      前端：Vuex 状态管理                                      │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │ ⚠️ getServerConfig() - 调用 /api/config (不是 getConfig)              │  │  │
│  │  │ getLists() - 调用 /api/lists                                          │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  │                                    │                                         │  │
│  │                                    ▼                                         │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │ setModelResponse() - 存储到 Vuex store                                 │  │  │
│  │  │  state.serverConfig = {...}                                            │  │  │
│  │  │  state.lists = {results: [...]}                                        │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                    │                                                 │
│                                    ▼                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │                      Forms.vue：表单生成                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │ computed: {                                                           │  │  │
│  │  │   publicLists() {                                                      │  │  │
│  │  │     return this.lists.results.filter(l => l.type === 'public')        │  │  │
│  │  │   }                                                                     │  │  │
│  │  │ }                                                                       │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  │                                    │                                         │  │
│  │                                    ▼                                         │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │ watch: {                                                              │  │  │
│  │  │   checked() {                                                          │  │  │
│  │  │     this.renderHTML()  // 用户勾选列表变化时重新生成                     │  │  │
│  │  │   }                                                                     │  │  │
│  │  │ }                                                                       │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  │                                    │                                         │  │
│  │                                    ▼                                         │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │ renderHTML() - 硬编码生成 HTML 字符串                                  │  │  │
│  │  │                                                                         │  │  │
│  │  │  固定字段（硬编码）：                                                    │  │  │
│  │  │  - <input type="email" name="email">                                  │  │  │
│  │  │  - <input type="text" name="name">                                    │  │  │
│  │  │                                                                         │  │  │
│  │  │  动态部分（基于配置和选择）：                                             │  │  │
│  │  │  - 列表 checkbox：循环 publicLists                                     │  │  │
│  │  │  - 验证码：基于 serverConfig 条件添加                                   │  │  │
│  │  │                                                                         │  │  │
│  │  │  ⚠️ 没有自定义字段渲染！                                                 │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 六、为什么这些链路都不属于自定义字段 Schema 动态渲染

### 6.1 核心区别对比

| 维度 | Forms.vue 表单生成 | 公共订阅页面（服务端渲染） | 自定义字段 Schema 动态渲染 |
|------|-------------------|--------------------------|---------------------------|
| **设计目标** | 生成可嵌入的 HTML 表单代码 | 渲染订阅页面供用户填写 | 定义和渲染自定义数据字段 |
| **数据来源** | 邮件列表配置 + 服务器全局配置 | 邮件列表配置 + 服务器全局配置 | 字段定义 Schema |
| **字段定义** | 硬编码（email, name, lists, captcha） | 硬编码（email, name, lists, captcha） | 动态从 schema 读取 |
| **字段类型** | 固定类型（email, text, checkbox） | 固定类型（email, text, checkbox） | 支持多种类型（string, number, select, radio, date 等） |
| **元数据存储** | 无独立存储，直接使用现有配置 | 无独立存储，直接使用现有配置 | 需要独立的字段定义表 |
| **数据验证** | 无字段级别验证（仅 HTML 属性） | 后端有基本验证，但无 schema 验证 | 基于 schema 的类型/规则验证 |
| **与 attribs 关联** | ❌ 完全无关 | ❌ 完全无关 | ✅ 直接映射到 attribs 字段 |

### 6.2 详细分析

#### 6.2.1 没有独立的字段定义存储

**当前系统使用的数据来源**：
1. **`lists` 表** - 邮件列表数据，用于生成 checkbox
2. **`settings` 表** - 全局配置，如 `root_url`、`captcha_enabled` 等

**真正的自定义字段 Schema 系统需要**：

```sql
-- 需要独立的字段定义表
CREATE TABLE subscriber_fields (
    id          SERIAL PRIMARY KEY,
    key         VARCHAR(100) NOT NULL UNIQUE,  -- 字段键名（如 "city"）
    name        VARCHAR(200) NOT NULL,          -- 显示名称（如 "城市"）
    type        VARCHAR(50) NOT NULL,           -- 类型: string, number, boolean, select, date
    required    BOOLEAN NOT NULL DEFAULT false,  -- 是否必填
    options     JSONB,                           -- 选项（用于 select 类型）
    default_val JSONB,                           -- 默认值
    validation  JSONB,                           -- 验证规则
    sort_order  INTEGER NOT NULL DEFAULT 0,      -- 排序
    is_public   BOOLEAN NOT NULL DEFAULT false,  -- 是否在公共表单显示
    status      VARCHAR(20) NOT NULL DEFAULT 'active'
);
```

#### 6.2.2 字段是硬编码的，不是动态渲染

**所有表单中的硬编码字段**：

```javascript
-- Forms.vue 中的硬编码
renderHTML() {
  let h = `...
    <input type="email" name="email" ...>  -- 硬编码
    <input type="text" name="name" ...>     -- 硬编码
  `;
  
  -- 列表 checkbox 也是硬编码为 checkbox 类型
  this.checked.forEach((i) => {
    h += `<input type="checkbox" name="l" ...>`;  -- 硬编码类型
  });
}
```

```html
<!-- 公共订阅页面模板中的硬编码 -->
<input id="email" name="email" type="email" ...>  <!-- 硬编码 -->
<input id="name" name="name" type="text" ...>     <!-- 硬编码 -->
```

**真正的动态渲染应该是**：

```javascript
-- 伪代码：基于 schema 动态渲染
renderDynamicFields(fields) {
  return fields
    .filter(f => f.is_public)
    .sort((a, b) => a.sort_order - b.sort_order)
    .map(field => {
      switch (field.type) {
        case 'string':
          return `<input type="text" name="${field.key}" 
            ${field.required ? 'required' : ''} 
            placeholder="${field.name}">`;
        case 'number':
          return `<input type="number" name="${field.key}" ...>`;
        case 'boolean':
          return `<input type="checkbox" name="${field.key}">`;
        case 'select':
          return `<select name="${field.key}">
            ${field.options.map(opt => `<option value="${opt.value}">${opt.label}</option>`).join('')}
          </select>`;
        case 'radio':
          return field.options.map(opt => 
            `<input type="radio" name="${field.key}" value="${opt.value}">${opt.label}`
          ).join('');
        -- ... 更多类型
      }
    });
}
```

#### 6.2.3 与订阅者属性（attribs）完全无关

**关键证据**：

**文件**: `cmd/public.go:754-759`

```go
-- processSubForm 中的插入代码
_, hasOptin, err := a.core.InsertSubscriber(models.Subscriber{
    Name:   req.Name,
    Email:  req.Email,
    Status: models.SubscriberStatusEnabled,
    -- ⚠️ 没有设置 Attribs！
}, nil, listUUIDs, false, true)
```

**所有生成的表单字段映射**：

| 表单字段名 | 映射到数据库 |
|-----------|-------------|
| `email` | `subscribers.email` |
| `name` | `subscribers.name` |
| `l` (列表) | `subscriber_lists` 关联表 |
| ⚠️ **无** | `subscribers.attribs` |

**真正的自定义字段系统映射**：

| 字段定义 Schema | 映射到数据库 |
|----------------|-------------|
| `{key: "city", type: "string"}` | `attribs.city` |
| `{key: "company", type: "string"}` | `attribs.company` |
| `{key: "role", type: "select"}` | `attribs.role` |
| `{key: "vip", type: "boolean"}` | `attribs.vip` |

#### 6.2.4 没有元数据和验证

**当前系统的限制**：
- 不知道字段类型（无法做类型转换）
- 不知道验证规则（无法在前端做验证）
- 不知道默认值（无法预填）
- 不知道排序顺序（无法按配置顺序排列）

**真正的 Schema 系统包含的元数据**：

```json
{
  "key": "age",
  "name": "年龄",
  "type": "number",
  "required": true,
  "default_val": 18,
  "validation": {
    "min": 0,
    "max": 150,
    "pattern": null
  },
  "sort_order": 3,
  "is_public": true
}
```

### 6.3 图示对比

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│            当前系统 vs 理想的自定义字段 Schema 系统                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  当前系统（Forms.vue + 公共订阅页面）                                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                           │  │
│  │  ┌──────────┐    ┌──────────┐    ┌─────────────────────────────────┐  │  │
│  │  │ lists 表 │    │ settings │    │        硬编码 HTML 模板           │  │  │
│  │  │ (邮件列表)│    │ (全局配置)│    │  ┌───────────────────────────┐  │  │  │
│  │  └────┬─────┘    └────┬─────┘    │  │ email: <input type="email">│  │  │  │
│  │       │               │          │  │ name:  <input type="text"> │  │  │  │
│  │       ▼               ▼          │  │ lists: <checkbox x N>       │  │  │  │
│  │  ┌──────────────────────────┐    │  │ captcha: (条件性添加)        │  │  │  │
│  │  │  renderHTML() / 模板渲染  │    │  └───────────────────────────┘  │  │  │
│  │  │  (硬编码字符串拼接)       │    └─────────────────────────────────┘  │  │
│  │  └────────────┬─────────────┘                                         │  │
│  │               │                                                          │  │
│  │               ▼                                                          │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  最终存储: subscribers 表                                           │  │  │
│  │  │  - email: 已设置                                                    │  │  │
│  │  │  - name: 已设置                                                     │  │  │
│  │  │  - attribs: '{}' (默认空对象)                                      │  │  │
│  │  │                                                                     │  │  │
│  │  │  ⚠️ 公共订阅无法设置自定义属性！                                       │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ─────────────────────────────────────────────────────────────────────────────  │
│                                                                                  │
│  理想的自定义字段 Schema 渲染系统                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                           │  │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │  │
│  │  │                subscriber_fields 表（字段定义 Schema）             │  │  │
│  │  │  ┌────────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │ key     │ name │ type   │ required │ options │ sort_order │  │  │  │
│  │  │  ├─────────┼──────┼────────┼──────────┼─────────┼────────────┤  │  │  │
│  │  │  │ city    │ 城市 │ string │ false    │ null    │ 1          │  │  │  │
│  │  │  │ company │ 公司 │ string │ true     │ null    │ 2          │  │  │  │
│  │  │  │ role    │ 角色 │ select │ false    │ {...}   │ 3          │  │  │  │
│  │  │  │ vip     │ 会员 │ boolean│ false    │ null    │ 4          │  │  │  │
│  │  │  └─────────┴──────┴────────┴──────────┴─────────┴────────────┘  │  │  │
│  │  └──────────────────────────────────────────────────────────────────┘  │  │
│  │                                     │                                     │  │
│  │                                     ▼                                     │  │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    动态渲染引擎                                      │  │  │
│  │  │  - 按 type 选择输入组件 (text, number, select, checkbox...)        │  │  │
│  │  │  - 按 sort_order 排序                                                │  │  │
│  │  │  - 应用 validation 规则                                              │  │  │
│  │  │  - 处理 default_val 默认值                                           │  │  │
│  │  └──────────────────────────────────────────────────────────────────┘  │  │
│  │                                     │                                     │  │
│  │                                     ▼                                     │  │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    数据映射                                          │  │  │
│  │  │  表单字段值  ──────►  subscribers.attribs                          │  │  │
│  │  │  city: "北京"   ──────►  {"city": "北京", ...}                    │  │  │
│  │  │  vip: true     ──────►  {"vip": true, ...}                        │  │  │
│  │  └──────────────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 七、前端表单处理机制

### 7.1 管理后台订阅者表单

**文件**: `frontend/src/views/SubscriberForm.vue`

#### 模板部分

```vue
<b-field :message="$t('subscribers.attribsHelp') + ' ' + egAttribs" class="mt-6">
  <div>
    <h5>{{ $t('globals.terms.attribs') }}</h5>
    <b-input v-model="form.strAttribs" name="attribs" type="textarea" />
    <a href="https://listmonk.app/docs/concepts" target="_blank" class="is-size-7">
      {{ $t('globals.buttons.learnMore') }}
      <b-icon icon="link-variant" size="is-small" />
    </a>
  </div>
</b-field>
```

#### 数据处理逻辑

```javascript
data() {
  return {
    form: {
      lists: [],
      strAttribs: '{}',  // 以字符串形式存储
      status: 'enabled',
      preconfirm: false,
    },
    egAttribs: '{"job": "developer", "location": "Mars", "has_rocket": true}',
  };
},

methods: {
  createSubscriber() {
    let attribs = {};
    if (this.form.strAttribs) {
      attribs = this.validateAttribs(this.form.strAttribs);
      if (!attribs) {
        return;
      }
    }

    const data = {
      email: this.form.email,
      name: this.form.name,
      status: this.form.status,
      attribs,  // 解析后的 JSON 对象
      preconfirm_subscriptions: this.form.preconfirm,
      lists: this.form.lists.map((l) => l.id),
    };

    this.$api.createSubscriber(data).then((d) => {
      -- ...
    });
  },

  validateAttribs(str) {
    let attribs = {};
    try {
      attribs = JSON.parse(str);
    } catch (e) {
      this.$utils.toast(
        `${this.$t('subscribers.invalidJSON')}: ${e.toString()}`,
        'is-danger',
        3000,
      );
      return null;
    }
    if (attribs instanceof Array) {
      this.$utils.toast('Attributes should be a map {} and not an array []', 'is-danger', 3000);
      return null;
    }
    return attribs;
  },
},
```

**表单特性**:
- 使用 `<textarea>` 接收 JSON 字符串输入
- 仅做基本的 JSON 格式验证
- 提供示例 `egAttribs` 帮助用户理解格式
- **没有基于 schema 的字段验证**（如类型检查、必填验证等）

---

## 八、数据传输流程

### 8.1 API 接口定义

#### 创建订阅者

**文件**: `docs/docs/content/apis/subscribers.md:300-340`

```json
-- POST /api/subscribers
{
  "email": "subscriber@domain.com",
  "name": "The Subscriber",
  "status": "enabled",
  "lists": [1],
  "attribs": {
    "city": "Bengaluru",
    "projects": 3,
    "stack": { "languages": ["go", "python"] }
  }
}
```

#### 公共订阅 API

**文件**: `docs/docs/content/apis/subscribers.md:364-398`

```json
-- POST /api/public/subscription
{
  "email": "subscriber@domain.com",
  "name": "The Subscriber",
  "list_uuids": ["eb420c55-4cfb-4972-92ba-c93c34ba475d"]
  -- ⚠️ 注意：此接口不支持直接传入 attribs
}
```

### 8.2 后端处理逻辑

#### 创建订阅者处理

**文件**: `internal/core/subscribers.go:286-349`

```go
func (c *Core) InsertSubscriber(sub models.Subscriber, listIDs []int, listUUIDs []string, preconfirm, assertOptin bool) (models.Subscriber, bool, error) {
    -- ... 省略前置代码 ...
    
    -- 直接将 sub.Attribs 存入数据库
    if err = c.q.InsertSubscriber.Get(&sub.ID,
        sub.UUID,
        sub.Email,
        strings.TrimSpace(sub.Name),
        sub.Status,
        sub.Attribs,  -- 直接传入，无 schema 验证
        pq.Array(listIDs),
        pq.Array(listUUIDs),
        subStatus); err != nil {
        -- ... 错误处理 ...
    }
    
    -- ... 省略后续代码 ...
}
```

---

## 九、数据流总结

### 9.1 数据流向图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据流概览                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐                    ┌──────────────────┐                   │
│  │ 管理后台表单  │                    │   公共订阅表单    │                   │
│  │ (JSON输入)   │                    │  (无自定义字段)   │                   │
│  └──────┬───────┘                    └────────┬─────────┘                   │
│         │                                     │                              │
│         ▼                                     ▼                              │
│  ┌──────────────────────────────────────────────────────────┐               │
│  │                    API 层                                  │               │
│  │  POST /api/subscribers        POST /api/public/subscription │           │
│  │  (支持 attribs 参数)           (不支持 attribs 参数)      │               │
│  └──────────────────────┬─────────────────────────────────────┘               │
│                         │                                                       │
│                         ▼                                                       │
│  ┌──────────────────────────────────────────────────────────┐               │
│  │                  后端 Core 层                              │               │
│  │  InsertSubscriber / UpdateSubscriber                       │               │
│  │  (直接存储 JSON，无 schema 验证)                          │               │
│  └──────────────────────┬─────────────────────────────────────┘               │
│                         │                                                       │
│                         ▼                                                       │
│  ┌──────────────────────────────────────────────────────────┐               │
│  │                    数据库层                                │               │
│  │  subscribers.attribs (JSONB 类型)                        │               │
│  │  - 支持 JSON 操作符 (->>, @>, etc.)                      │               │
│  │  - 无 schema 约束，自由格式存储                           │               │
│  └──────────────────────────────────────────────────────────┘               │
│                                                                              │
