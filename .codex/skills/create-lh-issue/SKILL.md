---
name: create-lh-issue
description: Draft Lighthouse GitHub issue text using the project's issue-writing conventions, code references, labels, and uncertainty style. Use when the user asks to create, draft, write, or refine a Lighthouse issue, bug report, feature request, or GitHub issue body.
---

# Create Lighthouse Issue

Use this skill from `/home/jimmy/workspace/lighthouse-workspace`.

## Required Context

Before drafting, read:

- `./CLAUDE.md`
- `./lighthouse/CLAUDE.md`
- `./lighthouse/.ai/ISSUES.md`

If `.ai/ISSUES.md` is missing, state that once and use the workspace and
Lighthouse guide as the fallback style source.

## Workflow

1. Understand the issue: problem, affected behavior, expected behavior, user
   impact, and relevant constraints.
2. Inspect relevant code when code references are useful. Use `./lighthouse/`
   for read-only exploration unless the user asks for code changes.
3. Use GitHub permalinks with commit hashes for code references:

```bash
cd lighthouse
git rev-parse unstable
```

4. Draft complete issue text ready to paste into GitHub.

## Issue Structure

Use this shape unless `.ai/ISSUES.md` says otherwise:

```markdown
## Description

<First paragraph: problem and brief solution direction.>

<Context about current behavior, related issues, PRs, specs, and trade-offs.>

## Steps to Resolve

<Options and considerations, without being overly prescriptive.>

## Code References

- <GitHub permalink with commit hash>

## Suggested Labels

- <type/component/effort labels>
```

Omit sections that do not apply, but always include a description.

## Style

- Write natural, concise, direct prose.
- Be technical and specific.
- Be honest about uncertainty.
- Present trade-offs instead of forcing a single solution when the right fix is
  not obvious.
- Avoid AI-sounding filler and attribution.

## Labels

Suggest labels when enough context exists:

- Type: `bug`, `enhancement`, `optimization`, `code-quality`.
- Component: `database`, `HTTP-API`, `fork-choice`, `beacon-processor`, or the
  closest project label.
- Effort: `good first issue`, `low-hanging-fruit`, `major-task`.

## After Feedback

If the user refines the text or gives a format preference:

1. Apply the feedback to the current issue text.
2. Ask whether to update `./lighthouse/.ai/ISSUES.md` to capture the preference.
3. If approved, add the reusable pattern to the docs.
