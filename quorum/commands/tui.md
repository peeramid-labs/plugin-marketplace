---
description: Launch the interactive TUI for live deliberation view.
allowed-tools: Bash
---

Launch the interactive TUI.

```
quorum tui
```

**What it shows:**

A ratatui-backed interface with a persistent top-level **tab bar**:

1. **Deliberate** — the landing screen. Lists the rooms you can submit to (config rooms, or runtime/remote rooms when running config-free) and launches a deliberation against the selected room with its bound policy.
2. **Rooms** — list / create / delete rooms. Each row shows the bound policy, panel **fill** (`eligible/desired`), tags, and a detail panel with the agents that would serve it. Create-room form picks the policy from a selector.
3. **Agents** — discovered agents + their current job / model / capability tags.
4. **Policies** — operator-visible policies the workspace can dispatch into.
5. **Settings** — Orchestrators (health/config) + workspace Config.

Selecting a running job opens its trace, where the per-round proposal + evaluation history live-streams over SSE.

**Panel fill:** a room whose eligible agents fall below its policy's target shows a red `✗` (green `✓` when met) on both the Rooms and Deliberate screens — it can't start a deliberation until enough matching agents are online.

**Keybindings:**

- `1`–`5` or `Tab` / `Shift+Tab` — switch top-level tab.
- `↑` / `↓` — navigate the current list.
- `Enter` — primary action for the tab (deliberate / open / select).
- `q` or `Esc` — back / quit.

**Requires:** the binary built with `--features tui` (default for `cargo install`).

**When to suggest the TUI over the streaming `quorum run`:**

- Operator wants to watch multiple jobs at once.
- Operator wants to mid-flight finalize a job that's clearly converged.
- Long-running deliberations where the operator needs to keep tabs without tailing logs.

**When to skip the TUI:**

- CI / scripted runs — stick with `quorum run` and parse the SSE stream.
- Very small terminals — the panes don't render well below 80x24.
