# Listmonk 前后端多语言同步机制分析报告

## 1. 概述

Listmonk 采用了一套精心设计的国际化（i18n）架构，实现了前后端共享同一套语言包，并通过多种机制确保语言包的同步和一致性。本报告深入分析其实现原理、加载机制和协作路径。

## 2. 语言包存储结构

### 2.1 统一数据源

语言包统一存储在项目根目录的 `i18n/` 文件夹下，前后端共用此目录：

```
i18n/
├── en.json          # 基准语言（英语）
├── zh-CN.json       # 简体中文
├── zh-TW.json       # 繁体中文
├── ja.json          # 日语
├── ko.json          # 韩语
├── fr.json          # 法语
├── de.json          # 德语
├── es.json          # 西班牙语
└── ... (共 40+ 种语言)
```

### 2.2 语言包格式

语言包采用 JSON 格式，具有以下特点：

**元数据字段**：
- `_.code` - 语言代码（如 "en", "zh-CN"）
- `_.name` - 语言显示名称（如 "English (en)", "简体中文 (zh-CN)"）

**翻译键格式**：
- 采用命名空间式键名：`module.submodule.key`
- 支持参数替换：`{paramName}`
- 支持复数形式：`"Singular | Plural"`

**示例** (`i18n/en.json:227-255`):
```json
{
    "_.code": "en",
    "_.name": "English (en)",
    
    "globals.terms.campaign": "Campaign | Campaigns",
    "globals.terms.list": "List | Lists",
    
    "globals.messages.errorCreating": "Error creating {name}: {error}",
    "globals.messages.notFound": "{name} not found"
}
```

### 2.3 复数形式规则

复数形式通过管道符 `|` 分隔：
- 单数形式：`n == 1` 时返回第一个部分
- 复数形式：`n > 1` 时返回第二个部分

示例：
```json
"globals.terms.subscriber": "Subscriber | Subscribers"
// Tc("globals.terms.subscriber", 1) → "Subscriber"
// Tc("globals.terms.subscriber", 5) → "Subscribers"
```

## 3. 后端国际化实现

### 3.1 核心包设计

后端 i18n 包位于 `internal/i18n/i18n.go`，其设计目标明确：

**模拟 vue-i18n 行为**（`i18n.go:1-3`）：
```go
// i18n is a simple package that translates strings using a language map.
// It mimics some functionality of the vue-i18n library so that the same JSON
// language map may be used in the JS frontend and the Go backend.
```

### 3.2 核心数据结构

```go
type I18n struct {
    Code    string            `json:"code"`
    Name    string            `json:"name"`
    langMap map[string]string // 内部翻译映射
}
```

### 3.3 翻译方法

后端提供与 vue-i18n 兼容的 API：

| 方法 | 功能 | 对应 vue-i18n |
|------|------|---------------|
| `T(key)` | 基础翻译 | `$t(key)` |
| `Ts(key, params...)` | 带参数替换的翻译 | `$t(key, {params})` |
| `Tc(key, n)` | 复数形式翻译 | `$tc(key, n)` |

### 3.4 参数替换机制

**正则表达式** (`i18n.go:20`):
```go
var reParam = regexp.MustCompile(`(?i)\{([a-z0-9-.]+)\}`)
```

**递归参数解析** (`i18n.go:148-164`):
```go
func (i *I18n) subAllParams(s string) string {
    // 递归解析所有 {param} 占位符
    // 支持嵌套引用：如 {other.key} 会被解析为 i.T("other.key")
}
```

### 3.5 语言加载策略

后端采用**分层加载**策略（`cmd/i18n.go:75-99`）：

1. **先加载默认语言**（en.json）作为基础
2. **再加载目标语言**覆盖对应键值
3. **未翻译的键保留英文默认值**

```go
func getI18nLang(lang string, fs stuffbin.FileSystem) (*i18n.I18n, bool, error) {
    const def = "en"
    
    // 1. 先加载默认语言
    b, _ := fs.Read(fmt.Sprintf("/i18n/%s.json", def))
    i, _ := i18n.New(b)
    
    // 2. 再加载目标语言进行覆盖
    b, err := fs.Read(fmt.Sprintf("/i18n/%s.json", lang))
    if err == nil {
        i.Load(b)
    }
    
    return i, true, nil
}
```

