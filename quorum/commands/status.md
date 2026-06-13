---
description: Health-check the orchestrator and list discovered agents.
allowed-tools: Bash
---

Surface the orchestrator's health + the agents it currently sees.

```
quorum status
```

**What it prints:**

- Orchestrator HTTP `/health` response (status + NATS connection state + timestamp).
- Agent listing from `GET /agents`: agent id, provider, model, current job (if any), capability tags, operator that registered it, last-seen timestamp.

**Useful flags:**

- `--orchestrator NAME` — when `nsed.yaml` has more than one orchestrator, name which to query. Defaults to the workspace's first.

**Read the output as:**

| signal | meaning |
|---|---|
| `nats_connection: ok` | Orchestrator's own NATS connection is alive — fleet can dispatch. |
| `nats_connection: degraded` | Orchestrator running but NATS bus is in trouble. Existing rounds may complete; new dispatch may fail. |
| `is_online: false` on an agent | The agent registered but its heartbeat lapsed. May still be there mid-restart. |
| Empty agent list despite a running `quorum serve` | Tenancy filter: the operator's grants don't intersect the agents' tags. Cross-check by hitting the API as an admin. |

**After:**

- If an agent that should be there is missing, suggest `/quorum:trace <job_id>` to see where the dispatch went, or remind the operator to bring up the fleet with `/quorum:serve`.
- If the orchestrator is unreachable, surface the curl command (`curl -sf https://api.peeramid.xyz/health`) so the operator can confirm the issue from outside the tool.
