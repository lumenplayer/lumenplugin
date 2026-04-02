# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## Project Overview

This is a plugin repository for the Lumen Player application. Plugins are written in JavaScript and use JSON configuration to define video source integrations for macOS, iOS, and tvOS platforms.

## Skills

When developing or modifying plugins, you **MUST** read and follow the skill documentation before writing any code:

- **[LumenPlugin Skill](skills/lumenplugin/SKILL.md)** — Complete plugin development specifications including config format, JS APIs (`$http`, `$next`), data structures, JavaScriptCore caveats, and performance optimization patterns.

## Plugin Structure

Each plugin is a standalone directory containing:

```
<plugin-name>/
├── config.json     # Plugin configuration (metadata, pages, API endpoints, JS function mapping)
├── main.js         # Main JavaScript logic
└── *.js            # Optional helper files (e.g. crypto-js.min.js)
```

## Development Workflow

1. Read `skills/lumenplugin/SKILL.md` before starting any plugin work
2. Plugins require no compilation — they are loaded directly in the player app
3. Use `console.log()` and the `print()` debug helper for debugging
4. Always implement error handling (JSON parse guards, missing field defaults)
5. Never pass `undefined` or `null` to `$next` callback methods
6. Set `DEBUG = false` before release

## Key Constraints

- **Runtime**: Apple JavaScriptCore (NOT V8/Node)
- **No browser APIs**: `fetch()`, `XMLHttpRequest`, `DOM` are unavailable
- **Must use**: Lumen-injected `$http.fetch()` for HTTP and `$next.*` for result callbacks
- **No `Promise.all`**: Use counter pattern for concurrent requests
- **No block-scoped `function`**: Use `var` function expressions instead
