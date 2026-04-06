# Installation

## Binary Name

As of v0.3.0 the primary binary is **`animus`**. The previous name `ao` is kept as a backward-compatible alias — both names invoke the same binary. All documentation uses `animus`.

## Build from Source

Animus is a Rust workspace. You need a working Rust toolchain (1.75+ recommended).

```bash
# Clone the repository
git clone https://github.com/launchapp-dev/ao.git
cd ao

# Build all runtime binaries (release mode)
cargo ao-bin-build-release

# Or build in debug mode for development
cargo ao-bin-build
```

The build produces the `animus` binary (with `ao` as an alias) along with supporting binaries (`agent-runner`, `workflow-runner`).

## Run Directly (Development)

If you want to run without installing:

```bash
cargo run -p orchestrator-cli -- --help
```

This compiles and runs the CLI in one step. Useful during development.

## Check Binaries

Verify that all required binaries are present and correctly linked:

```bash
cargo ao-bin-check
```

## Release Binaries

Pre-built binaries are available on the [GitHub Releases](https://github.com/launchapp-dev/ao/releases) page for the following targets:

| Target | Platform |
|--------|----------|
| `x86_64-unknown-linux-gnu` | Linux (x86_64) |
| `x86_64-apple-darwin` | macOS (Intel) |
| `aarch64-apple-darwin` | macOS (Apple Silicon) |
| `x86_64-pc-windows-msvc` | Windows (x86_64) |

Download the appropriate archive, extract it, and place the `animus` binary on your `PATH`. Each archive also contains the `ao` alias binary.

## Verify Installation

```bash
# Check the installed version
animus --version

# Run environment diagnostics
animus doctor
```

`animus doctor` checks for required dependencies, verifies configuration, and reports any issues. Use `animus doctor --fix` to attempt automatic remediation of common problems.

## Prerequisites

Animus orchestrates AI CLI tools. Depending on which agents and models you use, you may need one or more of:

- [Claude CLI](https://docs.anthropic.com/en/docs/claude-cli) (`claude`)
- [Codex CLI](https://github.com/openai/codex) (`codex`)
- [Gemini CLI](https://github.com/google-gemini/gemini-cli) (`gemini`)

These are not required to install Animus itself, but workflows that invoke AI agents will need the appropriate CLI tool available on your `PATH`.

## Next Steps

Once installed, proceed to the [Quick Start](quick-start.md) to configure your first project.
