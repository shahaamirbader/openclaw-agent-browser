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
metadata.openclaw:
  emoji: 🌐
  homepage: https://github.com/shahaamirbader/openclaw-agent-browser
  install:
    node: openclaw-agent-browser
  requires:
    bins: [agent-browser]
allowed-tools: Bash(npx agent-browser:*), Bash(agent-browser:*)
---

# Browser Automation with openclaw-agent-browser

A fork of [vercel-labs/agent-browser](https://github.com/vercel-labs/agent-browser)
with added features for Openclaw AI agent workflows. Install via:

```bash
npm install -g agent-browser
agent-browser install   # downloads Chrome for Testing (first time only)
```

Full source: https://github.com/shahaamirbader/openclaw-agent-browser

---

## Core Workflow

Every browser session follows this pattern:

1. **Navigate** — `agent-browser open <url>`
2. **Snapshot** — `agent-browser snapshot -i` (get element refs like `@e1`, `@e2`)
3. **Interact** — use refs to click, fill, select
4. **Re-snapshot** — after navigation or DOM changes, get fresh refs

```bash
agent-browser open https://example.com/login
agent-browser snapshot -i
# @e1 [input type="email"], @e2 [input type="password"], @e3 [button] "Sign In"

agent-browser fill @e1 "user@example.com"
agent-browser fill @e2 "secret"
agent-browser click @e3
agent-browser wait --load networkidle
agent-browser snapshot -i
```

---

## New Features (Openclaw additions)

### 1. Screenshot base64 passthrough

Return the screenshot as a base64 string in the JSON response so it can be
displayed inline in a chat UI without filesystem access.

```bash
# CLI flag
agent-browser screenshot --return-base64

# JSON daemon API
echo '{"action":"screenshot","returnBase64":true}' | agent-browser batch --json
```

Response includes `{ "path": "...", "base64": "iVBORw0KGgo..." }`.

---

### 2. Cursor position overlay

After a `click` or `hover`, take a screenshot with `--cursor` to draw a red
crosshair at the exact pixel the action landed. Use this to visually confirm
the AI interacted with the right element.

```bash
agent-browser click @e3
agent-browser screenshot --cursor --return-base64
```

JSON API: `{ "action": "screenshot", "cursor": true, "returnBase64": true }`

---

### 3. Page state — full context in one call

Returns the current URL, page title, accessibility snapshot, and a base64
screenshot all in a single round trip. Avoids 3–4 separate commands when an
AI agent needs the full picture of where it is.

```bash
# JSON daemon API
echo '{"action":"page-state"}' | agent-browser batch --json
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
echo '{"action":"form-state"}' | agent-browser batch --json

# Scoped to a specific form element
echo '{"action":"form-state","selector":"form#checkout"}' | agent-browser batch --json
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
echo '{"action":"websocket-messages"}' | agent-browser batch --json

# Filter by direction, limit results
echo '{"action":"websocket-messages","direction":"received","limit":20}' | agent-browser batch --json

# Clear the buffer
echo '{"action":"websocket-clear"}' | agent-browser batch --json
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
agent-browser open <url>          # Navigate (aliases: goto, navigate)
agent-browser back                # Browser back
agent-browser forward             # Browser forward
agent-browser reload              # Reload page
agent-browser get url             # Current URL
agent-browser get title           # Page title
agent-browser close               # Close browser
```

### Snapshots & Screenshots
```bash
agent-browser snapshot            # Accessibility tree
agent-browser snapshot -i         # Interactive elements only (best for AI)
agent-browser screenshot [path]   # Screenshot (--full for full page)
agent-browser screenshot --annotate          # Numbered element labels
agent-browser screenshot --return-base64     # Include base64 in response
agent-browser screenshot --cursor           # Draw crosshair at last action
```

### Interaction
```bash
agent-browser click <sel>         # Click element
agent-browser fill <sel> <text>   # Clear and fill
agent-browser type <sel> <text>   # Type (appends)
agent-browser press <key>         # Key press (e.g. Enter, Tab, Control+a)
agent-browser hover <sel>         # Hover element
agent-browser select <sel> <val>  # Select dropdown
agent-browser check <sel>         # Check checkbox
agent-browser drag <src> <tgt>    # Drag and drop
agent-browser upload <sel> <file> # File upload
agent-browser scroll <dir> [px]   # Scroll up/down/left/right
```

### Semantic Locators (preferred over CSS selectors)
```bash
agent-browser find role button click --name "Submit"
agent-browser find text "Sign In" click
agent-browser find label "Email" fill "me@example.com"
agent-browser find placeholder "Search..." fill "query"
```

### Wait
```bash
agent-browser wait <selector>              # Wait for element
agent-browser wait --text "Welcome"        # Wait for text
agent-browser wait --url "**/dashboard"    # Wait for URL pattern
agent-browser wait --load networkidle      # Wait for network idle
agent-browser wait --fn "window.ready"     # Wait for JS condition
```

### Get Info
```bash
agent-browser get text <sel>      # Text content
agent-browser get value <sel>     # Input value
agent-browser get attr <sel> <a>  # Attribute
agent-browser get count <sel>     # Count matches
agent-browser get box <sel>       # Bounding box
```

### Network & State
```bash
agent-browser requests            # List HTTP requests
agent-browser route <pattern> --mock-status 200 --mock-body '{"ok":true}'
agent-browser cookies get
agent-browser storage get local <key>
agent-browser state save ./auth.json
agent-browser state load ./auth.json
```

### Auth
```bash
agent-browser --auto-connect state save ./auth.json   # Import from running Chrome
agent-browser --profile ~/.myapp open <url>           # Persistent profile
agent-browser --session-name myapp open <url>         # Named session
```

---

## Handling Dialogs & Popups

```bash
agent-browser dialog status        # Check for pending alert/confirm/prompt
agent-browser dialog accept        # Accept dialog
agent-browser dialog accept "text" # Accept prompt with input text
agent-browser dialog dismiss       # Dismiss/cancel dialog
```

HTTP Basic Auth is handled automatically via env vars:
```bash
AGENT_BROWSER_PROXY_USERNAME=user AGENT_BROWSER_PROXY_PASSWORD=pass agent-browser open <url>
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
