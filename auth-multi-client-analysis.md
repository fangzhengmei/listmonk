# Listmonk 多客户端鉴权实现分析

## 一、重要澄清：术语与概念边界

### 1.1 关于 JWT 的澄清

**Listmonk 不使用 JWT（JSON Web Token）进行鉴权！**

经过完整代码审计，确认以下事实：
- 代码库中**没有任何 JWT 签发或验证逻辑**
- `go.sum` 中仅有的 JWT 相关依赖是 `github.com/coreos/go-oidc/v3` 的传递依赖
- 所谓的 "JWT 鉴权" 实际是以下两种机制的混淆：

| 实际机制 | 说明 | 代码位置 |
|---------|------|---------|
| **API Token** | 简单的 `api_username:api_token` 格式，明文存储 | `internal/auth/auth.go:431-464` |
| **OIDC ID Token** | OIDC 登录流程中使用，但仅用于身份验证，不用于后续 API 调用 | `internal/auth/auth.go:224-279` |

### 1.2 三种鉴权相关机制的明确边界

```
┌─────────────────────────────────────────────────────────────────┐
│                    Listmonk 鉴权体系架构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐ │
│  │  API Token   │      │ Session      │      │   OIDC       │ │
│  │  (API客户端)  │      │ Cookie       │      │ (仅登录用)   │ │
│  └──────┬───────┘      │ (Web管理界面) │      └──────┬───────┘ │
│         │              └──────┬───────┘             │         │
│         │                     │                      │         │
│         ▼                     ▼                      ▼         │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              Auth.Middleware (统一认证中间件)             │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 API Token vs OIDC 的关键区别

| 维度 | API Token | OIDC |
|------|-----------|------|
| **用途** | API 客户端持续鉴权 | Web 界面第三方登录 |
| **有效期** | 永不过期（需手动撤销） | 登录时验证一次 |
| **后续调用** | 每次请求都需携带 | 登录后转用 Session Cookie |
| **Token 格式** | `api_username:random_string` | JWT 格式（由 OIDC 提供商签发） |
| **存储方式** | 数据库明文存储 | 仅存储在会话中（`oidc_token` 字段） |
| **适用客户端** | 命令行工具、第三方应用 | 浏览器 Web 管理界面 |

---

## 二、核心鉴权实现分析

### 2.1 统一认证中间件

所有 HTTP 请求都经过 `Auth.Middleware` 处理（`internal/auth/auth.go:282-333`）：

```go
func (o *Auth) Middleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        // 1. 获取 Authorization 头
        hdr := strings.TrimSpace(c.Request().Header.Get("Authorization"))

        // 2. ⚠️ 关键：如果有 session Cookie，忽略 Authorization 头
        // 这是 v3 -> v4 的向后兼容措施
        if c := strings.TrimSpace(c.Request().Header.Get("Cookie")); strings.Contains(c, "session=") {
            hdr = ""  // 清空 Authorization 头
        }

        // 3. 如果有 Authorization 头，处理 API Token/Basic Auth
        if len(hdr) > 0 {
            key, token, err := parseAuthHeader(hdr)
            // ... 验证 API Token
            user, ok := o.GetAPIToken(key, token)
            // ...
        }

        // 4. 否则，尝试 Cookie 会话验证
        sess, user, err := o.validateSession(c)
        // ...
    }
}
```

### 2.2 Cookie 与 Authorization 并存时的优先顺序

**⚠️ 重要修正：实际是 Session Cookie 优先！**

| 场景 | 存在 Session Cookie | 存在 Authorization 头 | 实际使用的鉴权方式 |
|------|---------------------|----------------------|-------------------|
| 1 | ✅ 是 | ✅ 是 | **Session Cookie**（Authorization 被忽略） |
| 2 | ✅ 是 | ❌ 否 | Session Cookie |
| 3 | ❌ 否 | ✅ 是 | API Token / Basic Auth |
| 4 | ❌ 否 | ❌ 否 | 认证失败 |

**设计意图**（`internal/auth/auth.go:291-297` 注释）：
```
// If cookie is set, ignore BasicAuth. This is to preserve backwards compatibility
// in v3 -> v4 upgrade where the user browser sessions would still have old
// BasicAuth credentials, which no longer work in the new system which expects
// session cookies instead, which causes a redirect loop despite loggin in and session
// cookies being set.
```

这是一个**临时的向后兼容措施**，用于 v3 到 v4 的平滑升级（v3 使用 Basic Auth，v4 使用 Session Cookie）。

### 2.3 Authorization 头解析

`parseAuthHeader` 函数（`internal/auth/auth.go:431-464`）支持两种格式：

#### 格式 1：Token 格式（推荐）
```
Authorization: token api_username:api_token
```

#### 格式 2：HTTP Basic Auth（向后兼容）
```
Authorization: Basic base64(api_username:api_token)
```

### 2.4 API Token 验证机制

`GetAPIToken` 函数（`internal/auth/auth.go:135-146`）：

```go
func (o *Auth) GetAPIToken(user string, token string) (User, bool) {
    o.RLock()
    t, ok := o.apiUsers[user]  // 从内存缓存获取
    o.RUnlock()

    // 使用常量时间比较防止时序攻击
    if !ok || subtle.ConstantTimeCompare([]byte(t.Password.String), []byte(token)) != 1 {
        return User{}, false
    }

    return t, true
}
```

**安全特性**：
- API 用户缓存在内存中（`apiUsers` map），避免每次请求查询数据库
- 使用 `subtle.ConstantTimeCompare` 进行常量时间比较，防止时序攻击

### 2.5 API Token 与普通密码的存储差异

**关键差异：存储方式不同！**

从 `queries/users.sql:3-13`：
```sql
INSERT INTO users (username, password_login, password, ...)
VALUES($1, $2, (
    CASE
        -- 普通用户：bcrypt 哈希存储
        WHEN $6::user_type != 'api' AND $2 AND $3 != ''
            THEN CRYPT($3, GEN_SALT('bf'))
        -- API 用户：明文存储！
        WHEN $6 = 'api'
            THEN $3  -- 直接存储，不做哈希
        ELSE NULL
    END
), ...)
```

| 用户类型 | 存储方式 | 验证方式 | 代码位置 |
|---------|---------|---------|---------|
| `UserTypeUser`（普通用户） | bcrypt 哈希 | `CRYPT($2, password) = password` | `queries/users.sql:134-141` |
| `UserTypeAPI`（API 用户） | **明文** | 直接字符串比较 | `internal/auth/auth.go:135-146` |

**API Token 生成逻辑**（`internal/core/users.go:49-58`）：
```go
if u.Type == auth.UserTypeAPI {
    // 生成 32 位随机字符串作为 Token
    tk, err := utils.GenerateRandomString(32)
    // ...
    u.Email = null.String{String: u.Username + "@api", Valid: true}
    u.PasswordLogin = false
    u.Password = null.String{String: tk, Valid: true}  // 明文存储
}
```

---

## 三、两种主要鉴权方式对比

| 维度 | Session Cookie（Web 界面） | API Token（API 客户端） |
|------|---------------------------|------------------------|
| **用户类型** | `UserTypeUser` | `UserTypeAPI` |
| **传输方式** | Cookie 头 | Authorization 头 |
| **状态管理** | 有状态（服务端 PostgreSQL 存储） | 无状态 |
| **Token 存储** | 数据库中存储会话数据 | 数据库**明文**存储 Token |
| **普通用户密码** | bcrypt 哈希 | 不适用 |
| **过期机制** | Cookie MaxAge = 7 天 | 永不过期（需手动撤销） |
| **适用客户端** | 浏览器 Vue 管理界面 | 命令行工具、第三方应用 |
| **认证失败响应** | 307 重定向到登录页 | 403 JSON 错误 |

---

## 四、不同客户端鉴权分析

### 4.1 Vue 管理界面（Web 浏览器客户端）

#### 鉴权方式：Session Cookie

#### 完整鉴权流程

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Vue 管理界面鉴权流程                                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. 访问登录页面（未认证）                                              │
│     GET /admin                                                        │
│         │                                                              │
│         ▼                                                              │
│     Auth.Middleware 检测到无 session Cookie                            │
│         │                                                              │
│         ▼                                                              │
│     307 重定向到 /admin/login                                          │
│                                                                       │
│  2. 用户登录（POST /admin/login）                                      │
│     ┌─────────────────────────────────────────────────────────────┐  │
│     │ POST /admin/login                                            │  │
│     │ Content-Type: application/x-www-form-urlencoded             │  │
│     │                                                              │  │
│     │ username=admin&password=secret123                            │  │
│     └─────────────────────────────────────────────────────────────┘  │
│         │                                                              │
│         ▼                                                              │
│     cmd/auth.go:doLogin()                                              │
│         │                                                              │
│         ├──► core.LoginUser()                                          │
│         │       │                                                       │
│         │       └──► SQL: SELECT ... FROM users                       │
│         │            WHERE username = $1                               │
│         │            AND CRYPT($2, password) = password  (bcrypt 验证)│
│         │                                                              │
│         ├──► 检查 2FA（如果启用）                                       │
│         │                                                              │
│         └──► auth.SaveSession()                                        │
│                 │                                                       │
│                 └──► 生成 session Cookie:                               │
│                      Set-Cookie: session=abc123...;                   │
│                                 HttpOnly; Max-Age=604800; Path=/     │
│                                                                       │
│  3. 后续请求（携带 Cookie）                                             │
│     ┌─────────────────────────────────────────────────────────────┐  │
│     │ GET /api/lists                                                │  │
│     │ Cookie: session=abc123...                                     │  │
│     └─────────────────────────────────────────────────────────────┘  │
│         │                                                              │
│         ▼                                                              │
│     Auth.Middleware                                                    │
│         │                                                              │
│         ├──► 检测到 session Cookie                                    │
│         │                                                              │
│         ├──► validateSession()                                         │
│         │       │                                                       │
│         │       └──► 从 PostgreSQL 查询会话数据                        │
│         │            SELECT * FROM sessions WHERE id = $1             │
│         │                                                              │
│         └──► 获取 user_id，查询用户信息                                │
│                 │                                                       │
│                 └──► 设置 c.Set(UserHTTPCtxKey, user)                 │
│                                                                       │
│  4. OIDC 登录流程（可选）                                               │
│     POST /auth/oidc → 重定向到 OIDC 提供商 →                           │
│     GET /auth/oidc?code=... → 验证 ID Token →                         │
│     auth.SaveSession() → 设置 Session Cookie                           │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

#### 最小请求示例

##### 示例 1：表单登录请求
```http
POST /admin/login HTTP/1.1
Host: localhost:9000
Content-Type: application/x-www-form-urlencoded
Content-Length: 35

