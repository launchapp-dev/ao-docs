# ao cloud — CLI Reference

Complete reference for all `ao cloud` commands. Cloud management provides authentication, project deployment, remote daemon lifecycle control, and log streaming for AO projects hosted in the AO cloud.

For global flags (`--json`, `--project-root`) see [Global Flags](global-flags.md). For exit codes see [Exit Codes](exit-codes.md).

---

## Command Tree

```
ao cloud
├── login                     Authenticate with AO cloud
├── push                      Push project to AO cloud
├── start                     Start the cloud-hosted daemon
├── stop                      Stop the cloud-hosted daemon
├── status                    Show cloud deployment status
└── logs                      Stream or read cloud daemon logs
```

---

## Top-Level Commands

### `ao cloud login`

Authenticate with the AO cloud service. Opens a browser-based OAuth flow by default; use `--token` to supply a personal access token directly (suitable for CI environments).

```bash
ao cloud login
ao cloud login --token <TOKEN>
ao cloud login --json
```

**Flags:**

| Flag | Description |
|---|---|
| `--token <TOKEN>` | Personal access token — skips browser flow |
| `--org <ORG>` | Target organisation slug (required if account belongs to multiple orgs) |

**Output fields:**

| Field | Description |
|---|---|
| `authenticated` | `true` when login succeeded |
| `user` | Authenticated user email |
| `org` | Organisation the session is scoped to |
| `token_expires_at` | ISO 8601 expiry timestamp (`null` for non-expiring PATs) |

```json
{
  "authenticated": true,
  "user": "sam@example.com",
  "org": "acme",
  "token_expires_at": "2026-07-01T00:00:00Z"
}
```

> Credentials are stored in `~/.ao/cloud/credentials.json`. Re-run `ao cloud login` to refresh an expired session.

---

### `ao cloud push`

Push the current project to AO cloud. Packages project configuration, workflow definitions, and persona files, then uploads them to the cloud. Does **not** upload source code — the cloud daemon operates on the project metadata only.

```bash
ao cloud push
ao cloud push --env staging
ao cloud push --dry-run
ao cloud push --json
```

**Flags:**

| Flag | Description |
|---|---|
| `--env <ENV>` | Target environment (`production`, `staging`, `preview`). Default: `production` |
| `--dry-run` | Validate and package without uploading |
| `--force` | Skip confirmation when overwriting an existing deployment |

**Output fields:**

| Field | Description |
|---|---|
| `deployment_id` | Unique identifier for this deployment |
| `env` | Target environment |
| `pushed_at` | ISO 8601 timestamp |
| `artifacts` | Number of uploaded artifacts (workflow definitions, personas, packs) |
| `url` | Cloud dashboard URL for this deployment |

```json
{
  "deployment_id": "dep_01HXYZ",
  "env": "production",
  "pushed_at": "2026-04-01T10:00:00Z",
  "artifacts": 12,
  "url": "https://app.ao.dev/acme/my-project/deployments/dep_01HXYZ"
}
```

---

### `ao cloud start`

Start the cloud-hosted AO daemon for the current project. If the daemon is already running the command is a no-op (exit 0).

```bash
ao cloud start
ao cloud start --env staging
ao cloud start --wait
ao cloud start --json
```

**Flags:**

| Flag | Description |
|---|---|
| `--env <ENV>` | Target environment. Default: `production` |
| `--wait` | Block until the daemon reports `running` status (polls every 2 s, 60 s timeout) |

**Output fields:**

| Field | Description |
|---|---|
| `daemon_id` | Cloud daemon identifier |
| `status` | `starting` or `running` |
| `env` | Environment |
| `started_at` | ISO 8601 timestamp |

```json
{
  "daemon_id": "daemon_ABCD",
  "status": "starting",
  "env": "production",
  "started_at": "2026-04-01T10:05:00Z"
}
```

---

### `ao cloud stop`

Stop the cloud-hosted AO daemon. In-progress agent runs are allowed to finish (graceful drain) unless `--force` is passed.

