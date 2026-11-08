---
name: uprock
description: How to use the UpRock CLI for authentication, API key management (uprock keys list/use/create/revoke), daemon management, and AI tools (web crawling, multi-engine search, multi-region verification via uprock ai verify, resource fetching, and buying AI credits via uprock ai buy). Use this skill whenever the user asks to crawl a URL, research the web, verify site availability/performance across regions, top up AI credits, manage API keys, manage the UpRock daemon, or authenticate with UpRock.
---

# UpRock CLI

The `uprock` CLI connects to the UpRock network. This document describes every command,
its flags, and when to use each one. All commands follow `uprock <group> <action> [args]
[flags]`.

Install globally before use:

```bash
npm i -g uprock
```

## Authentication

Manage login state. Authentication is REQUIRED for earning and provides an API key for
AI tool commands (see [AI Tools](#ai-tools) for other key options).

### `uprock auth login`

Authenticate with UpRock. Prompts for an email address (or accepts `--email`), sends a
one-time code to that address, then prompts for the code on stdin. The session is stored
locally after confirmation.

**Requires an interactive terminal.** If stdin is not a TTY (automation, bots, CI), the code
prompt will receive EOF and the login will abort with a hint to use `auth request` +
`auth confirm` instead. See below.

| Flag | Default | Description |
|------|---------|-------------|
| `--email` | — | Pre-fill the email address in the login flow. |
| `--setup-ai` | false | Obtain an UpRock AI API key after login. Use this when you plan to call AI tool commands. |

### `uprock auth request`

**Non-interactive login, step 1.** Send a login code to the given email address and print
the session ID to stdout. No stdin interaction — designed for automation and bot-driven flows.

The session ID is printed as a bare string to stdout (one line, no label). Informational
messages go to stderr. This means `$(uprock auth request --email ...)` captures only the
session ID.

| Flag | Default | Description |
|------|---------|-------------|
| `--email` | — | Email address to send the login code to. **Required.** |

After calling `request`, obtain the verification code from the user through your own
channel (chat message, webhook, etc.), then pass it to `auth confirm`.

Example:
```bash
SESSION=$(uprock auth request --email user@example.com 2>/dev/null)
# → SESSION now holds the session ID
```

### `uprock auth confirm`

**Non-interactive login, step 2.** Confirm a login session using the session ID from
`auth request` and the verification code the user received by email. No stdin interaction.

| Flag | Default | Description |
|------|---------|-------------|
| `--session` | — | Session ID returned by `auth request`. **Required.** |
| `--code` | — | Verification code from the email. **Required.** |
| `--setup-ai` | false | Obtain an UpRock AI API key after login. Use this when you plan to call AI tool commands. |

On success, prints `"Logged in."` to stdout and returns exit code 0.

Example:
```bash
uprock auth confirm --session "$SESSION" --code 0018148
```

#### Non-interactive login decision rule

- **You have an interactive terminal** → use `uprock auth login`. Simpler, single command.
- **You are a bot, script, or CI pipeline** → use `auth request` + `auth confirm`. Two
  commands, no stdin required. The session ID bridges the two steps.

Full bot workflow:
```bash
# 1. Request code (captures session ID)
SESSION=$(uprock auth request --email user@example.com 2>/dev/null)

# 2. Obtain the code from the user via your own channel
#    (chat prompt, webhook callback, etc.)

# 3. Confirm login
uprock auth confirm --session "$SESSION" --code "$CODE"
```

### `uprock auth logout`

Clear the local session. After logout:
- The daemon continues running but stops earning.
- AI tool commands fail unless `UPROCK_API_KEY` is set.
- The user must run `uprock auth login` again to restore access.

### `uprock auth status`

Print whether a valid session exists, the associated email address, and details about the
active AI API key:
- Key name and first characters/prefix (never the full key)
- Key source (`UPROCK_API_KEY` environment variable or `saved login`)
- Organization ID
- Credits remaining (`credits_remaining`)

Returns exit code 1 when neither logged in nor configured with `UPROCK_API_KEY`.

When `--setup-ai` is passed to `uprock auth login` or `uprock auth confirm`, the CLI also
prints the configured key's name, prefix, and whether it was newly created (`[new key]`)
or reused (`[existing key]`).

## API Keys (`uprock keys`)

Manage your organization's UpRock AI API keys. Requires an active login session
(`uprock auth login`). Full API keys are never printed by `list`, `use`, or `revoke`.

### `uprock keys list`

List all active API keys in the organization, showing:
- `*` marker on the key the CLI uses by default
- `NAME`
- `PREFIX` (first characters of the key)
- `CREATED` date (`YYYY-MM-DD`)
- `LAST USED` date (`YYYY-MM-DD` or `never`)

### `uprock keys use <name or prefix>`

Set which API key the CLI uses by default (persisted to the local credential store and
updated on the running daemon if active). Matches by exact name or prefix first, then
case-insensitive name or prefix prefix-match. Note: the `UPROCK_API_KEY` environment
variable still takes precedence when set.

### `uprock keys create <name>`

Create a new API key in the organization with standard MCP permissions, print the full
key once, and offer to make it the default CLI key.

| Flag | Default | Description |
|------|---------|-------------|
| `--use` | false | Make the newly created key the default CLI key without prompting. |

### `uprock keys revoke <name or prefix>`

Revoke an API key by name or prefix. If the revoked key was the saved default CLI key,
the saved default key is cleared automatically.

## Daemon

The daemon is a long-running background process that earns while the machine is idle. It
communicates with the CLI over IPC (Unix socket on macOS/Linux, named pipe on Windows).

Decision rule for starting the daemon:
- **One-off session or debugging** → `uprock daemon start` (manual, does not survive reboot)
- **Persistent earning** → `uprock daemon install` (registers as an OS service, auto-starts
  on login, restarts on crash)

PREFER `daemon install` for any use case where the user wants the daemon running
continuously. `daemon start` is only appropriate when the user explicitly wants a
temporary session they will stop manually.

### `uprock daemon start`

Start the daemon as a background process. The process runs until explicitly stopped with
`daemon stop` or until the machine shuts down. Does NOT survive reboot — PREFER
`daemon install` for persistent earning.

### `uprock daemon stop`

Stop the running daemon. Attempts graceful shutdown over IPC first, falls back to SIGTERM,
and uses SIGKILL as a last resort if the process does not exit within 10 seconds.

### `uprock daemon status`

Print daemon state: PID, version, uptime, authentication status, earning status (with
earn rate when active), and whether the OS service is installed. Returns exit code 1 when
the daemon is not running, exit code 2 when running but not responding — use this to
check liveness in scripts.

### `uprock daemon logs`

Show daemon log output. Without flags, prints the last 10 lines and exits.

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--lines` | `-n` | 10 | Number of historical lines to show. |
| `--follow` | `-f` | false | Stream new log lines as they are written (like `tail -f`). Blocks until interrupted. |
| `--grep` | — | — | Filter lines by substring (case-insensitive). Applies to both historical and streamed lines. |
| `--path` | — | false | Print the absolute log file path and exit. Use this when you need to open the log in another tool. |

Decision rule:
- **Quick diagnostics** → `uprock daemon logs -n 50` or `uprock daemon logs --grep error`
- **Live monitoring** → `uprock daemon logs -f`
- **External tooling** → `uprock daemon logs --path` to get the file path, then use
  your own viewer.

### `uprock daemon install`

Register the daemon as an OS autolaunch service: launchd (macOS), systemd (Linux), or
Task Scheduler (Windows). After installation the daemon starts on login and restarts
automatically if it crashes.

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--force` | `-f` | false | Overwrite an existing service configuration. Use when upgrading the binary or changing install options. |
| `--ui` | — | false | Install the GUI desktop app as the daemon instead of the headless CLI. |

Do NOT use `npx` to install the daemon. `npx` runs from a temporary directory whose path
changes between invocations — the registered OS service will point to a stale binary
location. ALWAYS install the package globally first (`npm i -g uprock`), then run
`uprock daemon install`.

### `uprock daemon uninstall`

Remove the OS autolaunch service. The daemon stops immediately and will no longer start
on login.

## AI Tools

AI-powered tools that execute through the UpRock distributed network. Every AI command
requires an API key. Set `UPROCK_API_KEY` to use a key directly, or run
`uprock auth login` — the key is resolved automatically on first use.

All AI tool commands (`crawl`, `research`, `verify`) write JSON to stdout and progress indicators to stderr, so piped output
is always clean JSON. Use `jq` to extract fields.

### Quick Decision Matrix

| Task | Command |
|------|---------|
| Fetch a single web page (with JS rendering & anti-bot bypass) | `uprock ai crawl <url> --content` |
| Search the web across multiple engines & regions | `uprock ai research "<query>"` |
| Verify site availability & Core Web Vitals across regions | `uprock ai verify <url>` (`sweep` is a deprecated alias) |
| Retrieve a `crawl://` or `sweep://` resource URI | `uprock ai fetch "<uri>"` |
| Top up AI credits via Stripe Checkout (`$5`–`$1000` USD) | `uprock ai buy <amount>` |

### `uprock ai crawl <url>`

Fetch a URL via the UpRock crawl network.

This command fetches web pages through a distributed network of real browser instances in
different geographic locations. It supports multiple HTTP methods and can render JavaScript
for single-page applications.

Methods:
- **CRAWL_FULL_PAGE**: Full page rendering with JavaScript execution (default, recommended).
- **GET**: Standard HTTP GET (faster, but only use when you are certain the page does not
  require JavaScript rendering).
- **POST**: HTTP POST with body.
- **PUT**: HTTP PUT with body.

For most websites, CRAWL_FULL_PAGE is the most reliable method. Use GET only as an
optimization when you know the target is a static page or API endpoint.

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--method` | `-m` | CRAWL_FULL_PAGE | HTTP method: GET, POST, PUT, CRAWL_FULL_PAGE. |
| `--body` | `-b` | — | Request body for POST/PUT. |
| `--timeout` | `-t` | 60 | Timeout in seconds (max 300). For CRAWL_FULL_PAGE, values below 60 are raised to 60. |
| `--country` | — | — | Where to execute the crawl FROM (device geographic placement). This controls the device's physical location, NOT the content topic — a crawl from Germany fetches the same URL but may see region-specific content (e.g., cookie banners, localized pricing). Accepts meta-regions (NA, EU, APAC, LATAM, MEA) or ISO country codes (US, DE, JP). PREFER meta-regions — they give the placement algorithm more flexibility to find available devices. Use specific country codes only when you need exact country-level placement (e.g., to see local pricing, comply with regional regulations, or access geo-restricted content). |
| `--device` | `-d` | — | Device type: mobile, desktop. |
| `--retries` | `-r` | 2 | Speculative retries (max 3). When > 0, a new attempt starts every 7 seconds using a separate device session. First success wins. Default of 2 means up to 3 concurrent sessions. Set to 0 for a single attempt. |
| `--content` | — | false | Auto-fetch the markdown resource and inline it in the response under `inlined_markdown`. Use this when you need the full page text, not just the summary. |

Response structure:

```json
{
  "status": "success",
  "job_id": "...",
  "meta": { "url": "...", "status_code": 200, "title": "...", "content_type": "...", "time_ms": 1234 },
  "summary": "Brief summary of the page content...",
  "content": {
    "html":     { "resource": "crawl://{jobId}/html", "size_bytes": 12345, "mime_type": "text/html" },
    "markdown": { "resource": "crawl://{cacheKey}/markdown", "size_bytes": 6789, "mime_type": "text/markdown" }
  }
}
```

When `--content` is passed, an additional `inlined_markdown` field contains the full
markdown text directly in the response.

The `summary` field is present when content extraction succeeds (most pages). Full
markdown and HTML are available via the resource URIs in `content` — pass them to
`uprock ai fetch` to retrieve on demand. PREFER the markdown resource over HTML — it is
significantly more compact and easier to process.

Examples:

```bash
# Crawl a page with full JS rendering
uprock ai crawl example.com

# Fast static fetch
uprock ai crawl example.com -m GET

# Crawl from Europe with inlined content
uprock ai crawl example.com --country EU --content

# POST JSON to an API endpoint
uprock ai crawl api.example.com -m POST -b '{"query": "test"}'

# Single attempt, no retries
uprock ai crawl example.com -r 0

# Extract just the summary
uprock ai crawl example.com | jq -r '.summary'
```

### `uprock ai research <query...>`

Search the web using multiple search engines across geographic regions. This command
intelligently routes queries to the best-performing search engines for each region and
deduplicates results.

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--max-results` | `-n` | 20 | Maximum number of results. Actual count may be lower due to cross-source deduplication. Request 2–3x your target to compensate. |
| `--num-sources` | — | 5 | Number of provider+country combinations to query in parallel (max 20). More sources yield more diverse results but take longer. |
| `--timeout` | `-t` | 300 | Timeout in seconds. |
| `--countries` | — | — | Where to search FROM (the searcher's geographic perspective), NOT what to search ABOUT. When omitted, the server automatically selects optimal countries for broad global coverage — this is the correct default for most queries. Accepts meta-regions (NA, EU, APAC, LATAM, MEA) or ISO country codes. PREFER meta-regions — they give the scheduler more flexibility and better cache hit rates. Use specific country codes only when the query demands country-level precision. Do NOT mix meta-regions and country codes in a single call. |

GEOGRAPHIC TARGETING: `--countries` controls where searches execute FROM, not what they
search ABOUT. Use it to get results as locals in that region see them.

Decision rule:
- Geography is the TOPIC ("Thai cuisine", "Paris hotels") → keep it in the query, omit `--countries`.
- Geography is the PERSPECTIVE ("what locals see") → use `--countries`, keep query generic.
- BOTH ("what Germans think about Italian food") → "Italian food" in query, `--countries EU`.

Examples:
```
RIGHT: uprock ai research "best beach destinations" --countries EU
  → returns what Europeans see when they search for beach holidays

WRONG: uprock ai research "best European beach destinations"
  → searches globally for pages that mention "European beaches"

RIGHT: uprock ai research "best restaurants in Paris"
  → global results about Paris restaurants (geography is the topic)

WRONG: uprock ai research "best restaurants" --countries EU
  → returns what Europeans see when searching for restaurants (not Paris-specific)
```

If geography is the subject of the query (e.g., "history of the Berlin Wall"), keep it in
the query — `--countries` is for execution context, not query content.

If the user says "Germany" but the context is broadly European, use `--countries EU`. Use
`--countries DE` only when you specifically need German-language results, German law, or
data that differs between Germany and its EU neighbors.

To compare how different regions see a topic, make SEPARATE calls — one per region. Do
NOT combine regions in a single call expecting comparative results.

Response structure:

```json
{
  "query": "search terms",
  "count": 5,
  "results": [
    {
      "url": "https://example.com/page",
      "title": "Page Title",
      "description": "A brief description of the page content..."
    }
  ]
}
```

Results are unordered. Title and description are included when available.

### 4. Multi-Region Verification (`uprock ai verify <url>`)

Verify website reliability, availability, and performance across geographic regions (`uprock ai sweep <url>` remains available as a deprecated alias).

Use this after deploying a website or service to verify it is up, responsive, and
performing well from different parts of the world. Loads the target URL from multiple
regions simultaneously, capturing performance metrics from each check. All checks
(regions × tries) run concurrently, so total operation time is approximately equal to the
timeout value.

Each completed check returns Core Web Vitals (TTFB, FCP, LCP, CLS), load times, transfer
size, and HTTP protocol. Screenshots are captured and available as resource URIs in the
response — pass them to `uprock ai fetch` for visual verification, but they can be
ignored if you only need the metrics.

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--regions` | `-r` | NA,EU,APAC | Geographic regions to test from: NA (North America), EU (Europe), APAC (Asia Pacific), LATAM (Latin America), MEA (Middle East & Africa). Default regions have the fastest and most reliable device pools. |
| `--tries` | — | 5 | Number of checks per region. All checks across all regions run concurrently. |
| `--timeout` | `-t` | 60 | Global timeout in seconds. All checks run concurrently, so wall-clock time is approximately this value. |
| `--device` | `-d` | mobile | Device type: mobile, desktop. |

The response includes a `report_url` field with a link to a human-readable report page
that can be shared for visual review.

A verification run may complete with some failed checks. Inspect the `failed_jobs` count and
per-job `error` fields to identify issues, while completed regions still return their metrics.

Examples:

```bash
# Default verification: NA, EU, APAC — 5 checks each, mobile
uprock ai verify example.com

# Specific regions, desktop
uprock ai verify example.com -r NA,EU -d desktop

# Quick check: 2 tries, short timeout
uprock ai verify example.com --tries 2 -t 30

# All five regions
uprock ai verify example.com -r NA,EU,APAC,LATAM,MEA

# Extract the shareable report URL
uprock ai verify example.com | jq -r '.report_url'
```

### 5. Resource Fetch (`uprock ai fetch <uri>`)

Fetch the full content of a `crawl://` or `sweep://` resource URI from a prior tool
response.

Use this when you need to read actual content that a previous `crawl` or `verify` (`sweep`) command
returned as a resource URI. For crawl results, the response includes a summary for quick
analysis plus resource URIs pointing to full content in markdown and HTML formats. When
the summary is sufficient, you do not need this command. When you need the complete page
text — to extract specific data, parse tables, read full articles — pass the resource URI
here.

For crawl results, use the markdown resource URI unless you specifically need raw HTML
structure. Markdown is significantly more compact and easier to work with.

For verification results, pass a `sweep://` screenshot URI to retrieve the screenshot image.

Output is written directly to stdout: text for markdown/HTML resources, binary for images.
Pipe or redirect as needed.

Examples:

```bash
# Read the full markdown content of a crawled page
uprock ai fetch "crawl://abc123/markdown"

# Get raw HTML when you need DOM structure
uprock ai fetch "crawl://abc123/html"

# Save a verification screenshot to a file
uprock ai fetch "sweep://def456/NA/0/screenshot" > screenshot.png

# Pipe markdown through a processor
uprock ai fetch "crawl://abc123/markdown" | head -100
```

### 6. Buy AI Credits (`uprock ai buy <amount>`)

Create a Stripe Checkout session to top up your organization's UpRock AI credits, print the checkout link, and wait/poll until payment completes.

| Argument / Flag | Short | Default | Description |
|-----------------|-------|---------|-------------|
| `<amount>` | — | — | Whole USD amount between `5` and `1000` (e.g. `20`, `100`, or `'$20'`). **Required.** |
| `--timeout` | `-t` | 300 | Timeout in seconds while waiting for payment completion. |

#### Billing & Credit Management for Agents

When an AI command or API call fails due to exhausted credits (`402 Payment Required` / `BILLING_TOPUP_REQUIRED` or `429` daily cap reached) or when the user asks to top up credits:
1. Run `uprock ai buy <amount>` (for example, `uprock ai buy 20`). This endpoint is permitted even when the API key is currently blocked for `402 Payment Required`.
2. The CLI immediately prints the Stripe Checkout URL to stdout:
   ```
   Complete your $20 USD top-up (40000 credits) at:
   https://checkout.stripe.com/c/pay/cs_...
   ```
   and writes `Waiting for payment completion...` to stderr.
3. **Surface the Stripe Checkout URL directly to the user** so they can open it and complete payment in their browser.
4. Keep the command running (up to `-t/--timeout`, default `300`s): `uprock ai buy` polls `GET /v1/api-keys/self` every 3 seconds until the credit balance increases (or the `402` block clears with positive credits), prints `Payment complete. Credits remaining: <credits>`, and exits with code `0`.

Examples:

```bash
# Buy $20 USD in AI credits and wait for payment completion
uprock ai buy 20

# Buy $100 USD with a 10-minute timeout
uprock ai buy 100 -t 600
```

## Version

```bash
uprock version
```

Print version string and build metadata. No flags.

## Global Flags

These flags work with every command:

| Flag | Short | Description |
|------|-------|-------------|
| `--config` | `-c` | Path to a config file. Overrides the default config location. |
| `--verbose` | `-v` | Enable verbose logging to stderr. Useful for debugging authentication, IPC, or network issues. |

## Common Workflows

**First-time setup:**

```bash
npm i -g uprock
uprock auth login
uprock daemon install
```

**Crawl a page and read the full content:**

```bash
# Option 1: use --content to inline markdown in one call
uprock ai crawl example.com --content | jq -r '.inlined_markdown'

# Option 2: crawl first, then fetch the markdown separately
uprock ai crawl example.com > result.json
URI=$(jq -r '.content.markdown.resource' result.json)
uprock ai fetch "$URI"
```

**Research a topic and crawl the top results:**

```bash
uprock ai research "best static site generators 2026" -n 5 | jq -r '.results[].url'
# Then crawl individual URLs for full content
uprock ai crawl <url> --content
```

**Verify a site after deployment and share the report:**

```bash
uprock ai verify mysite.com | jq -r '.report_url'
# Share the printed URL with your team
```

**Top up AI credits when balance is low or 402 is returned:**

```bash
uprock ai buy 20
# Share the printed Stripe Checkout URL with the user; CLI waits until payment completes
```

**Check daemon health:**

```bash
uprock daemon status && echo "running" || echo "stopped"
uprock daemon logs --grep error -n 20
```
