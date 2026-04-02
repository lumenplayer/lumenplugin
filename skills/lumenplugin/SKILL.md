---
name: lumenplugin
description: Lumen Player plugin development specifications and guide for building video source plugins using JavaScript and JSON configuration.
---

# LumenPlugin Development Guide

This document defines the development specifications and best practices for building content source plugins (LumenPlugin) for Lumen Player. LumenPlugin is a lightweight cross-platform video plugin system powered by Apple's `JavaScriptCore` engine and `JSON` configuration parsing.

## 1. Plugin Structure

Each plugin exists as a standalone directory (e.g. `douban/`, `olevod/`) and must contain the following files:

- `config.json`: The plugin's configuration entry point, defining site metadata and page routing.
- `main.js` (or `main.min.js`): Core JavaScript business logic.

You can include third-party libraries (e.g. `crypto-js.min.js`) via the `"files"` field in `config.json`. Third-party libraries must be loaded before business logic files:

```json
{
  "files": ["crypto-js.min.js", "main.js"]
}
```

## 2. Configuration Spec (config.json)

`config.json` is the blueprint for plugin execution. Field definitions:

| Field | Type | Description |
|-------|------|-------------|
| `id` / `name` / `description` / `host` | string | Basic metadata. `host` is the target site's root domain. |
| `files` | string[] | Ordered list of JS source files to load. |
| `heroBanner` | object | Hero banner / carousel config (`{ "timeout": 20, "javascript": "HeroBanner" }`). |
| `pages` | array | Page configuration array for discovery/category pages. |
| `episodes` | object | Episode feature config. Empty `{}` means episodes not supported. |
| `detail` | object | (Optional) Additional movie detail config (director, actors, synopsis). |
| `player` | object | Playback config. Empty `{}` means metadata-only, no playback support. |
| `search` | object | Search config. Empty `{}` means search not supported. |
| `searchSuggestionsList` | object | Search suggestions aggregation using `keys` to aggregate page content. |

### 2.1 Page Item Fields (pages[])

Each page object in the `pages` array supports the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `key` | string | Unique page identifier for aggregation and internal routing. |
| `title` | string | Page title displayed in category lists and tab bars. |
| `url` | string | URL template supporting `${pageNumber}` (page number) and `${tag}` (tag value) placeholders. |
| `tags` | string[] | (Optional) Tag/sort option list. **The first item is the default tag.** See §2.3. |
| `keys` | string[] | (Optional) List of sub-page keys for `Aggregate` type pages. |
| `timeout` | number | (Optional) JS execution timeout in seconds, default 20. |
| `javascript` | string | Name of the JS function to invoke. Set to `"Aggregate"` for aggregate pages (no JS execution). |

### 2.2 URL Placeholders & Aggregation

In `pages` configuration, use the URL template placeholder `${pageNumber}` (starting from 1) for pagination:

```json
{
  "key": "movie-hot",
  "title": "Movies",
  "url": "https://api.example.com/movies?page=${pageNumber}",
  "timeout": 20,
  "javascript": "Category"
}
```

You can create aggregate pages (e.g. homepage) that reuse results from other existing sections without writing additional JS logic, by setting `keys`:

```json
{
  "key": "home",
  "title": "Featured",
  "keys": ["movie-hot", "tv-hot"],
  "timeout": 20,
  "javascript": "Aggregate"
}
```

The `searchSuggestionsList` config also commonly uses aggregation.

### 2.3 Tag Filtering & Sorting (tags + `${tag}`)

When a page supports multiple sort/filter options (e.g. "Recent", "Most Viewed", "Most Saved"), use the `tags` array + `${tag}` URL placeholder for tag switching.

#### Configuration

Add a `tags` array and the `${tag}` placeholder in the URL template for the corresponding page. Example with a **Douban plugin** filtering hot movies by region:

```json
{
  "key": "movie-hot",
  "title": "Hot Movies",
  "tags": ["All", "Chinese", "Western", "Korean", "Japanese"],
  "url": "https://m.douban.com/rexxar/api/v2/subject/recent_hot/movie?pageNumber=${pageNumber}&category=hot&type=${tag}",
  "timeout": 20,
  "javascript": "Category"
}
```

