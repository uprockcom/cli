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

### Other

```bash
uprock version          # Print version and build info
```

## Manual Download

Pre-built binaries for macOS, Linux, and Windows are available on the [Releases](https://github.com/uprockcom/cli/releases) page.

---

© 2026 UpRock. All rights reserved.
