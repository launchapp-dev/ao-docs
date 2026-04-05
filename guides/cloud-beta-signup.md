# Animus Cloud Beta: Landing Page and Signup

Animus cloud is currently in public beta. This guide explains how to join the beta, what is included, and what to expect during the beta period.

---

## Signing Up

1. Visit `ao.dev` in your browser.
2. Click **Get Early Access** (top-right of the landing page).
3. Enter your email address and click **Join Beta**.
4. You will receive a confirmation email within a few minutes. Click the link in that email to verify your address.
5. Once your account is provisioned (typically within 24 hours) you will receive a second email with a link to set your password and complete onboarding.

If you do not receive the confirmation email within 10 minutes, check your spam folder. To resend the confirmation, return to `ao.dev` and click **Resend confirmation**.

---

## Onboarding Wizard

After verifying your email and setting a password:

1. Log in at `app.ao.dev`.
2. The onboarding wizard walks you through:
   - Creating your first organisation
   - Installing the Animus CLI (`animus version` must be `0.5.0` or later for cloud support)
   - Running `animus cloud login` to link the CLI to your new account
   - Pushing a sample project with `animus cloud push`
3. The wizard completes when your first daemon reports `running` status.

You can skip steps in the wizard and complete them later from the CLI.

### Wizard Steps in Detail

The wizard is a six-step modal that persists across page reloads. Progress is stored server-side so you can close the browser and resume from any device.

#### Step 1 — Create Organisation

Enter an organisation name and optional slug (auto-generated from the name if left blank). The slug appears in your dashboard URL (`app.ao.dev/org/<slug>`) and cannot be changed after creation.

#### Step 2 — Install the CLI

The wizard detects your operating system and shows the appropriate install command. After running it in your terminal, click **Check version** — the wizard polls `GET /api/v1/onboarding/cli-check` every 5 seconds until it confirms `animus version` is `0.5.0` or later.

If the version check does not pass within 2 minutes, a fallback **Proceed anyway** link becomes available.

#### Step 3 — Log In via CLI

The wizard generates a one-time login URL and displays the corresponding CLI command:

```
animus cloud login --url https://app.ao.dev/auth/device/<code>
```

Running this command opens a browser tab and completes the device auth flow. Once the CLI has received a token, the wizard advances automatically.

Alternatively, click **Copy login command** and paste it into your terminal.

#### Step 4 — Create a Sample Project

The wizard provides a minimal sample project you can copy or download:

```
animus cloud push --project my-first-project
```

The project only requires a single workflow YAML in `.ao/workflows/`. Once the push succeeds the wizard advances and the project appears in the dashboard.

#### Step 5 — Start the Daemon

```
animus cloud start my-first-project
```

The wizard polls daemon status every 3 seconds. A progress bar and status badge update in real time as the daemon transitions through `starting` → `running`.

#### Step 6 — Done

A full-screen celebration screen confirms the setup is complete. It includes:

- A link to the **Projects** overview for your new project
- Quick-start suggestions: schedule your first workflow run, invite a team member, or explore billing
- A **Dismiss wizard** button — the wizard does not reappear but remains accessible at **Settings → Onboarding**

### Reopening the Wizard

If you dismissed the wizard before completing all steps, reopen it from **Settings → Onboarding**. Completed steps are shown with a green checkmark and skipped steps have a grey circle. You can re-run any step individually by clicking its **Redo** link.

### Wizard Completion Criteria

The wizard tracks completion state server-side. Each step is marked complete when:

| Step | Completion trigger |
|---|---|
| Create Organisation | Organisation record created in the database |
| Install CLI | `cli-check` API returns a version `≥ 0.5.0` |
| Log In via CLI | Device auth flow returns a valid access token |
| Create Sample Project | Deployment artifact received for the project |
| Start Daemon | Daemon status transitions to `running` |

---

## Beta Access Tiers

All beta accounts start on the **Starter** plan at no cost. Paid plan tiers (Pro, Team, Enterprise) are available to beta users and charge at the standard rate. See [Cloud Billing](cloud-billing.md) for plan details.

During the beta period:

- Starter plan compute limits may be temporarily increased for beta participants at Anthropic's discretion.
- Service-level agreements (SLAs) do not apply. The cloud service may have unplanned downtime.
- Features may change, be removed, or be added without prior notice.
- Stored data (projects, logs, deployments) is preserved across beta updates on a best-effort basis.

---

## What Is Included in the Beta

The beta includes the full Phase A–F feature set:

| Phase | Features |
|---|---|
| **A** | `animus cloud login`, `animus cloud push`, `animus cloud start`, `animus cloud stop` |
| **B** | `animus cloud status`, `animus cloud logs` — deployment and log management |
| **C** | React dashboard — projects, deployments, agent monitoring, log search |
| **D** | Stripe billing integration — plan management, usage tracking, invoices |
| **E** | CLI sync protocol — hot-reload, event-driven daemon configuration updates |
| **F** | This onboarding flow and beta program |

---

## Beta Limitations

The following features are not yet available in the beta:

- **Multi-region deployments** — all daemons run in `us-east-1`. Additional regions are planned for general availability.
- **Private networking / VPC peering** — daemons connect to AI provider APIs over the public internet.
- **SOC 2 certification** — in progress; expected before general availability.
- **Audit log export API** — available via the dashboard but not yet via CLI.
- **Custom domains** for the dashboard URL.

---

## Feedback

Beta feedback is critical to shaping the general availability release. There are three channels:

| Channel | Use for |
|---|---|
| **In-dashboard feedback button** (bottom-right of `app.ao.dev`) | Quick UI or UX observations |
| **GitHub Issues** (`github.com/ao-dev/ao/issues`) | Bug reports and reproducible errors |
| **Beta community (Discord)** — link in your welcome email | Discussion, questions, feature requests |

When filing a bug report, include the output of `animus version` and `animus cloud status --json` to help the team reproduce the issue.

---

## Leaving the Beta

To stop using Animus cloud:

1. Run `animus cloud stop` to drain and stop all cloud daemons.
2. Navigate to **Settings → Billing → Cancel Subscription** if you are on a paid plan.
3. To delete your organisation and all associated data, go to **Settings → Organisation → Delete Organisation**. This action is irreversible.

After cancellation, all local Animus CLI functionality (local daemon, offline workflows) continues to work without a cloud account.

---

## General Availability

Animus cloud is expected to exit beta when:

- Multi-region deployments are stable
- SOC 2 Type II certification is complete
- Uptime and reliability targets are met over a 90-day window

Beta users will be notified by email before GA pricing takes effect.

---

## Related

- [Cloud Deployment](cloud-deployment.md) — deploying your first project
- [Cloud Dashboard Guide](cloud-dashboard.md) — navigating the web UI
- [Cloud Billing](cloud-billing.md) — plans, usage, and invoices
- [ao cloud — CLI Reference](../reference/cli/cloud.md) — full CLI reference
