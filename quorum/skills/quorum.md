---
name: quorum
description: Multi-agent deliberation via the `quorum` CLI. Two yaml configs split the concerns. `nsed.yaml` is the workspace — it names the orchestrator(s), rooms, policies, and which agents staff each room. `agent.yml` is the fleet config — it names the LLM providers (openai-family, anthropic-family, ollama, claude-cli, mcp/stdio) and each agent's persona, model, and provider binding. `quorum run` reads `nsed.yaml` and dispatches a task; `quorum serve` reads `agent.yml` and runs every agent in it as a NATS-backed worker. Credentials live in `~/.nsed/agent.creds` (NATS NKey JWT, from `quorum redeem`) and `~/.nsed/operator.token` (bearer for orchestrator HTTP). The orchestrator the workspace points at advertises `GET /api/runtime/nats` so the agent fleet learns the right NATS URL at startup — no `--nats-url` flag in production.
when_to_use: Always load when the user asks about quorum, deliberation, nsed.yaml, agent.yml, an orchestrator at api.peeramid.xyz, a `~/.nsed/` credential, the `nsed-orchestrator` HTTP API, or any of the slash commands `/quorum:*`. Also when the workspace contains an `nsed.yaml` or `agent.yml` and the user starts a task that touches it.
---

# quorum — multi-agent deliberation toolkit

## Mental model

Three moving parts:

1. **Orchestrator** — a long-running HTTP + NATS server (`nsed-orchestrator` in the parent monorepo, runs at `api.peeramid.xyz` for shared use). Holds policies, room visibility / tenancy rules, deliberation history. Mints scoped NATS NKey JWTs for agents on `POST /credentials/register` (challenge-response) or via `POST /redeem` (single-use invite code).

2. **Agent fleet** — one or more `NatsNsedWorker` processes spun up by `quorum serve` from an `agent.yml`. Each agent advertises its capabilities to the orchestrator over NATS and pulls deliberation tasks when its capability tags match a policy role.

3. **Client** — usually just the operator's terminal. `quorum run --task "…"` dispatches into a room; the orchestrator routes it through propose → evaluate → finalize rounds; the verdict streams back over SSE.

## Two yaml files

| file | read by | configures |
|---|---|---|
| `nsed.yaml` (workspace) | `quorum run` / `status` / `trace` / `tui` | orchestrators (HTTP address + bearer token), policies, rooms, default_room |
| `agent.yml` (fleet) | `quorum serve` | providers (LLM endpoints), agents (per-worker config), telemetry, dashboard port |

`quorum init` (interactive wizard) writes both — defaulting `nsed.yaml` to point at `api.peeramid.xyz` and `agent.yml` to the operator's locally configured providers.

## Slash commands provided by this plugin

| command | what it does |
|---|---|
| `/quorum:init` | Interactive setup. Detects `.env` provider keys, picks defaults, runs `quorum init`, verifies with `quorum validate`. |
| `/quorum:redeem <code>` | Redeem a JWT invite for NATS creds. Writes `~/.nsed/agent.creds` + `~/.nsed/operator.token`. |
| `/quorum:run "<task>"` | Dispatch a deliberation task into the workspace's `default_room`. |
| `/quorum:serve` | Boot the agent fleet from `agent.yml`. Resolves NATS URL via the orchestrator's runtime endpoint. |
| `/quorum:status` | Health check the orchestrator and list discovered agents. |
| `/quorum:trace <job_id>` | Display the per-round deliberation trace for a finished job. |
| `/quorum:validate` | Schema-check `nsed.yaml` (and `agent.yml` when present). |
| `/quorum:tui` | Launch the interactive TUI for live deliberation view. |

## Files the plugin assumes exist (or guides the operator to create)

- `~/.nsed/operator.token` — bearer token for orchestrator HTTP. Mode `0600`. Read by every `quorum` subcommand that talks to the orchestrator.
- `~/.nsed/agent.creds` — NATS NKey JWT credentials. Mode `0600`. Read by `quorum serve` to connect agents to the orchestrator's NATS bus.
- `~/.nsed/agent.seed` — the NKey seed that signs the JWT. Mode `0600`. Kept so the same identity survives re-redeem.
- `nsed.yaml` at the workspace root — workspace config. Override with `--config PATH`.
- `agent.yml` at the workspace root — fleet config. Override with `--config PATH` on `quorum serve`.

The wizard rotates any pre-existing creds/seed/token to `.bak-<unix-ts>` sidecars when it overwrites them — so a failed redeem mid-flight doesn't lose the prior identity.

## Common error patterns (Claude should recognise these)

| symptom | actual cause | fix |
|---|---|---|
| `quorum serve` connects to `nats://localhost:4222` despite an orchestrator at api.peeramid.xyz | Missing workspace, or the workspace's orchestrator entry has no bearer token. SDK falls back to its localhost default. | Set `--workspace nsed.yaml` AND make sure the orchestrator entry has a `token: ${OPERATOR_TOKEN}` field. |
| `quorum redeem` succeeds server-side but refuses to write `~/.nsed/agent.creds` | A prior `.creds` file blocked the write. The SDK refused to overwrite, so the JTI was consumed without writing. | Run `quorum redeem --code <new-code>` again; the wizard auto-rotates the old `.creds` to `.bak-<ts>` now. |
| Round hangs in evaluate phase for full timeout | An agent's parse-retry budget exhausted. The agent published `agent_error`; orchestrator didn't see it because the consumer filter was missing. | Update orchestrator: post-`feat/orch-abstention-handling` patch subscribes to `result.{round}.*.evaluate.failed` and counts the agent as abstained. |
| `slop poke` flags every line of a ported file | The port carries inherited debt. Apply `slop apply` to splice `TODO(slop):` markers, then triage in follow-up. Do NOT strip markers. | See the sloppoke plugin's skill. |
| `GET /api/runtime/nats` returns 503 "no available server" | Orchestrator container not running on the host. Edge proxy has no upstream. | Restart the orchestrator deploy (coolify, docker compose) and re-probe. |

## Companion knowledge

- The full quorum-rs SDK + binary lives at <https://github.com/peeramid-labs/quorum>. Issue tracker is the same repo. The orchestrator is a separate crate in the parent `peeramid-labs/nsed` monorepo.
- Tenancy model: every agent + room + policy carries a tag set; operators have grants (tag globs). The orchestrator filters every list / dispatch by `caller.grants ∩ resource.tags`. Admins bypass.
- Persona stacking: `agent.yml` accepts either `persona: "string"` (back-compat) or `persona: [{type: text, prompt: "..."}, {type: md, prompt: ./prompts/x.md}, ...]` (layered, joined with `\n\n` at parse time). Use the layered form for fleets that share preambles.
- Dashboard at `--dashboard-port <N>` is unauthenticated by design; loopback default. `--dashboard-bind 0.0.0.0` exposes LAN — a warn line on boot makes the choice visible. Run behind a reverse proxy + auth for prod.
