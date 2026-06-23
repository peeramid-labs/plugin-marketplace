---
description: Send feedback to the slop learning loop via `slop learn`. Optionally combine with `--disable <path[:line]>` to also mute specific finding addresses upstream.
allowed-tools: Bash
---

Submit a short feedback note to the sloppoke server so the engine calibrates for the user's account and project.

Use this when:
- The most recent scan flagged something that is NOT slop in the user's codebase context (false positive)
- The most recent scan MISSED something the user expected it to catch (false negative)
- The user wants to share project-specific intent ("we always name our top-level handler `Manager` on purpose")

Plain feedback:

```
slop learn "<one-sentence note explaining the FP / FN / intent. include file/line if relevant.>"
```

Feedback **paired with a precise mute** so the server's RL loop joins "<reason>" to the exact patch address the operator wants muted:

```
slop learn --disable <path[:line],...> "<reason>"
```

Examples:

```bash
# Whole-file FP — Zama SDK alpha types churn, : any is deliberate
slop learn --disable 'src/sdk-alpha/**' \
  "Zama SDK 3.1.0-alpha.15 has unstable types; any-casts here are deliberate until the SDK stabilises"

# Line-level FP — explanation doc lists trailer patterns it crawls for
slop learn --disable 'docs/explanation/correlation-study.md:119' \
  "Line is technical content explaining which trailer patterns the crawler searches; catalog matched on the literal strings without checking context"

# Multi-target — same reason applies to several files
slop learn --disable 'src/db/queries.ts,src/db/migrations/*.ts' \
  "postgres.js tagged templates ARE prepared statements (\$1, \$2 binding under the hood) — sql\`SELECT … \${var}\` is not string interpolation"
```

Keep the note short and specific. Quote the exact identifier or pattern at issue. Do not paraphrase code blocks into the message. Targets are patch-notation addresses — `path/glob` (file-level mute) or `path/glob:LINE` (line-level mute) — the same shape `slop poke` prints in its stderr verdict per finding, so copy-paste from the verdict, do not synthesize ids.

Don't auto-submit feedback the user didn't actually ask for. Always confirm the wording before sending.

The `--disable` targets ride along inside the feedback context block. Nothing is written to disk locally — the mute is a server-side catalog adjustment, not a `.slop/disabled-findings` file. Future poke runs from the same org pick up the lower FP rate automatically once the RL loop accepts the signal.

For one-shot mutes that should NOT train the catalog (e.g. this specific scan only), use `slop poke --disable <target>` directly — see `/slop:poke`.
