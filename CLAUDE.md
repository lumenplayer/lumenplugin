# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a plugin repository for the Lumen Player application. Plugins are written in JavaScript and use JSON configuration to define video source integrations for macOS, iOS, and tvOS platforms.

## Required Reading

Before developing or modifying any plugin, you **MUST** read the full skill documentation:

```
skills/lumenplugin/SKILL.md
```

This file contains the complete plugin development specification including:
- `config.json` field definitions and page routing
- JavaScript API reference (`$http.fetch`, `$next.toMedias/toEpisodes/toPlayer`, etc.)
- Data structures (MediaData, EpisodeData, MetadataData)
- JavaScriptCore-specific caveats and pitfalls
- Performance optimization (`lumen_for_cover` protocol)
- Error handling patterns

## Plugin Structure

Each plugin is a standalone directory containing:

```
<plugin-name>/
├── config.json     # Plugin configuration (metadata, pages, API endpoints, JS function mapping)
├── main.js         # Main JavaScript logic
└── *.js            # Optional helper files (e.g. crypto-js.min.js)
```

## Key Constraints

- **Runtime**: Apple JavaScriptCore (NOT V8/Node)
- **No browser APIs**: `fetch()`, `XMLHttpRequest`, `DOM` are unavailable
- **Must use**: Lumen-injected `$http.fetch()` for HTTP and `$next.*` for result callbacks
- **No `Promise.all`**: Use counter pattern for concurrent requests
- **No block-scoped `function`**: Use `var` function expressions instead
- **Never pass `undefined`/`null`** to any `$next` method — use empty strings as defaults
- **Set `DEBUG = false`** before release
