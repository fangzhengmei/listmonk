# Listmonk 前后端多语言同步机制分析报告

## 校正说明

本报告已根据实际代码库进行以下校正：
1. **语言文件数量**：实际为 37 个文件（非 40+）
2. **日语代码**：使用非标准代码 `jp`（文件名 `jp.json`，内部 `_.code: "jp"`），而非标准 ISO 639-1 代码 `ja`
3. **键顺序行为**：同步脚本使用 `--sort-keys` 参数，输出按键名字母顺序排序，而非保留 en.json 原始顺序
4. **三阶段证据对照**：新增前端加载、后端合并、维护同步三个阶段的代码证据对照

---

## 1. 概述

Listmonk 采用了一套精心设计的国际化（i18n）架构，实现了前后端共享同一套语言包，并通过多种机制确保语言包的同步和一致性。本报告深入分析其实现原理、加载机制和协作路径。

## 2. 语言包存储结构

### 2.1 统一数据源

语言包统一存储在项目根目录的 `i18n/` 文件夹下，前后端共用此目录。

**实际文件列表**（共 37 个）：

```
i18n/
├── ar.json          # 阿拉伯语
├── bg.json          # 保加利亚语
├── ca.json          # 加泰罗尼亚语
├── cs-cz.json       # 捷克语
├── cy.json          # 威尔士语
├── da.json          # 丹麦语
├── de.json          # 德语
├── el.json          # 希腊语
├── en.json          # 基准语言（英语）
├── eo.json          # 世界语
├── es.json          # 西班牙语
├── fi.json          # 芬兰语
├── fr-CA.json       # 法语（加拿大）
├── fr.json          # 法语
├── he.json          # 希伯来语
├── hu.json          # 匈牙利语
├── it.json          # 意大利语
├── jp.json          # 日语 ⚠️ 注意：使用非标准代码 "jp"
├── ko.json          # 韩语
├── ml.json          # 马拉雅拉姆语
├── nl.json          # 荷兰语
├── no.json          # 挪威语
├── pl.json          # 波兰语
├── pt-BR.json       # 葡萄牙语（巴西）
├── pt.json          # 葡萄牙语
├── ro.json          # 罗马尼亚语
├── ru.json          # 俄语
├── se.json          # 北萨米语
├── sk.json          # 斯洛伐克语
├── sl.json          # 斯洛文尼亚语
├── tr.json          # 土耳其语
├── uk.json          # 乌克兰语
├── vi.json          # 越南语
├── zh-CN.json       # 简体中文
└── zh-TW.json       # 繁体中文
```

**统计**：37 个语言文件（含基准语言 en.json）

### 2.2 日语代码标识校正

**重要发现**：Listmonk 对日语使用了非标准语言代码。

**实际证据** (`i18n/jp.json:1-3`):
```json
{
    "_.code": "jp",
    "_.name": "日本語 (jp)",
    ...
}
```

**对比说明**：

| 项目 | Listmonk 实际值 | 标准 ISO 639-1 |
|------|-----------------|-----------------|
| 文件名 | `jp.json` | `ja.json` |
| `_.code` 字段 | `"jp"` | `"ja"` |
| `_.name` 字段 | `"日本語 (jp)"` | `"日本語 (ja)"` |
| Inlang 配置 | `"jp"` | - |

**Inlang 配置确认** (`project.inlang.json:4`):
```json
"languageTags": [..., "jp", ..., "zh-CN", "zh-TW"]
```

### 2.3 语言包格式

语言包采用 JSON 格式，具有以下特点：

**元数据字段**：
- `_.code` - 语言代码（如 "en", "zh-CN", "jp"）
- `_.name` - 语言显示名称（如 "English (en)", "日本語 (jp)"）

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

### 2.4 复数形式规则

复数形式通过管道符 `|` 分隔：
- 单数形式：`n == 1` 时返回第一个部分
- 复数形式：`n > 1` 时返回第二个部分

