---
name: uprock
description: How to use the UpRock CLI for authentication, daemon management, and AI tools (web crawling, multi-engine search, performance sweeps, Video Semantic Search). Use this skill whenever the user asks to crawl a URL, research the web, search inside video or find a moment in a video (VSS), test site performance, manage the UpRock daemon, or authenticate with UpRock.
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

Print whether a valid session exists, the associated email address, and whether an AI API
key is set. Returns exit code 1 when not authenticated.

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

All AI commands write JSON to stdout and progress indicators to stderr, so piped output
is always clean JSON. Use `jq` to extract fields.

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

### `uprock ai sweep <url>`

Test website reliability and performance across geographic regions.

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

A sweep may complete with some failed checks. Inspect the `failed_jobs` count and
per-job `error` fields to identify issues.

Examples:

```bash
# Default sweep: NA, EU, APAC — 5 checks each, mobile
uprock ai sweep example.com

# Specific regions, desktop
uprock ai sweep example.com -r NA,EU -d desktop

# Quick check: 2 tries, short timeout
uprock ai sweep example.com --tries 2 -t 30

# All five regions
uprock ai sweep example.com -r NA,EU,APAC,LATAM,MEA

# Extract the shareable report URL
uprock ai sweep example.com | jq -r '.report_url'
```

### `uprock ai fetch <uri>`

Fetch the full content of a `crawl://` or `sweep://` resource URI from a prior tool
response.

Use this when you need to read actual content that a previous `crawl` or `sweep` command
returned as a resource URI. For crawl results, the response includes a summary for quick
analysis plus resource URIs pointing to full content in markdown and HTML formats. When
the summary is sufficient, you do not need this command. When you need the complete page
text — to extract specific data, parse tables, read full articles — pass the resource URI
here.

For crawl results, use the markdown resource URI unless you specifically need raw HTML
structure. Markdown is significantly more compact and easier to work with.

For sweep results, pass a `sweep://` screenshot URI to retrieve the screenshot image.

Output is written directly to stdout: text for markdown/HTML resources, binary for images.
Pipe or redirect as needed.

Examples:

```bash
# Read the full markdown content of a crawled page
uprock ai fetch "crawl://abc123/markdown"

# Get raw HTML when you need DOM structure
uprock ai fetch "crawl://abc123/html"

# Save a sweep screenshot to a file
uprock ai fetch "sweep://def456/NA/0/screenshot" > screenshot.png

# Pipe markdown through a processor
uprock ai fetch "crawl://abc123/markdown" | head -100
```

### `uprock ai video-search <query...>`

Search inside video by meaning and get back the exact matching moments — UpRock **Video
Semantic Search (VSS)**.

Use this to find where something is said or shown across indexed video — "the part where
he opens the box", "a red car drifting in the rain". Each video is split into overlapping
~30 second moments; every moment is transcribed and fused into a single vector covering
both its frames and its speech. A query embeds into that same space, then a reranker
re-reads each candidate's keyframes and transcript together with the query. Matches
therefore come from what actually happens on screen and in the audio, NOT from titles,
descriptions, or tags. Results are returned best-first. A video that was never indexed
cannot appear — no result means "not in the index", not "not in the video".

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--max-results` | `-n` | 10 | Maximum moments to return. The service caps a response at roughly 10 segments regardless of what you ask for, so treat this as a lower bound, not a way to page. It is also applied AFTER reranking, so raising it will not surface fundamentally new matches. |
| `--tags` | — | — | Scope the search to a tagged corpus, as `key=value` (repeatable). Tag values are **CASE-SENSITIVE** — `PostHog` and `posthog` are different corpora. Omit to search everything visible to your key. |
| `--unique` | — | false | Return ONE moment per video — each video's best-scoring moment — with `occurrence_count` set. Useful because hits are per-segment: on a small corpus a single video can otherwise fill the whole page. |
| `--image` | — | — | Path to a PNG or JPEG to search by visual similarity. Combine with a text query to match both, or use alone. An image-only query keeps first-stage vector order and skips reranking, so scores are cosine similarity rather than reranker scores. |
| `--after` | — | — | Only moments from videos published on or after this date. Videos with an unknown publication date NEVER match a bounded range — they are excluded, not treated as old. |
| `--before` | — | — | Only moments from videos published on or before this date. Either bound may be used alone. |
| `--recent` | — | false | Re-sort the matched results newest-first by publication date, with relevance as the tiebreak. This re-orders matches only — it is NOT a chronological feed of everything published. |
| `--timeout` | `-t` | 120 | Timeout in seconds. |

QUERY PHRASING: describe the moment as you would *see* it, not as a video would be
*titled*. Metadata-style keyword queries match moments that happen to mention those words,
which is rarely what you want. PREFER one specific description over broad keywords — the
reranker is where the quality comes from, and it only ever sees candidates the first-stage
retrieval already found.

```
RIGHT: uprock ai video-search "he pulls the phone out of the box"
  → finds the moment that action happens, in any video

