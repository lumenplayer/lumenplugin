# LumenPlugin 開發文檔

**繁體中文** | [English](development-guide-en.md) | [简体中文](development-guide-cn.md)

> 本文檔為 Lumen Player 插件完整開發參考手冊，涵蓋配置規範、JS 介面、資料結構、開發模式和效能優化。

## 目錄

- [1. 概述](#1-概述)
- [2. 插件結構](#2-插件結構)
- [3. 配置檔案規範 (config.json)](#3-配置檔案規範-configjson)
- [4. JavaScript 介面規範](#4-javascript-介面規範)
- [5. 資料結構](#5-資料結構)
- [6. 開發模式與最佳實踐](#6-開發模式與最佳實踐)
- [7. 錯誤處理](#7-錯誤處理)
- [8. 效能優化](#8-效能優化)
- [9. JavaScriptCore 注意事項](#9-javascriptcore-注意事項)
- [10. 圖片品質最佳實踐](#10-圖片品質最佳實踐)
- [11. 宿主 App 整合參考](#11-宿主-app-整合參考)

---

## 1. 概述

LumenPlugin 是 Lumen Player 的內容源插件系統，基於 Apple **JavaScriptCore** 引擎執行 JavaScript 腳本，透過 JSON 配置檔案宣告式定義插件行為。

**核心流程：**

```
config.json（宣告頁面/搜尋/播放路由）
    ↓
main.js（實作對應的 JS 函式）
    ↓
$http.fetch()（請求目標站點 API）
    ↓
$next.toXxx()（將解析結果回傳給 App）
```

**執行環境：**
- JavaScript 引擎：Apple JavaScriptCore（非 V8/Node）
- 不支援瀏覽器 API（無 `fetch()`、`XMLHttpRequest`、`DOM`）
- 必須使用 Lumen 注入的 `$http` 和 `$next` 介面

---

## 2. 插件結構

每個插件是一個獨立目錄，至少包含以下檔案：

```
my-plugin/
├── config.json          # 插件配置入口（必需）
├── main.js              # 核心 JavaScript 邏輯（必需）
├── main.min.js          # 壓縮版本（發佈用，可選）
└── crypto-js.min.js     # 第三方依賴庫（按需）
```

透過 `config.json` 的 `files` 欄位控制載入順序，第三方庫必須排在業務檔案之前：

```json
{
  "files": ["crypto-js.min.js", "main.js"]
}
```

---

## 3. 配置檔案規範 (config.json)

`config.json` 是插件運行的藍圖，定義了插件的中繼資料和功能路由。

### 3.1 頂層欄位

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `id` | string | 否 | 插件唯一識別碼，如 `lumen.douban` |
| `name` | string | 是 | 插件顯示名稱 |
| `description` | string | 否 | 插件描述 |
| `host` | string | 是 | 目標站點根域名 |
| `files` | string[] | 是 | 按順序載入的 JS 檔案列表 |
| `heroBanner` | object | 否 | 首頁焦點圖/輪播圖配置 |
| `pages` | array | 是 | 頁面配置陣列 |
| `episodes` | object | 是 | 劇集功能配置，`{}` = 不支援 |
| `detail` | object | 否 | 額外影片詳情（導演、演員、簡介） |
| `player` | object | 是 | 播放功能配置，`{}` = 不支援播放 |
| `search` | object | 是 | 搜尋功能配置，`{}` = 不支援搜尋 |
| `searchSuggestionsList` | object | 否 | 搜尋建議聚合 |

### 3.2 完整配置範例

```json
{
  "id": "lumen.example",
  "name": "範例插件",
  "description": "一個完整的插件範例",
  "host": "https://example.com/",
  "files": ["main.js"],
  "heroBanner": {
    "timeout": 20,
    "javascript": "HeroBanner"
  },
  "pages": [
    {
      "key": "home",
      "title": "主頁",
      "keys": ["movie", "tv"],
      "timeout": 20,
      "javascript": "Aggregate"
    },
    {
      "key": "movie",
      "title": "電影",
      "url": "https://api.example.com/movies?page=${pageNumber}",
      "timeout": 20,
      "javascript": "Category"
    },
    {
      "key": "tv",
      "title": "劇集",
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

### 3.3 頁面配置 (pages[])

`pages` 陣列中每個頁面物件支援以下欄位：

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `key` | string | 是 | 頁面唯一識別碼 |
| `title` | string | 是 | 頁面標題，顯示在分類列表中 |
| `url` | string | 否 | URL 範本，支援佔位符 |
| `tags` | string[] | 否 | 標籤/排序選項，第一項為預設 |
| `keys` | string[] | 否 | 聚合子頁面的 key 列表 |
| `timeout` | number | 否 | JS 執行逾時（秒），預設 20 |
| `javascript` | string | 是 | 要呼叫的 JS 函式名 |

#### URL 範本佔位符

| 佔位符 | 說明 |
|--------|------|
| `${pageNumber}` | 當前頁碼（從 1 開始），用於分頁載入 |
| `${tag}` | 當前選中的標籤值，配合 `tags` 使用 |
| `${keyword}` | 搜尋關鍵詞（僅用於 `search.url`） |

#### 分頁範例

```json
{
  "key": "movie",
  "title": "電影",
  "url": "https://api.example.com/movies?page=${pageNumber}&limit=20",
  "timeout": 20,
  "javascript": "Category"
}
```

#### 聚合頁面

透過 `keys` 引用其他頁面，無需撰寫 JS 邏輯：

```json
{
  "key": "home",
  "title": "推薦主頁",
  "keys": ["movie", "tv", "variety"],
  "timeout": 20,
  "javascript": "Aggregate"
}
```

### 3.4 標籤篩選與排序

當頁面支援多種排序/篩選時，使用 `tags` + URL 中的 `${tag}` 佔位符。

#### 基本用法（標籤名即 API 值）

```json
{
  "key": "movie-hot",
  "title": "熱門電影",
  "tags": ["全部", "華語", "歐美", "韓國", "日本"],
  "url": "https://api.example.com/movies?type=${tag}&page=${pageNumber}",
  "timeout": 20,
  "javascript": "Category"
}
```

> **預設標籤**：`tags` 陣列的第一個元素即為預設標籤。

#### 標籤名與 API 值不一致時的映射

當 UI 標籤名與 API 參數值不同時，需在 JS 中做映射：

```json
{
  "tags": ["最近更新", "最多觀看", "最高收藏"],
  "url": "https://example.com/list/${pageNumber}/?sort_by=${tag}"
}
```

```javascript
var _tagMap = {
  "最近更新": "post_date",
  "最多觀看": "video_viewed",
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
  // ...繼續正常請求
}
```

> **建議**：當 API 天然接受標籤顯示名作為參數時，優先使用無映射模式，配置更簡潔。

---

## 4. JavaScript 介面規範

Lumen 執行時注入了兩個全域物件：`$http`（網路請求）和 `$next`（結果回傳）。

> ⚠️ LumenPlugin 不支援瀏覽器原生的 `fetch()`、`XMLHttpRequest` 等方法，必須使用 Lumen 注入的介面。

### 4.1 HTTP 請求：`$http.fetch(request)`

```javascript
$http.fetch({
  url: "https://api.example.com/data",   // 請求位址（必填）
  method: "GET",                          // 請求方法（可選，預設 GET）
  headers: { "User-Agent": "..." },       // 請求標頭（可選）
  body: '{"foo":"bar"}'                   // 請求主體（可選，用於 POST）
}).then(
  function(res) {
    // res.body — 回應主體（字串）
    var data = JSON.parse(res.body);
    // 處理資料...
  },
  function(error) {
    // 錯誤處理
  }
);
```

### 4.2 結果回傳：`$next`

每個 JS 函式執行完畢後，**必須**呼叫對應的 `$next` 回呼將結果回傳給宿主 App：

| 方法 | 場景 | 參數說明 |
|------|------|----------|
| `$next.toMedias(jsonString)` | 首頁 / 分類 / HeroBanner | MediaData JSON 陣列 |
| `$next.toEpisodes(jsonString)` | 劇集列表 | EpisodeData JSON 陣列 |
| `$next.toSearchMedias(jsonString, keyword)` | 搜尋結果 | MediaData JSON 陣列 + 原始搜尋詞 |
| `$next.toPlayer(url)` | 播放位址 | URL 字串或攜帶 header 的物件 |
| `$next.toMetadata(jsonString)` | 影片中繼資訊 | MetadataData JSON |
| `$next.emptyView(message)` | 空資料/錯誤提示 | 提示訊息字串 |

**範例 — 回傳媒體列表：**

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

## 5. 資料結構

### 5.1 MediaData（媒體項）

用於首頁、分類、搜尋結果的媒體展示。

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `id` | string | ✅ | 唯一識別碼，用於詳情/劇集查詢 |
| `coverURLString` | string | ✅ | 封面圖片 URL |
| `title` | string | ✅ | 顯示名稱 |
| `descriptionText` | string | ✅ | 備註/評分/年份描述 |
| `detailURLString` | string | ✅ | 劇集詳情入口連結 |
| `backdropURLString` | string | 否 | 大背景海報圖（用於 HeroBanner） |
| `land` | number | 否 | 設為 `1` 使用寬幅橫版封面（16:9），預設直版（2:3） |

### 5.2 EpisodeData（劇集項）

用於劇集列表展示。

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `id` | string | ✅ | 劇集識別碼 |
| `title` | string | ✅ | 集數名稱（如「第1集」） |
| `episodeDetailURL` | string | ✅ | 請求播放資訊的來源連結 |

### 5.3 MetadataData（影片中繼資訊）

用於影片詳情頁的額外資訊展示。

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| `directors` | string[] | 否 | 導演列表 |
| `actors` | string[] | 否 | 演員列表 |
| `intro` | string | 否 | 簡介內容 |

---

## 6. 開發模式與最佳實踐

### 6.1 純中繼資料插件

如果目標站點只提供發現和評分功能（如豆瓣），不支援播放：

```json
{
  "episodes": {},
  "player": {}
}
```

在 JS 中使用 `$next.emptyView()` 提示使用者：

```javascript
function Player(inputURL) {
  $next.emptyView("該插件僅提供影視資訊瀏覽，不支援線上播放");
}
```

### 6.2 API 簽章驗證

對於需要簽章驗證的站點，在 JS 中嵌入簽章邏輯：

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

### 6.3 並行請求模式

當需要合併多個 API 結果時（如 HeroBanner），使用**計數器模式**：

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

### 6.4 除錯開關

在 `main.js` 頂部新增統一的除錯開關：

```javascript
var DEBUG = false; // 發佈時設為 false

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

## 7. 錯誤處理

LumenPlugin 依託非同步 JSON 取值，必須做好例外捕獲，防止 JS 錯誤導致宿主 App 崩潰。

### 7.1 JSON 解析防護

```javascript
$http.fetch({ url: inputURL }).then(function(res) {
  try {
    var data = JSON.parse(res.body);
    // 正常處理...
  } catch(e) {
    $next.toMedias(JSON.stringify([]));
  }
});
```

### 7.2 缺失欄位補全

絕對不能傳入 `undefined` 或 `null` 給 `$next` 方法，應使用空字串填充：

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

## 8. 效能優化

### 8.1 `lumen_for_cover` 協議

當 App 僅需取得分類封面圖時，會在 URL 中追加 `lumen_for_cover=1` 參數。插件應偵測此參數並走快速路徑。

**偵測方式：**

```javascript
function Category(inputURL) {
  var isForCover = inputURL.indexOf("lumen_for_cover=1") >= 0;
  if (isForCover) {
    // 快速路徑：只回傳 1 筆資料
  } else {
    // 正常路徑：回傳完整列表
  }
}
```

**優化策略：**

| 場景 | 優化方式 |
|------|----------|
| 標準分頁 API | 將 `limit=20` 替換為 `limit=1` |
| 多次串列請求 | 取第一筆資料立即回傳，跳過後續請求 |
| HTML 解析排行榜 | 只保留第一項，跳過詳情請求 |

**減少 limit 範例：**

```javascript
if (isForCover && requestURL.indexOf("limit=") >= 0) {
  requestURL = requestURL.replace(/limit=\d+/, "limit=1");
}
```

**提前短路範例：**

```javascript
if (isForCover) {
  $http.fetch({ url: inputURL }).then(function(res) {
    var data = JSON.parse(res.body);
    var medias = parseMedias(data);
    $next.toMedias(JSON.stringify(medias.length > 0 ? [medias[0]] : []));
  });
  return; // 提前回傳，不執行完整載入
}
```

---

## 9. JavaScriptCore 注意事項

Lumen 使用 Apple JavaScriptCore 而非 V8/Node，存在以下關鍵差異：

### 9.1 ⚠️ 禁止在區塊作用域內宣告函式

在 `if`/`else`/`for` 區塊內部使用 `function` 宣告會導致函式參照遺失：

```javascript
// ❌ 錯誤 — 函式可能在回呼中不可存取
if (condition) {
  function myHelper() { /* ... */ }
  asyncFetch(function() { myHelper(); }); // 可能失敗！
}

// ✅ 正確 — 使用 var 函式運算式
var myHelper = null;
if (condition) {
  myHelper = function() { /* ... */ };
  asyncFetch(function() { myHelper(); }); // 正常運作
}
```

### 9.2 優先使用 `var` 宣告

在複雜多層巢狀回呼中，建議使用 `var` 而非 `const`/`let`，確保變數跨所有閉包可見。

### 9.3 無 `Promise.all` 支援

`$http.fetch` 不使用原生 Promise，並行請求應使用[計數器模式](#63-並行請求模式)。

---

## 10. 圖片品質最佳實踐

### 10.1 HeroBanner 優先使用橫版圖

焦點圖應優先使用橫版劇照（`picSlide`），而非直版海報：

```javascript
var backdropURL = item.picSlide || item.pic_slide || item.backdrop || "";
var coverURL = item.pic || item.cover || "";
return buildMediaData(id, coverURL, title, desc, detailURL, backdropURL);
```

### 10.2 未配置 heroBanner 時的行為

若 `config.json` 未配置 `heroBanner` 欄位，App 會自動跳過焦點圖顯示，無需額外處理。

---

## 11. 宿主 App 整合參考

以下表格說明 App（Swift 側）與插件 JS 的互動行為，供進階開發者參考：

| 場景 | App 行為 | 插件 JS 對應 |
|------|----------|-------------|
| 分類列表封面 | URL 追加 `lumen_for_cover=1` | 偵測參數進入快速路徑 |
| 首頁並行載入 | 並行呼叫所有分類頁 | 每個 `Category()` 獨立執行 |
| 切換站點 | 清空快取，重新載入 | 插件重新解析所有分類 |
| 標籤切換 | 替換 URL 中 `${tag}` | JS 需映射標籤名為 API 值（若不一致） |
| 寬幅封面 | `land >= 1` 時使用橫版版面 | 回傳 `land: 1` |
