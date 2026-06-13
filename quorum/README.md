# quorum — Claude Code plugin

Multi-agent deliberation toolkit. Wraps the [`quorum-rs`](https://github.com/peeramid-labs/quorum) CLI with smart prompts, `.env` auto-detection, post-run diagnosis, and slash commands for the entire lifecycle.

## Install

```
/plugin marketplace add peeramid-labs/plugin-marketplace
/plugin install quorum
```

## Prerequisites

The plugin shells out to the `quorum` binary. Install it first:

```
cargo install --git https://github.com/peeramid-labs/quorum quorum-rs --features cli,tui,status-server
```

Confirm with `quorum --version`.

## Slash commands

| Command | What it does |
|---|---|
| `/quorum:init` | Interactive workspace setup. Detects `.env` provider keys, picks defaults, runs `quorum init`, verifies. |
| `/quorum:redeem <code>` | Redeem a JWT invite for NATS creds. Writes `~/.nsed/agent.creds` + `~/.nsed/operator.token` (mode `0600`). |
| `/quorum:run "<task>"` | Dispatch a deliberation task into the workspace's `default_room`. Streams the verdict over SSE. |
| `/quorum:serve` | Boot the agent fleet from `agent.yml`. Resolves NATS URL via the orchestrator's runtime endpoint. |
| `/quorum:status` | Orchestrator health + agent listing. |
| `/quorum:trace <job_id>` | Per-round deliberation history for a finished job. |
| `/quorum:validate` | Schema-check the workspace yaml. |
| `/quorum:tui` | Launch the interactive TUI. |

## Auto-loaded skill

`skills/quorum.md` loads whenever the user touches an `nsed.yaml`, `agent.yml`, the `~/.nsed/` credentials directory, or mentions `quorum` / `nsed-orchestrator` / `api.peeramid.xyz`. It carries the mental model + common-failure-mode catalog so Claude doesn't relearn the architecture every session.

## Where the configs live

| File | Role | Mode |
|---|---|---|
| `nsed.yaml` | Workspace config. Names orchestrators + rooms + policies. Read by `quorum run`/`status`/`trace`/`tui`. | `0644` |
| `agent.yml` | Fleet config. Names providers + agents. Read by `quorum serve`. | `0644` |
| `~/.nsed/operator.token` | Bearer token for orchestrator HTTP. Written by `quorum redeem`. | `0600` |
| `~/.nsed/agent.creds` | NATS NKey JWT. Written by `quorum redeem`. Read by `quorum serve`. | `0600` |
| `~/.nsed/agent.seed` | NKey seed (signs the JWT). Written by `quorum redeem`. | `0600` |

## License

MIT. See `LICENSE` in the marketplace root once added.
