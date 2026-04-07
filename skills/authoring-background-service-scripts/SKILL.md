---
name: authoring-background-service-scripts
description: Use when building shell CLIs that manage long-running local services (start/stop/restart/update/install), interactive prompts via Charm gum, PID files under ~/.server/<name>, and port-based liveness checks alongside PID.
---

# Authoring background service scripts

## Overview

Turn ad‑hoc “run this in another terminal” workflows into a **small gum-driven CLI** that owns process lifecycle, logs, and health signals. One service → one directory under `~/.server/<server_name>` for PID and logs; **stop** and **restart** must consult **PID file** and **listening port** before treating a process as stopped or safe to replace.

## When to use

- Script wraps a daemon or dev server that should **outlive the launching shell**.
- Operators need predictable subcommands: at minimum **start**, **stop**, **restart**; usually also **install** (deps/binary) and **update** (pull/build/restart).
- UX should be **interactive menus/prompts** in the terminal (gum).
- You must avoid zombie processes and false “stopped” states when PID files are stale.

## When not to use

- One-off cron jobs or foreground-only tools with no daemon semantics.
- Production orchestration (systemd/Kubernetes) — different contracts.
- Requirements that are fully enforceable by a Makefile or package manager alone.

## Gum bootstrap

- **All** interactive UX goes through **gum** (choose, confirm, input, spin, style).
- If `gum` is missing, install via:

```bash
curl -LsSf https://raw.githubusercontent.com/farfarfun/nltdeploy/master/scripts/05-utils/utils-setup.sh | bash -s -- gum --force
```

Re-check `command -v gum` after install; fail fast with a clear message if still unavailable.

## Required CLI surface

Ship a single entrypoint (e.g. `mysvc` or `scripts/server`) that implements at least:

| Command   | Purpose |
|-----------|---------|
| `start`   | Launch service **in background**; write PID; tee or append logs. |
| `stop`    | Stop using **PID file**; confirm with **port** check when configured. |
| `restart` | `stop` then `start` (reuse the same rules). |
| `update`  | Refresh artifacts (git pull/build); typically ends with `restart`. |
| `install` | First-time setup (deps, dirs, defaults) — may call `start` after. |

Add `status`, `logs`, or `doctor` only if they reduce support load; document them in **Quick reference**.

## Data layout

Per service `server_name`:

- Root: `~/.server/<server_name>/`
- **`pid` file**: single line, numeric PID written atomically after successful background start.
- **Logs**: e.g. `logs/stdout.log`, `logs/stderr.log` or a single rotating file; include timestamps.
- Optional: `port` file or config fragment if port is dynamic but must be tracked for checks.

Never scatter PID/logs across `/tmp` without namespacing — the `~/.server/<server_name>` contract is the stable contract.

## Stop and restart rules

1. **Read PID** from `~/.server/<server_name>/pid` (if missing, treat as not running; still run port check if port known).
2. **Signal process** (`TERM`, then short wait, then `KILL` if needed).
3. **Port check**: if the service is known to bind a TCP port, verify nothing listens after stop (e.g. `lsof`/platform-specific). If port still open, surface error via gum and exit non‑zero.
4. **Cleanup**: remove or invalidate PID file only after success criteria (process gone **and** port free when applicable).
5. **restart**: must not start a second instance until stop succeeds.

## Quick reference

- **Start**: `nohup`/`setsid` or background `&` + disown — ensure controlling TTY does not kill the service; redirect logs under `~/.server/.../logs/`.
- **Idempotence**: `start` should refuse or offer “already running” when PID+port indicate health.
- **Observability**: log rotation or truncation policy; document where to `tail`.
- **Failures**: non‑zero exit; gum message states *what* failed (missing binary, port busy, stale PID).

## Common mistakes

- Writing PID before the child process is definitely running (race → wrong PID).
- Trusting PID alone when wrappers spawn children that hold the port.
- Using gum only in `install` but falling back to naked `echo/read` elsewhere (inconsistent UX).
- Leaving logs only on stdout of the parent shell (lost on disconnect).

## Verification

Before merging or publishing the CLI:

1. **Cold machine**: no `gum` → installer path runs → `gum` works.
2. **`start`**: service runs detached; PID file matches live process; port responds (if applicable); logs append.
3. **`stop`**: process ends; port free; PID file cleared or marked inactive.
4. **`restart`**: never two listeners on the same port; logs show cycle.
5. **`update`/`install`**: simulate failure (bad network) — script exits non‑zero, no partial PID state, gum shows cause.

**Org note:** Treat major edits like code: run at least one **baseline scenario without** this doc and one **with** it when changing enforcement-heavy sections (same discipline as `superpowers:writing-skills`).
