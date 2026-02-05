# Lighthouse Workspace

A development workspace configuration for [Lighthouse](https://github.com/sigp/lighthouse), optimized for use with [Claude Code](https://claude.ai/code).

## What's This?

This repo provides:

- **Git worktree workflow** for parallel Lighthouse development
- **Claude Code skills** for code review, issue creation, and releases
- **Coding guidelines** for Lighthouse contributions

It's designed to sit alongside (not inside) your Lighthouse clone, managing multiple feature branches via git worktrees.

## Directory Structure

```
lighthouse-workspace/
├── .cargo/config.toml       # Shared Rust build target (saves disk space)
├── .claude/commands/        # Claude Code skills
├── CLAUDE.md                # Worktree workflow instructions
├── CODE_REVIEW.md           # Code review guidelines
├── ISSUES.md                # Issue/PR writing guidelines
├── lighthouse/              # Main Lighthouse clone (you create this)
└── lighthouse-<branch>/     # Worktrees for feature branches
```

## Quick Start

```bash
git clone https://github.com/sigp/lighthouse-workspace.git
cd lighthouse-workspace
git clone https://github.com/sigp/lighthouse.git
claude
```

Then ask Claude to review a PR or work on an issue:

```
Review PR #1234
```

```
Look at issue #5678 and implement a fix
```

## Available Skills

- `/review` - Code review with Lighthouse-specific safety checks
- `/issue` - Create well-structured GitHub issues
- `/release` - Generate release notes and announcements

## Git Worktrees

For active development, the workspace uses git worktrees to manage multiple feature branches:

```bash
cd lighthouse
git worktree add -b feat-my-feature ../lighthouse-feat-my-feature unstable
```

All worktrees share a single `shared-target/` directory for Rust builds, saving 10-20GB per branch.

To push changes, add your fork as a remote:

```bash
cd lighthouse
git remote add <your-username> git@github.com:<your-username>/lighthouse.git
```

## Files

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Git worktree workflow, commit guidelines, PR process |
| `CODE_REVIEW.md` | Code review standards and checklist |
| `ISSUES.md` | Guidelines for writing issues and PR descriptions |
| `.claude/commands/` | Claude Code slash command definitions |

## Contributing

This workspace is tailored for Lighthouse development. Feel free to fork and adapt for your own workflow.
