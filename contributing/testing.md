# Testing Guide

## Running Tests

Run all workspace tests:

```bash
cargo test --workspace
```

Run tests for a specific crate:

```bash
cargo test -p protocol
cargo test -p orchestrator-core
cargo test -p orchestrator-cli
```

Run a single test by name:

```bash
cargo test -p orchestrator-cli -- rust_only_dependency_policy
```

## Unit Tests

Unit tests live in `#[cfg(test)]` modules within the source files they test. They use the `InMemoryServiceHub` for isolated testing without filesystem access.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn task_creation_sets_backlog_status() {
        let hub = InMemoryServiceHub::new();
        let task = hub.tasks().create(TaskCreateInput {
            title: "Test task".into(),
            ..Default::default()
        }).await.unwrap();
        assert_eq!(task.status, TaskStatus::Backlog);
    }
}
```

The `protocol` crate provides test utilities behind the `test-utils` feature flag, including `EnvVarGuard` for safely setting/unsetting environment variables in tests.

## Integration Tests

Integration tests live in `crates/orchestrator-cli/tests/` and exercise the CLI as a subprocess:

| Test File | What It Tests |
|-----------|--------------|
| `cli_smoke.rs` | Basic CLI invocation and help output |
| `cli_e2e.rs` | End-to-end task and workflow operations |
| `cli_json_contract.rs` | JSON envelope schema stability |
| `workflow_state_machine_e2e.rs` | Workflow state machine transitions |
| `setup_doctor_e2e.rs` | Setup and doctor command behavior |
| `cli_skill_lifecycle.rs` | Skill registration and execution |
| `session_continuation_e2e.rs` | Session continuation across CLI invocations |
| `rust_only_dependency_policy.rs` | Dependency policy enforcement |

## Test Harness

The `CliHarness` (defined in `crates/orchestrator-cli/tests/support/test_harness.rs`) provides a convenient wrapper for testing CLI commands:

- Creates a temporary project directory for isolation
- Runs `ao` commands as subprocesses
- Parses JSON output and validates envelope contracts
- Provides `run_json_ok()` for commands expected to succeed and `run_json_err()` for expected failures

Example usage:

```rust
let harness = CliHarness::new()?;
let result = harness.run_json_ok(&["task", "list"])?;
assert_success_envelope(&result);
```

The harness validates the `ao.cli.v1` envelope contract:
- `schema` field matches `CLI_SCHEMA_ID`
- `ok` is `true` for success, `false` for errors
- Success envelopes include `data`
- Error envelopes include `error.code`, `error.exit_code`, and `error.message`

## InMemoryServiceHub for Isolated Tests

The `InMemoryServiceHub` stores all state in memory, implementing the same `ServiceHub` trait as the production `FileServiceHub`. This means unit tests exercise real business logic without touching the filesystem:

- No temp directory cleanup needed
- Tests run in parallel without interference
- Fast execution (no I/O overhead)

## E2E Test Improvements

The E2E test suite was significantly expanded and stabilised across v39–v45 to cover cloud-integration paths that previously required manual verification.

### Cloud Trigger E2E Tests

`crates/orchestrator-cli/tests/cloud_trigger_e2e.rs` covers the full GitHub webhook → trigger engine → dispatch pipeline against a local stub server that mimics the GitHub App webhook receiver:

```rust
// Example: verifies that a pull_request.opened event fires the configured trigger
#[tokio::test]
async fn github_trigger_fires_on_pr_opened() {
    let harness = CloudHarness::new()?;
    harness.send_github_event("pull_request", "opened", fixture!("pr_opened.json")).await?;
    let dispatches = harness.wait_for_dispatches(1).await?;
    assert_eq!(dispatches[0].workflow_ref, "ao.task/standard");
}
```

The `CloudHarness` fixture (in `tests/support/cloud_harness.rs`) starts a local HTTP stub that validates HMAC signatures and injects events into the trigger engine without requiring a real GitHub App installation.

### Filter Expression Tests

`crates/orchestrator-cli/tests/trigger_filter_e2e.rs` exercises the `filter.expr` conditional evaluation added in v40. Tests cover all supported operators and edge cases including missing payload keys, nested dot-path access, and boolean short-circuit evaluation.

### Check Run E2E Tests

`crates/orchestrator-cli/tests/github_checks_e2e.rs` verifies that the check run lifecycle — `in_progress` on dispatch start, per-phase updates, and `completed` on workflow finish — produces the correct sequence of API calls to the GitHub Checks stub.

### Isolation Improvements

The harness now creates a fresh temporary directory for every test case, even within the same test file. Previously, some tests shared a project directory and could interfere with each other if run with `--test-threads > 1`. The `CliHarness::new()` and `CloudHarness::new()` constructors both guarantee isolation.

### Retry Logic in Assertions

`wait_for_dispatches` and similar assertion helpers use an exponential-backoff polling loop instead of a fixed `sleep`. This eliminates flaky failures on slow CI runners while keeping the fast path fast on developer machines.

### Running Cloud E2E Tests

Cloud E2E tests are gated behind the `cloud-e2e` feature flag because they require the stub server binary to be compiled:

```bash
cargo test --features cloud-e2e -p orchestrator-cli -- cloud
```

On CI, `web-ui-ci.yml` runs these tests against a deployed preview environment using the `AO_CLOUD_E2E_BASE_URL` environment variable.

---

## CI Workflows

The project uses several GitHub Actions workflows:

| Workflow | File | Purpose |
|----------|------|---------|
| Rust Workspace CI | `rust-workspace-ci.yml` | Build, test, and lint all crates |
| Dependency Policy | `rust-only-dependency-policy.yml` | Verify no prohibited dependencies |
| Web UI CI | `web-ui-ci.yml` | Build and test web UI assets |
| Release | `release.yml` | Build release binaries and publish |
