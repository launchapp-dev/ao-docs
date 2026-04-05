# Cloud Daemon Management

The AO Cloud dashboard provides a dedicated interface for managing cloud-hosted daemon instances — starting, stopping, restarting, scaling, and configuring auto-restart policies — without leaving the browser or using the CLI.

This guide covers the daemon management features shipped in v42. For CLI-based daemon control see [Cloud Deployment](cloud-deployment.md) and [Daemon Operations](daemon-operations.md).

---

## Accessing Daemon Management

Navigate to any project in the dashboard and click the **Daemon** tab. This tab is available to members with **Member** role or above.

The Daemon tab shows a summary panel at the top and a tabbed detail area below.

### Summary Panel

| Field | Description |
|---|---|
| **Status** | `running`, `stopped`, `starting`, `stopping`, `error`, or `draining` |
| **Uptime** | Time since the last successful start, or `—` if stopped |
| **Instance** | Instance tier name (e.g. `starter-1x`, `pro-2x`) |
| **Region** | Cloud region hosting this instance (e.g. `us-east-1`) |
| **Active runs** | Count of agent runs currently in progress |
| **Queue depth** | Number of dispatches waiting to be picked up |
| **Last started** | Absolute timestamp of the most recent start |
| **Last stopped** | Absolute timestamp of the most recent stop |

---

## Starting and Stopping

### Starting the Daemon

Click **Start** in the summary panel or from the project card on the Projects overview page. A confirmation modal shows the instance tier and estimated start time (typically under 10 seconds for warm instances).

Equivalent CLI command:

```bash
ao cloud start
```

### Stopping the Daemon

Click **Stop**. A modal presents two stop modes:

| Mode | Behaviour |
|---|---|
| **Graceful** | Daemon finishes all in-progress phases before shutting down; new dispatches queue but do not start |
| **Immediate** | Daemon stops at once; in-progress runs are interrupted and placed back in the queue |

The graceful stop mode displays a progress indicator and lists active runs as they complete. The daemon transitions through `draining` status before reaching `stopped`.

Equivalent CLI commands:

```bash
ao cloud stop            # graceful
ao cloud stop --force    # immediate
```

### Restarting

Click **Restart** to perform a graceful stop followed by an immediate start. Use this after updating environment variables or configuration that requires a daemon restart to take effect.

Restarting while runs are active puts the daemon into `draining` status until all active runs finish, then starts a fresh instance.

---

## Auto-Restart Policies

Auto-restart policies keep the daemon running without manual intervention after unexpected failures.

### Configuring a Policy

1. Open the **Daemon** tab for your project.
2. Click **Edit** next to **Auto-restart policy**.
3. Choose a policy preset or configure a custom policy.

**Available presets:**

| Preset | Behaviour |
|---|---|
| **Off** | No automatic restart; daemon stays stopped after any failure |
| **On failure** | Restart immediately after an unexpected exit (exit code ≠ 0) |
| **Always** | Restart after any stop — expected or unexpected |
| **Custom** | Full control over restart conditions, delay, and retry limits |

### Custom Policy Fields

| Field | Type | Description |
|---|---|---|
| `condition` | string | `on_failure`, `always`, or `never` |
| `initial_delay_seconds` | integer | Seconds to wait before the first restart (default: `5`) |
| `backoff_multiplier` | float | Multiply the delay by this factor on each subsequent attempt (default: `2.0`) |
| `max_delay_seconds` | integer | Cap the delay at this value (default: `300`) |
| `max_attempts` | integer | Stop attempting after this many consecutive failures (default: `10`; `0` for unlimited) |
| `reset_after_seconds` | integer | Reset the failure counter after the daemon runs stably for this many seconds (default: `3600`) |

**Example custom policy:**

```json
{
  "condition": "on_failure",
  "initial_delay_seconds": 10,
  "backoff_multiplier": 2.0,
  "max_delay_seconds": 600,
  "max_attempts": 5,
  "reset_after_seconds": 1800
}
```

