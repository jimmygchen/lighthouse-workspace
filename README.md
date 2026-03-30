# Lighthouse Workspace

## Setup

```bash
git clone https://github.com/sigp/lighthouse-workspace.git
cd lighthouse-workspace
git clone https://github.com/sigp/lighthouse.git
git submodule update --init --recursive
```

## Plugins

Install via `/plugin install` in Claude Code — select from the `lighthouse-workspace` marketplace.

- **review-lh** — `/review-lh:review <PR>`: PR briefing + deep multi-agent review

## Worktrees

```bash
cd lighthouse
git worktree add -b feat-foo ../worktrees/lighthouse-feat-foo unstable
```

Add your fork to push:

```bash
git remote add <you> git@github.com:<you>/lighthouse.git
```
