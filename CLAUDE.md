# Lighthouse Dev Workspace

**Context**: This is `<workspace>/`, a workspace for parallel development on Lighthouse using git worktrees.

## First-Time Setup

If `lighthouse/` doesn't exist, clone it before proceeding:

```bash
git clone https://github.com/sigp/lighthouse.git
```

## ⚠️ CRITICAL: Always Read Both Files

**Before implementing any code changes, you MUST read `./lighthouse/CLAUDE.md`** for essential coding standards, error handling patterns, and best practices.

**This file** (worktree): Git workflow, personal preferences, commit/PR guidelines
**Lighthouse file**: Development commands, architecture, code quality standards, testing patterns

## General Behavior

When asked to present a plan or review for user approval, STOP and present it. Do not autonomously explore the repo or begin implementation until the user explicitly approves.

## Project Context

This is a Rust-heavy Ethereum consensus client (Lighthouse) codebase. When debugging or investigating issues, always check the most recent logs first and confirm file paths before proceeding.

## Code Review

- Always fetch the latest diff from GitHub rather than relying on locally cached data. Verify the PR's current state before making claims about unaddressed comments or missing changes.
- Be critically analytical. Do not excuse potential issues just because they "follow existing patterns". Flag real concerns even if the codebase has precedent for the pattern.
- Always verify spec references against the specific spec version or PR the user indicates. Do not default to local/outdated spec copies.

## Directory Structure

```
<workspace>/
├── .cargo/                  # Root cargo config (shared by all worktrees)
│   └── config.toml          # Configures shared target directory (./shared-target)
├── .claude/                 # Claude Code config (commands, settings)
│   └── commands/            # Custom slash commands
├── lighthouse/              # Main repository - use for read-only exploration
│   ├── .git/                # Shared git directory for all worktrees
│   └── CLAUDE.md            # Lighthouse-specific dev docs (read this!)
├── shared-target/           # Shared Rust build artifacts (MUST stay within project root)
└── worktrees/               # All worktrees for active development
    └── lighthouse-<branch>*/
```

**CRITICAL**: `shared-target/` must be inside `<workspace>/` (the project root). Never create or modify directories outside the project root.

## When to Create a Worktree

**Create worktree when:**
- Making code changes (edits, new files, deletions)
- Starting a new feature or bug fix

**Do NOT create worktree when:**
- Just reading/exploring code (use `./lighthouse/`)
- Answering questions about the codebase
- Running searches or grep operations
- **Working on an existing PR** - ask the user where to make changes first!

## Creating a Worktree

**CRITICAL: Before creating a new worktree, ALWAYS:**

1. **Check existing worktrees**: `cd lighthouse && git worktree list`
2. **If working on an existing PR (especially user's own PR):**
   - Ask the user which worktree/location they want to use
   - The user may already have the PR branch checked out
   - DO NOT automatically create a new branch
3. **Only create a new worktree when:**
   - Starting a completely new feature/fix
   - User explicitly requests a new branch
   - No existing worktree is suitable

**Single command to create and configure:**
```bash
cd <workspace>/lighthouse && \
BRANCH="<type>-<short-description>" && \
git worktree add -b "$BRANCH" "../worktrees/lighthouse-$BRANCH" unstable && \
echo "gitdir: ../../lighthouse/.git/worktrees/lighthouse-$BRANCH" > "../worktrees/lighthouse-$BRANCH/.git" && \
cd "../worktrees/lighthouse-$BRANCH"
```

**Branch naming**: `<type>-<short-description>` (e.g., `fix-memory-leak`, `feat-sync-protocol`)

**Notes**:
- The .git fix uses relative path (`../../lighthouse/` because worktrees are nested one level deeper)
- Shared target directory via `<workspace>/.cargo/config.toml` applies automatically (saves 10-20GB per worktree)

## Git Commit Workflow

**Always leave committing to the user.** Stage changes and provide the commit command, but do not commit directly.

**Commit message format** — no Claude Code attribution footers or Co-Authored-By lines:
- First line: Brief summary (imperative mood)
- Blank line + additional details if needed

## Push and Create PR

**CRITICAL**: Never push to `origin` (sigp/lighthouse). Always use your fork remote.

```bash
# Recommended
gh pr create --draft --base unstable --head <your-github-username>:<branch-name> --title "Your PR Title" --body "PR description"

# Alternative
git push -u <your-fork-remote> <branch-name>
```

## Changing Base Branch

1. **Stay in the worktree directory** (not `./lighthouse/`!)
2. Reset and cherry-pick without committing:
   ```bash
   cd <workspace>/worktrees/lighthouse-<branch-name>
   git reset --hard origin/<new-base-branch>
   git cherry-pick --no-commit <commit-hash>
   ```
3. User commits, then force pushes
4. Create new PR against the correct base branch

## Worktree Management

```bash
# List all worktrees
cd lighthouse && git worktree list

# Remove a worktree (when user requests)
cd lighthouse && git worktree remove ../worktrees/lighthouse-<branch-name>

# Clean up stale references
cd lighthouse && git worktree prune
```

**Note**: Leave cleanup decision to the user. Don't auto-remove worktrees.

## Workflow-Specific Guidelines

### For Code Review
- **CRITICAL**: ALWAYS read `./lighthouse/.ai/CODE_REVIEW.md` AND `./lighthouse/CLAUDE.md` before starting any review
- **Format**: Follow the code review guidelines strictly - concise comments with natural, conversational language
- **Code comments**: Comment directly on specific lines using `gh pr review --comment` with file path and line number
  - No limit on number of code comments - add as many as needed for correctness, safety, and clarity issues
- **General review comments**: Use for non-code issues (PR description, missing tracking issues, etc.) - limit to 3-5 items
- **Absolutely NO**:
  - Review summary sections or overall assessments
  - Checkmarks (✅/❌) for positive/negative observations
  - Structured formatting with headers and subsections
  - Verbose explanations with multiple paragraphs
- **User approval required**: Always show proposed comments and wait for user approval before publishing to GitHub

#### PR Review Coverage Strategy

For thorough PR reviews, spawn **two parallel agents split by concern** rather than by crate:

1. **Agent 1 — Security**: Unsafe operations, panic paths, untrusted input handling, resource exhaustion, state corruption, cryptographic misuse, error variants that leak information
2. **Agent 2 — Quality/Resilience**: Error handling patterns, code clarity, test coverage, edge cases, API consistency, graceful degradation, codebase convention adherence

Both agents read the full diff but filter through their respective lens. Overlap between agents (e.g., missing error handling flagged by both) is a signal the issue is worth fixing.

**Split by crate instead only when** the PR is too large for a single agent to hold the full diff in context. In that case, give each per-crate agent both lenses.

### For Issues and PRs

Read `./lighthouse/.ai/ISSUES.md` before creating issues or PRs. Use natural, concise, direct language — no AI-sounding prose, no Claude Code attribution.

## Key Facts

- **Base branch**: Usually `unstable` — **ASK the user first** if targeting a release branch (e.g., `release-v8.1`)
- **Git hooks**: Shared across all worktrees (in `./lighthouse/.git/hooks`)
- **Branch isolation**: Git prevents same branch in multiple worktrees
- **Parallel builds**: Safe across worktrees; Cargo handles locking
