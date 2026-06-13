# peeramid-labs / plugin-marketplace

Claude Code plugin marketplace for Peeramid Labs developer tools.

## Install

In Claude Code:

```
/plugin marketplace add peeramid-labs/plugin-marketplace
/plugin install quorum
```

## Plugins

| Plugin | Description |
|---|---|
| [`quorum`](./quorum) | Multi-agent deliberation CLI wrapper. `/quorum:init`, `/quorum:redeem`, `/quorum:run`, `/quorum:serve`, `/quorum:status`, `/quorum:trace`, `/quorum:validate`, `/quorum:tui`. |

## Layout

```
.
├── .claude-plugin/marketplace.json   # marketplace manifest (consumed by Claude Code)
├── quorum/                            # plugin root
│   ├── .claude-plugin/plugin.json     # plugin manifest
│   ├── commands/                      # slash commands (/quorum:*)
│   ├── skills/                        # auto-loaded context
│   └── README.md                      # plugin docs
└── README.md
```

## Contributing

Open a PR against `main`. Add the plugin to `.claude-plugin/marketplace.json` so the marketplace surfaces it. Plugin manifests live under `<plugin>/.claude-plugin/plugin.json`.
