---
description: Schema-check the workspace yaml (`nsed.yaml`) and report a one-line summary.
allowed-tools: Bash
---

Schema-check the workspace yaml and surface a one-line summary.

```
quorum validate
```

**What it does:**

- Loads `nsed.yaml` (or `--config PATH`).
- Runs `WorkspaceConfig::load` (deserialise + structural validate — references resolve, default_room exists if rooms do, etc.).
- Prints `valid — N policies, M orchestrators, K rooms, default_room: <name|(none)>` and exits `0`.
- On failure: prints the error to stderr with a path + line + column, exits non-zero.

**Useful flags:**

- `--config PATH` — point at a non-default workspace file.

**When to invoke:**

- Right after `/quorum:init` finishes.
- Before `/quorum:run` if the operator hand-edited `nsed.yaml` in this session.
- In a pre-commit hook (`quorum validate` has no side effects — no network, no LLM, no filesystem mutation).

**Common failure shapes:**

| error | meaning | fix |
|---|---|---|
| `path not found` | The wrapper found neither `--config` nor `./nsed.yaml`. | Pass `--config PATH` or `cd` to the workspace root. |
| `room "main" references policy "default" not defined` | Operator deleted a policy block but a room still points at it. | Either restore the policy or repoint the room (or run `/quorum:init` to regenerate). |
| `multiple rooms defined but no default_room` | Workspace has > 1 room and `quorum run` won't know which one to dispatch into. | Add a `default_room: <name>` at the top level, or always pass `--room` to `quorum run`. |

**Never** auto-fix the yaml on the operator's behalf — surface the error verbatim and let them edit. The structure here is operator-owned and small enough to hand-edit.
