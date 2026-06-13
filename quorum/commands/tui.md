---
description: Launch the interactive TUI for live deliberation view.
allowed-tools: Bash
---

Launch the interactive TUI.

```
quorum tui
```

**What it shows:**

A multi-pane terminal interface (ratatui-backed) with:

1. **Orchestrators pane** — every orchestrator the workspace knows about + their health.
2. **Agents pane** — discovered agents + their current job / model / capability tags.
3. **Jobs pane** — running + recent deliberations with their phase, round, score.
4. **Trace pane** — when a job is selected, the per-round proposal + evaluation history live-streams as new events arrive over SSE.
5. **Policies pane** — admin / operator-visible policies the workspace can dispatch into.

**Keybindings:**

- `Tab` / `Shift+Tab` — cycle panes.
- `?` — show help.
- `q` — quit.
- `Enter` on a job — open the trace pane for that job.
- `s` while a job is selected — "Stop and pick this" (force-finalize).

**Requires:** the binary built with `--features tui` (default for `cargo install`).

**When to suggest the TUI over the streaming `quorum run`:**

- Operator wants to watch multiple jobs at once.
- Operator wants to mid-flight finalize a job that's clearly converged.
- Long-running deliberations where the operator needs to keep tabs without tailing logs.

**When to skip the TUI:**

- CI / scripted runs — stick with `quorum run` and parse the SSE stream.
- Very small terminals — the panes don't render well below 80x24.
