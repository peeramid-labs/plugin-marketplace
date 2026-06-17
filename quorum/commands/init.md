---
description: Bootstrap a quorum workspace. Detects `.env` provider keys, picks sensible defaults, runs `quorum init`, verifies with `quorum validate`.
allowed-tools: Bash, Read, Write, Edit
---

Bootstrap a quorum workspace in the current directory.

**Goal:** at the end of this run the operator has a working `nsed.yaml` + (optionally) `agent.yml`, `~/.nsed/agent.creds` if they redeemed an invite, and a working `quorum validate` pass.

**Preflight (do these silently before prompting the user):**

1. `command -v quorum` — confirm the binary is on PATH. If missing, tell them `cargo install --git https://github.com/peeramid-labs/quorum quorum-rs --features cli,tui,status-server` and stop.
2. Read `$PWD/.env` if it exists. Spot which provider keys are present:
   - `OPENAI_API_KEY` → suggest `openai-family` provider
   - `ANTHROPIC_API_KEY` → suggest `anthropic-family` provider
   - `OPENROUTER_API_KEY` → suggest `openrouter`
   - `GOOGLE_API_KEY` → suggest `gemini`
   - `PEERAMID_OPERATOR_TOKEN` → existing operator token, prefer the "Use existing token" auth path during redeem
3. Check whether `nsed.yaml` and/or `agent.yml` already exist in `$PWD`. If yes, ask whether to re-run init (the binary's wizard auto-rotates to `.bak-<ts>`).
4. Check for `~/.nsed/operator.token` — surfaces the third auth option in the wizard.
5. Check whether `api.peeramid.xyz` resolves / responds on `/health`. If unreachable, default suggestion is an embedded orchestrator instead of remote.

**Have an invite code? One-command onboarding:**

If the operator already has an invite code, prefer `quorum init --invite <code>`. It redeems the code (writing creds/token/endpoint, same as `/quorum:redeem`) and then scaffolds the matching config in one shot — auto-picked by the code's audience: an **agent** code writes `agent.yml`, an **operator** code writes `nsed.yaml`. Add `--out-dir <dir>` to redirect the redeemed credentials (default `~/.nsed`).

```
quorum init --invite <invite-code>
```

This collapses redeem + init; skip the wizard when it applies.

**Then run the wizard:**

The default `quorum init` (no flags, TTY stdin) drops the operator into the interactive wizard. Pass nothing — the binary handles the prompts. The wizard also asks, per agent, for **read access** (files/dirs the agent reads for context) and **write access** (directories it may manage) — these map into `agent.yml` as `builtin_tools: read_file` roots (native LLM), `add_dirs` + `writable` (Claude), or `working_dir` (exec). Leaving a prompt blank grants no access.

If the operator wants non-interactive (CI / Docker entrypoint), use:

```
quorum init --non-interactive \
  --orchestrator-url https://api.peeramid.xyz \
  --room main \
  --token-env PEERAMID_OPERATOR_TOKEN \
  --agents <comma-separated-list>
```

Add `--agent-fleet` if they want `agent.yml` instead of `nsed.yaml`.

**After the wizard completes:**

1. `quorum validate` — confirm the yaml passes schema check.
2. Read the emitted `nsed.yaml` and surface the resolved `default_room`, orchestrators, and policies count.
3. If `agent.yml` exists, also surface its provider count + agent count.
4. Suggest the next step: `/quorum:redeem <code>` if they have an invite, `/quorum:run "<task>"` if they want to dispatch immediately, or `/quorum:serve` if they need to bring up the agent fleet.

**Never:** auto-edit `nsed.yaml` / `agent.yml` after the wizard writes them; the wizard is the source of truth for the operator's choices. If the operator wants a tweak, re-run the wizard or have them edit by hand.
