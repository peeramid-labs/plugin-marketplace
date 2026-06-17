---
description: Redeem a JWT invite code for NATS credentials. Writes `~/.nsed/agent.creds` + `~/.nsed/operator.token` with `0o600` perms.
allowed-tools: Bash
---

Redeem the JWT invite code the user just pasted.

**Usage shape:** `/quorum:redeem <invite-code>` — the code is a JWT like `eyJhbGciOi...`. Pass it verbatim to `quorum redeem`.

```
quorum redeem <invite-code>
```

**One-command alternative:** if the operator also wants the workspace config scaffolded (not just creds), `quorum init --invite <invite-code>` redeems *and* writes the matching `agent.yml` / `nsed.yaml` in one shot — see `/quorum:init`. Use plain `quorum redeem` when they only need the credentials.

The binary:

1. Generates a fresh NKey on this host (or reuses `--seed-in PATH` if the operator wants to keep the same NATS identity across re-redeems).
2. POSTs `{code, user_pub_key}` to the orchestrator's `/redeem-agent` endpoint.
3. Writes `~/.nsed/agent.creds` (NATS NKey JWT, `0o600`) — only when the code grants NATS credentials.
4. Writes `~/.nsed/operator.token` (bearer for orchestrator HTTP, `0o600`) — only when the code is an operator invite.
5. Writes `~/.nsed/agent.seed` (the NKey seed, `0o600`).

**Default orchestrator URL** is `https://api.peeramid.xyz`. Override with `--url URL` or set `NSED_ENV=local` (or `dev`/`development`) to flip to `http://localhost:8080`.

**Pre-existing creds:** the binary auto-rotates any existing `~/.nsed/agent.{creds,seed}` and `~/.nsed/operator.token` to `.bak-<unix-ts>` sidecars before writing the new ones. Operators recover the previous identity from the sidecar if they need to.

**After redeem succeeds:**

- Tell the user the three file paths that got written.
- Suggest `/quorum:status` to verify the orchestrator sees them, or `/quorum:serve` if they want to bring up an agent fleet using the new creds.

**Common failure modes:**

| symptom | reason | recovery |
|---|---|---|
| `400 Bad Request: code expired` | TTL on the invite has lapsed. | Ask the admin for a fresh code. |
| `409 Conflict: jti already redeemed` | Code is single-use; another `quorum redeem` already consumed it. | Ask the admin to mint a new one. The current `.creds` (if written) is fine. |
| `403 Forbidden: tag X outside grants` | Operator's grants don't cover the requested agent tags (PR #445 gate). | Either narrow the request or ask the admin to widen the operator's grants. |
| `quorum redeem` succeeded server-side but wrote no `.creds` | Rare race: another process held the file. | Re-run `quorum redeem` with a freshly minted code. |

NEVER print the redeemed token / NKey seed / creds blob in chat output — the user can `cat ~/.nsed/agent.creds` if they need to inspect. The plugin treats those bytes as secrets.