示例：
```json
"globals.terms.subscriber": "Subscriber | Subscribers"
// Tc("globals.terms.subscriber", 1) → "Subscriber"
// Tc("globals.terms.subscriber", 5) → "Subscribers"
```

---

## 3. 三阶段同步机制：证据对照

### 阶段一：前端加载（Frontend Loading）

**核心行为**：前端不打包任何语言文件，应用启动时通过 API 动态获取。

**证据 1：main.js 初始化** (`frontend/src/main.js:34-40`):

```javascript
async function initConfig(app) {
  // 1. 并行获取用户配置和服务器配置
  const [profile, cfg] = await Promise.all([api.getUserProfile(), api.getServerConfig()]);

  // 2. 通过 API 从后端获取语言包
  const lang = await api.getLang(cfg.lang);
  
  // 3. 设置 vue-i18n 的当前语言和消息
  i18n.locale = cfg.lang;
  i18n.setLocaleMessage(i18n.locale, lang);
  // ...
}
```

**证据 2：API 调用定义** (`frontend/src/api/index.js:451-454`):

```javascript
export const getLang = async (lang) => http.get(
    `/api/lang/${lang}`,
    { loading: models.lang, camelCase: false },
);
```

**证据 3：vue-i18n 初始化** (`frontend/src/main.js:11-13`):

```javascript
// 安装 vue-i18n 插件
Vue.use(VueI18n);
// 创建空的 i18n 实例（不预置任何语言）
const i18n = new VueI18n();
```

**前端加载流程图**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    阶段一：前端加载                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  应用启动                                                         │
│     │                                                             │
│     ▼                                                             │
│  ┌─────────────────┐                                              │
│  │ initConfig()    │                                              │
│  │ (main.js:34)    │                                              │
│  └────────┬────────┘                                              │
│           │                                                        │
│           ├─── Promise.all([                                      │
│           │        api.getUserProfile(),                          │
│           │        api.getServerConfig()  ←── 获取 cfg.lang     │
│           │    ])                                                 │
│           │                                                        │
│           ▼                                                        │
│  ┌─────────────────────────────────────┐                         │
│  │ api.getLang(cfg.lang)               │                         │
│  │ ──▶ GET /api/lang/{code}            │                         │
│  │ (frontend/src/api/index.js:451)     │                         │
│  └──────────────────┬──────────────────┘                         │
│                     │                                              │
│                     ▼                                              │
│  ┌──────────────────────────────────────────────────┐            │
│  │ i18n.setLocaleMessage(locale, langData)          │            │
│  │ (vue-i18n 运行时注入，覆盖默认值)                  │            │
│  └──────────────────────────────────────────────────┘            │
│                                                                   │
│  关键特性：                                                        │
│  ✅ 前端无预置语言包                                               │
│  ✅ 完全依赖后端 API 返回                                           │
│  ✅ 语言包已由后端完成合并处理                                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

### 阶段二：后端合并（Backend Merging）

**核心行为**：采用"基准加载 + 目标覆盖"的分层策略，确保未翻译键有英文兜底。

**证据 1：getI18nLang 核心逻辑** (`cmd/i18n.go:75-98`):

```go
func getI18nLang(lang string, fs stuffbin.FileSystem) (*i18n.I18n, bool, error) {
    const def = "en"  // 基准语言代码

    // ── 步骤 1：加载基准语言 en.json ──
    b, err := fs.Read(fmt.Sprintf("/i18n/%s.json", def))
    if err != nil {
        return nil, false, fmt.Errorf("error reading default i18n language file: %s: %v", def, err)
    }
    // 用基准语言初始化 i18n 实例
    i, err := i18n.New(b)
    if err != nil {
        return nil, false, fmt.Errorf("error unmarshalling i18n language: %s: %v", lang, err)
    }

    // ── 步骤 2：加载目标语言并覆盖 ──
    b, err = fs.Read(fmt.Sprintf("/i18n/%s.json", lang))
    if err != nil {
        // 目标语言文件不存在时，返回基准语言（但标记 ok=true）
        return i, true, fmt.Errorf("error reading i18n language file: %s: %v", lang, err)
    }
    // 将目标语言的键值覆盖到基准语言之上
    if err := i.Load(b); err != nil {
        return i, true, fmt.Errorf("error loading i18n language file: %s: %v", lang, err)
    }

    return i, true, nil
}
```

