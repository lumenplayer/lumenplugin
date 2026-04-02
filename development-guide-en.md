# LumenPlugin Development Guide

**English** | [繁體中文](development-guide.md) | [简体中文](development-guide-cn.md)

> Complete developer reference for building Lumen Player content source plugins, covering configuration specs, JS APIs, data structures, development patterns, and performance optimization.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Plugin Structure](#2-plugin-structure)
- [3. Configuration Spec (config.json)](#3-configuration-spec-configjson)
- [4. JavaScript API Reference](#4-javascript-api-reference)
- [5. Data Structures](#5-data-structures)
- [6. Development Patterns & Best Practices](#6-development-patterns--best-practices)
- [7. Error Handling](#7-error-handling)
- [8. Performance Optimization](#8-performance-optimization)
- [9. JavaScriptCore Caveats](#9-javascriptcore-caveats)
- [10. Image Quality Best Practices](#10-image-quality-best-practices)
- [11. Host App Integration Reference](#11-host-app-integration-reference)

---

## 1. Overview

LumenPlugin is the content source plugin system for Lumen Player. It runs JavaScript via Apple's **JavaScriptCore** engine and uses JSON configuration files to declaratively define plugin behavior.

**Core Flow:**

```
config.json (declare page/search/player routes)
    ↓
main.js (implement the corresponding JS functions)
    ↓
$http.fetch() (request the target site's API)
    ↓
$next.toXxx() (return parsed results to the App)
```

**Runtime Environment:**
- JavaScript engine: Apple JavaScriptCore (not V8/Node)
- No browser APIs (`fetch()`, `XMLHttpRequest`, `DOM` are unavailable)
- Must use Lumen-injected `$http` and `$next` interfaces

---

## 2. Plugin Structure

Each plugin is a standalone directory containing at least the following files:

```
my-plugin/
├── config.json          # Plugin configuration entry (required)
├── main.js              # Core JavaScript logic (required)
├── main.min.js          # Minified version (for release, optional)
└── crypto-js.min.js     # Third-party dependencies (as needed)
```

The `files` field in `config.json` controls the loading order. Third-party libraries must be listed before business logic files:

```json
{
  "files": ["crypto-js.min.js", "main.js"]
}
```

---

## 3. Configuration Spec (config.json)

`config.json` is the blueprint of the plugin, defining its metadata and feature routing.

### 3.1 Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | No | Unique plugin identifier, e.g. `lumen.douban` |
| `name` | string | Yes | Plugin display name |
| `description` | string | No | Plugin description |
| `host` | string | Yes | Target site root domain |
| `files` | string[] | Yes | JS files to load, in order |
| `heroBanner` | object | No | Hero banner / carousel configuration |
| `pages` | array | Yes | Page configuration array |
| `episodes` | object | Yes | Episodes config; `{}` = not supported |
| `detail` | object | No | Additional detail info (director, actors, synopsis) |
| `player` | object | Yes | Playback config; `{}` = playback not supported |
| `search` | object | Yes | Search config; `{}` = search not supported |
| `searchSuggestionsList` | object | No | Search suggestions aggregation |

### 3.2 Full Configuration Example

```json
{
  "id": "lumen.example",
  "name": "Example Plugin",
  "description": "A complete plugin example",
  "host": "https://example.com/",
  "files": ["main.js"],
  "heroBanner": {
    "timeout": 20,
    "javascript": "HeroBanner"
  },
  "pages": [
    {
      "key": "home",
      "title": "Home",
      "keys": ["movie", "tv"],
      "timeout": 20,
      "javascript": "Aggregate"
    },
    {
      "key": "movie",
      "title": "Movies",
      "url": "https://api.example.com/movies?page=${pageNumber}",
      "timeout": 20,
      "javascript": "Category"
    },
    {
      "key": "tv",
      "title": "TV Shows",
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

### 3.3 Page Configuration (pages[])

Each page object in the `pages` array supports the following fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `key` | string | Yes | Unique page identifier |
| `title` | string | Yes | Page title shown in category list |
| `url` | string | No | URL template with placeholder support |
| `tags` | string[] | No | Tag/sort options; first item is the default |
| `keys` | string[] | No | List of page keys for aggregation |
| `timeout` | number | No | JS execution timeout (seconds), default 20 |
| `javascript` | string | Yes | Name of the JS function to invoke |

#### URL Template Placeholders

| Placeholder | Description |
|-------------|-------------|
| `${pageNumber}` | Current page number (starts from 1) for pagination |
| `${tag}` | Currently selected tag value, used with `tags` |
| `${keyword}` | Search keyword (only for `search.url`) |

#### Pagination Example

```json
{
  "key": "movie",
  "title": "Movies",
  "url": "https://api.example.com/movies?page=${pageNumber}&limit=20",
  "timeout": 20,
  "javascript": "Category"
}
```

#### Aggregate Pages

Use `keys` to reference other pages without writing JS logic:

```json
{
  "key": "home",
  "title": "Featured",
  "keys": ["movie", "tv", "variety"],
  "timeout": 20,
  "javascript": "Aggregate"
}
```

### 3.4 Tag Filtering & Sorting

When a page supports multiple sort/filter options, use `tags` + the `${tag}` URL placeholder.

#### Basic Usage (Tag Name = API Value)

```json
{
  "key": "movie-hot",
  "title": "Hot Movies",
  "tags": ["All", "Chinese", "Western", "Korean", "Japanese"],
  "url": "https://api.example.com/movies?type=${tag}&page=${pageNumber}",
  "timeout": 20,
  "javascript": "Category"
}
```

> **Default Tag**: The first element of the `tags` array is the default tag.

#### Mapping When Tag Names Differ from API Values

When UI tag names differ from API parameter values, map them in JS:

```json
{
  "tags": ["Recent", "Most Viewed", "Most Saved"],
  "url": "https://example.com/list/${pageNumber}/?sort_by=${tag}"
}
```

```javascript
var _tagMap = {
  "Recent": "post_date",
  "Most Viewed": "video_viewed",
  "Most Saved": "most_favourited"
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
  // ...proceed with the request
}
```

> **Tip**: When the API natively accepts the display name as a parameter value, prefer the no-mapping approach for simpler configuration.

---

## 4. JavaScript API Reference

Lumen runtime injects two global objects: `$http` (network requests) and `$next` (result callbacks).

> ⚠️ LumenPlugin does NOT support browser-native `fetch()`, `XMLHttpRequest`, or similar APIs. You must use Lumen's injected interfaces.

### 4.1 HTTP Requests: `$http.fetch(request)`

```javascript
$http.fetch({
  url: "https://api.example.com/data",   // Request URL (required)
  method: "GET",                          // HTTP method (optional, defaults to GET)
  headers: { "User-Agent": "..." },       // Request headers (optional)
  body: '{"foo":"bar"}'                   // Request body (optional, for POST)
}).then(
  function(res) {
    // res.body — response body (string)
    var data = JSON.parse(res.body);
    // Process data...
  },
  function(error) {
    // Error handling
  }
);
```

### 4.2 Result Callbacks: `$next`

After each JS function completes execution, it **must** call the corresponding `$next` callback to return results to the host App:

| Method | Scenario | Parameters |
|--------|----------|------------|
| `$next.toMedias(jsonString)` | Home / Category / HeroBanner | MediaData JSON array |
| `$next.toEpisodes(jsonString)` | Episode list | EpisodeData JSON array |
| `$next.toSearchMedias(jsonString, keyword)` | Search results | MediaData JSON array + original search keyword |
| `$next.toPlayer(url)` | Playback URL | URL string or object with headers |
| `$next.toMetadata(jsonString)` | Movie metadata | MetadataData JSON |
| `$next.emptyView(message)` | Empty state / error | Message string |

**Example — Returning a media list:**

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

## 5. Data Structures

### 5.1 MediaData

Used for media display on home, category, and search result pages.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique identifier for detail/episode queries |
| `coverURLString` | string | ✅ | Cover image URL |
| `title` | string | ✅ | Display name |
| `descriptionText` | string | ✅ | Notes / rating / year description |
| `detailURLString` | string | ✅ | Detail entry URL for episode retrieval |
| `backdropURLString` | string | No | Large backdrop poster (for HeroBanner) |
| `land` | number | No | Set to `1` for landscape cover (16:9); default is portrait (2:3) |

### 5.2 EpisodeData

Used for episode list display.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Episode identifier |
| `title` | string | ✅ | Episode name (e.g. "Episode 1") |
| `episodeDetailURL` | string | ✅ | Source URL for playback info |

### 5.3 MetadataData

Used for additional information on movie detail pages.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `directors` | string[] | No | List of directors |
| `actors` | string[] | No | List of actors |
| `intro` | string | No | Synopsis |

---

## 6. Development Patterns & Best Practices

### 6.1 Metadata-Only Plugins

If the target site only provides discovery and rating features (e.g. Douban) without playback support:

```json
{
  "episodes": {},
  "player": {}
}
```

Use `$next.emptyView()` in JS to inform the user:

```javascript
function Player(inputURL) {
  $next.emptyView("This plugin provides browsing only, playback is not supported");
}
```

### 6.2 API Signature Authentication

For sites requiring signature verification, embed the signing logic in JS:

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

### 6.3 Concurrent Request Pattern

When combining results from multiple APIs (e.g. HeroBanner), use the **counter pattern**:

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

### 6.4 Debug Toggle

Add a unified debug toggle at the top of `main.js`:

```javascript
var DEBUG = false; // Set to false for release

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

## 7. Error Handling

LumenPlugin relies on asynchronous JSON parsing. Proper exception handling is essential to prevent JS errors from crashing the host App.

### 7.1 JSON Parse Guard

```javascript
$http.fetch({ url: inputURL }).then(function(res) {
  try {
    var data = JSON.parse(res.body);
    // Normal processing...
  } catch(e) {
    $next.toMedias(JSON.stringify([]));
  }
});
```

### 7.2 Missing Field Defaults

Never pass `undefined` or `null` to `$next` methods. Use empty strings as fallbacks:

```javascript
function buildMediaData(item) {
  return {
    id: String(item.id || ""),
    coverURLString: item.cover || "",
    title: item.title || "Unknown",
    descriptionText: item.desc || "",
    detailURLString: item.url || ""
  };
}
```

---

## 8. Performance Optimization

### 8.1 `lumen_for_cover` Protocol

When the App only needs a category's cover image (not the full content list), it appends `lumen_for_cover=1` to the request URL. Plugins should detect this parameter and take a fast path.

**Detection:**

```javascript
function Category(inputURL) {
  var isForCover = inputURL.indexOf("lumen_for_cover=1") >= 0;
  if (isForCover) {
    // Fast path: return only 1 item
  } else {
    // Normal path: return full list
  }
}
```

**Optimization Strategies:**

| Scenario | Optimization |
|----------|-------------|
| Standard paginated API | Replace `limit=20` with `limit=1` |
| Multiple sequential requests | Return the first item immediately, skip remaining requests |
| HTML-parsed charts | Keep only the first item, skip detail requests |

**Reduce limit example:**

```javascript
if (isForCover && requestURL.indexOf("limit=") >= 0) {
  requestURL = requestURL.replace(/limit=\d+/, "limit=1");
}
```

**Early return example:**

```javascript
if (isForCover) {
  $http.fetch({ url: inputURL }).then(function(res) {
    var data = JSON.parse(res.body);
    var medias = parseMedias(data);
    $next.toMedias(JSON.stringify(medias.length > 0 ? [medias[0]] : []));
  });
  return; // Early return, skip full loading
}
```

---

## 9. JavaScriptCore Caveats

Lumen uses Apple JavaScriptCore instead of V8/Node. Be aware of the following key differences:

### 9.1 ⚠️ Do NOT Declare Functions Inside Block Scopes

Using `function` declarations inside `if`/`else`/`for` blocks causes function reference loss:

```javascript
// ❌ Wrong — function may be inaccessible in callbacks
if (condition) {
  function myHelper() { /* ... */ }
  asyncFetch(function() { myHelper(); }); // May fail!
}

// ✅ Correct — use var function expression
var myHelper = null;
if (condition) {
  myHelper = function() { /* ... */ };
  asyncFetch(function() { myHelper(); }); // Works correctly
}
```

### 9.2 Prefer `var` Declarations

In complex nested callbacks, prefer `var` over `const`/`let` to ensure variables are visible across all closures.

### 9.3 No `Promise.all` Support

`$http.fetch` does not use native Promises. For concurrent requests, use the [counter pattern](#63-concurrent-request-pattern).

---

## 10. Image Quality Best Practices

### 10.1 Use Landscape Images for HeroBanner

Hero banners should prefer landscape stills (`picSlide`) over portrait posters:

```javascript
var backdropURL = item.picSlide || item.pic_slide || item.backdrop || "";
var coverURL = item.pic || item.cover || "";
return buildMediaData(id, coverURL, title, desc, detailURL, backdropURL);
```

### 10.2 Behavior When heroBanner Is Not Configured

If `config.json` does not include a `heroBanner` field, the App automatically skips the hero banner display. No additional handling is needed.

---

## 11. Host App Integration Reference

The following table describes how the host App (Swift side) interacts with plugin JS, for advanced developers:

| Scenario | App Behavior | Plugin JS Counterpart |
|----------|-------------|----------------------|
| Category cover image | Appends `lumen_for_cover=1` to URL | Detect parameter and enter fast path |
| Home concurrent loading | Concurrently invokes all category pages | Each `Category()` executes independently |
| Switch site | Clears cache, reloads | Plugin re-parses all categories |
| Tag switch | Replaces `${tag}` in URL | JS must map tag names to API values (if different) |
| Landscape cover | Uses landscape layout when `land >= 1` | Return `land: 1` |