username=admin&password=secret123
```

**成功响应**：
```http
HTTP/1.1 302 Found
Location: /admin
Set-Cookie: session=abc123def456...; HttpOnly; Path=/; Max-Age=604800
```

##### 示例 2：后续 API 请求（携带 Cookie）
```http
GET /api/lists HTTP/1.1
Host: localhost:9000
Cookie: session=abc123def456...
Accept: application/json
```

**成功响应**：
```json
{
  "data": {
    "results": [...],
    "total": 5
  }
}
```

**认证失败响应**：
```http
HTTP/1.1 307 Temporary Redirect
Location: /admin/login?next=%2Fapi%2Flists
```

#### 前端实现要点

**API 配置**（`frontend/src/api/index.js:8-16`）：
```javascript
const http = axios.create({
  baseURL: import.meta.env.VUE_APP_ROOT_URL || '/',
  withCredentials: false,  // ⚠️ 不跨域发送凭据
  responseType: 'json',
  // ...
});
```

**关键**：前端不手动处理认证头，完全依赖浏览器自动发送 Cookie。

**初始化流程**（`frontend/src/main.js:34-88`）：
```javascript
async function initConfig(app) {
    // 启动时尝试获取用户 profile（依赖 Cookie）
    const [profile, cfg] = await Promise.all([
        api.getUserProfile(), 
        api.getServerConfig()
    ]);
    // ...
}
```

#### 路由配置与错误处理

**管理界面路由**（`cmd/handlers.go:52-77`）：
```go
// 认证的非 API 路由（管理界面）
g := e.Group("", a.auth.Middleware, func(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        u := c.Get(auth.UserHTTPCtxKey)

        // ⚠️ 认证失败时：重定向到登录页面
        if _, ok := u.(*echo.HTTPError); ok {
            u, _ := url.Parse(a.urlCfg.LoginURL)
            q := url.Values{}
            q.Set("next", c.Request().RequestURI)  // 记录原始请求 URI
            u.RawQuery = q.Encode()
            return c.Redirect(http.StatusTemporaryRedirect, u.String())
        }

        return next(c)
    }
})
```

---

### 4.2 命令行工具（CLI）

#### 鉴权方式：API Token

#### 完整鉴权流程

```
┌──────────────────────────────────────────────────────────────────────┐
│                    命令行工具鉴权流程                                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  前置条件：创建 API 用户                                                │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ 1. 通过管理界面或 API 创建 type=api 的用户                        │ │
│  │                                                                   │ │
│  │ POST /api/users                                                   │ │
│  │ Content-Type: application/json                                   │ │
│  │ Authorization: token admin:admin_password  (使用管理员认证)       │ │
│  │                                                                   │ │
│  │ {                                                                 │ │
│  │   "username": "mycli",                                            │ │
│  │   "name": "My CLI Tool",                                          │ │
│  │   "type": "api",                   // ⚠️ 必须是 "api"            │ │
│  │   "user_role_id": 2                // 分配角色                    │ │
│  │ }                                                                 │ │
│  │                                                                   │ │
│  │ 响应：                                                            │ │
│  │ {                                                                 │ │
│  │   "data": {                                                       │ │
│  │     "id": 5,                                                      │ │
│  │     "username": "mycli",                                          │ │
│  │     "type": "api",                                                │ │
│  │     "password": "abc123def456..."  // ⚠️ 仅此处显示 Token！     │ │
│  │   }                                                               │ │
│  │ }                                                                 │ │
│  │                                                                   │ │
│  │ 重要：Token 仅在创建时显示一次，后续查询不会返回！                   │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  鉴权流程：                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ CLI 发起请求                                                      │ │
│  │                                                                  │ │
│  │ GET /api/lists                                                   │ │
│  │ Host: localhost:9000                                             │ │
│  │ Authorization: token mycli:abc123def456...                      │ │
│  │ Accept: application/json                                         │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│         │                                                              │
│         ▼                                                              │
│     Auth.Middleware                                                    │
│         │                                                              │
│         ├──► 检查是否有 session Cookie（假设没有）                      │
│         │                                                              │
│         ├──► 解析 Authorization 头                                     │
│         │       parseAuthHeader("token mycli:abc123...")             │
│         │       → key="mycli", token="abc123..."                      │
│         │                                                              │
│         └──► GetAPIToken("mycli", "abc123...")                        │
│                 │                                                       │
│                 ├──► 从内存缓存 apiUsers 查找                           │
│                 │       apiUsers["mycli"] → User{}                    │
│                 │                                                              │
│                 └──► 常量时间比较：                                      │
│                      subtle.ConstantTimeCompare(                        │
│                          []byte(user.Password.String),                  │
│                          []byte(token)                                  │
│                      )                                                   │
│                                                                       │
│  ⚠️ 注意：API Token 是明文存储和比较的！                                 │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

#### 最小请求示例

##### 示例 1：使用 Token 格式（推荐）
```bash
# 使用 curl
curl -X GET http://localhost:9000/api/lists \
  -H "Authorization: token mycli:abc123def456ghij7890" \
  -H "Accept: application/json"
```

##### 示例 2：使用 Basic Auth 格式（向后兼容）
```bash
# 先编码 username:token
echo -n "mycli:abc123def456ghij7890" | base64
# 输出: bXljbGk6YWJjMTIzZGVmNDU2Z2hpajc4OTA=

curl -X GET http://localhost:9000/api/lists \
  -H "Authorization: Basic bXljbGk6YWJjMTIzZGVmNDU2Z2hpajc4OTA=" \
  -H "Accept: application/json"
```

##### 示例 3：使用 curl 的 -u 参数（等价于 Basic Auth）
```bash
curl -X GET http://localhost:9000/api/lists \
  -u "mycli:abc123def456ghij7890" \
  -H "Accept: application/json"
```

**成功响应**：
```json
{
  "data": {
    "results": [
      {
        "id": 1,
        "name": "Newsletter",
        "type": "public",
        "status": "enabled"
      }
    ],
    "total": 1
  }
}
```

**认证失败响应**（API 路由返回 JSON 错误）：
```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "message": "invalid API credentials"
}
```

#### API 用户缓存机制

**缓存时机**（`cmd/users.go:352-375`）：
```go
func cacheUsers(co *core.Core, a *auth.Auth) (bool, error) {
    users, err := co.GetUsers()
    // ...
    
    apiUsers := make([]auth.User, 0, len(users))
    for _, u := range users {
        // 只缓存 type=api 且 status=enabled 的用户
        if u.Type == auth.UserTypeAPI && u.Status == auth.UserStatusEnabled {
            apiUsers = append(apiUsers, u)
        }
        // ...
    }
    
    a.CacheAPIUsers(apiUsers)  // 更新内存缓存
    return hasUser, nil
}
```