**证据 2：i18n.Load 方法** (`internal/i18n/i18n.go:48-59`):

```go
// Load loads a JSON language map into the instance overwriting
// existing keys that conflict.
func (i *I18n) Load(b []byte) error {
    var l map[string]string
    if err := json.Unmarshal(b, &l); err != nil {
        return err
    }

    // 遍历并覆盖：目标语言存在的键才覆盖
    for k, v := range l {
        i.langMap[k] = v
    }

    return nil
}
```

**证据 3：API 端点返回** (`cmd/i18n.go:28-40`):

```go
func (a *App) GetI18nLang(c echo.Context) error {
    lang := c.Param("lang")
    // 安全检查：语言代码长度和格式
    if len(lang) > 6 || reLangCode.MatchString(lang) {
        return echo.NewHTTPError(http.StatusBadRequest, "Invalid language code.")
    }

    // 调用合并逻辑获取处理后的语言包
    i, ok, err := getI18nLang(lang, a.fs)
    if err != nil && !ok {
        return echo.NewHTTPError(http.StatusBadRequest, "Unknown language.")
    }

    // 返回合并后的完整 JSON（已包含所有键，未翻译键为英文）
    return c.JSON(http.StatusOK, okResp{json.RawMessage(i.JSON())})
}
```

**后端合并流程图**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    阶段二：后端合并                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  请求：GET /api/lang/zh-CN                                        │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Step 1: 加载基准语言 en.json                                  │ │
│  │ ────────────────────────────────────────────────────────── │ │
│  │                                                              │ │
│  │  en.json (基准，完整键集)                                    │ │
│  │  ┌─────────────────────────────────────────────────────┐   │ │
│  │  │ "_.code":        "en"                                │   │ │
│  │  │ "_.name":        "English (en)"                      │   │ │
│  │  │ "globals.terms.campaign": "Campaign | Campaigns"    │   │ │
│  │  │ "globals.terms.list":     "List | Lists"            │   │ │
│  │  │ "message.a":     "English A"                         │   │ │
│  │  │ "message.b":     "English B"  ←── zh-CN 无此翻译    │   │ │
│  │  │ "message.c":     "English C"                         │   │ │
│  │  └─────────────────────────────────────────────────────┘   │ │
│  │                            │                                 │ │
│  │                            ▼                                 │ │
│  │                   i18n.New(en.json)                         │ │
│  │                   (langMap 初始化为英文键值)                 │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Step 2: 加载目标语言 zh-CN.json 并覆盖                       │ │
│  │ ────────────────────────────────────────────────────────── │ │
│  │                                                              │ │
│  │  zh-CN.json (目标语言，部分键)                               │ │
│  │  ┌─────────────────────────────────────────────────────┐   │ │
│  │  │ "_.code":        "zh-CN"                             │   │ │
│  │  │ "_.name":        "简体中文 (zh-CN)"                  │   │ │
│  │  │ "globals.terms.campaign": "活动 | 活动"              │   │ │
│  │  │ "message.a":     "中文A"                              │   │ │
│  │  │ "message.c":     "中文C"                              │   │ │
│  │  │ (注意：message.b 不存在于此文件)                       │   │ │
│  │  └─────────────────────────────────────────────────────┘   │ │
│  │                            │                                 │ │
│  │                            ▼                                 │ │
│  │                   i.Load(zh-CN.json)                        │ │
│  │                   (遍历 zh-CN 的键，覆盖到 langMap)         │ │
│  │                                                              │ │
│  │  覆盖逻辑 (internal/i18n/i18n.go:54-56):                   │ │
│  │  for k, v := range l {                                      │ │
│  │      i.langMap[k] = v  // 只覆盖存在的键                   │ │
│  │  }                                                          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Step 3: 合并结果（返回给前端）                               │ │
│  │ ────────────────────────────────────────────────────────── │ │
│  │                                                              │ │
│  │  ┌─────────────────────────────────────────────────────┐   │ │
│  │  │ "_.code":        "zh-CN"      ←── 已覆盖            │   │ │
│  │  │ "_.name":        "简体中文 (zh-CN)" ←── 已覆盖      │   │ │
│  │  │ "globals.terms.campaign": "活动 | 活动" ←── 已覆盖  │   │ │
│  │  │ "globals.terms.list":     "List | Lists" ←── 未覆盖 │   │ │
│  │  │ "message.a":     "中文A"       ←── 已覆盖            │   │ │
│  │  │ "message.b":     "English B"   ←── 未覆盖（保留英文）│   │ │
│  │  │ "message.c":     "中文C"       ←── 已覆盖            │   │ │
│  │  └─────────────────────────────────────────────────────┘   │ │
│  │                                                              │ │
│  │  关键特性：                                                  │ │
│  │  ✅ 所有键都存在（来自 en.json）                            │ │
│  │  ✅ 目标语言有的键被覆盖                                    │ │
│  │  ✅ 目标语言没有的键保留英文（优雅降级）                     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

