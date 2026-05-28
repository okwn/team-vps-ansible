---
name: playwright-cli
description: Drive a real browser (navigate, click, fill forms, scrape, screenshot) using `playwright-cli` — the Microsoft CLI wrapper around Playwright. Use this skill whenever the user asks you to open a URL, fill a form, click through a site, extract visible data, take screenshots, record a user flow, or debug a selector. Prefer this over `curl`/`requests` for anything JS-rendered, login-gated, or where you need to see the rendered page. Chrome is preinstalled; `playwright-cli` v0.1.8+ is in PATH.
---

# playwright-cli

You have `playwright-cli` (Microsoft, v0.1.8+) and Google Chrome installed on this VPS. Use it for anything that needs a real browser.

## Why this over alternatives

| Task | Prefer | Why |
|------|--------|-----|
| Static HTML scrape | `curl` / `rg` | Faster, no browser needed |
| JS-rendered page / SPA | `playwright-cli` | Waits for render |
| Login-gated page (Gmail, Slack, internal tools) | `playwright-cli attach --cdp=chrome` | Reuses your live Chrome session with existing cookies |
| Form automation | `playwright-cli` | Handles modal / JS events |
| Screenshot for visual inspection | `playwright-cli` | Pixel-accurate |
| HTTP API call | `curl` / `requests` | Don't launch a browser for this |

## Two operating modes

### Mode A: sandbox Chrome (default)

Launches a fresh Chromium with no session. Use when the task is public or can be re-authenticated scripted.

```bash
playwright-cli open https://example.com
playwright-cli codegen https://example.com        # record a flow, print code
playwright-cli screenshot https://example.com /tmp/out.png --full-page
```

### Mode B: attach to user's real Chrome (`--cdp=chrome`)

Connects to an already-running Chrome (with all the user's logins). Essential for authenticated tasks.

Prerequisite: Chrome running with remote debugging. Start with:
```bash
google-chrome-stable --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-cdp &
```
Or use playwright-cli's built-in attach that resolves the channel automatically:
```bash
playwright-cli attach --cdp=chrome
```
(Also works with `msedge`, `chrome-canary`, `chrome-beta`, `chrome-dev`.)

Once attached, all subsequent commands run against that Chrome.

## Essential commands

```bash
playwright-cli --version                          # sanity check, should be >= 0.1.8
playwright-cli --help                             # full list

# Navigation / scraping
playwright-cli open <URL>                         # open in browser (manual inspection)
playwright-cli screenshot <URL> <out.png> [--full-page]
playwright-cli pdf <URL> <out.pdf>                # rendered page -> PDF

# Recording (user demonstrates -> code generated)
playwright-cli codegen <URL>                      # interactive, exits with code snippet
playwright-cli codegen <URL> --target python      # python output
playwright-cli codegen <URL> --target javascript  # js (default)

# Selector debugging
playwright-cli inspect                            # hover elements, get suggested selectors
```

## Common workflows

### 1. Scrape a JS-rendered page

```bash
# Headless, write rendered HTML to file
playwright-cli open https://example.com/spa-page --headless --dump-html=/tmp/page.html
rg 'pattern' /tmp/page.html
```

### 2. Log into an internal tool and extract data

```bash
# Start Chrome once, log in mannually via playwright-cli attach
google-chrome-stable --remote-debugging-port=9222 --user-data-dir=~/.chrome-workdata &
# In another terminal
playwright-cli attach --cdp=chrome
# Then: playwright-cli screenshot https://internal.example.com/dashboard ~/out.png
```

### 3. Automate a repeatable flow

```bash
# Record the flow by clicking through
playwright-cli codegen https://example.com/form --target python > flow.py
# Edit flow.py as needed, then run with plain Python + playwright
python -m pip install --user playwright && python -m playwright install chromium
python flow.py
```

### 4. Take a screenshot for a bug report

```bash
playwright-cli screenshot https://broken.example.com /tmp/bug.png --full-page
# Then: review with `claude` or attach to Slack message
```

## Hints for Claude Code

- **Do NOT install Playwright browsers separately** — use the system's Google Chrome via `--cdp=chrome` or playwright-cli's bundled Chromium.
- **Headless is the default** on this server; no display manager. Use `--headless` explicitly if a command defaults to headed.
- **Timeouts**: Add `--timeout=30000` (ms) for slow pages. Default is 30s.
- **Output inspection**: Prefer `--dump-html` / `--screenshot` / `--pdf` over manual stdout parsing.
- **For CI-like automation**: combine with `playwright` (the node/python package) for fine control. `playwright-cli` is for quick wins.
- **Don't leave zombie Chrome processes**: v0.1.8 fixed the orphaning bug, but if you see multiple `chrome` processes after a session, `pkill -f 'chrome.*playwright'` cleans up.

## Troubleshooting

- `playwright-cli: command not found` — mise activate probably didn't run. Prefix with `$HOME/.local/share/mise/installs/node/lts/bin/playwright-cli` or run inside an interactive bash.
- `Error: Failed to launch browser` — usually missing shared libs. Install with `sudo apt install -y libnss3 libxkbcommon0 libgbm1` (should already be present from the chrome-playwright role).
- `Target page, context or browser has been closed` — the attached Chrome was closed. Reattach.
- `Error: browserType.launch: Executable doesn't exist` — playwright-cli's bundled Chromium missing. Run `npx playwright install chromium` as the owner user.

## References

- Source: https://github.com/microsoft/playwright-cli
- Full Playwright docs (node/python API): https://playwright.dev
