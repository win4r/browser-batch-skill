---
name: browser-batch
description: "Optimize browser automation with batch actions and CSS selectors. Use this skill whenever performing 2+ sequential browser actions (navigate, click, type, wait, etc.), when the user asks to 'batch', 'speed up', 'optimize', or 'combine' browser steps, when filling forms, when performing multi-step web workflows, or when the user mentions 'batch actions', 'CSS selector', or 'reduce roundtrips'. Requires OpenClaw >= v2026.3.13. This skill teaches the agent to use the /act endpoint's batch kind and CSS selector targeting to dramatically reduce tool call count."
---

# Browser Batch Actions & CSS Selectors

Since OpenClaw v2026.3.13, the browser `/act` endpoint supports **batch actions** and **CSS selector targeting**. These features reduce browser tool calls by 3-5x.

**IMPORTANT**: These features work through the browser tool's `/act` endpoint (agent tool calls), NOT through CLI commands. The CLI (`openclaw browser click`) still only supports `ref` from snapshots.

## Core Rule: When to Batch

**ALWAYS batch when you have 2+ consecutive browser actions.** Instead of separate tool calls, combine them into one `batch` action.

## Batch Action Syntax

Use `kind: "batch"` with an `actions` array. Any valid browser action kind can go inside a batch:

```json
{
  "kind": "batch",
  "actions": [
    { "kind": "navigate", "url": "https://example.com" },
    { "kind": "wait", "selector": ".main-content" },
    { "kind": "click", "selector": "a.target-link" },
    { "kind": "wait", "selector": "article" }
  ],
  "stopOnError": true
}
```

**Always set `stopOnError: true`** for critical sequences so execution halts on first failure.

### Batch Response Format

```json
{
  "results": [
    { "ok": true },
    { "ok": true },
    { "ok": false, "error": "Selector not found within 5000ms" }
  ]
}
```

Each action returns `ok: true` or `ok: false` with an error message. When `stopOnError: true`, the results array stops at the first failure.

## CSS Selector Targeting

For element interaction actions, use `selector` (CSS) instead of `ref` (from snapshot) when the DOM structure is known:

```json
{ "kind": "click", "selector": "button[type='submit']" }
{ "kind": "type", "selector": "input[name='email']", "text": "user@example.com" }
{ "kind": "hover", "selector": ".dropdown-trigger" }
{ "kind": "select", "selector": "select#country", "values": ["US"] }
```

### Which actions support `selector`

Only these 6 element interaction actions accept the `selector` parameter: **click, type, hover, drag, scrollIntoView, select**.

Other action kinds (navigate, wait, evaluate, snapshot) do NOT use `selector` but CAN appear inside a batch.

### When to use `selector` vs `ref`

- **Use `selector`** when you know the page structure (stable pages, known apps, forms with named inputs).
- **Use `ref`** when exploring unknown pages — take a snapshot first, then act on refs.
- **After any batch**, take ONE snapshot to verify the final state.

## DelayMs for Human-Like Clicks

Add `delayMs` to click actions for hover-before-click behavior:

```json
{ "kind": "click", "selector": "button.submit", "delayMs": 300 }
```

This hovers the element, waits the specified milliseconds, then clicks. Replaces the old three-step hover → wait → click pattern.

## Constraints

- Maximum 100 actions per batch (recursive count, including nested batches).
- Maximum 5 levels of nesting.
- Selector timeout: 5000ms by default — if an element isn't found, returns `ok: false`.

## Decision Workflow

Follow this workflow for every browser automation task:

1. **Is the page structure known?** (previously visited, stable DOM, named form fields)
   - YES → Use batch + selector. Skip snapshot.
   - NO → Take ONE snapshot first to learn the DOM. Then batch subsequent actions using refs or selectors.

2. **Are there 2+ consecutive actions?**
   - YES → Combine into a single batch.
   - NO → Single action is fine.

3. **After batch completes:**
   - Take ONE snapshot to verify the final state.
   - If `ok: false` in results, diagnose via snapshot and retry.

## Common Patterns

### Pattern A: Form Fill + Submit (known structure)

```json
{
  "kind": "batch",
  "actions": [
    { "kind": "type", "selector": "input[name='username']", "text": "jeffrey" },
    { "kind": "type", "selector": "input[name='email']", "text": "jeff@example.com" },
    { "kind": "type", "selector": "input[name='company']", "text": "CortexReach" },
    { "kind": "click", "selector": "button[type='submit']", "delayMs": 200 }
  ],
  "stopOnError": true
}
```

### Pattern B: Navigate + Wait + Act (multi-step)

```json
{
  "kind": "batch",
  "actions": [
    { "kind": "navigate", "url": "https://github.com/user/repo/releases" },
    { "kind": "wait", "selector": "[data-testid='release-entry']" },
    { "kind": "click", "selector": "[data-testid='release-entry']:first-child a" },
    { "kind": "wait", "selector": ".markdown-body" }
  ],
  "stopOnError": true
}
```

Then take ONE snapshot to read the content.

### Pattern C: Paginated Navigation

```json
{
  "kind": "batch",
  "actions": [
    { "kind": "navigate", "url": "https://example.com/blog" },
    { "kind": "wait", "selector": "nav a[href*='page']" },
    { "kind": "click", "selector": "nav a:last-child" },
    { "kind": "wait", "selector": "article" }
  ],
  "stopOnError": true
}
```

### Pattern D: Nested Batch (navigation phase + action phase)

```json
{
  "kind": "batch",
  "actions": [
    { "kind": "navigate", "url": "https://mail.google.com" },
    { "kind": "wait", "selector": "[role='main']" },
    {
      "kind": "batch",
      "actions": [
        { "kind": "click", "selector": "[gh='cm']" },
        { "kind": "type", "selector": "[name='to']", "text": "team@example.com" },
        { "kind": "type", "selector": "[name='subjectbox']", "text": "Daily Report" },
        { "kind": "click", "selector": "[data-tooltip='Send']", "delayMs": 300 }
      ],
      "stopOnError": true
    }
  ],
  "stopOnError": true
}
```

Outer batch handles navigation. Inner batch handles the operation. Outer failure skips the inner batch entirely.

For more patterns and site-specific selector references, read `{baseDir}/references/advanced-patterns.md`.
