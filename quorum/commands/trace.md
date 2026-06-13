---
description: Display the per-round deliberation trace for a finished job.
allowed-tools: Bash
---

Display the full per-round trace for a finished deliberation job.

**Usage shape:** `/quorum:trace <job_id>` — the job_id is what `quorum run` printed on completion or what the orchestrator's history endpoint returns.

```
quorum trace <job_id>
```

**What it prints:**

Per round (`r1`, `r2`, …):

1. The proposers' candidates with their score + author.
2. The evaluators' verdicts per candidate: score in `[-1, +1]`, stance (`strong_agree` / `agree` / `neutral` / `disagree` / `strong_disagree`), per-category breakdown (correctness / completeness / novelty / feasibility / evidence_quality).
3. Round-level aggregates: convergence, quorum count, finalize trigger (if the round short-circuited).
4. Any abstention markers: agents that bailed mid-round and the reason (`parse_error`, `iter_budget_exhausted`, `timeout`, `tool_error`).

**Useful flags:**

- `-v` / `--verbose` — include each evaluator's full justification text (otherwise truncated to the first line).
- `--orchestrator NAME` — when the workspace has multiple orchestrators, name which one to query.

**Read the output as:**

| signal | meaning |
|---|---|
| Convergence = 1.00 in round 1 | Single-shot win — every evaluator agreed on the same candidate. |
| Convergence climbing across rounds | Agents are converging on the right answer; the orchestrator will finalize when the threshold trips. |
| One agent abstaining every round | Their LLM is failing — check their persona / model / capability tag set. The new `feat/orch-abstention-handling` handler counts the abstention so the round still advances. |
| "Mid-phase force-finalize observed" | Operator clicked "Stop and pick this" in the TUI — the verdict is whatever the orchestrator had at that moment. |

**After:**

- If a round looks wrong (one agent scoring everything `-1`, or a candidate winning despite contradicting peer review), suggest the operator look at that agent's persona via the dashboard or replay with `quorum run --output-dir audit/<job>` next time.
- If the trace shows repeated abstentions for the same agent, suggest re-deploying that agent's worker — its NATS connection or LLM client is likely stuck.
