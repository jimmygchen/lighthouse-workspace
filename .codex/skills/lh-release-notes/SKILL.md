---
name: lh-release-notes
description: Generate Lighthouse release notes, update priority guidance, all-changes lists, and announcement drafts from release branches and PR metadata. Use when the user asks for Lighthouse release notes, release announcements, upgrade priority text, or a release summary for a specific version.
---

# Lighthouse Release Notes

Use this skill from `/home/jimmy/workspace/lighthouse-workspace/lighthouse`
unless the user gives another Lighthouse checkout.

## Required Inputs

Collect or infer:

- Version number, for example `v8.1.0`.
- Base branch, usually `stable` for the previous release.
- Release branch, for example `release-v8.1`.
- Release name. It should be a Rick and Morty character and should not duplicate
  recent Lighthouse release names.

Ask for missing required inputs only when they cannot be inferred safely.

## Gather Changes

Fetch current branch and release metadata before making claims:

```bash
git fetch origin <base-branch> <release-branch>
git log --oneline origin/<base-branch>..origin/<release-branch>
gh release list --repo sigp/lighthouse --limit 50
```

Extract PR numbers from commit messages. For each PR, inspect labels and details:

```bash
gh pr view <PR> --repo sigp/lighthouse --json labels --jq '[.labels[].name] | join(",")'
gh pr view <PR> --repo sigp/lighthouse
```

Treat a `backwards-incompat` label as a breaking-change signal, but still verify
the user-facing impact from the PR body and code when needed.

## Categorize

Group notable changes into sections and skip empty sections:

- Breaking Changes: schema changes, CLI changes, API changes, or required
  operator action.
- Performance Improvements: user-noticeable speed, memory, CPU, or IO changes.
- Validator Client Improvements: VC-specific changes.
- Other Notable Changes: features, observability, metrics, and important
  behavior changes.
- CLI Changes: new or changed flags, noting Beacon Node or Validator Client.
- Bug Fixes: significant user-facing fixes only.

Prefer user impact over implementation detail.

## Release Notes Format

Draft release notes in this format:

```markdown
## <Release Name>

## Summary

Lighthouse v<VERSION> includes <brief description>.

This is a <recommended/mandatory> upgrade for <target users>.

## <Section>

- **<Title>** (#<PR>): <User-facing description>

## Update Priority

| User Class        | Beacon Node | Validator Client |
|:------------------|:------------|:-----------------|
| Staking Users     | Low/Medium/High | Low/Medium/High |
| Non-Staking Users | Low/Medium/High | ---              |

## All Changes

- <commit title> (#<PR>)

## Binaries

[See pre-built binaries documentation.](https://lighthouse-book.sigmaprime.io/installation_binaries.html)
```

## Announcement Drafts

After release notes, draft:

- Email: formal, includes the update priority table.
- Discord: shorter, includes `@everyone`.
- Twitter/X: one post with two or three highlights.

## Style

- State user impact, not internal implementation detail.
- Avoid jargon users will not understand.
- Mention whether CLI flags affect Beacon Node or Validator Client.
- Use PR descriptions for context and verify uncertain claims.
- Keep uncertainty explicit when impact or upgrade priority is not clear.
