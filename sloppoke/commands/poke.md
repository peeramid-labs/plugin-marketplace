---
description: Run `slop poke` against the current diff and report the verdict. Patches it proposes are TODO(slop) splices above semantic findings + line deletions for SafeDelete tier — never source rewrites.
allowed-tools: Bash
---

Scan the current git changes through sloppoke and report the verdict.

**Mental model:** `slop poke` is a debt-detector. The patch it proposes (if SLOP) splices `TODO(slop): …` markers above semantic findings and deletes lines only at SafeDelete tier (empty comment slop, hallucinated console.log). It does NOT rewrite logic, rename identifiers, or reformat. Closer in shape to a pre-commit TODO-injector than to `clippy --fix`.

Pick the scope based on context:
- Staged-only context → `slop poke --staged`
- Working-tree (default review) → `slop poke`
- Specific range → `slop poke --range <BASE>..<HEAD>`
- Remote repo → `slop poke --gh <org/repo> --range HEAD~5..HEAD`

After the scan:
- If exit code is 0 with empty stdout → reply with `LGTM` and one short line of context. **Then** run `git grep -n "TODO(slop)"` once — if any pre-existing markers exist they are queued work, surface them.
- If stdout has a patch → show the patch (or its hunk count) to the user, and offer `/slop:apply` as the next step. Remind that apply will splice `TODO(slop): …` markers for the semantic findings, which the agent then triages (fix-in-scope vs file-as-followup) — that's the point of the markers, not noise.
- If stderr has an error → surface it verbatim, do not try to fix it silently.

Stderr also carries one `  <file>:<line>  <category>` line per surviving finding. Copy a `<file>:<line>` straight from there into `--disable` or `slop learn --disable` when a finding is a false positive — no opaque ids to memorise.

**Per-finding mute when a finding is a clear FP** (e.g. SDK alpha types, `.env.example` placeholder creds, postgres.js tagged templates):

```bash
# One-shot: don't flag this in THIS scan
slop poke --staged --disable src/sdk-alpha/types.ts

# Multiple, comma-separated
slop poke --staged --disable 'src/sdk-alpha/**,src/db/queries.ts'

# Line-level — narrow to the specific line
slop poke --staged --disable src/lib.rs:42
```

Path-only `--disable` filters whole file blocks **before** the patch reaches the server (cheaper, fewer hits). `path:LINE` is a finer mute that applies post-receive to individual findings.

**Persistent FP suppression** lives in `.slopignore` (one glob per line, `#` for comments). Use this for repo-level patterns the operator has triaged once and never wants to see again: `.env.example`, `src/sdk-alpha/**`, generated bindings, etc. `.slopignore` is committed to git on purpose — every entry is reviewable.

NEVER commit, push, or amend HEAD on the user's behalf. The verdict is informational; the user runs `/slop:apply` when they decide.

`TODO(slop):` markers are precise (file, line, category) action items — the catalog already pattern-matched. The agent's job is to convert each into either a fix (small + in-scope) or a backlog entry (out-of-scope). Never strip a marker as cleanup.
