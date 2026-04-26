# Daemon Crash Recovery Runbook

When the ao-product daemon stops unexpectedly, this runbook walks you through confirming the crash, identifying what state can be trusted, and safely restarting work.

## Quick Reference

```
Confirm crash  → animus-v2 daemon status --json
Check logs     → animus-v2 daemon logs
Clean stale files → rm -f .ao/run/runner.sock .ao/run/runner.pid
Restart        → animus-v2 daemon start --autonomous
Verify healthy → animus-v2 daemon status --json
```

---

## Step 1: Confirm the Daemon Is Actually Crashed

Before doing anything else, get the authoritative status from the v2 probe:

```bash
animus-v2 daemon status --json
```

A crashed or stopped daemon returns:

```json
{"command":"daemon.status","data":{"http_port":3001,"status":"not_running"},"meta":{},"schema":"ao.cli.v1"}
```

A healthy daemon returns `"status":"running"` with a `pid` field.

> **Use `animus-v2 daemon status`, not `ao daemon status --project-root`.**
> The v1 probe misreports v2 daemon state and should not be used to diagnose
> v2 daemon health.

---

## Step 2: Distinguish Trusted from Untrusted State Surfaces

Not every signal that says "crashed" is reliable. Before acting, evaluate the source.

### Trusted surfaces

| Surface | Why trustworthy |
|---|---|
| `animus-v2 daemon status --json` | Direct probe against the v2 process socket |
| `animus-v2 daemon logs` / `.ao/daemon.log` | Structured, written by the daemon itself |
| `.ao/tasks.json`, `.ao/queue.json` on disk | Authoritative persistent store |
| `animus task list` / `animus workflow list` | CLI reads from the persistent store via daemon |

### Untrusted or unreliable surfaces

| Surface | Known issue |
|---|---|
| Scheduled Codex sweep reports | Codex sandbox runs with `HOME=/tmp/codex-home`, an isolated SQLite store that does not reflect live daemon state. A Codex report saying "daemon crashed" or "queue empty" may be a false positive — the live daemon may be healthy. |
| `ao daemon status --project-root` (v1) | Misreports v2 daemon state; do not use. |
| Dashboard cache | Can lag after a crash-and-restart; reload the page and re-check. |
| `HOME=/tmp/codex-home` queue output | Codex enqueues tasks into the fallback home. Those entries are not visible to the live daemon. Cross-check by reading `.ao/queue.json` directly. |

If a sweep report says `FAIL_CLOSED` (daemon crashed), confirm with `animus-v2 daemon status --json` before treating the report as ground truth. If the live probe returns `running`, the sweep report was a false positive — do not restart or hold back dispatches.

---

## Step 3: Read the Logs

Once you've confirmed the daemon is actually stopped, read the logs to find the exit reason before restarting:

```bash
# Via CLI (streams .ao/daemon.log)
animus-v2 daemon logs

# Or read directly for the last 200 lines
tail -200 .ao/daemon.log
```

Look for lines with `daemon_shutdown`, `panic`, or `error` near the end of the file. Common crash causes:

| Log pattern | Likely cause |
|---|---|
| `Not logged in · Please run /login` | Claude Code session expired. **Do not restart** until you refresh the login — new dispatches will fail immediately with the same error. |
| `Operation not permitted (os error 1)` | macOS codesign missing or sandbox restriction. See [below](#macos-codesign-fix). |
| `SQLite` / `temp-file` errors | Sandbox environment issue. Use `HOME=/tmp/codex-home` for direct AO commands. |
| `SIGKILL` at 0s uptime (recurring) | Binary needs codesign or a full source rebuild. |

---

## Step 4: Clean Stale Runtime Files

A crash can leave behind socket and PID files that block a clean restart:

```bash
ls -la .ao/run/
```

If you see `runner.sock` or `runner.pid` from a previous run, remove them:

```bash
rm -f .ao/run/runner.sock .ao/run/runner.pid
```

Also check for orphaned runner processes:

```bash
animus-v2 runner orphans detect
animus-v2 runner orphans cleanup
```

---

## Step 5: Restart the Daemon

Restart with the supervisor enabled so the daemon recovers from future transient failures automatically:

```bash
animus-v2 daemon start --autonomous
```

`--autonomous` enables:
- Automatic task scheduling
- Supervisor restart loop with exponential backoff (1 s → 2 s → 4 s → … → 60 s, max 5 restarts per 5-minute window)

Confirm the daemon came up:

```bash
animus-v2 daemon status --json
```

Expected response includes `"status":"running"` and a `pid`.

---

## Step 6: Audit In-Flight Work

A crash mid-workflow leaves some tasks stuck in `in-progress`. The daemon will not re-pick them automatically because it cannot safely distinguish "still running" from "abandoned".

Find abandoned tasks:

```bash
animus task list --status in-progress
```

For each task that should be re-queued:

```bash
animus workflow cancel --id WF-XXX          # cancel the orphaned workflow
animus task status --id TASK-XXX --status ready  # re-queue the task
```

Find any workflows that were running when the daemon crashed:

```bash
animus workflow list --status running
```

Cancel and re-queue those as well. The next sweep will pick them up.

---

## macOS Codesign Fix

If the daemon binary is killed immediately after launch (SIGKILL at 0s uptime), macOS Gatekeeper is rejecting the unsigned binary:

```bash
codesign -f -s - /path/to/animus-v2
```

If the SIGKILL persists after codesigning, the binary may be corrupt. Rebuild from source:

```bash
cd /path/to/ao-v2-source
cargo build --release -p orchestrator-cli
```

---

## Session Expiry — Do Not Restart Yet

If implement-ts or ao-cloud-deploy tasks all fail with:

```
Not logged in · Please run /login
```

and the duration is under 1 second, the daemon's Claude Code session has expired. Restarting the daemon process will not help — it will start up healthy but every dispatch will fail immediately.

**Resolution**: An operator must run `/login` in the Claude Code session that the daemon uses, then verify dispatches succeed before resuming normal sweep operations.

---

## Blocked-State Artifact

If you cannot restart the daemon (e.g. you are running inside the Codex sandbox, which cannot restart the process), record the current state so the next operator session can pick up cleanly:

1. List all `in-progress` tasks and `running` workflows.
2. Note any merge-ready PRs that cannot be merged while the daemon is down.
3. Write a blocked-state summary to `reports/blocked-<date>.md` rather than checking in a false GREEN sweep.

The Codex sandbox cannot restart the ao-product daemon. Recovery always requires an unsandboxed operator action.

---

## Related Guides

- [Daemon Operations](daemon-operations.md) — starting, stopping, and monitoring the daemon under normal conditions
- [Troubleshooting](troubleshooting.md) — runner health, workflow failures, and task state pitfalls
