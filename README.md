# Lighthouse Workspace

Personal development workspace and tooling for working on [Lighthouse](https://github.com/sigp/lighthouse).

## What's in This Repo

- **`lighthouse/`** — Lighthouse repo (cloned separately). Main working directory for development and IDE usage.
- **`worktrees/`** — Git worktrees for parallel branches, managed by Claude Code
- **`shared-target/`** — Shared Rust build artifacts across `lighthouse/` and all worktrees (configured via `.cargo/config.toml`)
- **`plugins/`** — Claude Code plugins (e.g., `review-lh` for PR reviews). Includes Ethereum spec submodules.
- **`.claude/`** — Claude Code config and custom commands

## Setup

```bash
git clone https://github.com/sigp/lighthouse-workspace.git
cd lighthouse-workspace
git submodule update --init --recursive
```

`lighthouse/` will be cloned automatically by Claude Code on first use if it doesn't exist.

## Plugins

Install via `/plugin install` in Claude Code — select from the `lighthouse-workspace` marketplace.