When the user selects "Chinese", `${tag}` is replaced with `Chinese`, resulting in:
```
.../recent_hot/movie?pageNumber=1&category=hot&type=Chinese
```

If the API natively accepts the display name as parameter values, JS can use `inputURL` directly without any mapping.

#### Default Tag

**The first element of the `tags` array is the default tag.** The App automatically selects the first item when the user enters a category page.

To change the default tag, simply reorder the array so the desired tag is first.

#### Mapping When Tag Names Differ from API Values

When UI tag names differ from API parameter values, you need to map them in JS. Example: tags display as Chinese text but the API requires English parameter values:

```json
{
  "tags": ["Recent", "Best", "Most Viewed", "Most Saved"],
  "url": "https://example.com/categories/list/${pageNumber}/?sort_by=${tag}"
}
```

`LumenPluginRuntime` replaces `${tag}` with the tag name as-is → `?sort_by=Recent`. JS must convert before making the request:

```javascript
var _tagMap = {
  "Best": "",              // empty = use site default sort
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
  if (mapped === "") {
    // empty value → remove parameter (use site default sort)
    return url.replace(/([?&])sort_by=[^&]*&?/, function(match, prefix) {
      return prefix === "?" ? "?" : "";
    }).replace(/\?$/, "");
  }
  return url.replace(/sort_by=[^&]*/, "sort_by=" + mapped);
}

function buildMedias(inputURL) {
  inputURL = _resolveSortBy(inputURL);
  // ...continue with normal request
}
```

> **Recommendation**: When the site API natively accepts display names as parameter values, prefer the **no-mapping approach** — simpler config with no extra JS code needed. Only use JS mapping when API parameter values differ from display names.

## 3. JavaScript API Reference

Lumen injects `$http` and `$next` closures for communication between JS and the Swift runtime layer.
> **Note**: LumenPlugin JS does NOT include browser-native `fetch()`, `XMLHttpRequest`, etc. You must use Lumen's injected interface objects.

### 3.1 HTTP Requests: `$http.fetch(request)`

```javascript
$http.fetch({
  url: "https://api.example.com/data",
  method: "GET",
  headers: { "User-Agent": "..." },
  body: '{"foo":"bar"}' // optional, for POST
}).then(
  function(res) {
    var data = JSON.parse(res.body);
    // success callback handling
  },
  function(error) {
    // failure/error handling
  }
);
```

### 3.2 Result Callbacks: `$next`

After execution, the plugin must call the corresponding `$next` callback method to return results to the host app. The specific method depends on the business scenario:

1. **Home / Navigation / Hot Lists / HeroBanner**
   Use `$next.toMedias(jsonString)`
   Must return a JSON array of `MediaData`.

2. **Episode Lists**
   Use `$next.toEpisodes(jsonString)`
   Must return a JSON array of `EpisodeData`.

3. **Search Requests**
   Use `$next.toSearchMedias(jsonString, keyword)`
   The second parameter must pass the original search keyword.

4. **Playback Requests**
   Use `$next.toPlayer(url)`
   Supports a direct `String` or an object (with headers).

5. **Movie Metadata Requests (director/actors/synopsis)**
   Use `$next.toMetadata(jsonString)`
   Must return a `MetadataData` JSON object.

6. **Empty Data / Error Interception**
   Use `$next.emptyView(message)`
   Displays an empty state message. For non-playable scenarios (e.g. metadata-only plugins), use this hook to show an error.

## 4. Data Structure Entities

| Entity | Required Field | Type | Description |
|--------|---------------|------|-------------|
| **MediaData** | `id` | string | Unique identifier, used as query parameter for detail/episodes |
| | `coverURLString` | string | Cover image URL |
| | `title` | string | Display name |
| | `descriptionText` | string | Notes / rating / year description text |
| | `detailURLString` | string | Detail entry URL for fetching episodes |
| | `backdropURLString` | string | (Optional) Large backdrop poster image |
| | `land` | number | (Optional) Set to `1` for landscape cover (16:9), default is portrait (2:3) |
| **EpisodeData** | `id` | string | Episode identifier |
| | `title` | string | Episode name (e.g. "Episode 1") |
| | `episodeDetailURL` | string | Source URL for requesting playback info |
| **MetadataData** | `directors` | string[] | List of directors |
| | `actors` | string[] | List of actors |
| | `intro` | string | Synopsis content |

