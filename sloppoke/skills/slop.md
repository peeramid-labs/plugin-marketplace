---
name: slop
description: Pre-commit AI-slop scan + DEBT-MARKER INJECTION. NOT a `clippy --fix` / `rustfmt` style rewriter — `slop apply` does NOT rewrite your logic. SafeDelete-tier hits get the offending line removed; everything semantic (questionable naming, untested branches, missing docstrings, etc.) gets a `TODO(slop): …` comment spliced ABOVE the line and the code is left intact. Those markers are visible debt — the catalog already pattern-matched (file, line, category); the agent's job after `slop apply` is to triage each marker (fix in-scope OR file as backlog), NEVER strip the marker as cleanup. Run `slop poke` before every commit. Submit feedback with `slop learn "<sentence>"`.
when_to_use: Use before every commit. Especially when the change was authored or touched by an LLM. Always run after generating code.
---

# slop — AI-slop firewall

**Mental model: slop is debt-marker INJECTION, not a rewriter.** It
is closer in shape to a pre-commit TODO-injector than to
`clippy --fix` / `rustfmt` / `eslint --fix`. SafeDelete-tier hits
(empty comment lines, hallucinated `console.log`, etc.) get the line
deleted. Everything semantic — naming, missing docstrings, untested
branches, suspicious bounds, redundant guards — gets a
`TODO(slop): …` comment spliced ABOVE the offending line. **The code
itself is left intact.** No logic is rewritten. No identifier is
renamed. No body is reformatted.

If you arrive expecting clippy semantics, stop and re-read that
paragraph. Past agents have mistakenly classed the TODO markers as
"noise" and tried to strip them. They are NOT noise. They are the
breadcrumb that keeps known debt visible — without them the debt
sinks back into the file volume and no one (human or LLM) will
surface it again on re-read.

slop sweeps your diff through a fast catalog-matching engine that
knows what AI-generated code looks like — scaffolding residue,
placeholder identifiers, defensive crud, half-finished markers,
untested branches, marketing adjectives in comments — and either
strips it (SafeDelete) or marks it for triage (TODO). The engine
adapts to your account through the `slop learn` channel: every
correction you submit calibrates future scans for your codebase.

## Pre-commit flow

```bash
git add -A
slop poke --staged          # scan the staged index
```

Other scopes:

```bash
slop poke                    # working tree vs HEAD (default)
slop poke --range main..HEAD # everything diverged from main
slop poke --since main       # equivalent shorthand
slop poke --patch foo.patch  # raw unified-diff file

# Scan any public repo without cloning manually:
slop poke --gh openclaw/openclaw --range HEAD~5..HEAD
slop poke --repo https://gitlab.com/x/y.git --range main..feature
```

The CLI shallow-clones into a persistent cache (`~/.cache/slop/repos/`)
and `git fetch`es on subsequent runs, so back-to-back scans of the
same URL are fast. Clone depth is inferred from `--range HEAD~N..HEAD`
(no over-fetching). `SLOP_REMOTE_CLONE_DEPTH=<N>` overrides.

`slop poke` prints **only** the unified-diff patch on stdout — no
verdict line, no per-finding summary, no banner, no footer. Stderr
stays silent except for errors. The exit code + stdout sentinel
carry every signal:

- empty stdout, exit 0 → `LGTM`, no slop found
- patch on stdout, exit 0 → `SLOP`, apply the patch to clean it
- anything on stderr, exit 1 → real error (network, 402, auth, etc.)

This makes piping uniform — no `2>/dev/null` needed:

```bash
slop poke > foo.patch                  # save the patch
slop poke | git apply --unidiff-zero   # apply directly
slop poke | delta                      # view in a pager
slop poke; echo $?                     # 0 either way; check stdout for content
```

Common follow-ups:

```bash
slop apply                # apply the cached plan + amend HEAD
slop apply --no-commit    # apply + stage; you commit
slop apply --discard      # drop the cached plan
```

