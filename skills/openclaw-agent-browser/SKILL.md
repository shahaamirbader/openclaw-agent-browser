---
name: openclaw-agent-browser
description: >
  Enhanced headless browser automation CLI for AI agents, built for Openclaw.
  Use when the user needs to interact with websites: navigating pages, filling
  forms, clicking buttons, taking screenshots (with optional base64 output for
  chat UIs), extracting form state, inspecting WebSocket traffic, or getting a
  full page snapshot in one call. Triggers include: "open a website", "fill out
  a form", "click a button", "take a screenshot", "scrape data", "test this web
  app", "login to a site", "automate browser actions", "show me what the page
  looks like", "what's on this page", "get page state", "inspect WebSocket
  messages", or any task requiring programmatic web interaction.
version: 1.0.0
metadata:
  emoji: 🌐
  homepage: https://github.com/shahaamirbader/openclaw-agent-browser
  install:
    node: openclaw-agent-browser
  requires:
    bins: [ocbrowser]
allowed-tools: Bash(npx openclaw-agent-browser:*), Bash(ocbrowser:*)
---

# Browser Automation with openclaw-agent-browser

A fork of [vercel-labs/agent-browser](https://github.com/vercel-labs/agent-browser)
with added features for Openclaw AI agent workflows. Install via:

```bash
npm install -g openclaw-agent-browser
ocbrowser install   # downloads Chrome for Testing (first time only)
```

Full source: https://github.com/shahaamirbader/openclaw-agent-browser

---

## Core Workflow

Every browser session follows this pattern:

1. **Navigate** — `ocbrowser open <url>`
2. **Snapshot** — `ocbrowser snapshot -i` (get element refs like `@e1`, `@e2`)
3. **Interact** — use refs to click, fill, select
4. **Re-snapshot** — after navigation or DOM changes, get fresh refs

```bash
ocbrowser open https://example.com/login
ocbrowser snapshot -i
# @e1 [input type="email"], @e2 [input type="password"], @e3 [button] "Sign In"

ocbrowser fill @e1 "user@example.com"
ocbrowser fill @e2 "secret"
ocbrowser click @e3
ocbrowser wait --load networkidle
ocbrowser snapshot -i
```

---

## New Features (Openclaw additions)

### 1. Screenshot base64 passthrough

Return the screenshot as a base64 string in the JSON response so it can be
displayed inline in a chat UI without filesystem access.

```bash
# CLI flag
ocbrowser screenshot --return-base64

# JSON daemon API
echo '{"action":"screenshot","returnBase64":true}' | ocbrowser batch --json
```

Response includes `{ "path": "...", "base64": "iVBORw0KGgo..." }`.

---

### 2. Cursor position overlay

After a `click` or `hover`, take a screenshot with `--cursor` to draw a red
crosshair at the exact pixel the action landed. Use this to visually confirm
the AI interacted with the right element.

```bash
ocbrowser click @e3
ocbrowser screenshot --cursor --return-base64
```

JSON API: `{ "action": "screenshot", "cursor": true, "returnBase64": true }`

---

### 3. Page state — full context in one call

Returns the current URL, page title, accessibility snapshot, and a base64
screenshot all in a single round trip. Avoids 3–4 separate commands when an
AI agent needs the full picture of where it is.

```bash
# JSON daemon API
echo '{"action":"page-state"}' | ocbrowser batch --json
```

Response:
```json
{
  "url": "https://example.com/dashboard",
  "title": "Dashboard — Example",
  "snapshot": "- main [landmark] ...",
  "screenshot": "iVBORw0KGgo..."
}
```

Optional params: `interactive` (bool), `compact` (bool).

---

### 4. Form state extraction

Dumps all `input`, `select`, and `textarea` fields on the page — or inside a
specific form — as structured JSON. Useful before filling a form to inspect
field names and current values, or after filling to verify accuracy.

```bash
# All fields on the page
echo '{"action":"form-state"}' | ocbrowser batch --json

# Scoped to a specific form element
echo '{"action":"form-state","selector":"form#checkout"}' | ocbrowser batch --json
```

Response:
```json
{
  "fields": [
    { "tag": "input", "type": "email", "name": "email", "id": "email",
      "value": "", "checked": null, "disabled": false, "required": true,
      "placeholder": "you@example.com" },
    { "tag": "select", "type": "select-one", "name": "country", "id": "country",
      "value": "US", "checked": null, "disabled": false, "required": false,
      "placeholder": null }
  ]
}
```

---

### 5. WebSocket message inspection

Captures WebSocket frames sent and received by the page in a ring buffer (max
500). Essential for real-time apps (chat, live dashboards, trading UIs) where
HTTP request tracking is blind to live data.

```bash
# Enable capture and list recent messages
echo '{"action":"websocket-messages"}' | ocbrowser batch --json

# Filter by direction, limit results
echo '{"action":"websocket-messages","direction":"received","limit":20}' | ocbrowser batch --json

# Clear the buffer
echo '{"action":"websocket-clear"}' | ocbrowser batch --json
```

Response:
```json
{
  "messages": [
    {
      "url": "wss://api.example.com/live",
      "direction": "received",
      "payload": "{\"type\":\"price_update\",\"value\":142.50}",
      "timestamp": 1748123456789,
      "opcode": 1
    }
  ],
  "total": 47
}
```

WebSocket capture is lazy — it enables on the first `websocket-messages` call
and runs until the browser is closed.

---

## Standard Commands Reference

### Navigation
```bash
ocbrowser open <url>          # Navigate (aliases: goto, navigate)
ocbrowser back                # Browser back
ocbrowser forward             # Browser forward
ocbrowser reload              # Reload page
ocbrowser get url             # Current URL
ocbrowser get title           # Page title
ocbrowser close               # Close browser
```

### Snapshots & Screenshots
```bash
ocbrowser snapshot            # Accessibility tree
ocbrowser snapshot -i         # Interactive elements only (best for AI)
ocbrowser screenshot [path]   # Screenshot (--full for full page)
ocbrowser screenshot --annotate          # Numbered element labels
ocbrowser screenshot --return-base64     # Include base64 in response
ocbrowser screenshot --cursor           # Draw crosshair at last action
```

### Interaction
```bash
ocbrowser click <sel>         # Click element
ocbrowser fill <sel> <text>   # Clear and fill
ocbrowser type <sel> <text>   # Type (appends)
ocbrowser press <key>         # Key press (e.g. Enter, Tab, Control+a)
ocbrowser hover <sel>         # Hover element
ocbrowser select <sel> <val>  # Select dropdown
ocbrowser check <sel>         # Check checkbox
ocbrowser drag <src> <tgt>    # Drag and drop
ocbrowser upload <sel> <file> # File upload
ocbrowser scroll <dir> [px]   # Scroll up/down/left/right
```

### Semantic Locators (preferred over CSS selectors)
```bash
ocbrowser find role button click --name "Submit"
ocbrowser find text "Sign In" click
ocbrowser find label "Email" fill "me@example.com"
ocbrowser find placeholder "Search..." fill "query"
```

### Wait
```bash
ocbrowser wait <selector>              # Wait for element
ocbrowser wait --text "Welcome"        # Wait for text
ocbrowser wait --url "**/dashboard"    # Wait for URL pattern
ocbrowser wait --load networkidle      # Wait for network idle
ocbrowser wait --fn "window.ready"     # Wait for JS condition
```

### Get Info
```bash
ocbrowser get text <sel>      # Text content
ocbrowser get value <sel>     # Input value
ocbrowser get attr <sel> <a>  # Attribute
ocbrowser get count <sel>     # Count matches
ocbrowser get box <sel>       # Bounding box
```

### Network & State
```bash
ocbrowser requests            # List HTTP requests
ocbrowser route <pattern> --mock-status 200 --mock-body '{"ok":true}'
ocbrowser cookies get
ocbrowser storage get local <key>
ocbrowser state save ./auth.json
ocbrowser state load ./auth.json
```

### Auth
```bash
ocbrowser --auto-connect state save ./auth.json   # Import from running Chrome
ocbrowser --profile ~/.myapp open <url>           # Persistent profile
ocbrowser --session-name myapp open <url>         # Named session
```

---

## Handling Dialogs & Popups

```bash
ocbrowser dialog status        # Check for pending alert/confirm/prompt
ocbrowser dialog accept        # Accept dialog
ocbrowser dialog accept "text" # Accept prompt with input text
ocbrowser dialog dismiss       # Dismiss/cancel dialog
```

HTTP Basic Auth is handled automatically via env vars:
```bash
AGENT_BROWSER_PROXY_USERNAME=user AGENT_BROWSER_PROXY_PASSWORD=pass ocbrowser open <url>
```

---

## Tips for AI Agents

- Always call `snapshot -i` after navigation before interacting — refs change on every page load.
- Use `page-state` at the start of a task to get URL + snapshot + screenshot in one call.
- Use `form-state` before filling a form to discover field names and current values.
- Prefer refs (`@e1`, `@e2`) over CSS selectors — they are more stable and semantic.
- If a command seems to hang, call `dialog status` — a JS dialog may be blocking.
- Use `screenshot --cursor --return-base64` after clicks to visually verify interactions.
- Chain commands with `&&` when you don't need to parse intermediate output.