**缓存触发场景**：
1. 应用启动时（`cmd/main.go:222`）
2. 创建/更新/删除用户后（`cmd/users.go:99, 184, 200`）

---

### 4.3 第三方应用

#### 鉴权方式：API Token（与命令行工具相同）

#### 与命令行工具的区别

| 维度 | 命令行工具 | 第三方应用 |
|------|-----------|-----------|
| **Token 管理** | 用户手动配置 | 应用开发者集成 |
| **权限需求** | 通常需要较全权限 | 通常需要最小权限（遵循最小权限原则） |
| **使用场景** | 自动化脚本、批量操作 | SaaS 集成、自定义工作流 |
| **错误处理** | 简单退出/重试 | 需要完整的错误处理和重试逻辑 |

#### 完整鉴权流程

```
┌──────────────────────────────────────────────────────────────────────┐
│                    第三方应用鉴权流程                                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. 创建专用 API 用户（最小权限原则）                                   │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ 步骤 1：创建限制权限的角色                                          │ │
│  │ POST /api/roles/users                                              │ │
│  │ {                                                                 │ │
│  │   "name": "Newsletter Sender",                                    │ │
│  │   "type": "user",                                                 │ │
│  │   "permissions": [                                                │ │
│  │     "campaigns:get",      // 只读邮件活动                         │ │
│  │     "campaigns:send",     // 发送邮件活动                         │ │
│  │     "subscribers:get"     // 只读订阅者                           │ │
│  │   ]                                                                │ │
│  │ }                                                                 │ │
│  │                                                                  │ │
│  │ 步骤 2：创建 API 用户并分配角色                                     │ │
│  │ POST /api/users                                                   │ │
│  │ {                                                                 │ │
│  │   "username": "newsletter-app",                                   │ │
│  │   "name": "Newsletter Integration",                               │ │
│  │   "type": "api",                                                  │ │
│  │   "user_role_id": 3  // 刚才创建的角色 ID                         │ │
│  │ }                                                                 │ │
│  │                                                                  │ │
│  │ ⚠️ 保存返回的 password（Token），仅显示一次！                        │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  2. 应用集成 API                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ 示例：Node.js + axios                                              │ │
│  │                                                                  │ │
│  │ const axios = require('axios');                                  │ │
│  │                                                                  │ │
│  │ const listmonk = axios.create({                                  │ │
│  │   baseURL: 'https://listmonk.example.com',                       │ │
│  │   headers: {                                                      │ │
│  │     'Authorization': `token ${process.env.LISTMONK_API_USER}:`  │ │
│  │                      + `${process.env.LISTMONK_API_TOKEN}`,      │ │
│  │     'Accept': 'application/json'                                  │ │
│  │   }                                                                │ │
│  │ });                                                                │ │
│  │                                                                  │ │
│  │ // 发送事务性邮件                                                 │ │
│  │ async function sendTxEmail(subscriberEmail, templateID, data) { │ │
│  │   try {                                                           │ │
│  │     const response = await listmonk.post('/api/tx', {           │ │
│  │       subscriber_email: subscriberEmail,                          │ │
│  │       template_id: templateID,                                    │ │
│  │       data: data                                                  │ │
│  │     });                                                           │ │
│  │     return response.data;                                         │ │
│  │   } catch (error) {                                               │ │
│  │     if (error.response) {                                         │ │
│  │       switch (error.response.status) {                            │ │
│  │         case 401:                                                 │ │
│  │           console.error('API 认证失败，请检查 Token');            │ │
│  │           break;                                                  │ │
│  │         case 403:                                                 │ │
│  │           console.error('权限不足，请检查用户角色');                │ │
│  │           break;                                                  │ │
│  │         default:                                                  │ │
│  │           console.error('请求失败:', error.response.data);       │ │
│  │       }                                                           │ │
│  │     }                                                             │ │
│  │     throw error;                                                  │ │
│  │   }                                                               │ │
│  │ }                                                                 │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  3. 权限检查流程（服务器端）                                           │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ POST /api/tx                                                      │ │
│  │                                                                  │ │
│  │ 执行顺序：                                                        │ │
│  │ 1. Auth.Middleware → 验证 API Token                              │ │
│  │ 2. Auth.Perm 中间件 → 检查权限 "tx:send"                         │ │
│  │                                                                  │ │
│  │ Auth.Perm 逻辑（internal/auth/auth.go:336-367）：               │ │
│  │                                                                  │ │
│  │ func (o *Auth) Perm(next echo.HandlerFunc, perms ...string) {  │ │
│  │   // ...                                                         │ │
│  │   // 超级管理员跳过权限检查                                        │ │
│  │   if u.UserRole.ID == SuperAdminRoleID { // ID = 1              │ │
│  │     return next(c)                                                │ │
│  │   }                                                               │ │
│  │                                                                  │ │
│  │   // 检查用户是否有指定权限                                        │ │
│  │   for _, perm := range perms {                                   │ │
│  │     if _, ok := u.PermissionsMap[perm]; ok {                     │ │
│  │       has = true                                                  │ │
│  │       break                                                       │ │
│  │     }                                                             │ │
│  │   }                                                               │ │
│  │                                                                  │ │
│  │   if !has {                                                       │ │
│  │     // 权限不足                                                   │ │
│  │     return echo.NewHTTPError(http.StatusForbidden,               │ │
│  │         fmt.Sprintf("permission denied: %s", perm))              │ │
│  │   }                                                               │ │
│  │ }                                                                 │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

#### 最小请求示例

##### 示例 1：使用 Python requests
```python
import requests
import os

API_USER = os.getenv('LISTMONK_API_USER', 'myapp')
API_TOKEN = os.getenv('LISTMONK_API_TOKEN', 'abc123...')

BASE_URL = 'https://listmonk.example.com'

headers = {
    'Authorization': f'token {API_USER}:{API_TOKEN}',
    'Accept': 'application/json',
    'Content-Type': 'application/json'
}

# 获取邮件列表
response = requests.get(
    f'{BASE_URL}/api/lists',
    headers=headers
)

if response.status_code == 200:
    lists = response.json()['data']
    print(f'Found {lists["total"]} lists')
elif response.status_code == 403:
    error = response.json()
    print(f'Auth failed: {error["message"]}')
