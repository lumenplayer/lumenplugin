# LumenPlugin 开发文档

**简体中文** | [English](development-guide-en.md) | [繁體中文](development-guide.md)

> 本文档为 Lumen Player 插件完整开发参考手册，涵盖配置规范、JS 接口、数据结构、开发模式和性能优化。

## 目录

- [1. 概述](#1-概述)
- [2. 插件结构](#2-插件结构)
- [3. 配置文件规范 (config.json)](#3-配置文件规范-configjson)
- [4. JavaScript 接口规范](#4-javascript-接口规范)
- [5. 数据结构](#5-数据结构)
- [6. 开发模式与最佳实践](#6-开发模式与最佳实践)
- [7. 错误处理](#7-错误处理)
- [8. 性能优化](#8-性能优化)
- [9. JavaScriptCore 注意事项](#9-javascriptcore-注意事项)
- [10. 图片质量最佳实践](#10-图片质量最佳实践)
- [11. 宿主 App 集成参考](#11-宿主-app-集成参考)

---

## 1. 概述

LumenPlugin 是 Lumen Player 的内容源插件系统，基于 Apple **JavaScriptCore** 引擎执行 JavaScript 脚本，通过 JSON 配置文件声明式定义插件行为。

**核心流程：**

```
config.json（声明页面/搜索/播放路由）
    ↓
main.js（实现对应的 JS 函数）
    ↓
$http.fetch()（请求目标站点 API）
    ↓
$next.toXxx()（将解析结果回传给 App）
```

**运行环境：**
- JavaScript 引擎：Apple JavaScriptCore（非 V8/Node）
- 不支持浏览器 API（无 `fetch()`、`XMLHttpRequest`、`DOM`）
- 必须使用 Lumen 注入的 `$http` 和 `$next` 接口

---

## 2. 插件结构

每个插件是一个独立目录，至少包含以下文件：

```
my-plugin/
├── config.json          # 插件配置入口（必需）
├── main.js              # 核心 JavaScript 逻辑（必需）
├── main.min.js          # 压缩版本（发布用，可选）
└── crypto-js.min.js     # 第三方依赖库（按需）
```

通过 `config.json` 的 `files` 字段控制加载顺序，第三方库必须排在业务文件之前：

```json
{
  "files": ["crypto-js.min.js", "main.js"]
}
```

---

## 3. 配置文件规范 (config.json)

`config.json` 是插件运行的蓝图，定义了插件的元数据和功能路由。

### 3.1 顶层字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 否 | 插件唯一标识符，如 `lumen.douban` |
| `name` | string | 是 | 插件显示名称 |
| `description` | string | 否 | 插件描述 |
| `host` | string | 是 | 目标站点根域名 |
| `files` | string[] | 是 | 按顺序加载的 JS 文件列表 |
| `heroBanner` | object | 否 | 首页焦点图/轮播图配置 |
| `pages` | array | 是 | 页面配置数组 |
| `episodes` | object | 是 | 剧集功能配置，`{}` = 不支持 |
| `detail` | object | 否 | 额外影片详情（导演、演员、简介） |
| `player` | object | 是 | 播放功能配置，`{}` = 不支持播放 |
| `search` | object | 是 | 搜索功能配置，`{}` = 不支持搜索 |
| `searchSuggestionsList` | object | 否 | 搜索建议聚合 |

### 3.2 完整配置示例

```json
{
  "id": "lumen.example",
  "name": "示例插件",
  "description": "一个完整的插件示例",
  "host": "https://example.com/",
  "files": ["main.js"],
  "heroBanner": {
    "timeout": 20,
    "javascript": "HeroBanner"
  },
  "pages": [
    {
      "key": "home",
      "title": "主页",
      "keys": ["movie", "tv"],
      "timeout": 20,
      "javascript": "Aggregate"
    },
    {
      "key": "movie",
      "title": "电影",
      "url": "https://api.example.com/movies?page=${pageNumber}",
      "timeout": 20,
      "javascript": "Category"
    },
    {
      "key": "tv",
      "title": "剧集",
      "url": "https://api.example.com/tv?page=${pageNumber}",
      "timeout": 20,
      "javascript": "Category"
    }
  ],
  "episodes": {
    "timeout": 20,
    "javascript": "Episodes"
  },
  "detail": {
    "timeout": 25,
    "javascript": "Detail"
  },
  "player": {
    "timeout": 20,
    "javascript": "Player"
  },
  "search": {
    "timeout": 20,
    "url": "https://api.example.com/search?q=${keyword}",
    "javascript": "Search"
  }
}
```

### 3.3 页面配置 (pages[])

`pages` 数组中每个页面对象支持以下字段：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `key` | string | 是 | 页面唯一标识符 |
| `title` | string | 是 | 页面标题，显示在分类列表中 |
| `url` | string | 否 | URL 模板，支持占位符 |
| `tags` | string[] | 否 | 标签/排序选项，第一项为默认 |
| `keys` | string[] | 否 | 聚合子页面的 key 列表 |
| `timeout` | number | 否 | JS 执行超时（秒），默认 20 |
| `javascript` | string | 是 | 要调用的 JS 函数名 |

#### URL 模板占位符

| 占位符 | 说明 |
|--------|------|
| `${pageNumber}` | 当前页码（从 1 开始），用于分页加载 |
| `${tag}` | 当前选中的标签值，配合 `tags` 使用 |
| `${keyword}` | 搜索关键词（仅用于 `search.url`） |

#### 分页示例

```json
{
  "key": "movie",
  "title": "电影",
  "url": "https://api.example.com/movies?page=${pageNumber}&limit=20",
  "timeout": 20,
  "javascript": "Category"
}
```

#### 聚合页面

通过 `keys` 引用其他页面，无需编写 JS 逻辑：

```json
{
  "key": "home",
  "title": "推荐主页",
  "keys": ["movie", "tv", "variety"],
  "timeout": 20,
  "javascript": "Aggregate"
}
```

### 3.4 标签过滤与排序

当页面支持多种排序/过滤时，使用 `tags` + URL 中的 `${tag}` 占位符。

#### 基本用法（标签名即 API 值）

```json
{
  "key": "movie-hot",
  "title": "热门电影",
  "tags": ["全部", "华语", "欧美", "韩国", "日本"],
  "url": "https://api.example.com/movies?type=${tag}&page=${pageNumber}",
  "timeout": 20,
  "javascript": "Category"
}
```

> **默认标签**：`tags` 数组的第一个元素即为默认标签。

#### 标签名与 API 值不一致时的映射

当 UI 标签名与 API 参数值不同时，需在 JS 中做映射：

```json
{
  "tags": ["最近更新", "最多观看", "最高收藏"],
  "url": "https://example.com/list/${pageNumber}/?sort_by=${tag}"
}
```

```javascript
var _tagMap = {
  "最近更新": "post_date",
  "最多观看": "video_viewed",
  "最高收藏": "most_favourited"
};

function _resolveSortBy(url) {
  var m = url.match(/[?&]sort_by=([^&]*)/);
  if (!m) return url;
  var raw = decodeURIComponent(m[1]);
  var mapped = _tagMap[raw];
  if (mapped === undefined) return url;
  return url.replace(/sort_by=[^&]*/, "sort_by=" + mapped);
}

function Category(inputURL) {
  inputURL = _resolveSortBy(inputURL);
  // ...继续正常请求
}
```

> **建议**：当 API 天然接受标签显示名作为参数时，优先使用无映射模式，配置更简洁。

---

## 4. JavaScript 接口规范

Lumen 运行时注入了两个全局对象：`$http`（网络请求）和 `$next`（结果回传）。

> ⚠️ LumenPlugin 不支持浏览器原生的 `fetch()`、`XMLHttpRequest` 等方法，必须使用 Lumen 注入的接口。

### 4.1 HTTP 请求：`$http.fetch(request)`

```javascript
$http.fetch({
  url: "https://api.example.com/data",   // 请求地址（必填）
  method: "GET",                          // 请求方法（可选，默认 GET）
  headers: { "User-Agent": "..." },       // 请求头（可选）
  body: '{"foo":"bar"}'                   // 请求体（可选，用于 POST）
}).then(
  function(res) {
    // res.body — 响应主体（字符串）
    var data = JSON.parse(res.body);
    // 处理数据...
  },
  function(error) {
    // 错误处理
  }
);
```

### 4.2 结果回传：`$next`

每个 JS 函数执行完毕后，**必须**调用对应的 `$next` 回调将结果回传给宿主 App：

| 方法 | 场景 | 参数说明 |
|------|------|----------|
| `$next.toMedias(jsonString)` | 首页 / 分类 / HeroBanner | MediaData JSON 数组 |
| `$next.toEpisodes(jsonString)` | 剧集列表 | EpisodeData JSON 数组 |
| `$next.toSearchMedias(jsonString, keyword)` | 搜索结果 | MediaData JSON 数组 + 原始搜索词 |
| `$next.toPlayer(url)` | 播放地址 | URL 字符串或携带 header 的对象 |
| `$next.toMetadata(jsonString)` | 影片元信息 | MetadataData JSON |
| `$next.emptyView(message)` | 空数据/错误提示 | 提示消息字符串 |

**示例 — 返回媒体列表：**

```javascript
function Category(inputURL) {
  $http.fetch({ url: inputURL }).then(
    function(res) {
      var data = JSON.parse(res.body);
      var medias = data.list.map(function(item) {
        return {
          id: String(item.id),
          coverURLString: item.cover || "",
          title: item.title || "",
          descriptionText: item.year || "",
          detailURLString: item.detail_url || ""
        };
      });
      $next.toMedias(JSON.stringify(medias));
    },
    function(error) {
      $next.toMedias(JSON.stringify([]));
    }
  );
}
```

---

## 5. 数据结构

### 5.1 MediaData（媒体项）

用于首页、分类、搜索结果的媒体展示。

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | ✅ | 唯一标识符，用于详情/剧集查询 |
| `coverURLString` | string | ✅ | 封面图片 URL |
| `title` | string | ✅ | 显示名称 |
| `descriptionText` | string | ✅ | 备注/评分/年份描述 |
| `detailURLString` | string | ✅ | 剧集详情入口链接 |
| `backdropURLString` | string | 否 | 大背景海报图（用于 HeroBanner） |
| `land` | number | 否 | 设为 `1` 使用宽幅横版封面（16:9），默认竖版（2:3） |

### 5.2 EpisodeData（剧集项）

用于剧集列表展示。

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | ✅ | 剧集标识符 |
| `title` | string | ✅ | 集数名称（如「第1集」） |
| `episodeDetailURL` | string | ✅ | 请求播放信息的源链接 |

### 5.3 MetadataData（影片元信息）

用于影片详情页的额外信息展示。

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `directors` | string[] | 否 | 导演列表 |
| `actors` | string[] | 否 | 演员列表 |
| `intro` | string | 否 | 简介内容 |

---

## 6. 开发模式与最佳实践

### 6.1 纯元数据插件

如果目标站点只提供发现和评分功能（如豆瓣），不支持播放：

```json
{
  "episodes": {},
  "player": {}
}
```

在 JS 中使用 `$next.emptyView()` 提示用户：

```javascript
function Player(inputURL) {
  $next.emptyView("该插件仅提供影视信息浏览，不支持在线播放");
}
```

### 6.2 API 签名鉴权

对于需要签名验证的站点，在 JS 中嵌入签名逻辑：

```javascript
function signature() {
  return generateMD5(Date.parse(new Date()) / 1e3);
}

function Category(inputURL) {
  var secureUrl = inputURL + "&_vv=" + signature();
  $http.fetch({ url: secureUrl, method: "GET" }).then(
    function(res) { /* ... */ },
    function(err) { /* ... */ }
  );
}
```

### 6.3 并发请求模式

当需要合并多个 API 结果时（如 HeroBanner），使用**计数器模式**：

```javascript
function HeroBanner() {
  var completed = 0;
  var movies = [], tvs = [];

  function tryResolve() {
    completed++;
    if (completed === 2) {
      var merged = movies.concat(tvs);
      $next.toMedias(JSON.stringify(merged));
    }
  }

  $http.fetch({ url: "api/movie" }).then(
    function(res) { movies = parse(res); tryResolve(); },
    function() { tryResolve(); }
  );
  $http.fetch({ url: "api/tv" }).then(
    function(res) { tvs = parse(res); tryResolve(); },
    function() { tryResolve(); }
  );
}
```

### 6.4 调试开关

在 `main.js` 顶部添加统一的调试开关：

```javascript
var DEBUG = false; // 发布时设为 false

var _perfStart = Date.now();

function _perf(label) {
  if (!DEBUG) return;
  console.log(JSON.stringify({ PERF: label, ms: Date.now() - _perfStart }));
}

function _perfReset() {
  if (!DEBUG) return;
  _perfStart = Date.now();
}

function print(params) {
  if (!DEBUG) return;
  console.log(JSON.stringify(params));
}
```

---

## 7. 错误处理

LumenPlugin 依托异步 JSON 取值，必须做好异常捕获，防止 JS 错误导致宿主 App 崩溃。

### 7.1 JSON 解析防护

```javascript
$http.fetch({ url: inputURL }).then(function(res) {
  try {
    var data = JSON.parse(res.body);
    // 正常处理...
  } catch(e) {
    $next.toMedias(JSON.stringify([]));
  }
});
```

### 7.2 缺失字段补全

绝对不能传入 `undefined` 或 `null` 给 `$next` 方法，应使用空字符串填充：

```javascript
function buildMediaData(item) {
  return {
    id: String(item.id || ""),
    coverURLString: item.cover || "",
    title: item.title || "未知",
    descriptionText: item.desc || "",
    detailURLString: item.url || ""
  };
}
```

---

## 8. 性能优化

### 8.1 `lumen_for_cover` 协议

当 App 仅需获取分类封面图时，会在 URL 中追加 `lumen_for_cover=1` 参数。插件应检测此参数并走快速路径。

**检测方式：**

```javascript
function Category(inputURL) {
  var isForCover = inputURL.indexOf("lumen_for_cover=1") >= 0;
  if (isForCover) {
    // 快速路径：只返回 1 条数据
  } else {
    // 正常路径：返回完整列表
  }
}
```

**优化策略：**

| 场景 | 优化方式 |
|------|----------|
| 标准分页 API | 将 `limit=20` 替换为 `limit=1` |
| 多次串行请求 | 取第一条数据立即返回，跳过后续请求 |
| HTML 解析榜单 | 只保留第一项，跳过详情请求 |

**减少 limit 示例：**

```javascript
if (isForCover && requestURL.indexOf("limit=") >= 0) {
  requestURL = requestURL.replace(/limit=\d+/, "limit=1");
}
```

**提前短路示例：**

```javascript
if (isForCover) {
  $http.fetch({ url: inputURL }).then(function(res) {
    var data = JSON.parse(res.body);
    var medias = parseMedias(data);
    $next.toMedias(JSON.stringify(medias.length > 0 ? [medias[0]] : []));
  });
  return; // 提前返回，不执行完整加载
}
```

---

## 9. JavaScriptCore 注意事项

Lumen 使用 Apple JavaScriptCore 而非 V8/Node，存在以下关键差异：

### 9.1 ⚠️ 禁止在块作用域内声明函数

在 `if`/`else`/`for` 块内部使用 `function` 声明会导致函数引用丢失：

```javascript
// ❌ 错误 — 函数可能在回调中不可访问
if (condition) {
  function myHelper() { /* ... */ }
  asyncFetch(function() { myHelper(); }); // 可能失败！
}

// ✅ 正确 — 使用 var 函数表达式
var myHelper = null;
if (condition) {
  myHelper = function() { /* ... */ };
  asyncFetch(function() { myHelper(); }); // 正常工作
}
```

### 9.2 优先使用 `var` 声明

在复杂多层嵌套回调中，推荐使用 `var` 而非 `const`/`let`，确保变量跨所有闭包可见。

### 9.3 无 `Promise.all` 支持

`$http.fetch` 不使用原生 Promise，并发请求应使用[计数器模式](#63-并发请求模式)。

---

## 10. 图片质量最佳实践

### 10.1 HeroBanner 优先使用横版图

焦点图应优先使用横版剧照（`picSlide`），而非竖版海报：

```javascript
var backdropURL = item.picSlide || item.pic_slide || item.backdrop || "";
var coverURL = item.pic || item.cover || "";
return buildMediaData(id, coverURL, title, desc, detailURL, backdropURL);
```

### 10.2 未配置 heroBanner 时的行为

若 `config.json` 未配置 `heroBanner` 字段，App 会自动跳过焦点图显示，无需额外处理。

---

## 11. 宿主 App 集成参考

以下表格说明 App（Swift 侧）与插件 JS 的交互行为，供高级开发者参考：

| 场景 | App 行为 | 插件 JS 对应 |
|------|----------|-------------|
| 分类列表封面 | URL 追加 `lumen_for_cover=1` | 检测参数进入快速路径 |
| 首页并发加载 | 并发调用所有分类页 | 每个 `Category()` 独立执行 |
| 切换站点 | 清空缓存，重新加载 | 插件重新解析所有分类 |
| 标签切换 | 替换 URL 中 `${tag}` | JS 需映射标签名为 API 值（若不一致） |
| 宽幅封面 | `land >= 1` 时使用横版布局 | 返回 `land: 1` |
