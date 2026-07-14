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

## Lighthouse Performance Experiments

Before running an expensive benchmark or E2E measurement, trace the real
production caller and scheduler. Classify the measured work as block-import
critical, synchronous fallback, or proactive/background work, and record:

- the representative production input and execution cadence;
- the span or timer that directly measures the optimized work;
- the production claim the measurement can support;
- the evidence needed to stop the investigation.

Prefer the smallest measurement that answers the open question. Do not add a
secondary harness when existing evidence is already sufficient unless it
resolves a specific uncertainty. Use Lighthouse symbols and established terms
in reports, and state clearly when a result does not measure normal block
processing or another critical path.

## Production PR Readability

Before the first production commit or push, review the final diff separately
from the experiment history:

- Keep temporary workflows, tracing, profiling drivers, orchestration, and
  other experiment-only code off the production branch.
- Remove redundant generated tests and unexplained test data. Keep the smallest
  test matrix that protects the relevant behavior and boundaries.
- Add comments only where they explain a non-obvious reason or invariant. Prefer
  clear names over comments that restate the code.
- Use exact Lighthouse names and terminology instead of introducing new phrases.
- Keep PR titles and bodies concise and readable. Focus on the production change,
  relevant performance evidence, and material trade-offs. Include routine test
  lists only when they add reviewer value.

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
