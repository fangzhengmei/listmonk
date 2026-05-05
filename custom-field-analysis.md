# Listmonk 自定义字段（Attributes）数据流分析报告

## 概述

本报告详细分析 listmonk 中订阅者自定义字段（Attributes）的存储结构、数据传输流程，以及与订阅表单的交互机制。

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
    Attribs JSON           `db:"attribs" json:"attribs"`  // 自定义属性
    Status  string         `db:"status" json:"status"`
    Lists   types.JSONText `db:"lists" json:"lists"`
}
```

**文件**: `models/common.go:112-134`

```go
// JSON 是 map[string]any 的类型别名，用于处理数据库的 JSONB 字段
type JSON map[string]any

// Value 实现 driver.Valuer 接口，将 JSON 序列化存入数据库
func (s JSON) Value() (driver.Value, error) {
    return json.Marshal(s)
}

// Scan 实现 sql.Scanner 接口，从数据库反序列化 JSON
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

## 二、数据传输流程

### 2.1 API 接口定义

#### 创建订阅者
**文件**: `docs/docs/content/apis/subscribers.md:300-340`

```json
// POST /api/subscribers
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
// POST /api/public/subscription
{
  "email": "subscriber@domain.com",
  "name": "The Subscriber",
  "list_uuids": ["eb420c55-4cfb-4972-92ba-c93c34ba475d"]
  // ⚠️ 注意：此接口不支持直接传入 attribs
}
```

### 2.2 后端处理逻辑

#### 创建订阅者处理
**文件**: `internal/core/subscribers.go:286-349`

```go
func (c *Core) InsertSubscriber(sub models.Subscriber, listIDs []int, listUUIDs []string, preconfirm, assertOptin bool) (models.Subscriber, bool, error) {
    // ... 省略前置代码 ...
    
    // 直接将 sub.Attribs 存入数据库
    if err = c.q.InsertSubscriber.Get(&sub.ID,
        sub.UUID,
        sub.Email,
        strings.TrimSpace(sub.Name),
        sub.Status,
        sub.Attribs,  // 直接传入，无 schema 验证
        pq.Array(listIDs),
        pq.Array(listUUIDs),
        subStatus); err != nil {
        // ... 错误处理 ...
    }
    
    // ... 省略后续代码 ...
}
```

#### 更新订阅者处理
**文件**: `internal/core/subscribers.go:352-383`

```go
func (c *Core) UpdateSubscriber(id int, sub models.Subscriber) (models.Subscriber, error) {
    // 格式化 JSON 属性
    attribs := []byte("{}")
    if len(sub.Attribs) > 0 {
        if b, err := json.Marshal(sub.Attribs); err != nil {
            return models.Subscriber{}, echo.NewHTTPError(...)
        } else {
            attribs = b
        }
    }

    _, err := c.q.UpdateSubscriber.Exec(id,
        sub.Email,
        strings.TrimSpace(sub.Name),
        sub.Status,
        json.RawMessage(attribs),  // 序列化为 JSON 存储
    )
    // ...
}
```

### 2.3 前端 API 层

**文件**: `frontend/src/api/index.js:162-170`

```javascript
export const getSubscribers = async (params) => http.get(
  '/api/subscribers',
  {
    params,
    loading: models.subscribers,
    store: models.subscribers,
    // 特殊处理：attribs 的键名不转换为驼峰命名
    camelCase: (keyPath) => !keyPath.startsWith('.results.*.attribs'),
  },
);
```

**关键设计**:
- `attribs` 对象的键名保持原样，不进行驼峰转换
- 这确保了用户定义的键名（如 `spoken_languages`）在前端保持不变

---

## 三、前端表单处理机制

### 3.1 管理后台订阅者表单

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
      // ...
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

### 3.2 公共订阅表单

**文件**: `static/public/templates/subscription-form.html`

```html
<form method="post" action="" class="form">
  <div>
    <p>
      <label for="email">{{ L.T "subscribers.email" }}</label>
      <input id="email" name="email" required="true" type="email" ...>
    </p>
    <p>
      <label for="name">{{ L.T "public.subName" }}</label>
      <input id="name" name="name" type="text" ...>
    </p>

    <!-- 邮件列表选择 -->
    <ul class="lists">
      {{ range $i, $l := .Data.Lists }}
        <li>
          <input checked="true" id="l-{{ $l.UUID}}" type="checkbox" name="l" value="{{ $l.UUID }}" >
          <label for="l-{{ $l.UUID}}">{{ $l.Name }}</label>
        </li>
      {{ end }}
    </ul>

    <!-- 验证码（如果启用） -->
    {{ if .Data.Captcha.Enabled }}
      <div class="captcha">...</div>
    {{ end }}

    <p>
      <button type="submit" class="button">{{ L.T "public.sub" }}</button>
    </p>
  </div>
</form>
```

