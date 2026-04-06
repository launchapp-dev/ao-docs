# CI/CD Guide

Animus uses GitHub Actions for continuous integration and release automation. This guide covers the CI workflows, build commands, and release process.

## CI Workflows

### Rust Workspace CI (`rust-workspace-ci.yml`)

Runs on every push and pull request. Checks and tests each crate in the workspace independently:

- `cargo check` for each crate: `protocol`, `orchestrator-cli`, `orchestrator-core`, `agent-runner`, `llm-cli-wrapper`, `orchestrator-web-contracts`, `orchestrator-web-api`, `orchestrator-web-server`
- `cargo test --workspace` for the full test suite
- Concurrency grouping cancels superseded runs on the same branch

### Rust-Only Dependency Policy (`rust-only-dependency-policy.yml`)

Enforces the project rule that Animus is Rust-only -- no desktop shell frameworks (Tauri, Wry, Tao, GTK, WebKit). This workflow rejects PRs that introduce prohibited dependencies.

### Web UI CI (`web-ui-ci.yml`)

Runs the web dashboard's npm test suite, Biome lint check, and build from `crates/orchestrator-web-server/web-ui/`. Since v68, Biome replaces ESLint and Prettier for formatting and linting (see [Biome Lint Integration](#biome-lint-integration-v68)).

### Continuous Deployment (`cd.yml`) — v67

The `cd.yml` workflow automates deployment to the Animus Cloud production environment immediately after a successful release build. It triggers on the same `v*` tags that trigger `release.yml` but runs as a dependent job that waits for all cross-platform builds to finish.

**Deployment steps:**

1. **Wait for release** — polls the `release.yml` workflow run status until all build jobs reach `success`.
2. **Authenticate** — uses the `ANIMUS_CLOUD_TOKEN` repository secret (a `deploy`-scoped API key) to authenticate with the cloud API.
3. **Push** — runs `animus cloud push --env production` from the checked-out tag, uploading the workflow definitions in `.ao/`.
4. **Start / restart daemon** — calls `animus cloud start --env production --wait` so the new deployment is active before the job exits.
5. **Health check** — sends a `GET /api/v1/health` request to `app.ao.dev` and asserts a `200` response. The job fails and creates a GitHub issue if the health check does not pass within 120 seconds.

**Configuring the CD workflow:**

Add the deploy token to your repository secrets:

```
Settings → Secrets and variables → Actions → New repository secret
Name: ANIMUS_CLOUD_TOKEN
Value: <deploy-scoped API key from app.ao.dev/settings/tokens>
```

The workflow only runs on tag pushes — branch pushes and PRs do not trigger a deployment.

**Dry-run mode:**

The workflow can be dispatched manually with `dry_run: true` via the GitHub Actions UI. In dry-run mode the push and start steps execute with `--dry-run` and no changes are made to the production daemon.

### Release Rollback Validation (`release-rollback-validation.yml`)

Validates that release artifacts can be produced correctly and that the release process is reversible.

## Build Commands

Animus provides cargo aliases for building the workspace binaries:

```bash
cargo ao-bin-check           # Check all runtime binaries compile
cargo ao-bin-build           # Debug build of all runtime binaries
cargo ao-bin-build-release   # Release (optimized) build
```

The workspace produces three binaries:

| Binary | Crate | Purpose |
|--------|-------|---------|
| `ao` | `orchestrator-cli` | Main CLI |
| `agent-runner` | `agent-runner` | Daemon agent runner |
| `llm-cli-wrapper` | `llm-cli-wrapper` | LLM CLI abstraction |

## Testing

Run all tests:

```bash
cargo test --workspace
```

Run tests for a specific crate:

```bash
cargo test -p protocol
cargo test -p orchestrator-cli
cargo test -p orchestrator-core
```

Integration tests live in `crates/orchestrator-cli/tests/` and cover:

- End-to-end smoke tests
- JSON output contract verification
- Workflow state machine transitions
- Dependency policy enforcement

## Release Process (`release.yml`)

Releases are triggered by pushing a tag matching `v*` or a branch matching `version/**`. Manual dispatch is also supported for dry-run validation.

### Release Steps

1. **Web UI gates** -- Runs npm tests, builds the web UI bundle, and runs smoke E2E tests
2. **Cross-platform builds** -- Compiles release binaries for all targets
3. **Packaging** -- Creates archives with binaries and metadata
4. **Publishing** -- Uploads artifacts (for tag pushes, creates a GitHub release)

### Build Targets

| Target | OS | Runner |
|--------|----|--------|
| `x86_64-unknown-linux-gnu` | Linux | `ubuntu-latest` |
| `x86_64-apple-darwin` | macOS (Intel) | `macos-15-intel` |
| `aarch64-apple-darwin` | macOS (Apple Silicon) | `macos-14` |
| `x86_64-pc-windows-msvc` | Windows | `windows-latest` |

### Creating a Release

Tag and push:

```bash
git tag v1.2.3
git push origin v1.2.3
```

The release workflow builds all targets, packages the archives, and creates a GitHub release with the artifacts.

## Local Release Build

Build a release locally:

```bash
cargo ao-bin-build-release
```

Binaries are placed in `target/release/` (or `target/<triple>/release/` for cross-compilation).

---

## Biome Lint Integration (v68)

Starting in v68, the web UI codebase uses [Biome](https://biomejs.dev) as a unified formatter and linter, replacing the previous ESLint + Prettier setup. Biome is a single Rust-based tool that handles both formatting and lint checks with significantly faster run times.

### Configuration

Biome is configured in `crates/orchestrator-web-server/web-ui/biome.json`. The default configuration enforces:

- Formatting: 2-space indentation, double quotes, trailing commas in multi-line contexts
- Lint rules: recommended ruleset plus `correctness`, `suspicious`, and `style` rule groups
- Organise imports: automatic import sorting on format

### Running Biome Locally

From the `web-ui/` directory:

```bash
# Format all files in place
npx biome format --write .

# Lint and auto-fix safe issues
npx biome lint --apply .

# Run both format and lint (check mode — no writes)
npx biome check .
```

The `biome check` command is what CI runs. It exits non-zero if any file is not correctly formatted or has lint errors that cannot be auto-fixed.

### CI Enforcement

The `web-ui-ci.yml` workflow runs `npx biome ci .` as the first step before the npm test suite and build. The `biome ci` command behaves like `biome check` but outputs diagnostics in a GitHub Actions-compatible format so lint errors appear as annotations on the PR diff.

Pull requests that introduce formatting drift or new lint errors will have a failing `web-ui / biome` check and cannot be merged until the issues are resolved.

### Migrating from ESLint / Prettier

If you have an older checkout with `.eslintrc.*` or `.prettierrc` files in the `web-ui/` directory, remove them — they are no longer used and may conflict with Biome's output. Run `npx biome format --write .` once after pulling v68 to realign your working tree with the new formatter settings before creating a branch.
