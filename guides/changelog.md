# Changelog

Release highlights for each Animus CLI version. For the full commit history see the [GitHub repository](https://github.com/launchapp-dev/ao).

---

## v0.3.0

### Binary Rename: `ao` → `animus`

The primary CLI binary is now **`animus`**. The previous name `ao` is kept as a backward-compatible alias — both names invoke the same binary. All documentation and examples use `animus`.

```bash
# Both of these work; animus is preferred
animus version
ao version
```

Upgrade path: no configuration changes are required. Any scripts using `ao` continue to work unchanged. Update them to `animus` at your own pace.

### CLI-to-Cloud Integration

v0.3.0 ships a complete set of commands for deploying and managing projects in Animus Cloud from the CLI.

#### `animus cloud login` — Device Auth Flow

`cloud login` now defaults to the **device auth flow**, which works on headless machines and remote servers without a local browser. The CLI prints a short URL and a one-time code; open the URL on any device, enter the code, and the terminal session is authenticated automatically.

```bash
# Device flow — default, works everywhere
animus cloud login

# Browser flow — opens the local system browser directly
animus cloud login --browser

# Token — CI/CD and service accounts
animus cloud login --token $AO_CLOUD_TOKEN
```

#### `animus cloud link` — Link Project to Cloud

New command. Links the current project to an Animus Cloud project. By default, Animus reads the `origin` git remote, normalises it to a canonical `owner/repo` identifier, and matches it against cloud projects registered in your organisation. If exactly one match is found the link is created automatically; otherwise you are prompted to select.

```bash
# Auto-detect from git remote (recommended)
animus cloud link

# Specify a cloud project slug explicitly
animus cloud link --project my-project
```

The link is stored in `.ao/config.json` under the `cloud` key. Run `animus cloud link` once per checkout — all subsequent `push`, `deploy`, and `status` commands use the stored link automatically.

#### `animus cloud push` — Push Project Metadata

Packages project configuration (workflow definitions, personas, packs) and uploads it to the cloud. Source code is **not** uploaded.

```bash
animus cloud push
animus cloud push --env staging
animus cloud push --dry-run          # validate without uploading
```

#### `animus cloud deploy` — Push + Start in One Step

New convenience command. Equivalent to `animus cloud push && animus cloud start`. Use this for the common case of shipping a configuration update and immediately restarting the cloud daemon.

```bash
animus cloud deploy
animus cloud deploy --env staging
animus cloud deploy --wait           # block until daemon reports running
```

#### `animus cloud status` — Deployment and Daemon Status

Shows the current state of the cloud deployment and daemon, including active agent count, queue depth, and region.

```bash
animus cloud status
animus cloud status --json | jq '.daemon_status'
```

### Upgrade Notes

| Area | Change | Action required |
|---|---|---|
| Binary name | `ao` renamed to `animus`; `ao` alias kept | None — update scripts at your convenience |
| Cloud auth | Device flow is now the default | None — existing token-based auth still works |
| Cloud link | Required before first `push` on a new checkout | Run `animus cloud link` once per checkout |

---

## Earlier Releases

For release notes prior to v0.3.0, see the [GitHub Releases](https://github.com/launchapp-dev/ao/releases) page.
