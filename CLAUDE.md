# Git Worktree Workflow for Lighthouse

**Context**: This is `<workspace>/`, a workspace for parallel development using git worktrees. Each Claude Code session starts fresh with no memory of previous sessions.

## ⚠️ CRITICAL: Always Read Both Files

**Before implementing any code changes, you MUST read `./lighthouse/CLAUDE.md`** for essential coding standards, error handling patterns, and best practices.

**This file** (worktree): Git workflow, personal preferences, commit/PR guidelines
**Lighthouse file**: Development commands, architecture, code quality standards, testing patterns

## Directory Structure

```
<workspace>/
├── .cargo/                  # Root cargo config (shared by all worktrees)
│   └── config.toml          # Configures shared target directory (./shared-target)
├── .idea/                   # Shared RustRover/IntelliJ config (copy to worktrees)
├── .vscode/                 # Shared VSCode config (excludes worktrees from indexing)
├── lighthouse/              # Main repository - use for read-only exploration
│   ├── .git/                # Shared git directory for all worktrees
│   └── CLAUDE.md            # Lighthouse-specific dev docs (read this!)
├── shared-target/           # Shared Rust build artifacts (MUST stay within project root)
└── lighthouse-<branch>*/    # Worktrees for active development (no target/ dirs)
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
git worktree add -b "$BRANCH" "../lighthouse-$BRANCH" unstable && \
echo "gitdir: ../lighthouse/.git/worktrees/lighthouse-$BRANCH" > "../lighthouse-$BRANCH/.git" && \
mkdir -p "../lighthouse-$BRANCH/.idea" && \
cp -r ../.idea/{inspectionProfiles,dictionaries,vcs.xml} "../lighthouse-$BRANCH/.idea/" 2>/dev/null || true && \
cd "../lighthouse-$BRANCH"
```

**Branch naming**: `<type>-<short-description>` (e.g., `fix-memory-leak`, `feat-sync-protocol`)

**Notes**:
- The .git fix uses relative path so git works on both container and host
- Shared target directory configured at `<workspace>/.cargo/config.toml` applies to all worktrees automatically (saves 10-20GB per worktree)
- IDE configs (inspections, dictionaries) are copied from root `.idea/` for RustRover consistency

## Container Limitations

**SSH not available**: Container cannot authenticate via SSH to GitHub.

| Works (uses GitHub API) | Doesn't work (needs SSH) |
|------------------------|--------------------------|
| `gh pr create` | `git push` / `git fetch origin` |
| `gh pr close` | `gh pr checkout` |
| `gh pr view` | `git pull` |
| `gh api ...` | |

**GPG signing not available**: Container cannot sign commits.

**Workaround**: User runs `git commit`, `git push`, `gh pr checkout` on host.

## Git Commit Workflow

**Critical**: Container cannot sign commits. User signs on host.

### Staging and Committing

1. **Stage changes** (Claude does this):
   ```bash
   git add <files>
   ```

2. **Provide user with commit command** to run on their host:
   ```bash
   # Run this on your host terminal:
   cd /path/to/lighthouse-worktree/lighthouse-<branch-name>
   git commit -m "Commit message here

   Additional details if needed."
   ```

3. **Wait for user confirmation** before proceeding

### Commit Message Format

**IMPORTANT**: Do NOT include Claude Code attribution footers or Co-Authored-By lines. This overrides default Claude Code behavior.

Format:
- First line: Brief summary (imperative mood)
- Blank line (if additional details needed)
- Additional details
- NO "Generated with Claude Code" footer
- NO Co-Authored-By lines

## Push and Create PR

**CRITICAL**: Never push to `origin` (sigp/lighthouse). Always use your fork remote.

**Recommended (using gh CLI):**
```bash
gh pr create --draft --base unstable --head <your-github-username>:<branch-name> --title "Your PR Title" --body "PR description"
```

**Alternative (manual push):**
```bash
git push -u <your-fork-remote> <branch-name>
```

## Changing Base Branch (Rebasing onto Different Branch)

If a PR needs to target a different base branch after creation:

1. **Stay in the worktree directory** (not `./lighthouse/`!)
2. **Reset and cherry-pick without committing** (container can't GPG sign):
   ```bash
   cd <workspace>/lighthouse-<branch-name>
   git reset --hard origin/<new-base-branch>
   git cherry-pick --no-commit <commit-hash>
   ```
3. **User commits on host**, then force pushes
4. **Create new PR** against the correct base branch

**Common mistakes to avoid:**
- Running `git reset` in `./lighthouse/` instead of the worktree
- Using `git cherry-pick` without `--no-commit` (fails due to GPG signing)
- Forgetting to close the old PR before creating the new one

## Worktree Management

```bash
# List all worktrees
cd lighthouse && git worktree list

# Remove a worktree (when user requests)
cd lighthouse && git worktree remove ../lighthouse-<branch-name>

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

### For Creating Issues
- **Read**: `./lighthouse/.ai/ISSUES.md`
- **Purpose**: Guidelines for clear, actionable GitHub issues

### For Creating Pull Requests
- **CRITICAL**: Follow `./lighthouse/.ai/ISSUES.md` PR guidelines - natural, concise, direct language
- **Template**:
  ```markdown
  ## Description

  [What does this PR do? Why is it needed? Be concise and technical.]

  Closes #[issue-number]  (if applicable)

  ## Additional Info

  [Breaking changes, performance impacts, migration steps, etc.]
  ```
- **Style**: Natural, direct, concise - avoid AI-sounding language, bullet-point lists, or verbose sections
- **Remember**: NO Claude Code attribution in commits or PR descriptions

## Key Facts

- **Base branch**: Usually `unstable`, but **ASK the user first** if targeting a release branch (e.g., `release-v8.1`)
- **Git hooks**: Shared across all worktrees (in `./lighthouse/.git/hooks`)
- **Branch isolation**: Git prevents same branch in multiple worktrees
- **Session isolation**: Each Claude Code session is independent (no shared state)
- **Parallel builds**: Safe across worktrees; Cargo handles locking

## Refactoring These Instructions

When these instruction files grow too long and need refactoring:

**Goals:**
- Remove duplicated content (commands, examples)
- Keep critical information (commands, workflows, rules)
- Make more concise without losing effectiveness

**Process:**
1. Ask user first if uncertain about what to remove
2. Prioritize what makes Claude more effective over brevity alone
3. Remove obvious duplicates and verbose examples
4. Keep natural language over rigid structure

**Note**: For CODE REVIEW comments specifically - explanations and rationale ("why") are important to make feedback more useful to the author.