## 5. Common Development Patterns & Best Practices

### 5.1 Metadata-Only Plugins (e.g. Douban Plugin)
If the target site only provides discovery and rating features with **no** playback capability:
Set the corresponding closures to empty `{}` in `config.json`, and use `$next.emptyView` in JS to return guidance or skip:
```json
// config.json
"episodes": {},
"player": {}
```

### 5.2 API Authentication & Anti-Scraping (e.g. Olevod Plugin)
For sites with API defense mechanisms, embed signing logic (Signature) in JS code, then append to the `url` request. Example of timestamp-salted anti-hotlinking:

```javascript
// main.js - Signature example
function signature() {
  return generateMD5(Date.parse(new Date()) / 1e3); // generate hash via crypto.js
}
function buildMedias(url) {
  var secureUrl = url + "?_vv=" + signature();
  $http.fetch({ url: secureUrl, method: "GET" }).then(...)
}
```

### 5.3 Data Aggregation & Enrichment (Concurrent Request Pattern)

When executing `HeroBanner` or similar features that combine movies and TV shows (or advanced scenarios requiring multiple API calls for poster images), use a "completion counter" to wait for multiple async `$http` callbacks before submitting `$next.toMedias`, preventing async packet loss:

```javascript
// Merge movie and TV show async calls for HeroBanner
function HeroBanner() {
  var completed = 0;
  var movies = [], tvs = [];

  function tryResolve() {
    completed++;
    if (completed === 2) { // both requests finished, then render
       var merged = movies.concat(tvs);
       $next.toMedias(JSON.stringify(merged));
    }
  }

  $http.fetch({ url: "api/movie" }).then(res => { movies = parse(res); tryResolve(); }, tryResolve);
  $http.fetch({ url: "api/tv" }).then(res => { tvs = parse(res); tryResolve(); }, tryResolve);
}
```

### 5.4 Debug Toggle

Use logging during plugin development and disable it for release to reduce runtime overhead. Add a unified toggle at the top of `main.js`:

```javascript
var DEBUG = false; // set to false for release

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

Set `DEBUG = true` during development, `false` before release/minification. All debug logs go through `_perf()` and `print()`, ensuring zero log noise in production.

## 6. Error Handling & Fault Tolerance
LumenPlugin relies on async JSON parsing. Implement exception handling wherever possible to prevent JS parsing errors from crashing the process:
- **JSON parse guard**: `try { var res = JSON.parse(res.body); } catch(e) { /* return fallback */ }`.
- **Missing field defaults**: If the API response lacks some cover images or details, fill with empty strings inside `buildMediaData`. Never pass `undefined` or `null` to `$next` methods, as this will break the host Swift parser.

## 7. Performance Optimization: `lumen_for_cover` Protocol

When the Lumen App only needs **category list cover images** (not full content lists), the Swift side appends `lumen_for_cover=1` to the request URL. Plugin JS should detect this parameter and adopt the following best practices to significantly reduce unnecessary network requests.

### 7.1 Detection

```javascript
function Category(inputURL) {
  var isForCover = inputURL.indexOf("lumen_for_cover=1") >= 0;
  // choose fast path based on isForCover
}
```

### 7.2 Standard List Optimization: Reduce Limit

For standard paginated APIs, replace `limit=20` with `limit=1`:

```javascript
if (isForCover && requestURL.indexOf("limit=") >= 0) {
  requestURL = requestURL.replace(/limit=\d+/, "limit=1");
}
```

### 7.3 Heavy Data Source Optimization: Early Short-Circuit

For complex sources requiring multiple sequential API calls (e.g. annual charts), return immediately with just the first item when `isForCover`, avoiding full loading:

```javascript
if (mode === "annual") {
  if (isForCover) {
    // fetch only the first item from the fast endpoint
    fetchJSON(neuURL, referer, function(payload) {
      var medias = parseAnnualNeuMedias(payload);
      if (medias.length > 0) {
        $next.toMedias(JSON.stringify([medias[0]]));
      } else {
        // fallback to secondary endpoint
        fetchJSON(ithilURL, referer, function(p2) {
          var m2 = parseAnnualIthilMedias(p2);
          $next.toMedias(JSON.stringify(m2.length > 0 ? [m2[0]] : []));
        }, function() { $next.toMedias(JSON.stringify([])); });
      }
    }, function() { $next.toMedias(JSON.stringify([])); });
    return;
  }
  // continue with full loading...
}
```

### 7.4 Chart/Ranking Optimization: Truncate List

For ranking pages that require downloading HTML and parsing (e.g. Douban rankings), keep only the first item in `isForCover` mode, skipping subsequent detail requests:

```javascript
function fetchSimpleChartMedias(html, chartSection, isForCover, onComplete) {
  var allItems = parseSimpleChartItems(html, chartSection);
  var items = isForCover ? allItems.slice(0, 1) : allItems;
  // ...subsequent processing
}
```

## 8. JavaScriptCore Caveats

Lumen uses Apple `JavaScriptCore` rather than V8/Node. The following behavioral differences exist:

### 8.1 Block-Scoped Function Declarations (Critical)

**Do NOT** use `function` declarations inside `if`/`else`/`for` blocks. This is undefined behavior in non-strict mode, and in JavaScriptCore it causes function references to be lost silently:

```javascript
// ❌ Wrong — function may be inaccessible in callback
if (condition) {
  function myHelper() { ... }
  asyncFetch(function() { myHelper(); }); // may fail
}