### 3.6 后端使用示例

在 `cmd/users.go` 和其他 handler 中广泛使用：

```go
// 基础翻译
a.i18n.T("globals.messages.invalidID")

// 带参数翻译
a.i18n.Ts("globals.messages.invalidFields", "name", "username")

// 复数形式
a.i18n.Tc("globals.terms.subscriber", count)
```

## 4. 前端国际化实现

### 4.1 技术栈

前端使用 `vue-i18n` 库进行国际化处理（`frontend/package.json:41`）：
```json
"vue-i18n": "^8.28.2"
```

### 4.2 初始化流程

**主入口文件** (`frontend/src/main.js:11-40`):

```javascript
// 1. 安装插件
Vue.use(VueI18n);
const i18n = new VueI18n();

// 2. 应用启动时异步加载语言包
async function initConfig(app) {
    // 并行获取用户配置和服务器配置
    const [profile, cfg] = await Promise.all([
        api.getUserProfile(), 
        api.getServerConfig()
    ]);
    
    // 3. 通过 API 从后端获取语言包
    const lang = await api.getLang(cfg.lang);
    
    // 4. 设置当前语言和消息
    i18n.locale = cfg.lang;
    i18n.setLocaleMessage(i18n.locale, lang);
    
    // ...
}
```

### 4.3 语言包 API 调用

**API 定义** (`frontend/src/api/index.js:451-454`):
```javascript
export const getLang = async (lang) => http.get(
    `/api/lang/${lang}`,
    { loading: models.lang, camelCase: false },
);
```

**后端接口** (`cmd/i18n.go:28-40`):
```go
func (a *App) GetI18nLang(c echo.Context) error {
    lang := c.Param("lang")
    // 安全检查：语言代码长度和格式
    if len(lang) > 6 || reLangCode.MatchString(lang) {
        return echo.NewHTTPError(http.StatusBadRequest, "Invalid language code.")
    }
    
    i, ok, err := getI18nLang(lang, a.fs)
    // 返回完整的语言包 JSON
    return c.JSON(http.StatusOK, okResp{json.RawMessage(i.JSON())})
}
```

### 4.4 前端使用方式

**模板中使用** (`frontend/src/App.vue` 及各组件):
```vue
<!-- 基础翻译 -->
{{ $t('globals.buttons.refresh') }}

<!-- 带参数 -->
{{ $t('settings.updateAvailable', { version: version }) }}

<!-- 复数形式 -->
{{ $tc('globals.terms.subscriber', count) }}

<!-- 检查翻译是否存在 -->
i18n.te(key) ? i18n.tc(key, 0) : ''
```

**脚本中使用**：
```javascript
// 通过 Vue 原型访问
Vue.prototype.$utils = new Utils(i18n);

// 直接使用 i18n 实例
i18n.t('settings.messengers.messageSaved')
```

## 5. 前后端同步机制

### 5.1 运行时同步

**核心原则**：前端不打包任何语言包，完全依赖后端提供

```
┌─────────────────────────────────────────────────────────────┐
│                        运行时同步流程                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  前端                                                        │
│  ┌──────────┐    API 请求      ┌─────────────────────────┐ │
│  │ main.js  │ ───────────────▶ │ GET /api/lang/{lang}   │ │
│  └──────────┘                  │ (cmd/i18n.go:28-40)    │ │
│       │                        └─────────────────────────┘ │
│       │                                  │                   │
│       │    返回 JSON 语言包              │                   │
│       │◀─────────────────────────────────┘                   │
│       │                                                       │
│       ▼                                                       │
│  ┌─────────────────────────────────────────┐                │
│  │ i18n.setLocaleMessage(locale, langData) │                │
│  │ (vue-i18n 运行时注入)                     │                │
│  └─────────────────────────────────────────┘                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 开发/维护同步

Listmonk 提供了两个重要的维护脚本来确保语言包同步：

#### 脚本 1: `refresh-i18n.sh` - 键结构同步

**功能**：以 `en.json` 为基准，同步所有其他语言文件的键结构

```bash
#!/bin/bash
BASE_DIR=../i18n
BASE_FILE="en.json"