```bash
ao cloud stop
ao cloud stop --env staging
ao cloud stop --force
ao cloud stop --json
```

**Flags:**

| Flag | Description |
|---|---|
| `--env <ENV>` | Target environment. Default: `production` |
| `--force` | Terminate immediately without draining active agents |
| `--wait` | Block until the daemon reports `stopped` (polls every 2 s, 120 s timeout) |

**Output fields:**

| Field | Description |
|---|---|
| `daemon_id` | Cloud daemon identifier |
| `status` | `draining`, `stopping`, or `stopped` |
| `stopped_at` | ISO 8601 timestamp (`null` if still draining) |

```json
{
  "daemon_id": "daemon_ABCD",
  "status": "stopping",
  "stopped_at": null
}
```

---

### `ao cloud status`

Show the current status of the cloud deployment and daemon.

```bash
ao cloud status
ao cloud status --env staging
ao cloud status --json
```

**Flags:**

| Flag | Description |
|---|---|
| `--env <ENV>` | Target environment. Default: `production` |

**Output fields:**

| Field | Description |
|---|---|
| `deployment_id` | Most recent deployment identifier |
| `env` | Environment |
| `daemon_status` | `running`, `stopped`, `starting`, `stopping`, `error` |
| `pushed_at` | Timestamp of last `ao cloud push` |
| `daemon_started_at` | Timestamp daemon last transitioned to `running` |
| `active_agents` | Number of agent runs currently executing |
| `queue_depth` | Dispatches waiting in the cloud queue |
| `region` | Cloud region the daemon is hosted in |
| `url` | Cloud dashboard URL |

```json
{
  "deployment_id": "dep_01HXYZ",
  "env": "production",
  "daemon_status": "running",
  "pushed_at": "2026-04-01T10:00:00Z",
  "daemon_started_at": "2026-04-01T10:05:12Z",
  "active_agents": 2,
  "queue_depth": 0,
  "region": "us-east-1",
  "url": "https://app.ao.dev/acme/my-project"
}
```

---

### `ao cloud logs`

Stream live logs from the cloud daemon, or read recent log history.

```bash
ao cloud logs
ao cloud logs --tail 100
ao cloud logs --follow
ao cloud logs --since 1h
ao cloud logs --level error
ao cloud logs --json
```

**Flags:**

| Flag | Description |
|---|---|
| `--env <ENV>` | Target environment. Default: `production` |
| `--follow` | Stream logs in real time (like `tail -f`). Press `Ctrl-C` to stop. |
| `--tail <N>` | Number of recent log lines to show before streaming (default: `50`, max: `1000`) |
| `--since <DURATION>` | Fetch logs from the past duration, e.g. `30m`, `2h`, `1d` |
| `--level <LEVEL>` | Filter by log level: `debug`, `info`, `warn`, `error` |
| `--run-id <RUN_ID>` | Scope output to a single agent run |

**Log line fields (with `--json`):**

| Field | Description |
|---|---|
| `ts` | ISO 8601 timestamp |
| `level` | `debug`, `info`, `warn`, `error` |
| `msg` | Log message |
| `run_id` | Agent run ID if log originated from an agent |
| `component` | Internal component that emitted the log |

```json
{ "ts": "2026-04-01T10:06:00Z", "level": "info",  "msg": "Daemon ready", "run_id": null, "component": "daemon" }
{ "ts": "2026-04-01T10:06:15Z", "level": "info",  "msg": "Agent started", "run_id": "run_1234", "component": "scheduler" }
{ "ts": "2026-04-01T10:07:02Z", "level": "error", "msg": "Model timeout",  "run_id": "run_1234", "component": "runner" }
```

---

## Related

- [Cloud Deployment Guide](../../guides/cloud-deployment.md) — end-to-end workflow: login → push → start → monitor
- [Daemon Operations](../../guides/daemon-operations.md) — local daemon equivalent
- [Fleet Management](../../guides/fleet-management.md) — multi-node distributed deployments
- [Global Flags](global-flags.md)
- [Exit Codes](exit-codes.md)
