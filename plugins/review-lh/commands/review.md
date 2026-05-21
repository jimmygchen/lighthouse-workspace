---
name: review
description: Review a Lighthouse PR with multi-agent analysis
user_invocable: true
argument_description: "PR number, e.g. '9138'"
---

Review Lighthouse PR: $ARGUMENTS

You are a Lighthouse PR review assistant. Execute the phases below sequentially without stopping for user input between them. The user can interrupt with questions at any point — this is a normal conversation.

**Before starting**: Read `./lighthouse/CLAUDE.md` and `./lighthouse/.ai/CODE_REVIEW.md` for review standards and coding guidelines.

**Spec submodules**: If `plugins/eth-spec/specs/consensus-specs/` is empty, run `git submodule update --init --recursive -- plugins/eth-spec/specs/` from the workspace root. Then pull latest: `git submodule update --remote -- plugins/eth-spec/specs/`.

---

## Phase 1: Briefing

Fetch the PR diff and metadata using `gh pr view` and `gh pr diff`. Output a concise briefing (target: 1-2 minutes to read):

### Summary
What the PR does, why, which subsystems it touches. Assess risk level (low/medium/high) based on: consensus-critical code, lock changes, database schema changes, API surface changes.

### Spec Context
If the PR implements or modifies spec behavior:
- Identify relevant spec sections from `plugins/eth-spec/specs/` (consensus-specs, beacon-APIs, builder-specs, execution-apis, keymanager-APIs)
- Explain what the code *should* do per spec in plain language, not just links
- Note any deviations or implementation choices vs spec

If no spec relevance, say so briefly and move on.

### Design
How the pieces fit together: data flow, key abstractions, non-obvious interactions between changed components. Keep it structural, not line-by-line.

### Review Guide
- **Reading order**: which files to read first and why
- **Key areas**: where bugs are most likely to hide
- **Edge cases**: specific scenarios to watch for

After outputting the briefing, say: *"Starting deep review now. I'll present findings when done."*

---

## Phase 2: Deep Review

Spawn a Claude team for thorough parallel review. Follow the PR Review Coverage Strategy from CLAUDE.md.

> **Why multi-agent**: A single agent reviewing a full diff tends to lose focus and miss details. Splitting by concern keeps each agent focused on a narrower lens. Do not consolidate into one agent.

**For normal PRs** — split by concern (3 agents):
- **Agent 1 — Security**: Unsafe operations, panic paths (.unwrap(), .expect(), array indexing), untrusted input handling, resource exhaustion, state corruption, cryptographic misuse, error variants that leak information, lock safety and deadlock potential
- **Agent 2 — Quality/Resilience**: Error handling patterns (silent swallowing, missing context), code clarity and naming, test coverage gaps, edge cases, API consistency, graceful degradation, codebase convention adherence, safe math in consensus code
- **Agent 3 — Simplification**: Missed reuse of existing helper methods or trait implementations in the codebase (grep for them), redundant match arms that could use wildcards, manual iteration replaceable by iterator combinators, duplicated logic across the PR that should be extracted, unnecessary clones or allocations, over-engineering relative to what the PR needs to do. Suggest concrete rewrites with code snippets, not vague "consider simplifying" notes. Reference CODE_REVIEW.md anti-patterns as your primary rubric.

**For large PRs** (10+ changed files spanning multiple subsystems) — split by crate/layer instead, with all three lenses per agent.

**Each agent must**:
- Read `./lighthouse/CLAUDE.md` and `./lighthouse/.ai/CODE_REVIEW.md` first
- Fetch the full PR diff with `gh pr diff $PR_NUMBER`
- Check spec compliance against `plugins/eth-spec/specs/` submodules where relevant
- Compare new code against analogous existing patterns in the codebase
- Apply the "Before Approval Checklist" from CODE_REVIEW.md

**Present findings in this format:**

### Coverage Map
Which areas were thoroughly checked vs areas that need more human attention (e.g., "Networking changes: high confidence. Consensus math: reviewed but complex - recommend manual verification.").

### Issues (must fix)
Correctness bugs, panics, security problems, consensus safety violations, missing error handling that could cause data loss. Each finding: `file:line` — what's wrong, why it matters.

### Suggestions (should consider)
Design improvements, missing test coverage, unclear naming, non-idiomatic patterns. Each finding: `file:line` — what could be better, concrete suggestion.

### Observations (informational)
Style notes, minor improvements, things that are fine but worth noting. Keep brief.

For any category with no findings, output "Nothing found" so the user knows it was checked.

**Important**: Each review agent must include evidence inline with each finding (e.g., "this duplicates `fn foo` at `bar.rs:42`", "CODE_REVIEW.md requires X") so that confidence scorers can evaluate without needing codebase access.

**Do not present raw agent findings to the user.** Collect them internally and pass them to confidence scoring below. Only the filtered results are shown.

---

## Phase 2.5: Confidence Scoring

The **main orchestrating agent** (not the review agents) performs this step after collecting all agent results. If no Issues or Suggestions were found across all agents, skip this phase and proceed to Phase 3.

1. **Deduplicate**: Merge findings that reference the same code location and issue. When multiple agents independently flagged the same problem, note the overlap and keep the most detailed version.
2. Collect all Issues and Suggestions into a numbered list. Observations pass through unscored.
3. Spawn parallel **Haiku agents** to score confidence 0-100. Batch findings per review agent (one Haiku call per agent's findings, not one per finding). Each scorer receives the agent's findings with their inline evidence, the relevant diff hunks, and the CODE_REVIEW.md. Use this rubric (pass it verbatim to each scorer):
   - **0**: False positive. Doesn't hold up to scrutiny, or is a pre-existing issue not introduced by this PR.
   - **25**: Might be real, but could also be a false positive. Unable to verify. Stylistic issues not explicitly called out in CODE_REVIEW.md.
   - **50**: Real issue, but minor or unlikely to hit in practice. Not important relative to the rest of the PR.
   - **75**: Very likely real. The evidence provided by the reviewer confirms the issue. Important and will directly impact functionality, or explicitly mentioned in CODE_REVIEW.md/CLAUDE.md.
   - **100**: Definitely real. Strong evidence confirms this. Will happen frequently in practice.
4. **Filter out findings scoring below 75.** For each surviving finding, append its score in brackets, e.g. `[confidence: 85]`.
5. Present the filtered findings to the user using the Coverage Map / Issues / Suggestions / Observations format from Phase 2.

**False positive examples to filter:**
- Pre-existing issues not introduced by this PR
- Issues a compiler, linter, or type checker would catch
- Pedantic nitpicks a senior Rust engineer wouldn't flag
- General quality concerns not backed by CODE_REVIEW.md
- Changes in functionality that are clearly intentional
- Issues on lines the PR did not modify

---

## Phase 3: Comment Review

After deep review completes (to avoid biasing findings):

1. Fetch all existing PR comments and review threads:
   ```
   gh api repos/{owner}/{repo}/pulls/{number}/comments
   gh api repos/{owner}/{repo}/pulls/{number}/reviews
   gh api repos/{owner}/{repo}/issues/{number}/comments
   ```

2. Present separately from your findings:
   - **Unresolved comment threads**: summarize each with context (who said what, what's pending)
   - **Overlap**: note where your findings overlap with existing comments (to avoid double-commenting)
   - **Resolved threads**: briefly note count, skip details

3. Ask: *"Want me to post any findings as review comments, or reply to any open threads?"*
