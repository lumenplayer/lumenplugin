# LumenPlugin

**简体中文** | [English](README-en.md) | [繁體中文](README.md)

为 [Lumen Player](https://lumenplayer.app) 开发视频源插件的官方仓库。

LumenPlugin 是一套基于 **JavaScript + JSON 配置** 的轻量级插件系统，让开发者能够快速将任意影视网站/API 接入 Lumen Player，支持 macOS、iOS 和 tvOS 平台。

## ✨ 特性

- **纯 JavaScript 开发** — 无需编译，无需 SDK，修改即生效
- **JSON 声明式配置** — 通过 `config.json` 定义页面路由、搜索、播放等功能
- **AI 辅助开发** — 内置 AI Agent 技能包，借助 Claude/Cursor 等工具快速生成插件
- **内置示例插件** — 提供多种类型的参考实现，开箱即用

## 📁 目录结构

```
plugin/
├── README.md                 # 本文件
├── development-guide-cn.md   # 完整开发文档
├── skills/                   # AI Agent 技能包
│   └── lumenplugin/          # 插件开发技能
└── xxx/                      # 示例：纯元数据插件
```

## 🚀 快速开始

### 方式一：AI 辅助开发（推荐）

本仓库内置了专为 AI Agent 设计的 **技能包**（`skills/` 目录），可配合 [Claude Code](https://claude.ai/code)、[Cursor](https://cursor.com) 等 AI 编程工具使用，大幅降低开发门槛。

#### 准备工作

1. 克隆本仓库
2. 在 AI 编程工具中打开本仓库目录

#### 使用方式

AI Agent 会自动读取 `skills/lumenplugin/` 中的开发规范，你只需用自然语言描述需求即可：

**示例对话：**

```
你：帮我为 example.com 创建一个插件，它的 API 接口是 https://api.example.com/v1/movies

AI：好的，我来分析这个 API 并为你创建插件...
    [自动生成 config.json 和 main.js]
```

```
你：这个插件需要支持搜索功能，搜索接口是 POST https://api.example.com/search

AI：我来添加搜索功能...
    [更新 config.json 和 main.js]
```

#### 技能包包含

| 技能 | 路径 | 用途 |
|------|------|------|
| `lumenplugin` | `skills/lumenplugin/SKILL.md` | 插件开发核心规范：配置格式、JS 接口、数据结构、最佳实践 |

> **提示**：AI Agent 在开发插件时会自动参考技能包中的规范，确保生成的代码符合 Lumen Player 的要求。你也可以直接阅读技能包文档来了解完整的开发规范。

### 方式二：手动开发

如果你更喜欢手动编写代码，请参阅 **[完整开发文档](development-guide-cn.md)**，文档涵盖：

- 插件结构与配置文件规范
- JavaScript 接口（`$http`、`$next`）详细说明
- 数据结构定义（MediaData、EpisodeData 等）
- 开发模式与最佳实践
- JavaScriptCore 注意事项
- 性能优化指南

#### 快速创建插件

1. 创建插件目录：
   ```bash
   mkdir my-plugin/
   ```

2. 编辑 `my-plugin/config.json`，修改基本信息和页面配置

3. 编写 `my-plugin/main.js`，实现数据获取和解析逻辑

4. 在 Lumen Player 中加载测试

## 🔧 插件结构概览

每个插件目录至少包含以下文件：

```
my-plugin/
├── config.json          # 插件配置（必需）
├── main.js              # 业务逻辑（必需）
├── main.min.js          # 压缩版（发布用）
└── crypto-js.min.js     # 第三方库（按需）
```

### config.json 核心结构

```json
{
  "name": "插件名称",
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
| `$http.fetch(request)` | 发起 HTTP 请求 |
| `$next.toMedias(json)` | 返回媒体列表 |
| `$next.toEpisodes(json)` | 返回剧集列表 |
| `$next.toSearchMedias(json, keyword)` | 返回搜索结果 |
| `$next.toPlayer(url)` | 返回播放地址 |
| `$next.toMetadata(json)` | 返回影片元信息 |
| `$next.emptyView(message)` | 显示空状态提示 |

## 📖 文档

- **[完整开发文档](development-guide-cn.md)** — 详细的接口规范、配置说明和最佳实践
- **[AI 技能包](skills/lumenplugin/SKILL.md)** — AI Agent 使用的开发规范

## 🤝 贡献

欢迎为 Lumen Player 开发新的插件！

## 📄 许可证

MIT License
