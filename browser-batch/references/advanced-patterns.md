# Advanced Batch Patterns & Site-Specific Selectors

## Edge Cases & Error Handling

### Retry on Failure

When a batch returns `ok: false` for a step, diagnose via snapshot and construct a new batch:

```
1. Run batch → step 3 fails with "Selector not found within 5000ms"
2. Take snapshot → discover the DOM changed (SPA routing, modal overlay, etc.)
3. Identify correct selector from snapshot
4. Construct new batch from the failed step onward with corrected selectors
```

Do NOT re-run the entire batch from step 1 — earlier steps (navigate, type) already succeeded.

### Dynamic Pages (React, Vue, SPA)

For SPAs where DOM mutates after interactions:

1. Use batch for the navigation + initial wait phase.
2. Take a snapshot after the batch to discover the new DOM.
3. Use a second batch for actions on the new DOM.

This two-batch pattern is the standard for SPAs:

```
Batch 1: navigate + wait for page shell
Snapshot: learn the dynamic DOM
Batch 2: interact with discovered elements (using refs or selectors)
```

### contenteditable Fields (Draft.js, Slate, ProseMirror)

Some rich text editors don't respond to `type`. Use `evaluate` with `document.execCommand`:

```json
{
  "kind": "evaluate",
  "fn": "const el = document.querySelector('[contenteditable]'); el.focus(); document.execCommand('insertText', false, 'Your text here');"
}
```

`evaluate` cannot use `selector` but CAN be placed inside a batch alongside other actions.

### Selectors That Don't Work

If a CSS selector consistently fails (ok: false), common causes:

- Element is inside an iframe → batch cannot cross iframe boundaries; use `evaluate` with `contentDocument` or switch frames first
- Element is inside Shadow DOM → use `evaluate` to pierce shadow roots
- Element hasn't rendered yet → add a `wait` action with that selector before the interaction
- Element is off-screen → add `scrollIntoView` before clicking

### Anti-Bot / Rate Limiting

For sites that detect automation:

- Add `delayMs: 200-500` to click actions
- Space out batches (don't chain 5 batches instantly)
- Use `existing-session` (profile="user") to inherit real cookies/auth

## Site-Specific Selector References

### GitHub

```
Repository releases list:     .release-entry, [data-testid="release-entry"]
Release title link:            .release-entry:first-child a
Markdown content:              .markdown-body
Issue/PR comment box:          #new_comment_field
Submit comment button:         button[type="submit"].btn-primary
Tab navigation:                a.UnderlineNav-item
File content:                  .blob-code-inner
```

### X (Twitter) — as of March 2026

```
Compose button:                a[href="/compose/post"]
Tweet textarea:                [data-testid="tweetTextarea_0"]
Send tweet button:             [data-testid="tweetButtonInline"]
Toast notification:            [data-testid="toast"]
Profile menu:                  [data-testid="AppTabBar_Profile_Link"]
```

NOTE: X uses Draft.js for the compose box. Standard `type` will NOT work. Use `evaluate` with `document.execCommand('insertText', false, 'text')` after focusing the textarea.

### Google Services (with existing-session)

Gmail:
```
Compose button:                [gh="cm"], div[role="button"][data-tooltip="Compose"]
To field:                      [name="to"], input[aria-label="To"]
Subject field:                 [name="subjectbox"], input[aria-label="Subject"]
Send button:                   [data-tooltip="Send"], div[role="button"][aria-label*="Send"]
```

Google Search:
```
Search input:                  input[name="q"], textarea[name="q"]
Search button:                 input[name="btnK"]
```

### Stripe Dashboard (with existing-session)

```
Sidebar navigation:            nav a[href*="/payments"], nav a[href*="/invoices"]
Data table rows:               tr[data-testid]
Status badge:                  span[class*="Badge"]
Action button (per row):       td button, [data-testid="overflow-menu-button"]
Filter dropdown:               button[aria-label*="Filter"]
```

### Notion (with existing-session)

```
New page button:               [role="button"][aria-label="New page"]
Page title:                    [placeholder="Untitled"]
Block content:                 [data-block-id] [contenteditable]
Sidebar toggle:                button[aria-label*="sidebar"]
```

NOTE: Notion uses a custom block editor. Content input requires `evaluate` with focus + insertText, similar to X/Twitter.

## Combining with evaluate

`evaluate` is the escape hatch for anything selectors can't handle. Use it inside batches for:

- Reading page data: `evaluate` returns the function's result
- Setting form values that `type` can't handle (React controlled inputs)
- Extracting structured data after navigation

Example — extract all links from a page after navigating:

```json
{
  "kind": "batch",
  "actions": [
    { "kind": "navigate", "url": "https://example.com/links" },
    { "kind": "wait", "selector": "main" }
  ],
  "stopOnError": true
}
```

Then:

```json
{
  "kind": "evaluate",
  "fn": "JSON.stringify(Array.from(document.querySelectorAll('main a')).map(a => ({text: a.textContent.trim(), href: a.href})))"
}
```

## Performance Comparison

| Workflow | Without Batch | With Batch | Savings |
|---|---|---|---|
| Fill 4-field form + submit | 9 calls (snapshot + type × 4 + snapshot + click + snapshot + verify) | 2 calls (batch + verify snapshot) | 78% |
| Navigate + wait + click + wait | 5 calls | 2 calls (batch + verify snapshot) | 60% |
| 3-page navigation chain | 9 calls | 4 calls (3 batches + final snapshot) | 56% |

## Anti-Patterns to Avoid

1. **Batching everything blindly**: Do NOT batch when you need to inspect a result mid-sequence. If step 2 depends on step 1's output, split into separate batches with a snapshot in between.

2. **Huge batches**: Keep batches to 3-8 actions. Massive batches are hard to debug when a step fails mid-way.

3. **Mixing known and unknown pages**: If navigating to a new unknown page, end the batch at the `wait` step. Take a snapshot. Then start a new batch with what you learned.

4. **Forgetting verification**: ALWAYS take a snapshot after the final batch to confirm success. Batch results only tell you actions didn't error — they don't tell you the page is in the right state.
