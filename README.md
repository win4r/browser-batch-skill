# [🚀观看我的哔哩哔哩视频](https://space.bilibili.com/3493277319825652)

# browser-batch-skill

An OpenClaw skill that teaches your agent to use **batch actions** and **CSS selector targeting** for dramatically faster browser automation.

## What This Solves

OpenClaw v2026.3.13 added batch actions and CSS selectors to the browser `/act` API ([PR #45457](https://github.com/openclaw/openclaw/pull/45457), [PR #45506](https://github.com/openclaw/openclaw/pull/45506)). But your agent doesn't know these features exist — its training data predates them, and no built-in skill teaches the syntax.

This skill injects that knowledge. After installing, your agent will automatically combine sequential browser actions into batches and use CSS selectors to skip unnecessary snapshots.

## Before vs After

| | Without this skill | With this skill |
|---|---|---|
| Fill 4-field form + submit | ~9 tool calls (snapshot → type → snapshot → type → ...) | **2 tool calls** (1 batch + 1 verify snapshot) |
| Navigate + wait + click + wait | ~5 tool calls | **2 tool calls** |
| 3-page navigation chain | ~9 tool calls | **4 tool calls** |

## Requirements

- **OpenClaw >= v2026.3.13** (batch actions were added in this release)
- Browser tool enabled (`browser.enabled: true` in config)

## Installation

### Option A: Direct install

```bash
cp -r browser-batch ~/.openclaw/workspace/skills/browser-batch
openclaw gateway restart
```

### Option B: From GitHub

```bash
git clone https://github.com/win4r/browser-batch-skill.git
cp -r browser-batch-skill/browser-batch ~/.openclaw/workspace/skills/
openclaw gateway restart
```

### Verify

```bash
openclaw skills info browser-batch
```

## Files

```
browser-batch/
├── SKILL.md                           # Main skill (trigger rules + core syntax)
└── references/
    └── advanced-patterns.md           # Edge cases, site-specific selectors, anti-patterns
```

## Usage

After installation, your agent automatically uses batch actions when appropriate. You can also explicitly request it:

```
帮我用 batch action 打开 GitHub releases 页面并点击最新版本
```

```
Fill out the signup form on example.com — use batch to combine all the steps
```

```
优化这个浏览器操作流程，减少 roundtrip
```

The skill triggers on keywords like "batch", "optimize", "speed up", "combine", "CSS selector", "reduce roundtrips", as well as general browser automation tasks like form filling and multi-step navigation.

## How It Works

The skill teaches the agent three capabilities:

1. **Batch Actions** — Combine multiple browser actions (navigate, click, type, wait, etc.) into a single tool call using `kind: "batch"`
2. **CSS Selectors** — Target elements directly with `selector: "button[type='submit']"` instead of requiring a snapshot to get ref numbers first
3. **DelayMs** — Add human-like hover-before-click timing with `delayMs: 300`

These features exist in the Gateway's `/act` HTTP endpoint. The CLI (`openclaw browser click`) still only supports refs — this is by design.

## Key Design Decisions

- **Description-as-trigger**: The YAML `description` field contains all trigger phrases because OpenClaw only reads `name` + `description` to decide whether to load a skill
- **Progressive disclosure**: Core syntax in SKILL.md (~180 lines), site-specific selectors and edge cases in `references/advanced-patterns.md` — loaded on demand to save context window
- **Decision workflow**: SKILL.md includes a step-by-step decision tree so the agent knows *when* to batch vs when to snapshot first

## Related

- [OpenClaw Browser Docs](https://docs.openclaw.ai/tools/browser)
- [Issue #44431](https://github.com/openclaw/openclaw/issues/44431) — Original feature request
- [PR #45457](https://github.com/openclaw/openclaw/pull/45457) — Batch + selector implementation
- [PR #45506](https://github.com/openclaw/openclaw/pull/45506) — Batch failure handling fix
- [Release v2026.3.13](https://github.com/openclaw/openclaw/releases/tag/v2026.3.13-1)

## License

MIT
