# 5-Minute First Autonomous PR Guide

Get from a fresh Animus installation to your first autonomous pull request in under 5 minutes. This guide provides the fastest path to seeing Animus work autonomously on your codebase.

## Prerequisite Checklist

Before starting, ensure you have the following ready. Each item includes a time estimate for verification:

| Prerequisite | Verify Command | Time |
|--------------|----------------|------|
| **Animus installed** | `animus --version` | ~10s |
| **Git repository** | `git status` | ~5s |
| **AI CLI available** | `claude --version` | ~10s |
| **API key configured** | `claude` (opens interactive) | ~30s |

**Total verification time: ~1 minute**

### Prerequisite Details

1. **Animus Binary**: Download from [releases](https://github.com/launchapp-dev/ao/releases) or [build from source](../getting-started/installation.md).

2. **Git Repository**: Animus works within a Git repository. Create one if needed:
   ```bash
   mkdir my-project && cd my-project
   git init
   git commit --allow-empty -m "Initial commit"
   ```

3. **AI CLI Tool**: Animus requires at least one AI CLI tool. Claude CLI is recommended:
   ```bash
   # Install Claude CLI (macOS/Linux)
   curl -fsSL https://claude.ai/install.sh | sh
   ```

4. **API Authentication**: Ensure your AI CLI is authenticated:
   ```bash
   claude  # First run triggers authentication flow
   ```

---

## Step 1: Initialize Your Project

**⏱️ Time: ~30 seconds**

Initialize Animus in your repository:

```bash
cd /path/to/your/project
animus setup
```

This creates the `.ao/` directory structure with default configuration.

Verify initialization:

```bash
animus pack list
```

You should see bundled packs listed (e.g., `ao.task/standard`).

---

## Step 2: Configure Autonomous Mode

**⏱️ Time: ~20 seconds**

Enable automatic PR creation and merging for fully autonomous operation:

```bash
animus daemon config --set auto_pr=true
animus daemon config --set auto_merge=true
```

View your configuration:

```bash
animus daemon config
```

The key settings for autonomous PRs:

| Setting | Value | Purpose |
|---------|-------|---------|
| `auto_pr` | `true` | Creates PRs after workflow completion |
| `auto_merge` | `true` | Merges PRs after passing checks |
| `auto_run_ready` | `true` | Automatically starts ready tasks |

---

## Step 3: Create Your First Task

**⏱️ Time: ~30 seconds**

Create a simple, well-scoped task for your first autonomous PR:

```bash
animus task create \
  --title "Add contributing guide with setup instructions" \
  --type docs \
  --priority high \
  --description "Create CONTRIBUTING.md with: prerequisites, installation steps, development workflow, and PR guidelines."
```

Note the task ID from the output (e.g., `TASK-001`).

Move the task to ready status so the daemon can pick it up:

```bash
animus task status --id TASK-001 --status ready
```

---

## Step 4: Start Autonomous Mode

**⏱️ Time: ~10 seconds**

Start the daemon in autonomous mode:

```bash
animus daemon start --autonomous
```

The daemon will:
1. Detect ready tasks
2. Dispatch workflows automatically
3. Execute phases (implementation, testing, PR creation)
4. Merge upon success (if `auto_merge=true`)

Verify the daemon is running:

```bash
animus daemon status
```

---

## Step 5: Monitor the Autonomous Flow

**⏱️ Time: ~2 minutes**

Watch your task progress through the workflow:

### Real-Time Output

Stream agent output in real-time:

```bash
animus output tail --task-id TASK-001
```

### Workflow Status

Check workflow execution state:

```bash
animus workflow list
```

Get detailed workflow info:

```bash
animus workflow get --id <workflow-id>
```

### Daemon Events

View daemon activity:

```bash
animus daemon events
```

### Task Status

Track task progression:

```bash
# Watch task status (run in another terminal)
watch -n 5 'ao task get --id TASK-001'
```

Or check periodically:

```bash
animus task stats
```

### What You'll See

The autonomous flow progresses through these phases:

```
1. Implementation  → Agent writes code
2. Push Branch     → Creates feature branch
3. Create PR       → Opens pull request
4. PR Review       → Runs checks, reviews
5. Merge           → Merges to main (if auto_merge)
```

---

## Step 6: View Your Completed PR

**⏱️ Time: ~30 seconds**

Once the workflow completes:

### Check Task Completion

```bash
animus task get --id TASK-001
```

Status should show `done`.

### View PR Details

List recent workflows to find the PR:

```bash
animus workflow list --status completed
```

Get the PR URL from workflow details:

```bash
animus workflow get --id <workflow-id>
```

### View Generated Files

See what the agent created:

```bash
animus output artifacts --execution-id <execution-id>
```

### Check Git History

View the merged commit:

```bash
git log --oneline -5
```

---

## Complete Timeline

| Step | Action | Time |
|------|--------|------|
| Prerequisites | Verify environment | ~1 min |
| Step 1 | Initialize project | ~30s |
| Step 2 | Configure autonomous | ~20s |
| Step 3 | Create task | ~30s |
| Step 4 | Start daemon | ~10s |
| Step 5 | Monitor progress | ~2 min |
| Step 6 | View PR | ~30s |
| **Total** | | **~5 min** |

---

## Troubleshooting

### Daemon Won't Start

```bash
animus doctor --fix
```

### Task Not Picked Up

Check task is in `ready` status:

```bash
animus task get --id TASK-001
```

Verify daemon is running:

```bash
animus daemon status
```

### Workflow Stuck

View workflow state:

```bash
animus workflow list
animus workflow get --id <workflow-id>
```

Check daemon logs:

```bash
animus daemon logs
```

### Agent Errors

Verify AI CLI is working:

```bash
claude --version
```

Check runner health:

```bash
animus runner health
```

---

## Next Steps

- **[Task Management](task-management.md)** -- Learn advanced task operations
- **[Daemon Operations](daemon-operations.md)** -- Deep dive into daemon configuration
- **[Writing Workflows](writing-workflows.md)** -- Create custom workflows
- **[A Typical Day](../getting-started/typical-day.md)** -- Daily Animus usage patterns

---

## Quick Reference Commands

```bash
# Initialize
animus setup

# Configure
animus daemon config --set auto_pr=true
animus daemon config --set auto_merge=true

# Create task
animus task create --title "..." --type docs --priority high
animus task status --id TASK-001 --status ready

# Start
animus daemon start --autonomous

# Monitor
animus output tail --task-id TASK-001
animus workflow list
animus daemon events

# View results
animus task get --id TASK-001
animus workflow list --status completed
```
