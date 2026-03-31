# Lighthouse Workspace

Personal development workspace and tooling for working on [Lighthouse](https://github.com/sigp/lighthouse).

## What's in This Repo

- **`lighthouse/`** — Lighthouse repo (cloned separately). Main working directory for development and IDE usage.
- **`worktrees/`** — Git worktrees for parallel branches, managed by Claude Code
- **`shared-target/`** — Shared Rust build artifacts across `lighthouse/` and all worktrees (configured via `.cargo/config.toml`)
- **`plugins/`** — Claude Code plugins (install via `/plugin install`)
  - [`review-lh`](plugins/review-lh) — Lighthouse PR review workflow with multi-agent analysis. Bundles consensus-specs for spec cross-referencing.
  - [`kurtosis-verify`](plugins/kurtosis-verify) — Spin up a Kurtosis local testnet and verify behavior via traces, metrics, logs, or beacon API
  - [`remote-verify`](plugins/remote-verify) — Verify behavior on remote nodes via SSH and Grafana
- **`.claude/`** — Claude Code config and custom commands
  - [`/deploy-pr-test`](.claude/commands/deploy-pr-test.md) — Provision a Hetzner dev box, start a Kurtosis testnet with a PR branch, and optionally verify behavior

## Setup

```bash
git clone https://github.com/sigp/lighthouse-workspace.git
cd lighthouse-workspace
git submodule update --init --recursive
```

`lighthouse/` will be cloned automatically by Claude Code on first use if it doesn't exist.