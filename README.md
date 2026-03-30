# Lighthouse Workspace

A development workspace for [Lighthouse](https://github.com/sigp/lighthouse), optimized for parallel development with git worktrees and [Claude Code](https://claude.ai/code).

## Directory Structure

```
lighthouse-workspace/
├── .cargo/config.toml       # Shared Rust build target (saves disk space)
├── CLAUDE.md                # Worktree workflow, commit/PR guidelines
├── lighthouse/              # Main Lighthouse clone (you create this)
├── shared-target/           # Shared Rust build artifacts
└── worktrees/               # Worktrees for feature branches
    └── lighthouse-<branch>/
```

Coding guidelines, review standards, and Claude Code skills live in the lighthouse repo under `.ai/` and `.claude/`.

## Quick Start

```bash
git clone https://github.com/sigp/lighthouse-workspace.git
cd lighthouse-workspace
git clone https://github.com/sigp/lighthouse.git
claude
```

## Plugins

The workspace includes Claude Code plugins under `plugins/`. To install:

```bash
# Initialize spec submodules (one-time)
git submodule update --init --recursive

# Install the review plugin via /plugin install in Claude Code
# Select "review-lh" from the "Lighthouse Workspace Plugins" marketplace
```

Available plugins:
- **review-lh** — `/review-lh:review <PR>`: Generates a PR briefing, runs deep multi-agent review (security + quality), then summarizes existing comment threads.

## Git Worktrees

The workspace uses git worktrees to manage multiple feature branches simultaneously:

```bash
cd lighthouse
git worktree add -b feat-my-feature ../worktrees/lighthouse-feat-my-feature unstable
```

All worktrees share a single `shared-target/` directory for Rust builds, saving 10-20GB per branch.

To push changes, add your fork as a remote:

```bash
cd lighthouse
git remote add <your-username> git@github.com:<your-username>/lighthouse.git
```
