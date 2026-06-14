# peeramid-labs / plugin-marketplace

Plugin marketplace for Peeramid Labs developer tools. Single tap,
two clients — Claude Code and Codex CLI.

## Install

Claude Code:

```
/plugin marketplace add peeramid-labs/plugin-marketplace
/plugin install <plugin-name>@peeramid-labs
```

Codex CLI:

```sh
codex plugin marketplace add github:peeramid-labs/plugin-marketplace
codex plugin install <plugin-name>@peeramid-labs
```

## Plugins

| Plugin | Description | Clients |
|---|---|---|
| [`quorum`](./quorum) | Multi-agent deliberation CLI wrapper. `/quorum:init`, `/quorum:redeem`, `/quorum:run`, `/quorum:serve`, `/quorum:status`, `/quorum:trace`, `/quorum:validate`, `/quorum:tui`. | Claude Code |
| [`sloppoke`](./sloppoke) | Pre-commit AI-slop firewall. Wraps the `slop` CLI with slash commands + a hook that auto-gates every `git commit`. See [sloppoke.me](https://sloppoke.me). | Claude Code · Codex CLI |

## Layout

```
.
├── .claude-plugin/marketplace.json   # Claude Code marketplace manifest
├── .codex-plugin/marketplace.json    # Codex CLI marketplace manifest
├── quorum/                            # plugin root (Claude only today)
│   ├── .claude-plugin/plugin.json
│   ├── commands/
│   ├── skills/
│   └── README.md
├── sloppoke/                          # plugin root (Claude + Codex)
│   ├── .claude-plugin/plugin.json
│   ├── .codex-plugin/plugin.json
│   ├── commands/
│   ├── hooks/
│   │   ├── hooks.json                 # Claude hook config
│   │   ├── codex-hooks.json           # Codex hook config
│   │   └── pre-commit-poke.sh         # shared script body
│   ├── skills/
│   └── README.md
└── README.md
```

## Contributing

Open a PR against `main`. Add the plugin's entry to whichever
marketplace manifest(s) it should surface in
(`.claude-plugin/marketplace.json`, `.codex-plugin/marketplace.json`,
or both). Plugin manifests live under
`<plugin>/.claude-plugin/plugin.json` and/or
`<plugin>/.codex-plugin/plugin.json`.
