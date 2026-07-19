# kazi Claude Code plugin marketplace

This repository is the [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
for [kazi](https://github.com/kazi-org/kazi). It is generated and updated
automatically by the kazi release pipeline (ADR-0077) -- do not hand-edit.

## Install

```
/plugin marketplace add kazi-org/claude-plugins
/plugin install kazi@kazi
```

The `kazi` plugin bundles, in one install:

- the `kazi` skill (SKILL.md + AUTHORING.md + RECIPES.md),
- the `kazi` MCP server registration (the same shape `kazi init --with-mcp` writes),
- the session-bus hooks (the same set `kazi install-hooks` registers).

## Lockstep versioning

The plugin version always equals the kazi binary release it was generated from.
Every kazi release regenerates the bundle under `plugins/kazi/` and bumps the
`version` field of the `kazi` entry in `.claude-plugin/marketplace.json` to the
release tag, in the same CI run that publishes the binary. Marketplace content
can therefore never lag the binary.

The explicit installers (`kazi install-skill`, `kazi init --with-mcp`,
`kazi install-hooks`) remain a parallel channel for other harnesses and for
operators who prefer explicit commands.
