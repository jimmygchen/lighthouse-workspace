# GitHub Issue Guidelines for Lighthouse

Guidelines for creating well-structured GitHub issues (and PRs) based on patterns from Lighthouse maintainers.

## Issue Structure

### Start with Description

Always begin with a `## Description` section. This is the foundation.

**Guidelines:**
- First paragraph should outline the problem and brief description of the solution
- Provide context about current behavior and why it's an issue
- Link to related issues, PRs, or external resources
- Include relevant error messages, logs, or symptoms
- Be technical and specific - assume reader has Lighthouse context

**Example:**
```markdown
## Description

We presently prune all knowledge of non-canonical blocks once they conflict with
finalization. The pruning is not always immediate, fork choice currently prunes
once the number of nodes reaches a threshold of 256.

It would be nice to develop a simple (easy to reason about) system for handling
messages relating to blocks that are non-canonical.
```

### Add Steps to Resolve (when applicable)

For bugs, features, or improvements where you have implementation ideas:

```markdown
## Steps to resolve

1. [Specific actionable step]
2. [Next step with technical detail]
3. [Final step]
```

**Guidelines:**
- Don't be overly prescriptive - present options and considerations
- If there are multiple approaches, acknowledge them without strongly favoring one
- Mention relevant constraints (e.g., rayon thread pools, existing architecture patterns)
- Examples are helpful but clarify they're just examples
- It's fine to write "Please describe the steps required to resolve this issue, if known" if uncertain

**Example:**
```markdown
## Steps to resolve

I see two ways to fix this: a strict approach, and a pragmatic one.

The strict approach would only check once the slot is finalized. This would have
0 false positives, but would be slower to detect missed blocks.

The pragmatic approach might be to only process `BeaconState`s from the canonical
chain. I don't have a strong preference between approaches.
```

### Optional Sections

Use additional sections as needed:

- `## Additional Info` - Edge cases, related issues, version info
- `## Metrics` - Performance data, observational evidence
- `## Version` - For bug reports
- `## Testing` - How to verify the fix

### Use Bullet Points

Use bullet points for clarity when it makes sense - easier to scan and digest.

## Code References

**Critical:** Use GitHub permalinks with commit hashes (not branch names) so GitHub can render the code properly in issues.

**Format:**
```
https://github.com/sigp/lighthouse/blob/<commit-hash>/<file-path>#L<line-number>
```

**Get commit hash from unstable:**
```bash
git rev-parse unstable
```

**Example:**
```
https://github.com/sigp/lighthouse/blob/261322c3e3ee467c9454fa160a00866439cbc62f/beacon_node/beacon_processor/src/lib.rs#L809
```

**For line ranges:**
```
#L809-L825
```

### When to Include Code References

- Link to the section where the problem exists
- Link to sections that need changing
- Provide reference code to help readers understand context
- Show examples of similar patterns already in the codebase

## Writing Style

### Be Natural and Concise

- Language should be natural, concise, simple
- Avoid sounding AI-generated or overly formal
- Be direct and objective
- Get to the point quickly
- Use precise technical terminology

### Be Honest About Uncertainty

- Don't guess - ask questions if unclear
- Mark things as `TODO` when more investigation is needed
- Use tentative language when appropriate ("might", "likely", "I think")
- It's okay to present multiple options without picking one

**Example:**
```markdown
Before committing to this change, I would be keen to know the time it takes
to decompress the state at startup. Perhaps getting numbers for mainnet and
Prater would be good.
```

### Think About Trade-offs

- Present multiple approaches when applicable
- Discuss pros and cons
- Consider backward compatibility
- Note performance implications
- Mention coordination needs with other clients

## Before Creating Issues

**Research and validate first:**

- Research and validate all claims about the code
- Verify line numbers and file paths are accurate on the unstable branch
- Understand the broader context (don't just focus on one example)
- Check if similar issue already exists
- Ask clarifying questions about timeline, scope, or implementation preferences

## Labels

Use appropriate labels:

**Type:**
- `bug` - Something isn't working
- `enhancement` - New feature or request
- `optimization` - Performance improvements
- `code-quality` - Cleanup or refactoring
- `test improvement` - Improve or add tests
- `security` - Security-related
- `RFC` - Request for comment (community input needed)

**Component:**
- `database`, `HTTP-API`, `UX-and-logs`, `fork-choice`, `dependencies`, `infra-ci`, `hardening`, `beacon-processor`

**Effort:**
- `good first issue` - Good for newcomers
- `low-hanging-fruit` - Easy to resolve
- `major-task` - Significant work required

## Examples

### Bug Report

```markdown
## Description

The validator monitor's monitoring of missed blocks uses arbitrary `BeaconState`s
with a tolerance of `MISSED_BLOCK_LAG_SLOTS = 4` to avoid false positives.

The problem is, this is still vulnerable to false positives when processing
random side chains. E.g. on Hoodi:

> Sep 18 02:36:22.561 ERROR Validator missed a block    slot: 1329123

This is a false positive because that slot was indeed proposed.

## Steps to resolve

Two approaches: strict (only check after finalization) or pragmatic (only process
canonical chain states). I don't have a strong preference.

## Additional Info

Thanks to the community member who found this bug in the logs!
```

### Enhancement

```markdown
## Description

We include uncompressed genesis states in our binary. These total 57.4M, which
goes straight to binary size (currently ~110M). We could significantly reduce
this by storing compressed `genesis.ssz` bytes and decompressing on-demand.

I propose using snappy compression since it's already in the binary for P2P.

Before committing, I'd like to know decompression time at startup.

## Details

The bytes are added here:
https://github.com/sigp/lighthouse/blob/.../common/eth2_config/src/lib.rs#L129-L137

The file is generated here:
https://github.com/sigp/lighthouse/blob/.../build.rs#L30-L47
```

### Simple Coordination Issue

```markdown
## Description

Sepolia is scheduled to fork on Tuesday 14 October:
- https://github.com/eth-clients/sepolia/pull/111
```

## Pull Request Guidelines

PR descriptions should follow similar patterns:

```markdown
## Description

[What does this PR do? Why is it needed?]

Closes #[issue-number]

## Additional Info

[Breaking changes, performance impacts, migration steps, etc.]
```

### Commit Messages

**Format:**
- First line: Brief summary (imperative mood)
- Blank line
- Additional details if needed
- **NO attribution footer** (no "Generated with Claude Code")
- **NO Co-Authored-By lines**

**Good example:**
```
Add custody info API for data columns

Implements `/lighthouse/custody/info` endpoint that returns custody group
count, custodied columns, and earliest available data column slot.
```

## General Tips

1. **One issue per problem** - Don't mix multiple unrelated issues
2. **Update as you learn** - Edit the issue as you discover more
3. **Use checklists** - For multi-step tasks, use `- [ ]` checkboxes
4. **Credit others** - Acknowledge people who helped identify issues
5. **Link liberally** - Reference related issues, PRs, specs, discussions
6. **Think about deployment** - Consider coordination and rollout

## Anti-Patterns

- Vague descriptions without details
- No code references when describing code
- Premature solutions without understanding the problem
- Over-apologizing or demanding urgency
- Making claims without validating against codebase

## Good Examples

Real issues that follow these patterns well:
- https://github.com/sigp/lighthouse/issues/6120
- https://github.com/sigp/lighthouse/issues/4388
- https://github.com/sigp/lighthouse/issues/8216
- https://github.com/sigp/lighthouse/issues/8073
- https://github.com/sigp/lighthouse/issues/4564