### 阶段三：维护同步（Maintenance Sync）

**核心行为**：通过脚本确保所有语言文件的键结构与基准语言一致。

#### 3.1 refresh-i18n.sh 脚本分析

**脚本源码** (`scripts/refresh-i18n.sh:1-17`):

```bash
#!/bin/bash

# "Refresh" all i18n language files by merging and syncing keys with the base file.
BASE_DIR=$(dirname "$0")"/../i18n" # Exclude the trailing slash.
BASE_FILE="en.json"

# Iterate through all i18n files and sync them with the base file.
for fpath in "$BASE_DIR/"*.json; do
    if [ "$(basename -- "$fpath")" = "$BASE_FILE" ]; then
        continue  # Skip the base file itself
    fi
    echo "$(basename -- "$fpath")"
    jq -s --indent 4 --sort-keys \
        '.[0] as $base | .[1] as $target |
        $base | with_entries(.value = ($target[.key] // .value))' \
        "$BASE_DIR/$BASE_FILE" "$fpath" > "$fpath.tmp" && mv "$fpath.tmp" "$fpath"
done
```

#### 3.2 jq 表达式逐段解析

| 参数/表达式 | 作用 | 证据位置 |
|-------------|------|----------|
| `-s` | 将多个输入文件读入一个数组 | `jq -s ...` |
| `--indent 4` | 输出时使用 4 空格缩进 | `--indent 4` |
| `--sort-keys` | ⚠️ 输出按键名字母顺序排序 | `--sort-keys` |
| `.[0] as $base` | 第一个文件（en.json）赋值给 $base | `.[0] as $base` |
| `.[1] as $target` | 第二个文件（目标语言）赋值给 $target | `.[1] as $target` |
| `$base \| ...` | 以 $base 的键结构为输出基础 | `$base \| with_entries(...)` |
| `with_entries(.value = ($target[.key] // .value))` | 对每个键，优先用 $target 的值，否则用 $base 的值 | 核心逻辑 |

#### 3.3 键顺序行为校正

**重要发现**：脚本使用了 `--sort-keys` 参数，这会改变键的输出顺序。

**行为对比**：

| 假设的 en.json 原始顺序 | 无 --sort-keys 输出 | 有 --sort-keys 输出（实际行为） |
|------------------------|---------------------|---------------------------------|
| `"_.code"`            | `"_.code"`         | `"_.code"` (字母序保留)         |
| `"_.name"`            | `"_.name"`         | `"_.name"` (字母序保留)         |
| `"globals.terms.z"`   | `"globals.terms.z"` | `"globals.terms.a"` ← **被重排** |
| `"globals.terms.a"`   | `"globals.terms.a"` | `"globals.terms.b"` ← **被重排** |
| `"globals.terms.b"`   | `"globals.terms.b"` | `"globals.terms.z"` ← **被重排** |