**表单字段**:
1. `email` - 邮箱（必填）
2. `name` - 姓名
3. `l` - 邮件列表（多选）
4. 验证码（可选）

**⚠️ 关键限制**:
- **没有自定义字段输入** - 公共订阅表单不支持输入 `attribs` 数据
- **没有动态渲染机制** - 表单字段是硬编码的，无法基于配置动态生成

### 3.3 公共订阅表单后端处理

**文件**: `cmd/public.go:706-788`

```go
func (a *App) processSubForm(c echo.Context) (bool, error) {
    var req struct {
        Name          string   `form:"name" json:"name"`
        Email         string   `form:"email" json:"email"`
        FormListUUIDs []string `form:"l" json:"list_uuids"`
        // ⚠️ 注意：没有 attribs 字段！
    }
    if err := c.Bind(&req); err != nil {
        return false, err
    }

    // ... 验证代码 ...

    // 插入订阅者时，attribs 为空
    _, hasOptin, err := a.core.InsertSubscriber(models.Subscriber{
        Name:   req.Name,
        Email:  req.Email,
        Status: models.SubscriberStatusEnabled,
        // ⚠️ 没有设置 Attribs！
    }, nil, listUUIDs, false, true)
    
    // ...
}
```

---

## 四、属性的使用场景

### 4.1 查询与分段

**文件**: `docs/docs/content/apis/subscribers.md:55`

```shell
curl -u 'api_username:access_token' -X GET 'http://localhost:9000/api/subscribers' \
    --url-query 'page=1' \
    --url-query 'per_page=100' \
    --url-query "query=subscribers.name LIKE 'Test%' AND subscribers.attribs->>'city' = 'Bengaluru'"
```

### 4.2 邮件模板渲染

属性可以在邮件模板中使用：

```html
<!-- 模板示例 -->
<p>Hello {{ .Subscriber.Name }},</p>
<p>You are from {{ .Subscriber.Attribs.city }}.</p>
{{ if .Subscriber.Attribs.likes_tea }}
  <p>We have special tea offers for you!</p>
{{ end }}
```

### 4.3 管理后台查看

在订阅者详情页面，可以查看和编辑属性的 JSON 数据。

---

## 五、数据流总结

### 5.1 数据流向图

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
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 与"理想"动态表单系统的对比

| 特性 | listmonk 当前实现 | 理想的动态表单系统 |
|------|------------------|-------------------|
| 字段定义存储 | ❌ 无独立存储 | ✅ 独立的字段定义表 |
| 字段元数据 | ❌ 无类型/标签/验证规则 | ✅ 包含类型、标签、验证、默认值等 |
| 表单动态渲染 | ❌ 硬编码字段 | ✅ 基于 schema 动态生成 |
| 数据验证 | ❌ 仅 JSON 格式验证 | ✅ 基于 schema 的类型/规则验证 |
| 公共表单支持 | ❌ 不支持自定义字段 | ✅ 支持配置的字段 |

---

## 六、API 端点与属性相关的处理

### 6.1 管理 API

| 端点 | 方法 | 支持 attribs | 说明 |
|------|------|-------------|------|
| `/api/subscribers` | POST | ✅ | 创建时可设置属性 |
| `/api/subscribers/{id}` | PUT | ✅ | 全量更新属性 |
| `/api/subscribers/{id}` | PATCH | ✅ | 合并更新属性 |
| `/api/subscribers` | GET | ✅ | 返回包含属性的订阅者数据 |
| `/api/subscribers/{id}` | GET | ✅ | 返回单个订阅者的属性 |

### 6.2 公共 API

| 端点 | 方法 | 支持 attribs | 说明 |
|------|------|-------------|------|
| `/api/public/subscription` | POST | ❌ | 不支持设置属性 |
| `/api/public/lists` | GET | ❌ | 仅返回列表信息 |

---

## 七、扩展建议

如果需要实现"自定义字段 schema 驱动订阅表单动态渲染"，可以考虑以下扩展方案：

### 7.1 数据库扩展