With this policy, the first restart attempt waits 10 seconds, the second waits 20 seconds, the third 40 seconds, and so on up to 600 seconds. After five consecutive failures, the policy stops retrying and sends a [notification](cloud-dashboard.md#notifications) to the configured channels.

---

## Instance Sizing

Instance size controls the CPU and memory allocated to the daemon process. Larger instances support more concurrent agent runs and higher queue throughput.

### Available Tiers

| Tier | vCPU | Memory | Max concurrent runs | Plans |
|---|---|---|---|---|
| `starter-1x` | 0.5 | 512 MB | 2 | Starter and above |
| `pro-2x` | 1 | 1 GB | 5 | Pro and above |
| `team-4x` | 2 | 2 GB | 12 | Team and above |
| `team-8x` | 4 | 4 GB | 24 | Team and above |
| `enterprise-custom` | Custom | Custom | Custom | Enterprise |

### Changing Instance Tier

1. Stop the daemon (graceful stop recommended).
2. In the **Daemon** tab, click **Edit** next to **Instance**.
3. Select the new tier.
4. Click **Save and Start**. The daemon starts on the new tier immediately.

Resizing is billed at the new rate from the moment the daemon restarts on the new tier. Partial-hour billing applies. See [Cloud Billing](cloud-billing.md) for per-tier pricing.

---

## Run Concurrency

The daemon limits the number of simultaneously executing agent runs. Runs that exceed the concurrency limit wait in the dispatch queue.

### Viewing the Queue

The **Queue** sub-tab in the Daemon tab shows all pending dispatches:

| Column | Description |
|---|---|
| Dispatch ID | Unique identifier |
| Subject | Task or workflow subject title |
| Workflow | Workflow definition name |
| Priority | `high`, `normal`, or `low` |
| Queued at | Time the dispatch was created |
| Wait time | Duration since the dispatch was queued |

Click any dispatch to see its full subject and variable payload.

### Adjusting the Concurrency Limit

The concurrency limit defaults to the maximum for the selected instance tier. To set a lower limit:

```bash
ao cloud project config --set daemon.max_concurrent_runs=3
```

Or update it from the **Daemon → Settings** sub-tab in the dashboard. The new limit takes effect immediately without restarting the daemon.

---

## Daemon Logs

The **Logs** sub-tab in the Daemon tab streams daemon log output in near-real time (5-second polling interval). This view shows only daemon-process log lines, not agent run output.

Use the **Level** filter to show `info`, `warn`, or `error` lines only. The log viewer displays the last 500 lines; click **Open in Logs** to switch to the full [Logs](cloud-dashboard.md#logs) search page with full-text search and export.

---

## Drain Mode

Drain mode pauses new dispatch pickup without stopping the daemon process. In-progress runs continue; new dispatches queue up. Use drain mode to perform maintenance (such as pushing a new deployment) without interrupting active work.

### Entering Drain Mode

Click **Drain** in the summary panel, or:

```bash
ao cloud pause
```

The daemon status changes to `draining`. The status badge turns yellow. New dispatches are accepted and queued but not started.

### Exiting Drain Mode

Click **Resume**, or:

```bash
ao cloud resume
```

The daemon immediately resumes picking up queued dispatches.

---

## Daemon Health Timeline

The **Uptime** sub-tab shows a 30-day history of daemon status as a colour-coded bar chart:

| Colour | State |
|---|---|
| Green | Running |
| Yellow | Draining or starting |
| Red | Error or unexpected stop |
| Grey | Stopped (intentional) |

Hover over any bar segment to see the start and end timestamp of that state and (for red segments) the exit reason. Click a segment to jump to the corresponding daemon log entries for that time window.

---

## Notifications for Daemon Events

Configure notification delivery for daemon status changes at **Settings → Notifications**. Daemon-related events:

| Event | Description |
|---|---|
| `daemon.started` | Daemon successfully started |
| `daemon.stopped` | Daemon stopped after an explicit `ao cloud stop` or dashboard stop |
| `daemon.crashed` | Daemon exited with a non-zero exit code |
| `daemon.restart_limit_reached` | Auto-restart gave up after `max_attempts` consecutive failures |
| `daemon.queue_depth_threshold` | Queue depth exceeded the configured warning threshold |

For notification delivery configuration see [Cloud Dashboard → Notifications](cloud-dashboard.md#notifications).

---

## Related

- [Cloud Deployment](cloud-deployment.md) — pushing projects and CLI-driven cloud management
- [Cloud Dashboard](cloud-dashboard.md) — full dashboard navigation reference
- [Cloud Billing](cloud-billing.md) — per-instance-tier pricing and usage metering
- [Daemon Operations](daemon-operations.md) — local daemon management with the CLI