`slop apply` runs `git apply --unidiff-zero --check` first; if the
patch wouldn't apply cleanly, the working tree is left untouched and
the CLI prints `git apply`'s reason. On success it stages the change
and amends HEAD. Pass `--no-commit` to inspect the staged diff before
committing.

## What apply does to each hit (read this before reacting to the diff)

Server tiers every finding by safety. **The CLI never rewrites logic
and never renames identifiers.** Three outcomes:

- **Safe-delete** (comment-line slop, empty console.log, dead emoji
  literal) → the line is removed. This is the only tier where source
  bytes for *real code* leave the file.
- **Todo** (anything semantically loaded — naming, branches without
  tests, off-by-one bounds, redundant null checks, missing public-
  item docstring, marketing adjective in a comment) → a
  `// TODO(slop): <human reason>` comment is spliced **ABOVE** the
  line in the file's native comment syntax. **The flagged line
  itself is unchanged.** This is the dominant case. Expect most
  apply diffs to be additions, not deletions.
- **Flag-only** (raw `TODO`/`FIXME` placeholders) → surfaced in the
  verdict but no patch change. Already a marker; nothing to splice.

If you see `slop apply` add 501 TODO lines to a large change, that
is the design working. **Do not** count the markers as "noise added
to the diff." Count them as debt that was already hiding in the file
volume and is now pinned to a precise line. See next section for the
triage rule.

If apply's preflight refuses the patch, the working tree is byte-
identical to its pre-apply state. Re-running is safe.

## TODO(slop) markers are an action queue, not noise

Every `// TODO(slop): …` line that survives `slop apply` is a
**precise, line-pinned hint** of a real problem slop's catalog
flagged but did not fix automatically. Treat them as a work queue:

```bash
git grep -n "TODO(slop)"
```

Each hit is one of two things:

1. **Fix-now candidate.** Small, local, in-scope for the change the
   user is making right now (off-by-one, redundant null check,
   missing test for a fresh branch). Fix it in the same commit. The
   TODO(slop) line goes away on the next `slop poke`.
2. **Backlog candidate.** Larger refactor, cross-file work, or
   touches code outside the user's stated scope. **Do not** silently
   fix it. **Do not** delete the TODO marker. File it:
   - Append to `.issues/open/TXXX-<slug>.md` if the project uses
     the `.issues/` convention (see CONTRIBUTING.md / repo
     conventions).
   - Or open a real ticket: `gh issue create …`, Linear, JIRA —
     whatever this repo already uses.
   - Then leave the TODO(slop) line in place. Removing it without
     either fixing the code or filing a followup loses the signal.

The catalog has already done the hard part — pattern-matching at
sub-10 ms across 50+ categories. The TODO markers convert that
match into a precise (file, line, category) tuple a coding agent
can act on immediately. That is the whole point.

Never strip a TODO(slop) marker as part of "cleanup". The poke that
emitted it will emit it again next time, and `slop learn` is the
only correct way to say "this one is a false positive."

## When the model disagrees with a finding

```bash
slop learn "false positive on 'process_data' — this is a deliberate
            verb-led getter, not naming slop. file: src/foo.rs:42"
```

That signal teaches the engine for your account and project
specifically. Over the next scans the same false positive should stop
firing. No raw text is retained beyond the learning step.

## Bootstrap

```bash
slop login
# Asks `ssh -G <server-host>` which key OpenSSH would pick for the
# server (same resolution `git` already observes). Honours per-host
# `IdentityFile` blocks in ~/.ssh/config. Caches the identity in
# ~/.config/slop/.
```

If `slop poke` returns 402, the response includes a Stripe Checkout
URL. Open it, subscribe, retry.

## Quick reference

