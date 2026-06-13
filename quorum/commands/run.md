---
description: Dispatch a deliberation task into the workspace's default room. Streams the verdict over SSE.
allowed-tools: Bash, Read
---

Dispatch a deliberation task and stream the verdict.

**Usage shape:** `/quorum:run "<task description>"` — quoting matters; the task is a single argument.

```
quorum run "<task description>"
```

**Defaults:**

- Reads `nsed.yaml` from the workspace root. Override with `--config PATH`.
- Dispatches into the workspace's `default_room`. Override with `--room <name>` (or `--policy <id>` for ad-hoc rounds that bypass a room).
- Streams the per-round verdict over SSE; prints the final solution + score on completion.

**Useful flags:**

| flag | when to use |
|---|---|
| `--task-file PATH` | task is too long for a shell arg; reads from a file. Resolves relative to the workspace config's directory. |
| `--files <PATH...>` | attach shared context files to the dispatched task. |
| `--output-dir DIR` | capture `verdict.md` + `prompt.md` + `dispatch.json` for audit. |
| `--output-file PATH` | dump only the verdict markdown. |
| `--force-output` | allow overwrite of an existing `verdict.md` under `--output-dir`. |
| `--tui` | open the interactive TUI instead of streaming text. Mutually exclusive with `--files`. |

**Preflight:**

1. `quorum validate` first if `nsed.yaml` was touched in this session. Bail with a clean error if it fails.
2. Confirm `~/.nsed/operator.token` exists; suggest `/quorum:redeem` if not.
3. If the workspace points at api.peeramid.xyz, `curl -sf https://api.peeramid.xyz/health` to confirm the orchestrator is up before dispatching — the task should NOT hang on an unreachable server.

**After completion:**

- Print the verdict's score + winning author.
- Offer `/quorum:trace <job_id>` to inspect the per-round history.
- If the run errored (parse failure, agent timeout, policy not found), surface the orchestrator's error verbatim — those messages already name the right corrective action.

**Common gotchas:**

- "Policy not found" usually means the operator's grants don't intersect any policy's tags — they see the room (because room tenancy is more permissive) but no policy is dispatchable. Surface `/quorum:status` to inspect what they CAN dispatch.
- "Default room ambiguous" — workspace has 2+ rooms and no `default_room`. Either set it in `nsed.yaml` or pass `--room <name>`.