for fpath in "$BASE_DIR/"*.json; do
    if [ "$(basename -- "$fpath")" = "$BASE_FILE" ]; then
        continue  # 跳过基准文件
    fi
    
    # 使用 jq 进行智能合并：
    # 1. 保留基准文件的所有键和顺序
    # 2. 对于存在的翻译，保留目标语言的值
    # 3. 对于缺失的翻译，使用英文值
    jq -s --indent 4 --sort-keys \
        '.[0] as $base | .[1] as $target |
         $base | with_entries(.value = ($target[.key] // .value))' \
        "$BASE_DIR/$BASE_FILE" "$fpath" > "$fpath.tmp" && mv "$fpath.tmp" "$fpath"
done
```

**同步效果**：
```
en.json (基准)              zh-CN.json (同步前)          zh-CN.json (同步后)
──────────────────          ──────────────────          ──────────────────
"a": "English A"     ──▶    "a": "中文A"          ──▶    "a": "中文A"
"b": "English B"              (缺失)                        "b": "English B"  ← 补全
"c": "English C"              "c": "中文C"                   "c": "中文C"
(新增键 "d")                   (无此键)                       "d": "English D"  ← 新增
```

#### 脚本 2: `translate-i18n.py` - 智能翻译辅助

**功能**：使用 GPT-4 自动翻译未翻译的键（值与英文相同的键）

```python
# 1. 加载基准语言和目标语言
BASE = json.loads(open(os.path.join(DIR, DEFAULT_LANG), "r").read())
data = json.loads(open(f, "r").read())

# 2. 找出未翻译的键（值与英文相同）
diff = {k: v for k, v in data.items() if BASE.get(k) == v}

# 3. 调用 GPT 进行批量翻译
def translate(data, lang):
    completion = client.chat.completions.create(
        model="gpt-4.1-mini",
        messages=[
            {"role": "system", 
             "content": "You are an i18n language pack translator for listmonk..."},
            {"role": "user",
             "content": f"Translate the untranslated English strings to {lang}..."},
            {"role": "user", "content": json.dumps(data)}
        ]
    )
    return json.loads(str(completion.choices[0].message.content))

# 4. 更新语言文件
new = translate(diff, data["_.name"])
data.update(new)
```

### 5.3 加载策略一致性

前后端采用**完全相同**的语言加载策略：

| 层面 | 策略 | 实现位置 |
|------|------|----------|
| **后端** | 先加载 en.json，再加载目标语言覆盖 | `cmd/i18n.go:getI18nLang()` |
| **前端** | 通过 API 获取后端处理后的完整语言包 | `frontend/src/main.js:initConfig()` |

**关键保证**：
- 缺失的翻译自动使用英文默认值
- 前后端看到的翻译结果完全一致
- 无需前端处理降级逻辑

## 6. 协作路径与工具链

### 6.1 单一数据源原则

```
                    ┌─────────────────┐
                    │   i18n/*.json   │
                    │  (单一数据源)    │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │  后端    │  │  前端    │  │  维护    │
        │ 直接读取 │  │ API 获取 │  │  工具    │
        └──────────┘  └──────────┘  └──────────┘
```

### 6.2 Inlang 集成

Listmonk 集成了 **Inlang** 平台，支持在线协作翻译：

**配置文件** (`project.inlang.json`):
```json
{
    "$schema": "https://inlang.com/schema/project-settings",
    "sourceLanguageTag": "en",
    "languageTags": ["en", "zh-CN", "zh-TW", "de", "fr", ...],
    
    "modules": [
        "@inlang/plugin-json",           // JSON 文件解析
        "@inlang/message-lint-rule-*",   // 翻译质量检查
    ],
    
    "plugin.inlang.json": {
        "pathPattern": "./i18n/{languageTag}.json",
        "variableReferencePattern": ["{", "}"]  // 匹配 {param} 格式
    }
}
```

**Inlang 提供的能力**：
1. 在线可视化翻译编辑器
2. 翻译质量检查（空翻译、重复翻译等）
3. Git 集成，直接推送 PR
4. 社区贡献者友好的界面

### 6.3 完整协作流程图

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Listmonk i18n 完整协作流程                           │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐                                                       │
│  │ 开发者/贡献者 │                                                       │
│  └──────┬───────┘                                                       │
│         │                                                                │
│         ├──────────────────────────────────────────────────────────┐   │
│         │                                                          │   │
│         ▼                                                          ▼   │
│  ┌──────────────┐                                          ┌──────────┐│
│  │  手动编辑     │                                          │ Inlang   ││
│  │ JSON 文件    │                                          │ 在线编辑  ││
│  └──────┬───────┘                                          └────┬─────┘│
│         │                                                        │      │
│         └──────────────────┬─────────────────────────────────────┘      │
│                            │                                              │
│                            ▼                                              │
│                   ┌─────────────────┐                                     │
│                   │  i18n/*.json    │                                     │
│                   │  (Git 托管)      │                                     │
│                   └────────┬────────┘                                     │
│                            │                                               │
│          ┌─────────────────┼─────────────────┐                           │
│          │                 │                 │                           │
│          ▼                 ▼                 ▼                           │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │
│   │ refresh.sh  │  │translate.py │  │  构建/部署   │                    │
│   │  键结构同步  │  │  AI 翻译辅助 │  │             │                    │
│   └─────────────┘  └─────────────┘  └──────┬──────┘                    │
│                                               │                            │
│                                               ▼                            │
│                                      ┌──────────────┐                      │
│                                      │   运行时     │                      │
│                                      │  前后端同步   │                      │
│                                      └──────────────┘                      │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

## 7. 关键设计亮点

### 7.1 零前端语言包打包

**创新点**：前端不内置任何语言文件，完全运行时加载

**优势**：
1. **构建产物更小**：无需打包 40+ 语言的 JSON
2. **一致性保证**：前后端使用完全相同的翻译数据
3. **热更新能力**：替换语言文件无需重新构建前端
4. **集中管理**：所有翻译维护只需关注 `i18n/` 目录

### 7.2 API 兼容的后端实现

**设计理念**：后端 i18n 包刻意模拟 vue-i18n 的行为

**代码证据** (`i18n.go:1-3`):
```go
// It mimics some functionality of the vue-i18n library so that the same JSON
// language map may be used in the JS frontend and the Go backend.
```

**API 对照表**：

| vue-i18n (前端) | Go i18n (后端) | 功能 |
|-----------------|----------------|------|
| `$t(key)` | `T(key)` | 基础翻译 |
| `$t(key, {p:v})` | `Ts(key, "p", v)` | 参数替换 |
| `$tc(key, n)` | `Tc(key, n)` | 复数选择 |
| `{param}` | `{param}` | 占位符格式 |
| `Singular \| Plural` | `Singular \| Plural` | 复数分隔符 |

### 7.3 分层加载 + 优雅降级

**策略**：以英文为基准，目标语言覆盖

```
用户请求语言: zh-CN
                 
    ┌─────────────────────────────────────────┐
    │  Step 1: 加载 en.json (完整键集)        │
    │  ┌─────────────────────────────────┐    │
    │  │ "a": "English A"                │    │
    │  │ "b": "English B"                │    │
    │  │ "c": "English C"                │    │
    │  └─────────────────────────────────┘    │
    └─────────────────────────────────────────┘
                        ↓
    ┌─────────────────────────────────────────┐
    │  Step 2: 加载 zh-CN.json 并覆盖        │
    │  ┌─────────────────────────────────┐    │
    │  │ "a": "中文A"      ← 已翻译      │    │
    │  │ "b": "English B"  ← 未翻译      │    │
    │  │ "c": "中文C"      ← 已翻译      │    │
    │  └─────────────────────────────────┘    │
    └─────────────────────────────────────────┘
                        ↓
    最终结果: "b" 显示英文（优雅降级，不报错）
```

**优势**：
- 新功能新增的翻译键不会破坏现有语言
- 贡献者只需翻译部分键即可
- 永远有可显示的文本，不会出现空键

### 7.4 递归参数解析

**高级特性**：参数值可以包含其他翻译键的引用

**实现** (`i18n.go:148-164`):
```go
func (i *I18n) subAllParams(s string) string {
    // 递归查找所有 {xxx} 占位符
    parts := reParam.FindAllStringSubmatch(s, -1)
    
    for _, p := range parts {
        // 将 {key} 替换为 i.T("key") 的结果
        s = strings.ReplaceAll(s, p[0], i.T(p[1]))
    }
    
    // 递归处理，支持嵌套
    return i.subAllParams(s)
}
```

**使用场景示例**：
```json
{
    "error.notFound": "{name} not found",
    "entity.campaign": "Campaign"
}
```

调用：
```go
i18n.Ts("error.notFound", "name", "entity.campaign")
// 结果: "Campaign not found"
```

## 8. 潜在改进空间

### 8.1 复数形式的局限性

**当前实现**：仅支持简单的单/复数二元选择

```go
// i18n.go:109-121
func (i *I18n) Tc(key string, n int) string {
    // Plural.
    if n > 1 {
        return i.getPlural(s)
    }
    return i.getSingular(s)
}
```

**问题**：某些语言有复杂的复数规则
- 俄语：3 种形式（1, 2-4, 5+）
- 波兰语：3 种形式
- 阿拉伯语：6 种形式

**建议**：考虑引入更完整的 CLDR 复数规则支持

### 8.2 前端语言切换体验

**当前实现**：语言切换需要刷新页面或重新初始化

```javascript
// main.js 中的 awaitRestart 示例
awaitRestart(response) {
    // ...
    this.loadConfig();  // 重新加载配置和语言
    // ...
}
```

**建议**：实现更平滑的语言切换体验
- 前端缓存已加载的语言包
- 支持运行时切换无需页面刷新
- 考虑添加语言切换动画

### 8.3 翻译键的类型安全

**当前问题**：翻译键是纯字符串，无编译时检查

```go
// 运行时才能发现的错误
a.i18n.T("globals.messages.tpyo")  // 拼写错误
```

**建议方向**：
- 考虑生成 TypeScript/Go 的类型定义文件
- 或使用代码生成工具从 JSON 生成常量

## 9. 总结

Listmonk 的国际化架构体现了以下核心设计哲学：

### 9.1 架构优势

| 维度 | 实现策略 | 收益 |
|------|----------|------|
| **数据源** | 单一 JSON 目录 | 维护简单，一致性高 |
| **格式** | 标准 vue-i18n 兼容 | 前后端零转换成本 |
| **运行时** | 前端 API 拉取 | 构建产物小，热更新友好 |
| **维护** | 脚本 + Inlang 平台 | 自动化同步，社区友好 |
| **降级** | 分层加载策略 | 新键不破坏，优雅回退 |

### 9.2 关键文件索引

| 文件路径 | 功能 | 关键行 |
|----------|------|--------|
| `internal/i18n/i18n.go` | 后端 i18n 核心实现 | 1-164 |
| `cmd/i18n.go` | 语言 API 接口 | 28-99 |
| `frontend/src/main.js` | 前端 i18n 初始化 | 34-88 |
| `frontend/src/api/index.js` | 语言 API 调用 | 451-454 |
| `scripts/refresh-i18n.sh` | 键结构同步脚本 | 1-17 |
| `scripts/translate-i18n.py` | AI 翻译辅助脚本 | 1-55 |
| `project.inlang.json` | Inlang 平台配置 | 1-19 |
| `i18n/en.json` | 基准语言包 | 1-702 |

### 9.3 协作流程最佳实践

对于 Listmonk 的贡献者和维护者：

1. **新增翻译键**：仅需修改 `i18n/en.json`，其他语言通过脚本同步
2. **维护现有翻译**：直接编辑对应语言的 JSON 文件，或使用 Inlang 在线编辑
3. **同步键结构**：运行 `./scripts/refresh-i18n.sh` 确保所有语言文件键一致
4. **批量翻译**：配置 OpenAI API Key 后运行 `python ./scripts/translate-i18n.py`

这套架构在简洁性、一致性和可维护性之间取得了良好的平衡，是中小型项目国际化的优秀实践参考。

---

*报告生成时间：2026-05-05*
*分析版本：基于当前代码库状态*