WRONG: uprock ai video-search "iPhone unboxing 2026"
  → title-style keywords; matches moments that merely say those words

RIGHT: uprock ai video-search "a red car drifting in the rain"
  → matches what is visible on screen, not just what is spoken
```

Decision rule:
- **"Where is this discussed?"** → default per-moment search. Returns every strong moment,
  and one video may appear several times.
- **"Which videos cover this?"** → `--unique`. One entry per video, with `occurrence_count`
  showing how many of that video's moments matched.
- **"What is new on this topic?"** → `--after` to bound the range, plus `--recent` to order
  by publication date.

Response structure:

```json
{
  "results": [
    {
      "id": "9f2c1e34-…",
      "video_id": "3a7e88b2-…",
      "t0": 812.0,
      "t1": 840.0,
      "score": 0.87,
      "title": "Static fire anomaly",
      "transcript": "…extract of what is said during the moment…",
      "occurrence_count": 0,
      "origin_url": "https://www.youtube.com/watch?v=…",
      "source": "youtube",
      "published_at": 1767225600
    }
  ]
}
```

`t0` and `t1` are the moment's start and end in SECONDS. Combine `origin_url` with `t0` to
deep-link straight to the moment. `source` is the platform the video came from: youtube,
tiktok, x, instagram, facebook, or upload. `transcript` is a truncated extract, present
when the moment contains speech. `published_at` is the video's publication time in unix
seconds, or 0 when unknown. `occurrence_count` is meaningful only with `--unique`, where
it is a lower bound on that video's matching moments; it is 0 otherwise.

Each hit is a matched SEGMENT, so one video can legitimately return several results —
use `--unique` when you want distinct videos instead.

Scope comes from your API key (your own indexed videos plus the public catalog), narrowed
further by `--tags` when given. An empty result set can also mean a recently-ingested video
has not finished indexing yet — indexing runs asynchronously after ingest completes, and
searching is the only readiness signal. Retry before concluding a video is absent.

Examples:

```bash
# Find the moment an action happens
uprock ai video-search "the part where he opens the box"

# Which videos cover this topic at all
uprock ai video-search "rocket engine test failure" --unique -n 5

# Scope to one tagged corpus (tag values are case-sensitive)
uprock ai video-search "pricing objection" --tags cust-demo=PostHog

# Search by what a frame looks like
uprock ai video-search --image ./red-car.jpg

# Newest coverage first, this year only
uprock ai video-search "quarterly earnings" --after 2026-01-01 --recent

# Deep-link the best moment
uprock ai video-search "keynote demo" | jq -r '.results[0] | "\(.origin_url)&t=\(.t0|floor)"'
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

**Find a moment in video and share a deep link:**

```bash
uprock ai video-search "the part where he opens the box" \
  | jq -r '.results[0] | "\(.origin_url)&t=\(.t0|floor)"'
# Share the printed URL — it opens the video at that moment
```

**Sweep a site after deployment and share the report:**

```bash
uprock ai sweep mysite.com | jq -r '.report_url'
# Share the printed URL with your team
```

**Check daemon health:**

```bash
uprock daemon status && echo "running" || echo "stopped"
uprock daemon logs --grep error -n 20
```
