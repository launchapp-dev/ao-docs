# Privacy & Data Policy

Animus is a local CLI tool. It does not collect telemetry, does not send your code to Animus servers, and does not use your data to train AI models.

## What Animus Does With Your Data

Animus orchestrates agent workflows locally on your machine. All persistent state — tasks, requirements, workflows, worktrees — lives in the `.ao/` directory inside your project. Nothing is transmitted to Animus infrastructure.

When a workflow phase runs, Animus launches an AI CLI tool (e.g. `claude`, `codex`, `gemini`) with your code as context. That data goes to **the AI provider you configured** — Anthropic, OpenAI, Google, or whichever endpoint your model routes to. Animus itself is not in that data path.

## AI Provider Data Policies

Your code reaches only the AI provider whose API key you supply. Review each provider's current data and training policy before use:

| Provider | Data & privacy policy |
|---|---|
| Anthropic (Claude) | [anthropic.com/legal/privacy](https://www.anthropic.com/legal/privacy) |
| OpenAI (GPT / Codex) | [openai.com/policies/privacy-policy](https://openai.com/policies/privacy-policy) |
| Google (Gemini) | [ai.google.dev/gemini-api/terms](https://ai.google.dev/gemini-api/terms) |

API-tier usage typically has different (stricter) training-opt-out terms than consumer-tier products. Confirm the current terms with each provider.

## Why This Matters: Copilot's April 2024 Policy Change

In April 2024, GitHub Copilot updated its data policy to enable code-snippet collection for model improvement by default for certain account tiers. Users who had not reviewed their settings found their code included in training datasets unless they explicitly opted out.

Animus's architecture prevents this class of issue entirely:

- **No Animus cloud layer.** There is no Animus server that your code passes through. There is no Animus account, no Animus telemetry endpoint, and no Animus training pipeline.
- **You choose the provider.** The AI provider that sees your code is determined by which API key you configure. If you want zero cloud exposure, route to a locally-served model.
- **Opt-out is not required.** Because Animus has no data collection of its own, there is nothing to opt out of on the Animus side. Manage training preferences directly with each AI provider.

## Using Local Models for Zero-Cloud Operation

To keep all code processing on your own hardware, point Animus at a locally-served model:

```bash
# Set a custom OpenAI-compatible endpoint
export OPENAI_BASE_URL=http://localhost:11434/v1
export OPENAI_API_KEY=ollama

# Route all phases to a local model
animus workflow agent-runtime set --input-json '{
  "agents": {
    "default": {
      "model": "qwen2.5-coder:32b",
      "tool": "opencode"
    }
  }
}'
```

With a local endpoint, no code leaves your machine at any point during workflow execution.

## Animus's Data Guarantees

| Guarantee | Status |
|---|---|
| No Animus telemetry collection | Yes |
| No Animus training on your code | Yes |
| No Animus cloud account required | Yes |
| All project state stored locally in `.ao/` | Yes |
| You control which AI provider receives code | Yes |
| Local-model operation supported | Yes |

## Auditing Data Flow

To inspect exactly what each phase sends to an AI provider, enable verbose logging:

```bash
AO_LOG=debug animus daemon start
```

Phase prompts, model selections, and tool invocations are logged to stderr. You can verify that no data is routed anywhere other than the configured model endpoint.

## Related

- [Model Routing](model-routing.md) — control which provider handles each phase
- [Agent Personas](agent-personas.md) — restrict what context agents receive
- [Configuration Reference](../reference/configuration.md) — full environment variable reference
