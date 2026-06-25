---
description: Apply the last cached `slop poke` patch — splices TODO(slop) markers above semantic findings, strips SafeDelete tier. NOT a `clippy --fix` style rewriter; logic is never modified. The spliced markers are designed to be greppable cleanup queues, not noise.
allowed-tools: Bash
---

Apply the cached slop-poke patch produced by the most recent `/slop:poke` invocation.

**Mental model — read first:** `slop apply` is debt-marker INJECTION, not source rewriting. Expect most of the diff to be `+// TODO(slop): …` additions above flagged lines. The flagged line itself is unchanged. SafeDelete tier (empty comments, hallucinated console.log) deletes the offending line outright. Nothing renames identifiers; nothing rewrites bodies.

**`TODO(slop):` markers are an asset, not noise.** They are deliberately greppable, precise (file, line, category) breadcrumbs designed to make future attention cheap. `git grep -n "TODO(slop)"` enumerates every queued cleanup spot in the tree. The whole point of `slop apply` is to convert hidden LLM-introduced debt into surface-visible action items that any future agent or reviewer can pick up. If the diff looks "noisy" because many TODOs were added, that is the system working as designed.

Default behaviour:
```
slop apply --no-commit
```

`--no-commit` is the safer default for plugin use: it applies + stages the patch and leaves committing to the user (and their normal workflow), so we never amend a HEAD that has already been pushed.

If the user explicitly asks to amend HEAD ("squash it in", "amend the last commit"), drop the flag:
```
slop apply
```

**Skipping specific findings at apply time.** Each finding the `/slop:poke` summary printed carries a `[<8hex>]` checksum next to its `<file>:<line>` row. To land the cached patch without the TODO markers piling on findings the user has already triaged as false positives, pass the checksums to `--skip`:

```
slop apply --skip 3e1cc88d,fc10f7dd          # drop two specific hunks
slop apply --skip 3e1cc88d --no-commit       # combined with the safer default
slop apply --skip 3e1cc88d \
  --skip-reason "Zama SDK alpha types churn; any-casts are deliberate"
```

`--skip` filters the cached patch BEFORE `git apply` runs — the dropped hunks never touch the working tree. Same 8-hex checksums also work as `--disable` targets on the next `/slop:poke` if the user wants the line redacted pre-send instead of cleaned up post-apply.

**Auto-learn**: every `--skip` invocation that actually dropped a hunk **also ships a single batched learn entry to the server** so tomorrow's catalog de-ranks the pattern for this org. The operator does NOT have to run a separate `slop learn` call — the apply IS the training signal. Confirmation prints on stderr ("slop: shipped N skip signal(s)…"). One entry per apply (not per checksum) so the operator's monthly learn quota stays predictable. Pair with `--skip-reason "<text>"` to attach an explicit WHY so the catalog learns the reason, not just the checksum.

If apply fails its preflight (`git apply --check`), the working tree is byte-identical. Surface the reason verbatim. Don't loop, don't retry, don't mutate anything else.

When the user wants to drop the cached plan instead of applying it, run `slop apply --discard`.

## After apply: the TODO(slop) markers are an action queue

Once `slop apply` succeeds, **immediately** run:

```
git grep -n "TODO(slop)"
```

Each hit is a precise (file, line, category) flag the catalog matched but did not auto-fix. Treat the list as a backlog the agent (or any future agent reading the tree) is expected to convert:

- **In-scope + small** (off-by-one, redundant guard, missing test for a branch the user just added) → fix it in the same change. The marker disappears on the next `/slop:poke`. Don't ask permission for trivial in-scope fixes; just do them and report.
- **Out-of-scope or larger refactor** → file a followup. Use whatever issue convention this repo already has (`.issues/open/TXXX-…md` if the repo uses the convention, otherwise `gh issue create`). Leave the TODO(slop) line in the code. Mention the new ticket reference next to it so the marker stays load-bearing for future grep.

**Never** delete a TODO(slop) marker as cleanup. The catalog will re-emit it on the next poke and the operator loses the historical breadcrumb. If the marker is a clear false positive, the right tool is:

```
slop learn --disable '<path>:<8hex>' "<one-line reason, name the pattern>"
```

The checksum is the `[<8hex>]` value `/slop:poke` printed for that finding. `slop learn --disable` trains the server-side catalog so the same shape stops getting flagged for this org; future scans return fewer FPs without anyone having to touch the local mute list again.

Pattern: `apply → grep → triage → either fix in scope, file a followup, or `slop learn` if FP → poke again`. The markers are work, not noise.
