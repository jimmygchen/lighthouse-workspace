---
name: review-lh
description: Review Lighthouse GitHub pull request code with the code-review workflow, current GitHub state, repository guidance, and Ethereum specification evidence. Use for Lighthouse pull request code reviews and explicit review-lh/lh-review/lh-reeview requests. Do not use for pull request title or body copy, status, links, or metadata unless explicitly invoked.
---

# Lighthouse PR Review Adapter

Use this skill from `/home/jimmy/workspace/lighthouse-workspace`.

The target is a Lighthouse GitHub pull request, usually provided as a number.
This skill supplies Lighthouse context to the `code-review` skill. It does not
define a second review workflow.

## Authoritative Workflow

Load and apply the `code-review` skill. It owns the value and design gates,
fixed review contract, adaptive reviewer topology, candidate verification,
readiness verdict, and convergence rules.

Use the workspace and repository guidance for Lighthouse-specific rules and
publishing constraints. Do not replace `code-review` gates or verification with
reviewer agreement, a fixed reviewer count, file-count thresholds, or
confidence scores.

## Lighthouse Inputs

Read these files before the implementation review:

- `./CLAUDE.md`
- `./lighthouse/CLAUDE.md`
- `./lighthouse/.ai/CODE_REVIEW.md`

If `CODE_REVIEW.md` is missing, state that once. Use the other two files and
the workspace review guidance as the Lighthouse rubric.

Fetch the current pull request metadata from GitHub before the value gate. Do
not treat the pull request description as independent evidence of need:

```bash
gh pr view <PR_NUMBER> --repo sigp/lighthouse \
  --json number,url,title,body,baseRefName,baseRefOid,headRefName,headRefOid,isDraft,state
```

After the value and design gates pass, fetch the current diff from GitHub. Do
not use a cached local diff:

```bash
gh pr diff <PR_NUMBER> --repo sigp/lighthouse
```

Record the exact base and head commits in the fixed review contract. Give every
finder and verifier the same base-to-head diff. Before the final verdict, fetch
the head commit again. If it changed, apply the convergence rules from
`code-review`. Do not combine evidence from different heads.

Use the pull request comment endpoints when `code-review` creates its comment
inventory:

```bash
gh api --paginate repos/sigp/lighthouse/pulls/<PR_NUMBER>/comments
gh api --paginate repos/sigp/lighthouse/pulls/<PR_NUMBER>/reviews
gh api --paginate repos/sigp/lighthouse/issues/<PR_NUMBER>/comments
```

## Specification Evidence

If the change is specification-relevant, load and apply the `eth-spec` skill.
Use it to select and prepare the applicable local specification repository.
Record the repository, fork or API version, and Git revision in the authority
evidence for `code-review`.

Use the exact specification version or upstream pull request that the user
identifies. If the user does not identify one, resolve the authoritative source
under the `code-review` rules. Do not assume that a local checkout is current.

Treat consensus transitions, fork choice, validator duties, networking, SSZ,
Beacon APIs, builder APIs, Engine APIs, and keymanager APIs as
specification-relevant unless the diff proves otherwise.

## Lighthouse Risk Context

Use `lighthouse/.ai/CODE_REVIEW.md` as the project rubric, not as another review
process. Raise the review risk and trigger the applicable `code-review` lenses
for changes to consensus behaviour, validator behaviour, locks or async work,
storage or schemas, networking, public APIs, unsafe code, or serialization.

Compare changed code with analogous Lighthouse code by using `rg` or
`git grep`. Check the production callers and affected fork paths, not only the
changed function.

After the value and design gates pass and the review contract is fixed, give a
concise briefing before candidate finding. Include the change, affected
subsystems, authoritative specification evidence, data flow, triggered risks,
and the planned coverage. Use the final report and readiness format from
`code-review`.

Do not post GitHub review comments or replies without explicit user approval.
