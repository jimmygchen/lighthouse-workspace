# Lighthouse Workspace

Personal development workspace and tooling for working on [Lighthouse](https://github.com/sigp/lighthouse).

## What's in This Repo

- **`lighthouse/`** — Lighthouse repo (cloned separately). Main working directory for development and IDE usage.
- **`worktrees/`** — Git worktrees for parallel branches, managed by Claude Code
- **`shared-target/`** — Shared Rust build artifacts across `lighthouse/` and all worktrees (configured via `.cargo/config.toml`)
- **`plugins/`** — Claude Code plugins (install via `/plugin install`)
  - [`/review-lh:review`](plugins/review-lh) — Lighthouse PR review workflow with multi-agent analysis. Bundles consensus-specs for spec cross-referencing.
  - [`/node-check:kurtosis`](plugins/node-check) — Spin up a Kurtosis local testnet and verify behavior via traces, metrics, logs, or beacon API
  - [`/node-check:remote-ssh`](plugins/node-check) — Verify behavior on remote nodes via SSH and Grafana
- **`.claude/`** — Claude Code config and custom commands
  - [`/deploy-pr`](.claude/commands/deploy-pr.md) — Provision a Hetzner dev box, start a Kurtosis testnet with a PR branch, and optionally verify behavior

## Setup

```bash
git clone https://github.com/sigp/lighthouse-workspace.git
cd lighthouse-workspace
git submodule update --init --recursive
```

`lighthouse/` will be cloned automatically by Claude Code on first use if it doesn't exist.

## Usage

Claude Code automatically manages worktrees — just describe what you want to do:

```
# Check out a PR into a worktree
> checkout #9138 into a worktree

# Implement a feature (automatically creates a worktree)
> add a --foo flag to the beacon node CLI

# Review a PR
> /review-lh:review 9138

# Spin up a local testnet and verify behavior
> /node-check:kurtosis blobs are available on the beacon API after 2 epochs

# Deploy a PR branch to a remote testnet
> /deploy-pr 9138
```

For read-only exploration (searching code, answering questions), Claude Code uses `lighthouse/` directly without creating a worktree.