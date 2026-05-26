# Lighthouse Workspace Agent Guide

This is the workspace root for parallel Lighthouse development using git
worktrees. The actual Lighthouse repository is `./lighthouse/`; active
development worktrees live under `./worktrees/`.

## Required Context

Before making code changes in Lighthouse, read:

- `./CLAUDE.md` for workspace workflow, git, PR, and review conventions.
- `./lighthouse/CLAUDE.md` for Lighthouse coding standards, test commands, and
  architecture notes.

For code review work, also read `./lighthouse/.ai/CODE_REVIEW.md`. For issues
and PR descriptions, read `./lighthouse/.ai/ISSUES.md`.

## Worktree Rules

- Use `./lighthouse/` for read-only exploration.
- Create or use a worktree before editing Lighthouse code.
- Keep all new Lighthouse worktrees under `./worktrees/`.
- Check existing worktrees first:

```bash
cd lighthouse && git worktree list
```

- For a new feature or fix, create from `unstable` unless the user specifies a
  different base:

```bash
cd <workspace>/lighthouse
BRANCH="<type>-<short-description>"
git worktree add -b "$BRANCH" "../worktrees/lighthouse-$BRANCH" unstable
echo "gitdir: ../../lighthouse/.git/worktrees/lighthouse-$BRANCH" > "../worktrees/lighthouse-$BRANCH/.git"
cd "../worktrees/lighthouse-$BRANCH"
```

- For an existing PR, prefer the PR branch if it already exists locally or fetch
  it from the PR head remote, then add a worktree for that branch.
- Do not remove or prune worktrees unless the user asks.

## Shared Build State

The workspace-level `.cargo/config.toml` points all worktrees at
`./shared-target`. Keep `shared-target/` inside this workspace root.

## Git Conventions

- Leave commits to the user unless explicitly asked.
- Never push to `origin` (`sigp/lighthouse`) for personal branches; use the
  user's fork remote.
- Do not add AI attribution footers or `Co-Authored-By` lines.
