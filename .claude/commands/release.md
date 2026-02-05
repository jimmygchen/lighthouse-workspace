# Release Notes Generation Task

You are generating release notes for a new Lighthouse version.

## Input Required

The user should provide:
- **Version number** (e.g., v8.1.0)
- **Base branch** (typically `stable` for the previous release)
- **Release branch** (e.g., `release-v8.1`)
- **Release name** (Rick and Morty character - check existing releases to avoid duplicates)

## Step 1: Gather Changes

```bash
# Get all commits between branches
git log --oneline origin/<base-branch>..origin/<release-branch>

# Check existing release names to avoid duplicates
gh release list --repo sigp/lighthouse --limit 50
```

## Step 2: Analyze PRs

For each PR in the release:

1. **Extract PR numbers** from commit messages
2. **Check for `backwards-incompat` label** - these need special attention:
   ```bash
   gh pr view <PR_NUMBER> --repo sigp/lighthouse --json labels --jq '[.labels[].name] | join(",")'
   ```
3. **Get PR details** for context on user impact:
   ```bash
   gh pr view <PR_NUMBER> --repo sigp/lighthouse --json title,body
   ```
4. **Check linked issues** for additional context when needed

## Step 3: Categorize Changes

Group PRs into sections (skip sections with no items):

### Breaking Changes (if any `backwards-incompat` PRs)
- Database schema changes
- CLI flag changes/removals
- API changes
- Platform support changes

### Performance Improvements
- Optimizations with measurable user impact
- Focus on **what users will notice**, not implementation details

### Validator Client Improvements
- Changes affecting validator duties
- API call reductions
- New VC features

### Other Notable Changes
- New features
- API additions
- New metrics

### CLI Changes
- New flags (specify if BN or VC)
- Changed/deprecated flags

### Bug Fixes (if significant)
- Only include if user-facing
- Trivial fixes can be omitted from highlights

## Step 4: Write Release Notes

**Format:**
```markdown
## <Release Name>

## Summary

Lighthouse v<VERSION> includes <brief description>.

This is a <recommended/mandatory> upgrade for <target users>.

## <Section>

- **<Title>** (#<PR>): <User-facing description of impact>

## Update Priority

| User Class        | Beacon Node | Validator Client |
|:------------------|:------------|:-----------------|
| Staking Users     | <Low/Medium/High> | <Low/Medium/High> |
| Non-Staking Users | <Low/Medium/High> | ---              |

*See [Update Priorities](https://lighthouse-book.sigmaprime.io/installation_priorities.html) for more information.*

## All Changes

- <commit title> (#<PR>)
- ...

## Binaries

[See pre-built binaries documentation.](https://lighthouse-book.sigmaprime.io/installation_binaries.html)

<binaries table - copy from previous release>
```

**Writing Guidelines:**
- State **user impact**, not implementation details
- Avoid technical jargon users won't understand
- If describing a fix, explain what was broken
- For CLI flags, mention if it's BN or VC
- Link PR numbers for reference
- Check PR descriptions and linked issues for context on what to write
- "All Changes" section: list all PRs in commit order (most recent first)
- "Binaries" section: copy table format from previous release, update version numbers

## Step 5: Generate Announcements

Create drafts for:

### Email
- Formal, professional tone
- Include: version, release notes link, summary, highlights, update priority table

### Discord (#announcements)
- Tag @everyone
- Shorter, less formal
- Include: release link, key highlights, recommendation

### Twitter (@sigp_io)
- Single tweet
- Professional but concise
- Include: version, 2-3 key highlights, release link

## Reference: Prior Release Format

Fetch a recent release for format reference:
```bash
gh release view v8.0.0 --repo sigp/lighthouse
```

## Output Files

Save drafts to:
- `release-notes-v<VERSION>.md`
- `release-v<VERSION>-email.md`
- `release-v<VERSION>-discord.md`
- `release-v<VERSION>-twitter.md`