// ✅ Correct — use var function expression
var myHelper = null;
if (condition) {
  myHelper = function() { ... };
  asyncFetch(function() { myHelper(); }); // works correctly
}
```

### 8.2 `const`/`let` Scoping in Nested Closures

In complex multi-level nested callbacks (e.g. multi-level fallbacks), prefer `var` declarations to ensure variables are visible across all closures.

### 8.3 Concurrent Pattern Without `Promise.all`

JavaScriptCore's injected `$http.fetch` does not use native Promises. For concurrent requests, use the **counter pattern**:

```javascript
var completed = 0;
var results = {};

function tryResolve(key, data) {
  results[key] = data;
  completed++;
  if (completed === 2) {
    $next.toMedias(JSON.stringify(results.movies.concat(results.tvs)));
  }
}

$http.fetch({ url: movieAPI }).then(
  function(r) { tryResolve("movies", parse(r)); },
  function()  { tryResolve("movies", []); }
);
$http.fetch({ url: tvAPI }).then(
  function(r) { tryResolve("tvs", parse(r)); },
  function()  { tryResolve("tvs", []); }
);
```

## 9. Image Quality Best Practices

### 9.1 Use Landscape Images for HeroBanner

Hero banners should prefer `picSlide` (landscape stills) over `pic` (portrait posters) for better visual impact:

```javascript
var backdropURL = item.picSlide || item.pic_slide || item.backdrop || "";
var coverURL    = item.pic || item.cover || "";
return buildMediaData(id, coverURL, title, desc, detailURL, backdropURL);
```

### 9.2 When heroBanner Is Not Configured

If `config.json` does not include a `heroBanner` field or `heroBanner.javascript` is empty, the App **automatically skips** the hero banner display. No low-quality portrait covers will be used as banners. No additional handling needed.

## 10. Swift-Side Integration Reference

| Scenario | Swift Behavior | Plugin JS Counterpart |
|----------|---------------|----------------------|
| Category list covers | `extend["lumen_for_cover"] = "1"` appended to URL | Detect `lumen_for_cover=1` and enter fast path |
| Home concurrent loading | `withTaskGroup` concurrently invokes all category pages | Each `Category(url)` executes independently, calling `$next.toMedias` |
| Switch site | `siteCategoryGroups` cleared, reload | Plugin re-parses all categories each time |
| Annual chart covers | Same as above, appends `lumen_for_cover=1` | NEU endpoint first, fallback to ithil on failure |
| Tag switch | `parameters["tag"]` replaces `${tag}` in URL | JS must map tag names to API values (if different) |
| Landscape covers | Uses landscape layout when `land` field ≥ 1 | `buildMediaData` returns `land: 1` |
