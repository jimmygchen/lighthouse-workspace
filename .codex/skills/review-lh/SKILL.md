---
name: review-lh
description: Review Lighthouse GitHub pull requests from the Lighthouse workspace with a current PR briefing, spec-aware analysis, focused multi-pass or multi-agent review, confidence filtering, and existing comment triage. Use when the user asks to review a Lighthouse PR, run the Lighthouse review workflow, or invokes review-lh/lh-review/lh-reeview.
---

# Lighthouse PR Review

Use this skill from `/home/jimmy/workspace/lighthouse-workspace`.

The target is a Lighthouse GitHub PR, usually provided as a number. Always fetch
the current PR state with `gh`; do not rely on cached local diffs.

## Required Context

Before reviewing, read:

- `./CLAUDE.md`
- `./lighthouse/CLAUDE.md`
- `./lighthouse/.ai/CODE_REVIEW.md` if it exists

If `CODE_REVIEW.md` is missing, state that once and use `CLAUDE.md`, the
workspace review conventions, and Lighthouse coding standards as the review
rubric.

For spec-relevant changes, use the `eth-spec` skill workflow and local specs
under `plugins/eth-spec/specs/`. If the specs are missing, initialize them:

```bash
git submodule update --init --recursive -- plugins/eth-spec/specs/
```

If the user asks for latest/current spec state, update them:

```bash
git submodule update --remote -- plugins/eth-spec/specs/
```

## Phase 1: Briefing

Fetch PR metadata and diff:

```bash
gh pr view <PR_NUMBER> --repo sigp/lighthouse
gh pr diff <PR_NUMBER> --repo sigp/lighthouse
```

Give a concise briefing before deep review:

- Summary: what the PR changes, why, touched subsystems, and risk level
  (low/medium/high). Raise risk for consensus-critical code, locks, database
  schema changes, API surface changes, networking, storage, unsafe code, or
  behavior that affects validators.
- Spec context: relevant spec files and the rule the implementation should
  satisfy. If no spec relevance is apparent, say so briefly.
- Design: data flow, key abstractions, and non-obvious component interactions.
- Review guide: recommended reading order, likely bug locations, and concrete
  edge cases to check.

End the briefing with: "Starting deep review now. I'll present findings when done."

## Phase 2: Deep Review

Use parallel sub-agents when available. Discover multi-agent tools with
`tool_search` if they are not already exposed. If sub-agents are unavailable,
perform the same passes yourself, keeping the lenses separate.

For normal PRs, split by concern:

- Security: unsafe operations, panics (`unwrap`, `expect`, indexing), untrusted
  input, resource exhaustion, state corruption, cryptographic misuse, leaked
  sensitive information, lock safety, and deadlock potential.
- Quality/resilience: error handling, context in errors, code clarity, naming,
  edge cases, API consistency, graceful degradation, convention adherence, test
  coverage, and safe math in consensus code.
- Simplification: existing helper reuse, duplicated logic, redundant match arms,
  iterator opportunities, unnecessary clones or allocations, and over-engineered
  control flow. Suggestions must include concrete rewrites or exact existing
  helpers where possible.

For large PRs with 10+ changed files spanning multiple subsystems, split by
crate or layer instead, and apply all three lenses in each pass.

Each pass must:

- Read the required context files first.
- Inspect the full current diff using `gh pr diff`.
- Compare new code against analogous Lighthouse patterns using `rg`/`git grep`.
- Check spec compliance against local specs when relevant.
- Focus on issues introduced or exposed by the PR.

Do not show raw pass or agent output. Consolidate it first.

## Phase 2.5: Confidence Filtering

Before presenting findings:

1. Deduplicate issues that reference the same location or root cause.
2. Discard false positives, pre-existing issues not introduced by the PR,
   compiler/linter/type-checker catches, and intentional behavior changes.
3. Score each remaining issue from 0 to 100:
   - 0: false positive.
   - 25: uncertain or stylistic without local rubric support.
   - 50: real but minor or unlikely.
   - 75: very likely real and important.
   - 100: definitely real and likely to occur.
4. Show only findings with confidence >= 75, appending `[confidence: N]`.

Present results in this order:

- Coverage Map: what was thoroughly checked and where residual human attention
  is useful.
- Issues (must fix): correctness, security, consensus safety, panic, data loss,
  or serious error handling problems.
- Suggestions (should consider): tests, design, naming, idioms, clarity, or
  simplification backed by concrete evidence.
- Observations: brief informational notes.

For an empty category, write "Nothing found."

Each finding should use `file:line` and explain what is wrong, why it matters,
and the concrete fix or validation needed.

## Phase 3: Existing Comment Review

After the deep review, fetch existing PR comments and reviews:

```bash
gh api repos/sigp/lighthouse/pulls/<PR_NUMBER>/comments
gh api repos/sigp/lighthouse/pulls/<PR_NUMBER>/reviews
gh api repos/sigp/lighthouse/issues/<PR_NUMBER>/comments
```

Present this separately from your findings:

- Unresolved comment threads: summarize who said what and what is pending.
- Overlap: note where your findings are already covered.
- Resolved threads: give a count and skip details unless needed.

Finish by asking whether the user wants any findings posted as GitHub review
comments or replies. Do not post comments without explicit approval.