```

##### 示例 2：发送事务性邮件
```python
# 发送事务性邮件（需要 "tx:send" 权限）
response = requests.post(
    f'{BASE_URL}/api/tx',
    headers=headers,
    json={
        'subscriber_email': 'user@example.com',
        'template_id': 1,
        'data': {
            'name': 'John Doe',
            'order_id': 'ORD-12345'
        }
    }
)
```

**权限不足响应**：
```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "message": "permission denied: tx:send"
}
```

---

## 五、两种路由组的认证失败返回差异

### 5.1 管理界面路由（`/admin/*`）

**注册代码**（`cmd/handlers.go:52-77`）：
```go
// 认证的非 API 路由（管理界面）
g := e.Group("", a.auth.Middleware, func(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        u := c.Get(auth.UserHTTPCtxKey)

        // ⚠️ 认证失败：重定向到登录页面
        if _, ok := u.(*echo.HTTPError); ok {
            u, _ := url.Parse(a.urlCfg.LoginURL)
            q := url.Values{}
            q.Set("next", c.Request().RequestURI)  // 保存原始请求 URI
            u.RawQuery = q.Encode()
            return c.Redirect(http.StatusTemporaryRedirect, u.String())  // 307
        }

        return next(c)
    }
})

// 注册的端点
g.GET(path.Join(uriAdmin, ""), a.AdminPage)           // /admin
g.GET(path.Join(uriAdmin, "/*"), a.AdminPage)         // /admin/*
```

**响应特征**：
- HTTP 状态码：`307 Temporary Redirect`
- `Location` 头：指向 `/admin/login?next=...`
- 适合浏览器自动重定向

### 5.2 API 路由（`/api/*`）

**注册代码**（`cmd/handlers.go:79-98`）：
```go
// 认证的 /api/* 路由
{
    var (
        pm = a.auth.Perm  // 权限检查中间件

        g = e.Group("", a.auth.Middleware, func(next echo.HandlerFunc) echo.HandlerFunc {
            return func(c echo.Context) error {
                u := c.Get(auth.UserHTTPCtxKey)

                // ⚠️ 认证失败：直接返回 JSON 错误
                if err, ok := u.(*echo.HTTPError); ok {
                    return err  // 直接返回 403
                }

                return next(c)
            }
        })
    )

    // 注册的端点
    g.GET("/api/health", a.HealthCheck)
    g.GET("/api/lists", a.GetLists)
    g.POST("/api/subscribers", pm(a.CreateSubscriber, "subscribers:manage"))
    // ... 更多 API 端点
}
```

**响应特征**：
- HTTP 状态码：`403 Forbidden`
- `Content-Type: application/json`
- 响应体：`{"message": "invalid API credentials"}` 或 `{"message": "permission denied: xxx"}`
- 适合 API 客户端程序化处理

### 5.3 完整对比表

| 维度 | 管理界面路由 (`/admin/*`) | API 路由 (`/api/*`) |
|------|--------------------------|-------------------|
| **认证失败状态码** | 307 Temporary Redirect | 403 Forbidden |
| **响应类型** | 重定向（HTML） | JSON 错误 |
| **响应体** | 空（由浏览器处理重定向） | `{"message": "..."}` |
| **适用场景** | 浏览器 Web 界面 | 命令行工具、第三方应用 |
| **中间件后处理** | 检查错误并重定向 | 检查错误并直接返回 |

---

## 六、权限系统详解

### 6.1 权限模型

系统采用基于角色的访问控制（RBAC），权限定义于 `internal/auth/models.go:44-75`：

```go
const (
    // 列表权限
    PermListGetAll            = "lists:get_all"
    PermListManageAll         = "lists:manage_all"
    PermListManage            = "list:manage"
    PermListGet               = "list:get"

    // 订阅者权限
    PermSubscribersGet        = "subscribers:get"
    PermSubscribersGetAll     = "subscribers:get_all"
    PermSubscribersManage     = "subscribers:manage"
    PermSubscribersImport     = "subscribers:import"

    // 邮件活动权限
    PermCampaignsGet          = "campaigns:get"
    PermCampaignsManage       = "campaigns:manage"
    PermCampaignsSend         = "campaigns:send"

    // 事务性邮件权限
    PermTxSend                = "tx:send"

    // 管理权限
    PermUsersGet              = "users:get"
    PermUsersManage           = "users:manage"
    PermSettingsGet           = "settings:get"
    PermSettingsManage        = "settings:manage"
    // ... 更多权限
)
```

### 6.2 权限检查流程

```
请求到达
    │
    ▼
┌─────────────────┐
│ Auth.Middleware │  ──► 身份验证（Cookie 或 API Token）
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ Auth.Perm       │  ──► 权限检查（检查用户角色的 permissions）
│ (路由级别)       │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ 业务逻辑内部     │  ──► 细粒度权限检查（如列表级权限）
└─────────────────┘
```

### 6.3 超级管理员豁免

`SuperAdminRoleID = 1` 的用户跳过所有权限检查（`internal/auth/auth.go:344-347`）：
```go
// 如果当前用户是超级管理员，不进行权限检查
if u.UserRole.ID == SuperAdminRoleID {
    return next(c)
}
```

---

## 七、安全考虑与最佳实践

### 7.1 已实现的安全措施

| 措施 | 说明 | 代码位置 |
|------|------|---------|
| **时序攻击防护** | API Token 比较使用 `subtle.ConstantTimeCompare` | `internal/auth/auth.go:141` |
| **Cookie 安全** | `HttpOnly` 标志防止 XSS 窃取 | `internal/auth/auth.go:92` |
| **密码哈希** | 普通用户密码使用 bcrypt 存储 | `queries/users.sql:6-7` |
| **会话清理** | 每 12 小时清理过期会话 | `internal/auth/auth.go:75, 105-110` |
| **登录计时保护** | 登录请求至少耗时 100ms，防止时序攻击 | `cmd/auth.go:458-462` |
| **密码更改后会话失效** | 密码更改后销毁所有会话 | `cmd/auth.go:691-693` |

### 7.2 安全关注点

#### ⚠️ API Token 明文存储

**问题**：API 用户的 Token 以明文形式存储在数据库中。

**代码证据**（`queries/users.sql:8-10`）：
```sql
WHEN $6 = 'api'
    THEN $3  -- 直接存储，不做哈希
```

**风险**：
- 如果数据库泄露，所有 API Token 立即暴露
- 无法像密码那样通过哈希不可逆性保护

**建议**：
1. 考虑对 API Token 进行加密存储（使用应用层密钥）
2. 实现 Token 过期机制
3. 实现 Token 轮换机制
4. 定期审计 API 用户的使用情况

#### ⚠️ API Token 永不过期

**问题**：API Token 没有内置过期机制，一旦泄露可永久使用。

**建议**：
1. 在用户模型中添加 `token_expires_at` 字段
2. 实现 Token 轮换 API
3. 在中间件中检查 Token 过期时间

#### ⚠️ Session Cookie 未设置 `Secure` 标志

**检查**：从代码看，Cookie 配置只设置了 `HttpOnly`，未明确设置 `Secure`。

**建议**：
- 在生产环境（HTTPS）中启用 `Secure` 标志
- 防止 Cookie 通过 HTTP 明文传输

### 7.3 最佳实践建议

#### 对于 API 客户端：

1. **使用最小权限原则**：
   ```go
   // ❌ 不推荐：给 API 用户分配超级管理员角色
   user_role_id: 1
   
   // ✅ 推荐：创建专用角色，只分配必要权限
   permissions: ["subscribers:get", "campaigns:send"]
   ```

2. **安全存储 Token**：
   ```bash
   # ❌ 不推荐：硬编码在代码中
   curl -H "Authorization: token myapp:abc123..." ...
   
   # ✅ 推荐：使用环境变量
   export LISTMONK_API_TOKEN="abc123..."
   curl -H "Authorization: token myapp:$LISTMONK_API_TOKEN" ...
   ```

3. **定期轮换 Token**：
   - 更新 API 用户的密码（即 Token）
   - 更新后所有使用旧 Token 的客户端将失效

#### 对于 Web 管理界面：

1. **启用 2FA**：
   - 为所有管理员用户启用 TOTP 双因素认证
   - 路径：`/user/profile` → Two-Factor Authentication

2. **使用 OIDC 集成**：
   - 配置企业身份提供商（如 Google Workspace、Azure AD）
   - 集中管理用户身份和访问权限

3. **会话管理**：
   - 定期检查活跃会话
   - 密码更改后会话自动失效

---

## 八、完整代码引用

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 核心认证中间件 | `internal/auth/auth.go` | 282-333 |
| API Token 验证 | `internal/auth/auth.go` | 135-146 |
| Authorization 头解析 | `internal/auth/auth.go` | 431-464 |
| 权限检查中间件 | `internal/auth/auth.go` | 336-367 |
| 用户模型定义 | `internal/auth/models.go` | 84-122 |
| 用户类型常量 | `internal/auth/models.go` | 33-37 |
| 登录处理 | `cmd/auth.go` | 449-499 |
| 路由注册与错误处理 | `cmd/handlers.go` | 52-98 |
| 用户创建（含 API Token 生成） | `internal/core/users.go` | 44-74 |
| SQL 用户创建（存储逻辑） | `queries/users.sql` | 1-13 |
| SQL API Token 查询 | `queries/users.sql` | 131-132 |
| SQL 登录验证（bcrypt） | `queries/users.sql` | 134-141 |
| 前端 API 配置 | `frontend/src/api/index.js` | 8-16 |
| 前端初始化流程 | `frontend/src/main.js` | 34-88 |

---

## 九、总结

### 9.1 关键澄清

1. **JWT 不存在**：Listmonk 不使用 JWT 进行鉴权，所谓的 "JWT" 实际是：
   - **API Token**：简单的 `api_username:random_string` 格式，明文存储
   - **OIDC ID Token**：仅用于第三方登录验证，不用于后续 API 调用

2. **Cookie 与 Authorization 并存时**：
   - **Session Cookie 优先**！如果存在 `session` Cookie，Authorization 头会被忽略
   - 这是 v3 到 v4 的向后兼容措施

3. **认证失败响应差异**：
   - `/admin/*` 路由：**307 重定向**到登录页
   - `/api/*` 路由：**403 JSON 错误**

### 9.2 三种客户端鉴权方式对照

| 客户端类型 | 鉴权机制 | 用户类型 | 请求示例 | 失败响应 |
|-----------|---------|---------|---------|---------|
| **Vue 管理界面** | Session Cookie | `UserTypeUser` | `Cookie: session=abc123...` | 307 重定向 |
| **命令行工具** | API Token | `UserTypeAPI` | `Authorization: token user:token` | 403 JSON |
| **第三方应用** | API Token | `UserTypeAPI` | `Authorization: token user:token` | 403 JSON |

### 9.3 安全建议

1. **API Token 安全**：
   - API Token 明文存储，需特别注意保护
   - 使用最小权限原则分配角色
   - 定期轮换 Token

2. **生产环境配置**：
   - 启用 HTTPS
   - 考虑设置 Cookie 的 `Secure` 标志
   - 为管理员启用 2FA

3. **监控与审计**：
   - 定期检查 API 用户列表
   - 监控异常 API 调用
   - 及时撤销不再使用的 API Token

---

## 十、高风险冲突场景分析

### 10.1 问题描述

**核心问题：当请求同时携带**：
1. **过期或无效的 `session` Cookie
2. **有效的** `Authorization` 鉴权头

**当前行为**：中间件返回 `"invalid session"` 错误，**不会回退到 API Token 验证。

**预期行为**：当 Session 无效时，应该尝试使用有效的 API Token。

---

### 10.2 当前中间件逻辑详细分析

#### 10.2.1 完整执行流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    当前 Auth.Middleware 执行流程                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  请求携带：                                                                  │
│  Cookie: session=expired_token_invalid                                     │
│  Authorization: token myapi:valid_token_123                                │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤 1: 获取 Authorization 头                                       │   │
│  │ hdr = "token myapi:valid_token_123"                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                     │
│         ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤 2: 检查 Cookie 中是否有 "session="                             │   │
│  │                                                                      │   │
│  │ if strings.Contains(cookie, "session=") {                           │   │
│  │     hdr = ""  // ⚠️ 清空 Authorization 头！                         │   │
│  │ }                                                                    │   │
│  │                                                                      │   │
│  │ 结果：hdr = "" (即使 session 已过期/无效！)                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                     │
│         ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤 3: 检查 hdr 是否为空                                           │   │
│  │                                                                      │   │
│  │ if len(hdr) > 0 {     // hdr 现在是空的，跳过这一步              │   │
│  │     // 尝试 API Token 验证（永远不会执行！）                         │   │
│  │ }                                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                     │
│         ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤 4: 尝试会话验证（唯一路径！）                                     │   │
│  │                                                                      │   │
│  │ sess, user, err := o.validateSession(c)                            │   │
│  │                                                                      │   │
│  │ // validateSession 内部：                                             │   │
│  │ sess, err := o.sess.Acquire(...)      // 尝试从数据库获取会话      │   │
│  │                                                                      │   │
│  │ // 结果：err != nil（session 不存在或已过期）                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                     │
│         ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤 5: 设置错误并返回                                               │   │
│  │                                                                      │   │
│  │ c.Set(UserHTTPCtxKey, echo.NewHTTPError(                          │   │
│  │     http.StatusForbidden, "invalid session"  // ⚠️ 误导性错误！  │   │
│  │ ))                                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  最终结果：                                                                  │
│  - 有效的 API Token 被完全忽略                                               │
│  - 返回 "invalid session" 错误（不是 "invalid API credentials"）            │
│  - 用户/开发者困惑：为什么带了正确的 Token 还被拒绝？                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 10.2.2 关键代码位置

**问题代码**（`internal/auth/auth.go:282-333`）：

```go
func (o *Auth) Middleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        // 步骤 1: 获取 Authorization 头
        hdr := strings.TrimSpace(c.Request().Header.Get("Authorization"))

        // 步骤 2: ⚠️ 问题所在！
        // 如果有 session Cookie，无条件清空 Authorization 头
        // 不检查 session 是否有效！
        if c := strings.TrimSpace(c.Request().Header.Get("Cookie")); strings.Contains(c, "session=") {
            hdr = ""  // 清空！
        }

        // 步骤 3: 尝试 API Token 验证（只有 hdr 非空时执行）
        if len(hdr) > 0 {
            key, token, err := parseAuthHeader(hdr)
            // ... 验证 API Token
        }

        // 步骤 4: 尝试会话验证（唯一可能的路径）
        sess, user, err := o.validateSession(c)
        if err != nil {
            // 步骤 5: 返回会话错误
            c.Set(UserHTTPCtxKey, echo.NewHTTPError(http.StatusForbidden, "invalid session"))
            return next(c)
        }
        // ...
    }
}
```

---

### 10.3 真实触发场景

#### 场景 1：开发人员混用浏览器和 API 工具

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  时间线                                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Day 1:                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 开发人员用浏览器登录管理界面                                            │   │
│  │                                                                      │   │
│  │ 1. POST /admin/login (username=admin, password=secret)            │   │
│  │ 2. 响应 Set-Cookie: session=valid_session_abc123; Max-Age=604800  │   │
│  │ 3. 浏览器保存 Cookie 7 天                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Day 8: (7 天后)                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Session 已过期，但浏览器仍在发送 Cookie！                               │   │
│  │                                                                      │   │
│  │ Cookie: session=valid_session_abc123 (已过期，数据库中已不存在)    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  同一时刻：                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 开发人员用 Postman 测试 API，配置了有效的 API Token                  │   │
│  │                                                                      │   │
│  │ 但是！Postman 运行在同一浏览器中（Chrome 扩展）                     │   │
│  │ 或者：开发人员从浏览器复制 Cookie 到 Postman 调试                   │   │
│  │                                                                      │   │
│  │ 实际发送的请求：                                                    │   │
│  │ GET /api/lists                                                      │   │
│  │ Cookie: session=valid_session_abc123 (已过期)                    │   │
│  │ Authorization: token myapi:valid_token_123 (有效)                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  结果：                                                                      │
│  - Authorization 头被清空                                                          │
│  - 会话验证失败                                                              │
│  - 返回 "invalid session" 错误                                             │
│  - 开发人员困惑：为什么带了正确的 Token 还被拒绝？                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 场景 2：浏览器扩展调用 API

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  场景：用户同时使用：                                                       │
│  1. 浏览器中的 Listmonk 管理界面（有 session Cookie）                       │
│  2. 第三方浏览器扩展（配置了 API Token）                                   │
│                                                                             │
│  问题：                                                                      │
│  - 扩展发起 API 请求时：                                                    │
│    - 浏览器自动附带所有 Cookie（包括 session）                               │
│    - 扩展同时设置 Authorization 头                                          │
│                                                                             │
│  时间线：                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 1. 用户在浏览器中访问 Listmonk 管理界面                                │ │
│  │    → 获得 session Cookie                                              │ │
│  │                                                                      │ │
│  │ 2. 用户安装浏览器扩展，配置 API Token                                 │ │
│  │    扩展代码：                                                         │ │
│  │    fetch('https://listmonk.example.com/api/lists', {               │ │
│  │      headers: {                                                       │ │
│  │        'Authorization': 'token myext:valid_token'                   │ │
│  │      }                                                                │ │
│  │    })                                                                 │ │
│  │                                                                      │ │
│  │ 3. 浏览器实际发送的请求：                                             │ │
│  │    GET /api/lists                                                    │ │
│  │    Cookie: session=abc123... (来自管理界面)                         │ │
│  │    Authorization: token myext:valid_token (来自扩展)                 │ │
│  │                                                                      │ │
│  │ 4. 服务器处理：                                                        │ │
│  │    - 检测到 session Cookie → 清空 Authorization 头                  │ │
│  │    - 尝试验证 session → 可能过期/无效                                │ │
│  │    - 返回 "invalid session"                                           │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  结果：扩展无法正常工作，用户困惑为什么配置了正确的 Token                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 场景 3：嵌入式 iframe / 微前端

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  架构：                                                                      │
│  - 主应用（Portal）使用 session Cookie 认证                                │
│  - Listmonk 作为微前端嵌入在 iframe 中，使用 API Token                     │
│                                                                             │
│  问题：                                                                      │
│  - 主应用的 Cookie 会被发送到 iframe 中的请求                              │
│  - 即使 iframe 设置了 Authorization 头                                      │
│                                                                             │
│  请求流程：                                                                  │
│  主应用 (Portal)                                                             │
│       │                                                                     │
│       │ Set-Cookie: session=portal_session...                             │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  iframe (Listmonk 微前端)                                         │   │
│  │                                                                      │   │
│  │  发起 API 请求：                                                    │   │
│  │  GET /api/lists                                                      │   │
│  │  Cookie: session=portal_session... (来自父级，对 Listmonk 无效)   │   │
│  │  Authorization: token microfrontend:valid_token (iframe 设置)      │   │
│  │                                                                      │   │
│  │  结果：                                                               │   │
│  │  - 服务器检测到 session Cookie (虽然是其他应用的)                   │   │
│  │  - 清空 Authorization 头                                             │   │
│  │  - 尝试验证 session → 失败                                           │   │
│  │  - 返回 "invalid session"                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 10.4 影响面分析

| 影响维度 | 描述 | 严重程度 |
|---------|------|---------|
| **可用性** | 合法的 API 请求被错误拒绝 | **高** |
| **可调试性** | 错误信息误导（"invalid session" 而非 "invalid API credentials"） | **高** |
| **用户体验** | 用户/开发者困惑：为什么带了正确的 Token 还被拒绝？ | **中** |
| **安全性** | 无直接安全问题，但可能导致不当 workaround | **低** |

#### 具体影响场景

1. **开发环境**：
   - 开发人员调试 API 时困惑
   - 浪费时间排查问题
   - 可能导致开发者使用不当的 workaround（如手动清除 Cookie）

2. **生产环境**：
   - 浏览器扩展无法正常工作
   - 微前端架构集成失败
   - 自动化脚本在特定条件下失败

3. **升级场景**：
   - v3 -> v4 升级后，用户可能同时有：
     - 旧的 Basic Auth 凭证（浏览器保存）
     - 新的 Session Cookie
     - 新的 API Token（用于自动化）
   - 这可能导致复杂的交互问题

---

### 10.5 最小可复现请求示例

#### 前置条件

1. 创建一个 API 用户并获取 Token：
   ```http
   POST /api/users
   Authorization: token admin:admin_password
   Content-Type: application/json
   
   {
     "username": "testapi",
     "name": "Test API User",
     "type": "api",
     "user_role_id": 1
   }
   
   响应：
   {
     "data": {
       "id": 5,
       "username": "testapi",
       "password": "abc123def456valid"  // 保存这个 Token
     }
   }
   ```

2. 确认 API Token 单独有效：
   ```bash
   curl http://localhost:9000/api/lists \
     -H "Authorization: token testapi:abc123def456valid"
   # 应该返回 200 OK
   ```

#### 可复现请求

```bash
# 场景：同时携带无效 session Cookie 和有效 API Token

# 使用 curl 模拟：
curl -v http://localhost:9000/api/lists \
  -H "Cookie: session=this_is_an_invalid_session_token_that_does_not_exist_in_db" \
  -H "Authorization: token testapi:abc123def456valid" \
  -H "Accept: application/json"
```

**预期行为**（应该使用有效的 API Token，返回 200 OK。

**实际行为**（当前代码）：返回 403 Forbidden。

#### 不同场景测试

```bash
# 场景 1: 只有有效 API Token（应该成功）
curl http://localhost:9000/api/lists \
  -H "Authorization: token testapi:abc123def456valid"
# ✅ 预期：200 OK
# ✅ 实际：200 OK

# 场景 2: 只有无效 session Cookie（应该失败）
curl http://localhost:9000/api/lists \
  -H "Cookie: session=invalid_session"
# ✅ 预期：403
# ✅ 实际：403

# 场景 3: 无效 session Cookie + 有效 API Token（BUG！）
curl http://localhost:9000/api/lists \
  -H "Cookie: session=invalid_session" \
  -H "Authorization: token testapi:abc123def456valid"
# ❌ 预期：200 OK（应该使用有效的 API Token）
# ❌ 实际：403 Forbidden（返回 "invalid session"）
```

---

### 10.6 v3->v4 兼容意图的深入分析

#### 原始兼容问题

从代码注释（`internal/auth/auth.go:291-297`）：

```
// If cookie is set, ignore BasicAuth. This is to preserve backwards compatibility
// in v3 -> v4 upgrade where the user browser sessions would still have old
// BasicAuth credentials, which no longer work in the new system which expects
// session cookies instead, which causes a redirect loop despite loggin in and session
// cookies being set.
```

#### v3 vs v4 认证机制对比

| 版本 | 认证机制 | 说明 |
|------|---------|------|
| **v3** | HTTP Basic Auth | 浏览器保存用户名密码，每次请求自动发送 |
| **v4** | Session Cookie | 登录后获得 Cookie，7 天过期 |

#### v3->v4 升级时的问题场景

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  v3 -> v4 升级时的问题场景                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  v3 时代：                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 用户访问管理界面                                                        │   │
│  │ 1. 浏览器弹出 Basic Auth 对话框                                       │   │
│  │ 2. 用户输入 admin / secret                                           │   │
│  │ 3. 浏览器保存这些凭证，每次请求自动发送                                │   │
│  │                                                                      │   │
│  │ 每次请求：                                                          │   │
│  │ GET /admin                                                          │   │
│  │ Authorization: Basic YWRtaW46c2VjcmV0 (base64(admin:secret))        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  升级到 v4 后：                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 用户用新的 Session Cookie 机制登录                                       │   │
│  │                                                                      │   │
│  │ 1. POST /admin/login (username=admin, password=secret)            │   │
│  │ 2. 响应 Set-Cookie: session=new_session_abc123                     │   │
│  │                                                                      │   │
│  │ 问题！浏览器仍然保存着旧的 Basic Auth 凭证！                          │   │
│  │                                                                      │   │
│  │ 后续请求：                                                          │   │
│  │ GET /admin                                                          │   │
│  │ Cookie: session=new_session_abc123 (新的，有效)                       │   │
│  │ Authorization: Basic YWRtaW46c2VjcmV0 (旧的，v4 不再使用)        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  v4 中间件处理（如果没有兼容逻辑）：                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. 检测到 Authorization 头（Basic Auth）                               │   │
│  │ 2. 尝试验证 API Token                                             │   │
│  │ 3. Basic Auth 格式的用户可能：                                           │   │
│  │    - v3: admin 用户现在需要 bcrypt 哈希验证                               │   │
│  │    - v4: API 用户需要明文比较                                      │   │
│  │ 4. 验证失败（可能失败或行为异常）                                      │   │
│  │ 5. 返回 403 错误                                                   │   │
│  │ 6. 管理界面中间件看到 403 → 重定向到登录页                         │   │
│  │ 7. 用户已经登录了！又被重定向到登录页 → 重定向循环！                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  这就是为什么需要兼容逻辑：                                                 │
│  - 如果有 session Cookie，忽略 Authorization 头                              │
│  - 避免重定向循环                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 兼容逻辑的关键假设

**原始设计假设**：
- 如果有 session Cookie → 用户已经通过 v4 机制登录
- 应该优先使用 session Cookie
- 忽略 Authorization 头（可能是旧的 Basic Auth 凭证）

**这个假设的问题**：
- ❌ session Cookie 可能是**无效的/过期的**
- ❌ Authorization 头可能是**有效的 API Token**（不是旧的 Basic Auth）
- ❌ 没有回退机制

#### 需要区分的两种情况

| 情况 | Authorization 头类型 | 应该忽略？ |
|------|------------------|---------|
| A | 旧的 Basic Auth（v3 遗留）| ✅ 应该忽略（有有效 session 时） |
| B | 新的 Token 格式（`token user:token`）| ❌ 不应该忽略（应该作为回退） |

**关键洞察**：
- v3->v4 兼容只需要忽略 **Basic Auth** 格式
- **Token 格式**是 v4 的新格式，应该被尊重

---

### 10.7 改进方案设计

#### 方案评估标准

任何改进方案必须满足：

| 标准 | 说明 |
|------|------|
| **不破坏 v3->v4 兼容** | 有有效 session Cookie + 旧 Basic Auth → 使用 session |
| **修复当前问题** | 有无效 session Cookie + 有效 API Token → 使用 API Token |
| **保持现有行为** | 无 session Cookie → 优先 API Token，然后 session |
| **错误信息准确** | 哪种认证方式失败，返回对应错误信息 |

#### 方案 1：最简修复 - 会话失败后回退

**核心思想**：
- 保持现有优先顺序（有 session Cookie 时优先尝试 session）
- 但 session 验证**失败后**，回退尝试 API Token

**修改后的逻辑流程**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    改进后的 Auth.Middleware 执行流程                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  请求携带：                                                                  │
│  Cookie: session=expired_invalid                                              │
│  Authorization: token myapi:valid_token_123                                │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤 1: 保存原始 Authorization 头（用于可能的回退）                  │   │
│  │ originalHdr = "token myapi:valid_token_123"                      │   │
│  │                                                                      │   │
│  │ 步骤 2: 检查是否有 session Cookie                                    │   │
│  │ hasSessionCookie = true                                             │   │
│  │                                                                      │   │
│  │ 步骤 3: 有 session Cookie 时，优先尝试 session                      │   │
│  │                                                                      │   │
│  │ sess, user, err := o.validateSession(c)                              │   │
│  │                                                                      │   │
│  │ 分支 A: session 有效（v3->v4 兼容场景）                               │   │
│  │ ┌─────────────────────────────────────────────────────────────┐   │
│  │ │ if err == nil {                                               │   │
│  │ │   // session 有效，使用它                                        │   │
│  │ │   c.Set(UserHTTPCtxKey, user)                                  │   │
│  │ │   // ✅ 保持 v3->v4 兼容                                       │   │
│  │ │   return next(c)                                               │   │
│  │ │ }                                                              │   │
│  └─┴─────────────────────────────────────────────────────────────────┘   │
│         │                                                                     │
│         │ session 无效（err != nil）                                          │
│         ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤 4: session 无效，回退尝试 API Token（新增！）                    │   │
│  │                                                                      │   │
│  │ if len(originalHdr) > 0 {                                          │   │
│  │     key, token, err := parseAuthHeader(originalHdr)              │   │
│  │     if err == nil {                                                │   │
│  │         user, ok := o.GetAPIToken(key, token)                     │   │
│  │         if ok {                                                     │   │
│  │             // API Token 有效！使用它                               │   │
│  │             c.Set(UserHTTPCtxKey, user)                            │   │
│  │             return next(c)                                           │   │
│  │         }                                                            │   │
│  │     }                                                                │   │
│  │ }                                                                    │   │
│  │                                                                      │   │
│  │ // 两种方式都失败                                                    │   │
│  │ c.Set(UserHTTPCtxKey, echo.NewHTTPError(                            │   │
│  │     http.StatusForbidden, "invalid session or API credentials"       │   │
│  │ ))                                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**代码实现**：

```go
func (o *Auth) Middleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        // 保存原始 Authorization 头，用于可能的回退
        originalHdr := strings.TrimSpace(c.Request().Header.Get("Authorization"))

        // 检查是否有 session Cookie
        hasSessionCookie := strings.Contains(
            strings.TrimSpace(c.Request().Header.Get("Cookie")),
            "session=",
        )

        // ============================================
        // 决策逻辑：
        // - 有 session Cookie：先尝试 session，失败后回退到 API Token
        // - 无 session Cookie：先尝试 API Token，失败后尝试 session
        // ============================================

        if hasSessionCookie {
            // 有 session Cookie：优先尝试 session（保持 v3->v4 兼容）
            sess, user, sessErr := o.validateSession(c)
            if sessErr == nil {
                // session 有效，使用它
                c.Set(UserHTTPCtxKey, user)
                c.Set(SessionKey, sess)
                return next(c)
            }

            // session 无效，回退尝试 API Token（如果有）
            if len(originalHdr) > 0 {
                key, token, err := parseAuthHeader(originalHdr)
                if err == nil {
                    apiUser, ok := o.GetAPIToken(key, token)
                    if ok {
                        // API Token 有效，使用它
                        c.Set(UserHTTPCtxKey, apiUser)
                        return next(c)
                    }
                }
            }

            // 两种方式都失败
            c.Set(UserHTTPCtxKey, echo.NewHTTPError(http.StatusForbidden, "invalid session or API credentials"))
            return next(c)
        }

        // 无 session Cookie：保持原有行为
        // 先尝试 API Token，再尝试 session

        if len(originalHdr) > 0 {
            key, token, err := parseAuthHeader(originalHdr)
            if err != nil {
                c.Set(UserHTTPCtxKey, echo.NewHTTPError(http.StatusForbidden, err.Error()))
                return next(c)
            }

            user, ok := o.GetAPIToken(key, token)
            if !ok {
                c.Set(UserHTTPCtxKey, echo.NewHTTPError(http.StatusForbidden, "invalid API credentials"))
                return next(c)
            }

            c.Set(UserHTTPCtxKey, user)
            return next(c)
        }

        // 尝试 session
        sess, user, err := o.validateSession(c)
        if err != nil {
            c.Set(UserHTTPCtxKey, echo.NewHTTPError(http.StatusForbidden, "invalid session"))
            return next(c)
        }

        c.Set(UserHTTPCtxKey, user)
        c.Set(SessionKey, sess)
        return next(c)
    }
}
```

#### 方案 2：更精确的修复 - 区分 Basic Auth 和 Token 格式

**核心思想**：
- 只有当 Authorization 头是 **Basic Auth** 格式时才忽略（v3 遗留）
- **Token 格式**（`token user:token`）应该始终被尊重

**代码实现**：

```go
func (o *Auth) Middleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        hdr := strings.TrimSpace(c.Request().Header.Get("Authorization"))

        hasSessionCookie := strings.Contains(
            strings.TrimSpace(c.Request().Header.Get("Cookie")),
            "session=",
        )

        // 区分 Authorization 头类型
        isBasicAuth := strings.HasPrefix(hdr, "Basic ")
        isTokenAuth := strings.HasPrefix(hdr, "token ")

        // ============================================
        // v3->v4 兼容逻辑：
        // - 有 session Cookie + Basic Auth（v3 遗留）→ 忽略 Basic Auth
        // - 有 session Cookie + Token 格式（v4 新格式）→ 不忽略
        // ============================================

        // 保存原始 hdr 用于可能的回退
        effectiveHdr := hdr

        if hasSessionCookie && isBasicAuth {
            // 只有 Basic Auth 格式才忽略（v3 遗留）
            effectiveHdr = ""
        }

        // 尝试 API Token 验证
        if len(effectiveHdr) > 0 {
            key, token, err := parseAuthHeader(effectiveHdr)
            if err != nil {
                c.Set(UserHTTPCtxKey, echo.NewHTTPError(http.StatusForbidden, err.Error()))
                return next(c)
            }

            user, ok := o.GetAPIToken(key, token)
            if !ok {
                c.Set(UserHTTPCtxKey, echo.NewHTTPError(http.StatusForbidden, "invalid API credentials"))
                return next(c)
            }

            c.Set(UserHTTPCtxKey, user)
            return next(c)
        }

        // 尝试 session 验证
        sess, user, err := o.validateSession(c)
        if err != nil {
            // session 失败

            // ⚠️ 新增：如果是因为忽略了 Basic Auth，回退尝试它
            // 但这可能导致与 v3->v4 兼容的原始意图冲突...
            // 需要更仔细的考虑

            // 简单处理：返回错误
            c.Set(UserHTTPCtxKey, echo.NewHTTPError(http.StatusForbidden, "invalid session"))
            return next(c)
        }

        c.Set(UserHTTPCtxKey, user)
        c.Set(SessionKey, sess)
        return next(c)
    }
}
```

**方案 2 的问题**：
- 只解决了 Token 格式被错误忽略的问题
- 但没有解决"有效 session + 有效 API Token"的优先级问题
- 也没有解决"无效 session + 有效 API Token"的回退问题

#### 方案对比

| 维度 | 方案 1（推荐） | 方案 2 |
|------|---------------|--------|
| **修复核心问题** | ✅ 会话失败后回退 | ⚠️ 部分修复（只区分格式） |
| **v3->v4 兼容** | ✅ 有有效 session 时优先使用 | ✅ 忽略 Basic Auth |
| **Token 格式支持** | ✅ 作为回退 | ✅ 不被忽略 |
| **代码改动量** | 中等 | 较小 |
| **行为一致性** | ✅ 逻辑清晰（优先 session，失败回退） | ⚠️ 较复杂（按格式区分） |

**推荐方案**：**方案 1**

理由：
1. 逻辑更清晰：有 session Cookie 时优先尝试 session
2. session 无效时回退是合理的预期行为
3. 完全保持 v3->v4 兼容（有效 session 时不回退）
4. 修复了所有已知问题场景

---

### 10.8 方案 1 的详细测试场景

#### 测试场景矩阵

| # | 场景 | Session Cookie | API Token | 预期行为 | 方案 1 行为 |
|---|------|----------------|-----------|---------|------------|
| 1 | 只有有效 session | ✅ 有效 | ❌ 无 | 使用 session | ✅ 使用 session |
| 2 | 只有无效 session | ❌ 无效 | ❌ 无 | 403 错误 | ✅ 403 错误 |
| 3 | 只有有效 API Token | ❌ 无 | ✅ 有效 | 使用 API Token | ✅ 使用 API Token |
| 4 | 有效 session + 有效 API Token | ✅ 有效 | ✅ 有效 | 使用 session（v3->v4 兼容） | ✅ 使用 session |
| 5 | **无效 session + 有效 API Token**（核心问题） | ❌ 无效 | ✅ 有效 | 使用 API Token | ✅ 使用 API Token（修复！） |
| 6 | 有效 session + Basic Auth（v3 遗留） | ✅ 有效 | Basic Auth | 使用 session（v3->v4 兼容） | ✅ 使用 session |
| 7 | 无效 session + Basic Auth | ❌ 无效 | Basic Auth | 403 或回退？ | ✅ 回退尝试（可配置） |

#### 场景 5 详细验证（核心问题）

```
改进前（当前代码）：
┌─────────────────────────────────────────────────────────────────────┐
│ 请求：                                                              │
│ Cookie: session=invalid_expired                                      │
│ Authorization: token myapi:valid_token                                  │
│                                                                      │
│ 处理流程：                                                          │
│ 1. 检测到 session Cookie → 清空 Authorization 头                  │
│ 2. 尝试验证 session → 失败                                          │
│ 3. 返回 "invalid session"                                           │
│                                                                      │
│ 结果：❌ 403 Forbidden（应该成功！）                              │
└─────────────────────────────────────────────────────────────────────┘

改进后（方案 1）：
┌─────────────────────────────────────────────────────────────────────┐
│ 请求：                                                              │
│ Cookie: session=invalid_expired                                      │
│ Authorization: token myapi:valid_token                                  │
│                                                                      │
│ 处理流程：                                                          │
│ 1. 保存 originalHdr = "token myapi:valid_token"                      │
│ 2. 检测到 hasSessionCookie = true                                   │
│ 3. 尝试验证 session → 失败（err != nil）                             │
│ 4. 检查 originalHdr 非空 → 回退尝试 API Token                      │
│ 5. 解析 API Token → 验证成功                                          │
│ 6. 使用 API Token，返回 200 OK                                       │
│                                                                      │
│ 结果：✅ 200 OK（正确行为）                                       │
└─────────────────────────────────────────────────────────────────────┘
```

#### 场景 4 验证（v3->v4 兼容保持）

```
场景：有效 session + 旧 Basic Auth（v3 遗留）

改进前（当前代码）：
┌─────────────────────────────────────────────────────────────────────┐
│ 请求：                                                              │
│ Cookie: session=valid_session                                   │
│ Authorization: Basic YWRtaW46c2VjcmV0（旧 Basic Auth）              │
│                                                                      │
│ 处理流程：                                                          │
│ 1. 检测到 session Cookie → 清空 Authorization 头                  │
│ 2. 尝试验证 session → 成功                                          │
│ 3. 使用 session，返回 200 OK                                         │
│                                                                      │
│ 结果：✅ 200 OK（正确行为，避免重定向循环）                           │
└─────────────────────────────────────────────────────────────────────┘

改进后（方案 1）：
┌─────────────────────────────────────────────────────────────────────┐
│ 请求：                                                              │
│ Cookie: session=valid_session                                      │
│ Authorization: Basic YWRtaW46c2VjcmV0（旧 Basic Auth）                  │
│                                                                      │
│ 处理流程：                                                          │
│ 1. 保存 originalHdr = "Basic YWRtaW46c2VjcmV0"                       │
│ 2. 检测到 hasSessionCookie = true                                   │
│ 3. 尝试验证 session → 成功（err == nil）                            │
│ 4. 使用 session，返回 200 OK                                           │
│ 5. ⚠️ 不会回退到 API Token（因为 session 成功了）                   │
│                                                                      │
│ 结果：✅ 200 OK（保持 v3->v4 兼容）                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 10.9 完整改进代码

#### 推荐的最终实现

```go
// Middleware is the HTTP middleware used for wrapping HTTP handlers registered on the echo router.
// It authorizes token (BasicAuth/token) based and cookie based sessions and on successful auth,
// sets the authenticated User{} on the echo context on the key UserKey. On failure, it sets an Error{}
// instead on the same key.
func (o *Auth) Middleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        // 保存原始 Authorization 头，用于可能的回退
        originalHdr := strings.TrimSpace(c.Request().Header.Get("Authorization"))

        // 检查是否有 session Cookie
        hasSessionCookie := strings.Contains(
            strings.TrimSpace(c.Request().Header.Get("Cookie")),
            "session=",
        )

        // ============================================
        // 决策逻辑：
        // - 有 session Cookie：先尝试 session
        //   - session 有效：使用 session（v3->v4 兼容）
        //   - session 无效：回退尝试 API Token
        // - 无 session Cookie：先尝试 API Token，再尝试 session
        // ============================================

        if hasSessionCookie {
            // 有 session Cookie：优先尝试 session
            sess, user, sessErr := o.validateSession(c)
            if sessErr == nil {
                // session 有效，使用它
                // 这保持了 v3->v4 兼容：有有效 session 时忽略 Authorization
                c.Set(UserHTTPCtxKey, user)
                c.Set(SessionKey, sess)
                return next(c)
            }

            // session 无效，回退尝试 API Token（如果有）
            if len(originalHdr) > 0 {
                key, token, err := parseAuthHeader(originalHdr)
                if err == nil {
                    apiUser, ok := o.GetAPIToken(key, token)
                    if ok {
                        // API Token 有效，使用它
                        c.Set(UserHTTPCtxKey, apiUser)
                        return next(c)
                    }
                }
            }

            // 两种方式都失败
            c.Set(UserHTTPCtxKey, echo.NewHTTPError(http.StatusForbidden, "invalid session or API credentials"))
            return next(c)
        }

        // 无 session Cookie：保持原有行为
        // 先尝试 API Token，再尝试 session

        if len(originalHdr) > 0 {
            key, token, err := parseAuthHeader(originalHdr)
            if err != nil {
                c.Set(UserHTTPCtxKey, echo.NewHTTPError(http.StatusForbidden, err.Error()))
                return next(c)
            }

            user, ok := o.GetAPIToken(key, token)
            if !ok {
                c.Set(UserHTTPCtxKey, echo.NewHTTPError(http.StatusForbidden, "invalid API credentials"))
                return next(c)
            }

            c.Set(UserHTTPCtxKey, user)
            return next(c)
        }

        // 尝试 session
        sess, user, err := o.validateSession(c)
        if err != nil {
            c.Set(UserHTTPCtxKey, echo.NewHTTPError(http.StatusForbidden, "invalid session"))
            return next(c)
        }

        c.Set(UserHTTPCtxKey, user)
        c.Set(SessionKey, sess)
        return next(c)
    }
}
```

#### 关键改进点总结

| 改进点 | 原始代码 | 改进后代码 |
|--------|---------|-----------|
| **保存原始 Header** | 不保存 | `originalHdr` 保存原始值 |
| **Session 失败后** | 直接返回错误 | 回退尝试 API Token |
| **错误信息** | "invalid session"（误导） | "invalid session or API credentials"（准确） |
| **v3->v4 兼容** | 有 session Cookie 就忽略 Authorization | 有**有效** session 才忽略 Authorization |

---

### 10.10 风险与缓解措施

#### 潜在风险

| 风险 | 描述 | 缓解措施 |
|------|------|---------|
| **行为变化** | 某些边缘场景的行为可能改变 | 全面测试所有场景 |
| **错误信息变化** | 错误信息从 "invalid session" 变为更通用的消息 | 更新文档，说明新的错误信息 |
| **性能影响** | session 失败后额外进行一次 API Token 验证 | API Token 验证是内存操作，性能影响可忽略 |

#### 测试建议

1. **单元测试**：
   - 测试所有 7 种场景
   - 特别关注场景 5（核心问题）和场景 4（v3->v4 兼容）

2. **集成测试**：
   - 测试浏览器扩展场景
   - 测试微前端/iframe 场景
   - 测试 v3->v4 升级场景

3. **手动测试**：
   ```bash
   # 场景 5：无效 session + 有效 API Token（核心问题）
   curl http://localhost:9000/api/lists \
     -H "Cookie: session=invalid_session" \
     -H "Authorization: token testapi:valid_token"
   # 应该返回 200 OK
   
   # 场景 4：有效 session + Basic Auth（v3->v4 兼容）
   curl http://localhost:9000/api/lists \
     -H "Cookie: session=valid_session" \
     -H "Authorization: Basic YWRtaW46c2VjcmV0"
   # 应该返回 200 OK（使用 session）
   ```

---

### 10.11 总结

#### 问题根源

1. **根本原因**：当前代码在**验证之前**就清空了 Authorization 头，不考虑 session 是否有效。

2. **设计假设**：
   - 原始假设：有 session Cookie → 用户已登录 → Authorization 头是旧的 Basic Auth
   - 实际情况：session Cookie 可能无效，Authorization 头可能是有效的 API Token

3. **错误信息**：
   - 返回 "invalid session" 而不是 "invalid API credentials"
   - 误导用户和开发者

#### 改进方案效果

| 场景 | 原始行为 | 改进后行为 |
|------|---------|-----------|
| 无效 session + 有效 API Token | ❌ 403 "invalid session" | ✅ 200 OK（使用 API Token） |
| 有效 session + Basic Auth | ✅ 使用 session | ✅ 使用 session（保持兼容） |
| 有效 session + API Token | ❌ API Token 被忽略 | ✅ 使用 session（预期行为） |
| 无 session + API Token | ✅ 使用 API Token | ✅ 使用 API Token |

#### 最终建议

**推荐实施方案 1**（会话失败后回退），因为：

1. ✅ 完全修复核心问题
2. ✅ 保持 v3->v4 兼容
3. ✅ 逻辑清晰，易于理解
4. ✅ 改动量适中，风险可控
5. ✅ 错误信息更准确

**实施步骤**：
1. 实现改进后的中间件代码
2. 添加单元测试覆盖所有场景
3. 更新错误处理文档
4. 在测试环境验证
5. 生产环境部署