```sql
-- 新增字段定义表
CREATE TABLE subscriber_fields (
    id          SERIAL PRIMARY KEY,
    key         VARCHAR(100) NOT NULL UNIQUE,  -- 字段键名
    name        VARCHAR(200) NOT NULL,          -- 显示名称
    type        VARCHAR(50) NOT NULL,           -- 类型: string, number, boolean, select, date
    required    BOOLEAN NOT NULL DEFAULT false,  -- 是否必填
    options     JSONB,                           -- 选项（用于 select 类型）
    default_val JSONB,                           -- 默认值
    validation  JSONB,                           -- 验证规则
    description TEXT,                            -- 描述
    sort_order  INTEGER NOT NULL DEFAULT 0,      -- 排序
    is_public   BOOLEAN NOT NULL DEFAULT false,  -- 是否在公共表单显示
    status      VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### 7.2 后端模型扩展

```go
type SubscriberField struct {
    ID          int                    `db:"id" json:"id"`
    Key         string                 `db:"key" json:"key"`
    Name        string                 `db:"name" json:"name"`
    Type        string                 `db:"type" json:"type"`
    Required    bool                   `db:"required" json:"required"`
    Options     map[string]interface{} `db:"options" json:"options"`
    DefaultVal  map[string]interface{} `db:"default_val" json:"default_val"`
    Validation  map[string]interface{} `db:"validation" json:"validation"`
    SortOrder   int                    `db:"sort_order" json:"sort_order"`
    IsPublic    bool                   `db:"is_public" json:"is_public"`
    Status      string                 `db:"status" json:"status"`
}
```

### 7.3 表单渲染逻辑

```javascript
// 伪代码：基于 schema 动态渲染表单
function renderFormFields(fields) {
  return fields
    .filter(f => f.is_public)
    .sort((a, b) => a.sort_order - b.sort_order)
    .map(field => {
      switch (field.type) {
        case 'string':
          return `<input type="text" name="${field.key}" ${field.required ? 'required' : ''} placeholder="${field.name}">`;
        case 'number':
          return `<input type="number" name="${field.key}" ${field.required ? 'required' : ''}>`;
        case 'boolean':
          return `<input type="checkbox" name="${field.key}">`;
        case 'select':
          return `<select name="${field.key}" ${field.required ? 'required' : ''}>
            ${field.options.map(opt => `<option value="${opt.value}">${opt.label}</option>`).join('')}
          </select>`;
        // ... 更多类型
      }
    });
}
```

---

## 八、总结

### 8.1 当前系统的特点

1. **简单灵活**：`attribs` 作为自由格式 JSON，支持任意嵌套结构
2. **功能受限**：没有 schema 约束，无法进行类型安全验证
3. **公共表单不支持**：用户无法通过公共订阅表单提交自定义属性
4. **管理后台体验差**：需要手动编辑 JSON，容易出错

### 8.2 使用场景建议

当前的实现适合以下场景：
- 开发人员通过 API 集成外部系统
- 简单的属性存储需求，不需要复杂验证
- 通过管理后台少量维护订阅者数据

不适合以下场景：
- 需要终端用户通过表单填写自定义字段
- 需要严格的数据验证和类型约束
- 需要动态配置表单字段的 SaaS 平台

### 8.3 关键文件索引

| 文件路径 | 说明 |
|---------|------|
| `schema.sql:20-30` | 数据库表定义，包含 `attribs` 字段 |
| `models/subscribers.go:28-37` | Subscriber 模型定义 |
| `models/common.go:112-134` | JSON 类型的 Value/Scan 实现 |
| `internal/core/subscribers.go` | 订阅者核心业务逻辑 |
| `cmd/subscribers.go` | 订阅者 API 处理层 |
| `cmd/public.go:706-788` | 公共订阅表单处理 |
| `frontend/src/views/SubscriberForm.vue` | 管理后台订阅者表单 |
| `frontend/src/api/index.js:162-170` | 前端 API 定义（含 camelCase 特殊处理） |
| `static/public/templates/subscription-form.html` | 公共订阅表单模板 |

---

## 附录：API 示例

### 获取包含属性的订阅者

```shell
curl -u 'api_username:access_token' 'http://localhost:9000/api/subscribers/1'
```

响应：
```json
{
  "data": {
    "id": 1,
    "uuid": "ea06b2e7-4b08-4697-bcfc-2a5c6dde8f1c",
    "email": "john@example.com",
    "name": "John Doe",
    "attribs": {
      "city": "Bengaluru",
      "good": true,
      "type": "known"
    },
    "status": "enabled",
    "lists": [...]
  }
}
```

### 创建带属性的订阅者

```shell
curl -u 'api_username:access_token' 'http://localhost:9000/api/subscribers' \
  -H 'Content-Type: application/json' \
  --data '{
    "email": "new@example.com",
    "name": "New Subscriber",
    "status": "enabled",
    "lists": [1],
    "attribs": {
      "city": "Shanghai",
      "interests": ["tech", "design"],
      "vip": true
    }
  }'
```

---

*报告生成时间: 2026-05-05*  
*基于 listmonk 代码库分析*