**结论**：
- ❌ **不保留** en.json 的原始键顺序
- ✅ **按键名字母顺序**重新排序输出
- 这是 jq `--sort-keys` 参数的标准行为

#### 3.4 同步效果演示

**输入**：

```
en.json (基准)                    zh-CN.json (目标，同步前)
───────────────────────────────────  ───────────────────────────────────
{                                  {
  "_.code": "en",                   "_.code": "zh-CN",
  "_.name": "English (en)",         "_.name": "简体中文",
  "message.a": "English A",         "message.a": "中文A",
  "message.b": "English B",         (message.b 缺失)
  "message.c": "English C"          "message.c": "中文C",
}                                    "extra.key": "额外键"  ← 基准没有
                                   }
```

**执行**：
```bash
jq -s --indent 4 --sort-keys \
    '.[0] as $base | .[1] as $target |
     $base | with_entries(.value = ($target[.key] // .value))' \
    en.json zh-CN.json
```

**输出**（zh-CN.json 同步后）：

```json
{
    "_.code": "zh-CN",           // 来自目标语言
    "_.name": "简体中文",         // 来自目标语言
    "message.a": "中文A",         // 来自目标语言
    "message.b": "English B",     // ⚠️ 目标缺失，使用基准值
    "message.c": "中文C"          // 来自目标语言
    // ⚠️ "extra.key" 被丢弃（因为基准语言没有此键）
}
```

**同步规则总结**：

| 情况 | 处理方式 |
|------|----------|
| 基准有键，目标有值 | 使用目标值 |
| 基准有键，目标无此键 | 使用基准值（英文） |
| 基准无键，目标有此键 | **被丢弃**（以基准为准） |
| 键顺序 | 按字母顺序排序（`--sort-keys`） |

