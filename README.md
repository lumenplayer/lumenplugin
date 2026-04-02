# LumenPlugin

**繁體中文** | [English](README-en.md) | [简体中文](README-cn.md)

為 [Lumen Player](https://lumenplayer.app) 開發視訊源插件的官方倉庫。

LumenPlugin 是一套基於 **JavaScript + JSON 配置** 的輕量級插件系統，讓開發者能夠快速將任意影視網站/API 接入 Lumen Player，支援 macOS、iOS 和 tvOS 平台。

## ✨ 特性

- **純 JavaScript 開發** — 無需編譯，無需 SDK，修改即生效
- **JSON 宣告式配置** — 透過 `config.json` 定義頁面路由、搜尋、播放等功能
- **AI 輔助開發** — 內建 AI Agent 技能包，藉助 Claude/Cursor 等工具快速生成插件
- **內建範例插件** — 提供多種類型的參考實作，開箱即用

## 📁 目錄結構

```
plugin/
├── README.md                 # 本檔案
├── development-guide.md      # 完整開發文檔
├── skills/                   # AI Agent 技能包
│   └── lumenplugin/          # 插件開發技能
└── xxx/                      # 範例：純中繼資料插件
```

## 🚀 快速開始

### 方式一：AI 輔助開發（推薦）

本倉庫內建了專為 AI Agent 設計的 **技能包**（`skills/` 目錄），可配合 [Claude Code](https://claude.ai/code)、[Cursor](https://cursor.com) 等 AI 程式設計工具使用，大幅降低開發門檻。

#### 準備工作

1. 複製本倉庫
2. 在 AI 程式設計工具中開啟本倉庫目錄

#### 使用方式

AI Agent 會自動讀取 `skills/lumenplugin/` 中的開發規範，你只需用自然語言描述需求即可：

**範例對話：**

```
你：幫我為 example.com 建立一個插件，它的 API 介面是 https://api.example.com/v1/movies

AI：好的，我來分析這個 API 並為你建立插件...
    [自動生成 config.json 和 main.js]
```

```
你：這個插件需要支援搜尋功能，搜尋介面是 POST https://api.example.com/search

AI：我來新增搜尋功能...
    [更新 config.json 和 main.js]
```

#### 技能包包含

| 技能 | 路徑 | 用途 |
|------|------|------|
| `lumenplugin` | `skills/lumenplugin/SKILL.md` | 插件開發核心規範：配置格式、JS 介面、資料結構、最佳實踐 |

> **提示**：AI Agent 在開發插件時會自動參考技能包中的規範，確保生成的程式碼符合 Lumen Player 的要求。你也可以直接閱讀技能包文檔來瞭解完整的開發規範。

### 方式二：手動開發

如果你更喜歡手動撰寫程式碼，請參閱 **[完整開發文檔](development-guide.md)**，文檔涵蓋：

- 插件結構與配置檔案規範
- JavaScript 介面（`$http`、`$next`）詳細說明
- 資料結構定義（MediaData、EpisodeData 等）
- 開發模式與最佳實踐
- JavaScriptCore 注意事項
- 效能優化指南

#### 快速建立插件

1. 建立插件目錄：
   ```bash
   mkdir my-plugin/
   ```

2. 編輯 `my-plugin/config.json`，修改基本資訊和頁面配置

3. 撰寫 `my-plugin/main.js`，實作資料取得和解析邏輯

4. 在 Lumen Player 中載入測試

## 🔧 插件結構概覽

每個插件目錄至少包含以下檔案：

```
my-plugin/
├── config.json          # 插件配置（必需）
├── main.js              # 業務邏輯（必需）
├── main.min.js          # 壓縮版（發佈用）
└── crypto-js.min.js     # 第三方庫（按需）
```

### config.json 核心結構

```json
{
  "name": "插件名稱",
  "description": "插件描述",
  "host": "https://target-site.com/",
  "files": ["main.js"],
  "pages": [...],
  "episodes": { "javascript": "getEpisodes" },
  "player": { "javascript": "Player" },
  "search": { "url": "...", "javascript": "Search" }
}
```

### 核心 API

| API | 用途 |
|-----|------|
| `$http.fetch(request)` | 發起 HTTP 請求 |
| `$next.toMedias(json)` | 回傳媒體列表 |
| `$next.toEpisodes(json)` | 回傳劇集列表 |
| `$next.toSearchMedias(json, keyword)` | 回傳搜尋結果 |
| `$next.toPlayer(url)` | 回傳播放位址 |
| `$next.toMetadata(json)` | 回傳影片中繼資訊 |
| `$next.emptyView(message)` | 顯示空狀態提示 |

## 📖 文檔

- **[完整開發文檔](development-guide.md)** — 詳細的介面規範、配置說明和最佳實踐
- **[AI 技能包](skills/lumenplugin/SKILL.md)** — AI Agent 使用的開發規範

## 🤝 貢獻

歡迎為 Lumen Player 開發新的插件！

## 📄 授權條款

MIT License
