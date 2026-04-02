# LumenPlugin

**English** | [繁體中文](README.md) | [简体中文](README-cn.md)

The official repository for developing video source plugins for [Lumen Player](https://lumenplayer.app).

LumenPlugin is a lightweight plugin system based on **JavaScript + JSON configuration**, enabling developers to quickly integrate any video website or API into Lumen Player across macOS, iOS, and tvOS platforms.

## ✨ Features

- **Pure JavaScript** — No compilation, no SDK, changes take effect immediately
- **Declarative JSON Config** — Define page routes, search, and playback via `config.json`
- **AI-Assisted Development** — Built-in AI Agent skill packs for rapid plugin generation with Claude/Cursor
- **Built-in Examples** — Reference implementations for various plugin types, ready to use

## 📁 Directory Structure

```
plugin/
├── README.md                    # This file
├── development-guide-en.md      # Full development guide
├── skills/                      # AI Agent skill packs
│   └── lumenplugin/             # Plugin development skill
└── xxx/                         # Example: metadata-only plugin
```

## 🚀 Getting Started

### Option 1: AI-Assisted Development (Recommended)

This repository includes **skill packs** (`skills/` directory) designed for AI Agents, compatible with [Claude Code](https://claude.ai/code), [Cursor](https://cursor.com), and other AI coding tools — significantly lowering the development barrier.

#### Prerequisites

1. Clone this repository
2. Open the repository directory in your AI coding tool

#### Usage

The AI Agent will automatically read the development specs from `skills/lumenplugin/`. Simply describe your requirements in natural language:

**Example conversations:**

```
You: Create a plugin for example.com, the API endpoint is https://api.example.com/v1/movies

AI: Sure, let me analyze this API and create the plugin for you...
    [Automatically generates config.json and main.js]
```

```
You: This plugin needs search support, the search endpoint is POST https://api.example.com/search

AI: I'll add search functionality...
    [Updates config.json and main.js]
```

#### Included Skill Packs

| Skill | Path | Purpose |
|-------|------|---------|
| `lumenplugin` | `skills/lumenplugin/SKILL.md` | Core plugin development specs: config format, JS APIs, data structures, best practices |

> **Tip**: The AI Agent automatically references the skill pack specs when developing plugins, ensuring generated code meets Lumen Player's requirements. You can also read the skill pack documentation directly to learn the full development specs.

### Option 2: Manual Development

If you prefer writing code manually, refer to the **[Full Development Guide](development-guide-en.md)**, which covers:

- Plugin structure and configuration file specs
- JavaScript APIs (`$http`, `$next`) in detail
- Data structure definitions (MediaData, EpisodeData, etc.)
- Development patterns and best practices
- JavaScriptCore caveats
- Performance optimization guide

#### Quick Start

1. Create a plugin directory:
   ```bash
   mkdir my-plugin/
   ```

2. Edit `my-plugin/config.json` with your plugin metadata and page configuration

3. Write `my-plugin/main.js` to implement data fetching and parsing logic

4. Load and test in Lumen Player

## 🔧 Plugin Structure Overview

Each plugin directory contains at least the following files:

```
my-plugin/
├── config.json          # Plugin configuration (required)
├── main.js              # Business logic (required)
├── main.min.js          # Minified version (for release)
└── crypto-js.min.js     # Third-party library (as needed)
```

### config.json Core Structure

```json
{
  "name": "Plugin Name",
  "description": "Plugin description",
  "host": "https://target-site.com/",
  "files": ["main.js"],
  "pages": [...],
  "episodes": { "javascript": "getEpisodes" },
  "player": { "javascript": "Player" },
  "search": { "url": "...", "javascript": "Search" }
}
```

### Core APIs

| API | Purpose |
|-----|---------|
| `$http.fetch(request)` | Make HTTP requests |
| `$next.toMedias(json)` | Return media list |
| `$next.toEpisodes(json)` | Return episode list |
| `$next.toSearchMedias(json, keyword)` | Return search results |
| `$next.toPlayer(url)` | Return playback URL |
| `$next.toMetadata(json)` | Return movie metadata |
| `$next.emptyView(message)` | Display empty state message |

## 📖 Documentation

- **[Full Development Guide](development-guide-en.md)** — Detailed API specs, configuration reference, and best practices
- **[AI Skill Pack](skills/lumenplugin/SKILL.md)** — Development specs used by AI Agents

## 🤝 Contributing

We welcome new plugins for Lumen Player!

## 📄 License

MIT License
