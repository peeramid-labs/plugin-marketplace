---
description: Boot a fleet of agents from `agent.yml`. Each agent registers with the orchestrator over NATS and pulls deliberation tasks.
allowed-tools: Bash, Read
---

Boot the agent fleet defined in `agent.yml`.

**Usage shape:** `/quorum:serve` (no args for default) — or with explicit `--config PATH` and filter flags.

```
quorum serve
```

**Defaults:**

- Reads `agent.yml` from `$PWD`. Override with `--config PATH`.
- Reads `nsed.yaml` (workspace) from `$PWD` to learn which orchestrator HTTP endpoint to query for the agent-facing NATS URL. Override with `--workspace PATH`.
- Resolves the NATS URL by calling `GET /api/runtime/nats` on the workspace's resolved orchestrator (matching the resolved `--room`'s `orchestrator` entry, or the workspace's first orchestrator otherwise). Override with `--nats-url URL` only for offline / dev clusters where the runtime endpoint isn't reachable.
- Uses `~/.nsed/agent.creds` for NATS auth. Override with `--nats-creds PATH`.
- Traps SIGTERM / SIGINT and aborts every worker through a CancellationToken before the runner future drops.

**Useful flags:**

| flag | when to use |
|---|---|
| `--agent NAME` (repeatable) | run a subset instead of every agent in the fleet. Matches case-insensitively. |
| `--dashboard-port PORT` | start the LAN-visible unified dashboard. Falls back to `agent.yml`'s `dashboard_port`. |
| `--dashboard-bind ADDR` | bind address for the dashboard. Defaults to `127.0.0.1`. **Use `0.0.0.0` carefully — dashboard ships unauthenticated.** |
| `--stream-name NAME` | override the JetStream stream name. Only needed if the orchestrator was deployed with a non-default `$NSED_STREAM`. |

**Preflight:**

1. Confirm `agent.yml` exists and `quorum validate --config agent.yml` (or its analog) doesn't bail.
2. Confirm `~/.nsed/agent.creds` exists; suggest `/quorum:redeem` if not.
3. If the workspace points at api.peeramid.xyz, confirm `GET /api/runtime/nats` is up — otherwise `serve` will fail-loud with a structured error naming the workspace path + orchestrator address (no localhost fallback in production).

**After boot:**

- Tail the structured log output for the `agent ready` lines.
- If `--dashboard-port` is set, give the operator the dashboard URL (`http://<bind>:<port>/`).
- Suggest `/quorum:status` from another shell to verify the orchestrator sees the new agents.

**Common gotchas:**

| symptom | cause | fix |
|---|---|---|
| "no fleet config found" | `--config` not passed and neither `agent.yml` / `agent.yaml` / `config/default.yml` exists in `$PWD`. | Run `/quorum:init --agent-fleet` first. |
| "no --nats-url passed and workspace config not loadable" | The workspace file doesn't exist or doesn't have an orchestrator entry. | Either pass `--workspace PATH` to a valid `nsed.yaml` or set `--nats-url` directly. |
| Agent boots but never picks up tasks | Capability tags don't match any policy role on the orchestrator. | `/quorum:status` to inspect; widen the agent's tags or the policy's `capabilities`. |
| `quorum serve` connects to localhost despite a remote workspace | Old binary build (pre-#453) or workspace's orchestrator entry has no token. | Upgrade the binary; ensure `nsed.yaml` orchestrator has `token: ${...}`. |
