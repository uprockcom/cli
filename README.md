# UpRock CLI

Command-line interface for the [UpRock](https://uprock.com) network. Earn by contributing idle resources.

## Install

```bash
npm i -g uprock
```

Or run directly:

```bash
npx uprock
```

## Quick Start

```bash
npm i -g uprock
uprock auth login
uprock daemon install
```

This logs you in and installs the daemon as an OS service. The daemon starts automatically on login and restarts on crash — no need to manage it manually.

> **Do not use `npx` to start or install the daemon.** `npx` runs from a temporary directory, so the binary path changes between invocations and the OS service will point to a stale location.

## Commands

### Authentication

```bash
uprock auth login       # Log in with your UpRock account
uprock auth logout      # Log out
uprock auth status      # Check authentication state
```

You must be logged in to earn. The daemon can run without authentication, but it will not earn until you log in.

### Daemon

The daemon runs in the background and earns while your machine is idle.

```bash
uprock daemon start     # Start the daemon process
uprock daemon stop      # Stop the daemon
uprock daemon status    # Show daemon state (PID, uptime, earn rate, etc.)
uprock daemon logs      # Show daemon log output
uprock daemon install   # Install as an OS service (auto-start on login)
uprock daemon uninstall # Remove the OS service
```

Installing the service (`daemon install`) is recommended — it registers the daemon with launchd (macOS), systemd (Linux), or Task Scheduler (Windows) so it starts on login and restarts if it crashes.

`daemon logs` supports tail-like flags:

```bash
uprock daemon logs -n 50        # Show last 50 lines
uprock daemon logs -f           # Follow new output
uprock daemon logs --grep error # Filter lines by substring
uprock daemon logs --path       # Print log file path
```

### AI Tools

AI-powered tools that run through the UpRock network. Requires an API key — either log in
with `uprock auth login` or set the `UPROCK_API_KEY` environment variable directly.

```bash
uprock ai crawl <url>       # Fetch a URL via the crawl network
uprock ai research <query>  # Search the web across multiple engines and regions
uprock ai sweep <url>       # Test site performance across geographic regions
uprock ai fetch <uri>       # Fetch content from a crawl:// or sweep:// resource URI
```

`ai crawl` supports geographic targeting, HTTP methods, and JavaScript rendering:

```bash
uprock ai crawl example.com                         # Full-page crawl (default)
uprock ai crawl example.com -m GET                  # Plain HTTP GET (faster, no JS)
uprock ai crawl example.com --country EU            # Crawl from Europe
uprock ai crawl example.com -d desktop              # Desktop device
uprock ai crawl example.com --content               # Inline markdown in response
```

`ai research` searches across multiple engines with deduplication:

```bash
uprock ai research best cloud providers             # Multi-engine search
uprock ai research best restaurants --countries EU  # Search from Europe's perspective
uprock ai research climate data -n 30               # Up to 30 results
```

`ai sweep` tests performance from multiple regions simultaneously:

```bash
uprock ai sweep example.com                         # Default: NA, EU, APAC
uprock ai sweep example.com -r NA,EU                # Specific regions
uprock ai sweep example.com --tries 3 -d desktop    # 3 checks/region, desktop
```

All AI commands output JSON. Pipe to `jq` for filtering. See [SKILL.md](./SKILL.md) for detailed flag reference and usage guidance for LLM agents.

### Other

```bash
uprock version          # Print version and build info
```

## Manual Download

Pre-built binaries for macOS, Linux, and Windows are available on the [Releases](https://github.com/uprockcom/cli/releases) page.

---

© 2026 UpRock. All rights reserved.