#### 3.5 维护同步流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    阶段三：维护同步                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  触发时机：新增/删除翻译键后，运行 ./scripts/refresh-i18n.sh     │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ 遍历所有非基准语言文件                                        │ │
│  │ for fpath in "$BASE_DIR/"*.json; do                         │ │
│  │     if [ "$fpath" = "en.json" ]; then continue; fi         │ │
│  │     ...                                                      │ │
│  │ done                                                         │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ jq 处理逻辑（逐文件执行）                                    │ │
│  │ ────────────────────────────────────────────────────────── │ │
│  │                                                              │ │
│  │  jq -s --indent 4 --sort-keys \                             │ │
│  │      '.[0] as $base | .[1] as $target |                    │ │
│  │       $base | with_entries(.value = ($target[.key] // .value))' \ │
│  │      en.json zh-CN.json > zh-CN.json.tmp                    │ │
│  │                                                              │ │
│  │  表达式拆解：                                                 │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │ $base                    → 以 en.json 的键为模板     │  │ │
│  │  │ with_entries(...)        → 遍历每个键处理             │  │ │
│  │  │ $target[.key] // .value  → 优先取目标值，否则取基准值 │  │ │
│  │  │ --sort-keys              → 按键名字母顺序排序 ⚠️      │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ 同步结果示例                                                 │ │
│  │ ────────────────────────────────────────────────────────── │ │
│  │                                                              │ │
│  │  同步前：                                                    │ │
│  │  en.json: {a, b, c}         zh-CN.json: {a, c, extra}     │ │
│  │                                                              │ │
│  │  同步后：                                                    │ │
│  │  zh-CN.json: {                                              │ │
│  │    "a": "中文A",           ← 目标有值，保留                 │ │
│  │    "b": "English B",       ← 目标无值，使用英文            │ │
│  │    "c": "中文C"            ← 目标有值，保留                 │ │
│  │    // "extra" 被丢弃         ← 基准无此键                   │ │
│  │  }                                                           │ │
│  │                                                              │ │
│  │  关键特性：                                                  │ │
│  │  ✅ 以 en.json 为唯一真相源（SSOT）                         │ │
│  │  ✅ 缺失翻译自动补英文默认值                                  │ │
│  │  ✅ 冗余键（基准没有的）被清理                                │ │
│  │  ⚠️ 键顺序按字母重排（不保留原始顺序）                       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. 三阶段联动总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Listmonk i18n 三阶段同步总览                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         阶段三：维护同步                               │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐│   │
│  │  │ 开发者修改 en.json（新增/删除键）                                  ││   │
│  │  │         │                                                        ││   │
│  │  │         ▼                                                        ││   │
│  │  │ ./scripts/refresh-i18n.sh                                        ││   │
│  │  │         │                                                        ││   │
│  │  │         ▼                                                        ││   │
│  │  │ 所有语言文件键结构与 en.json 保持一致                              ││   │
│  │  │ - 缺失键 → 补英文默认值                                           ││   │
│  │  │ - 冗余键 → 被清理                                                 ││   │
│  │  │ - 键顺序 → 按字母重排 ⚠️                                          ││   │
│  │  └─────────────────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         阶段二：后端合并                               │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐│   │
│  │  │ 运行时请求：GET /api/lang/zh-CN                                   ││   │
│  │  │         │                                                        ││   │
│  │  │         ▼                                                        ││   │
│  │  │ getI18nLang(lang, fs)                                            ││   │
│  │  │         │                                                        ││   │
│  │  │         ├── Step 1: fs.Read("/i18n/en.json")                    ││   │
│  │  │         │         → i18n.New(enData)                             ││   │
│  │  │         │                                                        ││   │
│  │  │         ├── Step 2: fs.Read("/i18n/zh-CN.json")                 ││   │
│  │  │         │         → i.Load(zhCNData)  // 覆盖存在的键           ││   │
│  │  │         │                                                        ││   │
│  │  │         └── Step 3: return i.JSON()  // 完整合并后的 JSON       ││   │
│  │  │                                                                   ││   │
│  │  │ 结果：所有键都存在，未翻译键为英文                                ││   │
│  │  └─────────────────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         阶段一：前端加载                               │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐│   │
│  │  │ 应用启动：initConfig()                                            ││   │
│  │  │         │                                                        ││   │
│  │  │         ▼                                                        ││   │
│  │  │ api.getServerConfig() → 获取 cfg.lang                            ││   │
│  │  │         │                                                        ││   │
│  │  │         ▼                                                        ││   │
│  │  │ api.getLang(cfg.lang)                                            ││   │
│  │  │         │                                                        ││   │
│  │  │         ▼                                                        ││   │
│  │  │ i18n.setLocaleMessage(locale, langData)                          ││   │
│  │  │ (vue-i18n 运行时注入)                                            ││   │
│  │  │                                                                   ││   │
│  │  │ 前端视角："不知道"后端做了什么合并，直接使用 API 返回的数据       ││   │
│  │  └─────────────────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 其他辅助工具

### 5.1 translate-i18n.py - AI 翻译辅助

**功能**：使用 GPT-4 自动翻译未翻译的键（值与英文相同的键）

**核心逻辑** (`scripts/translate-i18n.py:35-55`):

```python
# 遍历每个语言文件
for f in glob(os.path.join(DIR, "*.json")):
    if os.path.basename(f) == DEFAULT_LANG:
        continue  # 跳过基准语言

    data = json.loads(open(f, "r").read())

    # 找出未翻译的键：值与基准语言相同的键
    if KEYS:
        diff = {k: BASE[k] for k in KEYS}
    else:
        diff = {k: v for k, v in data.items() if BASE.get(k) == v}

    # 调用 GPT 翻译
    new = translate(diff, data["_.name"])
    
    # 更新并写回文件
    data.update(new)
    with open(f, "w") as o:
        o.write(json.dumps(data, sort_keys=True, indent=4, ensure_ascii=False) + "\n")
```

**翻译逻辑**：
- 识别：`data[k] == BASE[k]` 的键视为"未翻译"
- 批量：一次性发送给 GPT-4.1-mini
- 提示词要求："Translate the untranslated English strings... Retain any technical terms or acronyms."

### 5.2 Inlang 平台集成

**配置文件** (`project.inlang.json`):

```json
{
    "$schema":"https://inlang.com/schema/project-settings",
    "sourceLanguageTag": "en",
    "languageTags": ["ca", "cs-cz", "cy", "de", "en", "es", "fi", "fr", 
                     "hu", "it", "jp", "ml", "nl", "pl", "pt-BR", "pt", 
                     "ro", "ru", "se", "sk", "tr", "vi", "zh-CN", "zh-TW"],
    "modules": [
        "https://cdn.jsdelivr.net/npm/@inlang/plugin-json@4/dist/index.js",
        "https://cdn.jsdelivr.net/npm/@inlang/message-lint-rule-empty-pattern@1/dist/index.js",
        "https://cdn.jsdelivr.net/npm/@inlang/message-lint-rule-identical-pattern@1/dist/index.js",
        "https://cdn.jsdelivr.net/npm/@inlang/message-lint-rule-without-source@1/dist/index.js",
        "https://cdn.jsdelivr.net/npm/@inlang/message-lint-rule-missing-translation@1/dist/index.js"
    ],
    "plugin.inlang.json": {
        "pathPattern": "./i18n/{languageTag}.json",
        "variableReferencePattern": ["{", "}"]
    }
}
```

**注意**：`languageTags` 中包含 `"jp"` 而非标准 `"ja"`，与实际代码一致。

**Inlang 能力**：
1. 在线可视化翻译编辑器
2. 翻译质量检查（空翻译、重复翻译、缺失翻译等）
3. Git 集成，直接推送 PR
4. 社区贡献者友好的界面

---

## 6. 关键设计亮点

### 6.1 单一真相源（SSOT）

**设计**：`i18n/en.json` 是唯一的真相源

| 维度 | 实现 |
|------|------|
| **键结构** | refresh-i18n.sh 以 en.json 为模板同步所有语言 |
| **运行时** | 后端先加载 en.json，再用目标语言覆盖 |
| **前端** | 完全依赖后端 API，不内置任何语言 |

**收益**：
- 新增键只需修改 en.json
- 删除键只需修改 en.json
- 所有语言自动同步

### 6.2 零前端语言包打包

**创新点**：前端不内置任何语言文件，完全运行时加载

**证据** (`frontend/src/main.js:11-13`):
```javascript
Vue.use(VueI18n);
const i18n = new VueI18n();  // 空实例，无预置消息
```

**收益**：
1. **构建产物更小**：无需打包 37 个语言的 JSON
2. **一致性保证**：前后端使用完全相同的翻译数据（都经过后端合并逻辑）
3. **热更新能力**：替换语言文件无需重新构建前端
4. **集中管理**：所有翻译维护只需关注 `i18n/` 目录

### 6.3 API 兼容的后端实现

**设计理念**：后端 i18n 包刻意模拟 vue-i18n 的行为

**代码证据** (`internal/i18n/i18n.go:1-3`):
```go
// i18n is a simple package that translates strings using a language map.
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

### 6.4 分层加载 + 优雅降级

**策略**：以英文为基准，目标语言覆盖

**流程图**（详见阶段二）：
1. 加载 en.json（完整键集）
2. 加载目标语言 JSON
3. 用目标语言的键值覆盖

**优势**：
- 新功能新增的翻译键不会破坏现有语言
- 贡献者只需翻译部分键即可
- 永远有可显示的文本，不会出现空键

---

## 7. 潜在改进空间

### 7.1 日语代码非标准

**现状**：使用 `"jp"` 而非标准 `"ja"`

**证据**：
- 文件名：`jp.json`
- 内部 `_.code`: `"jp"`
- Inlang 配置：`"jp"`

**影响**：
- 与标准 ISO 639-1 不一致
- 可能导致外部集成时的混淆

**建议**：考虑在未来版本中迁移为标准代码 `"ja"`，同时保持向后兼容。

### 7.2 复数形式的局限性

**当前实现**：仅支持简单的单/复数二元选择

**代码** (`internal/i18n/i18n.go:109-121`):
```go
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

**建议**：考虑引入更完整的 CLDR 复数规则支持。

### 7.3 键顺序被重排

**现状**：refresh-i18n.sh 使用 `--sort-keys`，导致键顺序按字母重排

**影响**：
- 与 en.json 原始顺序不一致
- Git diff 可能显示大量顺序变化（实际值未变）

**建议**：考虑移除 `--sort-keys` 参数，或使用能保留原始顺序的工具。

### 7.4 前端语言切换体验

**当前实现**：语言切换需要重新初始化

**代码** (`frontend/src/main.js:101-103`):
```javascript
loadConfig() {
    initConfig();  // 重新加载配置和语言
}
```

**建议**：实现更平滑的语言切换体验
- 前端缓存已加载的语言包
- 支持运行时切换无需完整重加载

---

## 8. 关键文件索引

| 文件路径 | 功能 | 关键行 |
|----------|------|--------|
| `internal/i18n/i18n.go` | 后端 i18n 核心实现 | 1-164 |
| `cmd/i18n.go` | 语言 API 接口 + 合并逻辑 | 28-99 |
| `frontend/src/main.js` | 前端 i18n 初始化 + 加载逻辑 | 34-40 |
| `frontend/src/api/index.js` | 语言 API 调用定义 | 451-454 |
| `scripts/refresh-i18n.sh` | 键结构同步脚本 | 1-17 |
| `scripts/translate-i18n.py` | AI 翻译辅助脚本 | 1-55 |
| `project.inlang.json` | Inlang 平台配置 | 1-19 |
| `i18n/en.json` | 基准语言包 | 1-702 |
| `i18n/jp.json` | 日语（非标准代码 `"jp"`） | 1-3 |

---

## 9. 总结

### 9.1 校正要点回顾

| 校正项 | 原报告 | 校正后 | 证据位置 |
|--------|--------|--------|----------|
| 语言文件数量 | 40+ 种 | 37 个文件 | `LS i18n/` |
| 日语代码 | `ja.json` | `jp.json`（非标准） | `i18n/jp.json:1-3` |
| 键顺序行为 | 保留 en.json 顺序 | 按字母重排（`--sort-keys`） | `refresh-i18n.sh:13` |
| 三阶段证据 | 描述性 | 代码级对照 | 本报告第 3 章 |

### 9.2 架构优势总结

| 维度 | 实现策略 | 收益 |
|------|----------|------|
| **数据源** | 单一 JSON 目录，en.json 为 SSOT | 维护简单，一致性高 |
| **格式** | 标准 vue-i18n 兼容 | 前后端零转换成本 |
| **运行时** | 前端 API 拉取 + 后端分层合并 | 构建产物小，一致性保证 |
| **维护** | refresh.sh + translate.py + Inlang | 自动化同步，社区友好 |
| **降级** | 先加载 en，再覆盖目标 | 新键不破坏，优雅回退 |

### 9.3 协作流程最佳实践

对于 Listmonk 的贡献者和维护者：

1. **新增翻译键**：仅需修改 `i18n/en.json`
2. **同步键结构**：运行 `./scripts/refresh-i18n.sh` 确保所有语言文件键一致
3. **维护现有翻译**：直接编辑对应语言的 JSON 文件，或使用 Inlang 在线编辑
4. **批量翻译**：配置 OpenAI API Key 后运行 `python ./scripts/translate-i18n.py`

这套架构在简洁性、一致性和可维护性之间取得了良好的平衡，是中小型项目国际化的优秀实践参考。

---

*报告校正时间：2026-05-05*
*分析版本：基于当前代码库实际状态*
*校正内容：语言数量、日语代码、键顺序行为、三阶段证据对照*