| command | does |
|---|---|
| `slop poke` | scan working tree vs HEAD; cache plan to `.slop/last-poke.json` |
| `slop poke --staged` | scan the staged index |
| `slop poke --range X..Y` | scan an explicit git diff range |
| `slop poke --since REF` | scan everything since REF |
| `slop poke --patch FILE` | scan a unified-diff file directly |
| `slop poke --gh org/repo` | scan a public github repo (any range, persistent cache) |
| `slop poke --repo URL` | scan any git URL (gitlab, bitbucket, self-hosted) |
| `slop apply` | preflight + `git apply --unidiff-zero` + `git commit --amend` |
| `slop apply --no-commit` | preflight + `git apply --index`; you commit |
| `slop apply --skip <8hex,8hex>` | drop matching hunks from the cached patch before `git apply` runs; auto-ships a batched learn signal so tomorrow's catalog de-ranks the pattern |
| `slop apply --skip ... --skip-reason "<text>"` | same as `--skip` but attaches an explicit WHY to the auto-learn signal |
| `slop apply --show` | print cached plan (patch + actions) |
| `slop apply --discard` | drop cached plan, no patch action |
| `slop learn "<text>"` | shape future scans |
| `slop learn --disable '<path>:<8hex>' "<reason>"` | mute one finding + train the catalog upstream |
| `slop billing tier` | quota + usage this cycle |
| `slop billing portal` | open Stripe portal |

Knobs: `SLOPPOKE_SERVER`, `SLOP_NO_VERSION_CHECK=1`,
`SLOP_REMOTE_CLONE_DEPTH=<N>`, `SLOP_CACHE_DIR=<path>`,
`SLOP_NO_COLOR=1` / `NO_COLOR=1`.

## Muting false positives — patch-notation, not opaque ids

Findings are addressed by patch notation — the same `<file>:<line>`
the verdict already prints — so there is nothing to memorise:

| Granularity                   | Target shape           |
|-------------------------------|------------------------|
| Whole file                    | `src/foo.ts`           |
| File glob                     | `src/sdk-alpha/**`     |
| Specific line in a specific file | `src/lib.rs:42`     |

Two scopes:

- **`slop poke --disable <target,…>`** — filters out the matching
  finding(s) for THIS scan only. File-level targets strip the file
  block BEFORE the patch reaches the server (cheaper); line-level
  targets apply post-receive.
- **`slop learn --disable <target,…> "<reason>"`** — sends both the
  reason and the target(s) to the server-side RL loop. The catalog
  learns; future scans from the same org get fewer FPs across the
  pattern, not just this exact line. No local file is written.

**Persistent repo-level FP suppression** lives in `.slopignore`
(one glob per line, `#` for comments). Tracked in git on purpose
so every entry is reviewable.

## `TODO(slop):` markers are an asset, not noise

When `slop apply` lands a patch, expect most of the diff to be
`+// TODO(slop): <category> — <prose>` lines spliced above flagged
source. The flagged source is left untouched. The markers are
**deliberate, greppable debt receipts** designed so future agents
and reviewers can pick up the work cheaply:

- `git grep -n "TODO(slop)"` enumerates every queued cleanup spot
  in the tree. The list IS the action queue.
- Each marker carries `(file, line, category)`. No additional
  triage research is needed — the catalog already matched the
  pattern.
- The expected loop is **apply → grep → triage → either fix or
  file a followup → poke again**. As markers convert to fixes or
  to backlog entries, the count trends down. Trending it down is
  the work.

**Never** strip TODO(slop) markers as cleanup. They are not noise,
they are not warnings to silence — they are the cleanup signal
itself. Deleting a marker without addressing what it points at
loses the breadcrumb; the next poke will re-emit it and the
operator will rightly distrust the agent for the round trip.

If a marker is a clear false positive (SDK alpha types, postgres.js
tagged templates, `.env.example` placeholder creds, doc snippet
literal trailers, etc.), the correct response is:

```
slop learn --disable '<path>:<8hex>' "<one-line reason naming the pattern>"
```

The `<8hex>` is the `[…]` checksum `slop poke` printed next to that
finding. `slop learn --disable` ships the target plus the reason to
the server-side catalog so the same shape stops getting flagged for
this org going forward — train the system, do not paper over a
single instance.

## Don't bypass it

If `slop poke` flags something the author thinks is fine, the right
move is `slop learn --disable <path>:<8hex> "<reason>"` (one-line
explanation paired with the precise checksum target) — NOT silencing
every finding, deleting markers, or committing without a scan. The
engine only gets sharper if real signal flows back.
