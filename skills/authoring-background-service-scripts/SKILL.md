---
name: authoring-background-service-scripts
description: Use when building scripts/setup.sh CLIs that manage long-running services across dev/prod, with runtime files under .run/ and port-based liveness checks alongside PID.
---

# Authoring background service scripts

## Overview

Turn ad‑hoc “run this in another terminal” workflows into a single `scripts/setup.sh` CLI that owns process lifecycle, logs, and health signals. Keep runtime state under the repository's `.run/` directory; **start**, **stop**, and **restart** must validate both the **PID file** and **listening port** before treating a service as running, stopped, or safe to replace.

## When to use

- Script wraps a daemon or dev server that should **outlive the launching shell**.
- Operators need predictable subcommands: at minimum **start**, **stop**, **restart**; usually also **install** (deps/binary) and **update** (pull/build/restart).
- Interactive UX, when provided, should use **gum**.
- You must avoid zombie processes and false “stopped” states when PID files are stale.

## When not to use

- One-off cron jobs or foreground-only tools with no daemon semantics.
- Production orchestration (systemd/Kubernetes) — different contracts.
- Requirements that are fully enforceable by a Makefile or package manager alone.

## Gum bootstrap

- **All** interactive UX goes through **gum** (choose, confirm, input, spin, style).
- If `gum` is missing, install via:

```bash
curl -LsSf https://raw.githubusercontent.com/farfarfun/fundeploy/master/scripts/tools/utils/setup.sh | bash -s -- gum --force
```

Re-check `command -v gum` after install; fail fast with a clear message if still unavailable.

## Required CLI surface

Ship `scripts/setup.sh` as the single service-management entrypoint. Non-interactive arguments put the action first and the environment last: `scripts/setup.sh <action> <env>` for one service, or `scripts/setup.sh <action> <service> <env>` for multiple services. Use `dev` and `prod` as explicit environments; commands such as `status` that do not select an environment may omit it.

| Command   | Purpose |
|-----------|---------|
| `run`     | Run the selected environment in the foreground. |
| `start`   | Launch in the background after PID and port preflight checks. |
| `stop`    | Stop the owned process and verify that its port is released. |
| `restart` | `stop` then `start`, reusing the same checks. |
| `status`  | Report PID validity, process identity, and port ownership. |
| `install` | Install the selected environment's dependencies or formal package. |

`start prod` must run an already installed formal package. It must fail with a non-zero status when that package is absent, rather than building from source or falling back to a development command.

## Data layout

Under the repository root:

- Root: `.run/` (or `.run/<service>/` when managing multiple services).
- **`pid` file**: single line, numeric PID written atomically after successful background start.
- **Logs**: files below `.run/`, with a documented rotation or truncation policy.
- **Lock**: a lock below `.run/` serializes lifecycle operations and prevents concurrent starts.
- Optional: `port` file or config fragment if port is dynamic but must be tracked for checks.

Resolve the repository root from `scripts/setup.sh` itself, not from the caller's current directory. Do not scatter PID, lock, or log files across `/tmp`, the user's home directory, or the source tree outside `.run/`.

## Stop and restart rules

1. **Start preflight**: under the lifecycle lock, reject a live owned PID, a matching listener, or a port owned by another process. Distinguish a stale PID file from the expected service by checking process identity; never signal an unrelated reused PID.
2. **Launch and confirm**: start the service, confirm that the expected process remains alive and owns the configured port, then atomically publish its PID. On failure, clean partial state and exit non-zero.
3. **Stop**: read the PID from `.run/`, confirm it belongs to the expected service, signal `TERM`, wait briefly, and use `KILL` only if the contract permits it.
4. **Port check**: verify that the configured port is no longer listening. If it remains open, report the owner and exit non-zero.
5. **Cleanup**: remove PID and lock state only after the process is gone and the port is free.
6. **Restart**: do not start a second instance unless stop completes successfully.

## Quick reference

- **Entry point**: use `scripts/setup.sh <action> <env>` or `scripts/setup.sh <action> <service> <env>`.
- **Start**: detach the process from the controlling terminal and redirect logs below `.run/`; publish the PID only after successful readiness checks.
- **Idempotence**: `start` should refuse or offer “already running” when PID+port indicate health.
- **Observability**: log rotation or truncation policy; document where to `tail`.
- **Failures**: non‑zero exit; state *what* failed (missing package, port busy, stale PID, or readiness timeout).

## Common mistakes

- Writing PID before the child process is definitely running (race → wrong PID).
- Trusting PID alone when wrappers spawn children that hold the port.
- Using gum only in `install` but falling back to naked `echo/read` elsewhere (inconsistent UX).
- Leaving logs only on stdout of the parent shell (lost on disconnect).

## Verification

Before merging or publishing the CLI:

1. **Cold machine**: no `gum` → installer path runs → `gum` works.
2. **Syntax and foreground**: `bash -n scripts/setup.sh`; `scripts/setup.sh run dev` starts in the foreground and propagates failure.
3. **`start dev`**: service runs detached; `.run/` contains the expected state; PID matches the process; port responds; logs append.
4. **Duplicate/stale state**: a second start is rejected; a stale or reused PID never causes an unrelated process to be signaled.
5. **`stop dev`**: process ends; port is free; PID state is cleared; repeated stop has documented idempotent behavior.
6. **Production**: `start prod` uses only the installed formal package and fails non-zero when it is missing.
7. **`restart`/`install`**: never create two listeners; simulate dependency or network failure and verify a non-zero exit with no partial runtime state.

**Org note:** Treat major edits like code: run at least one **baseline scenario without** this doc and one **with** it when changing enforcement-heavy sections (same discipline as `superpowers:writing-skills`).
