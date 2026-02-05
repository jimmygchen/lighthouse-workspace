# Lighthouse Code Review Guidelines

This document provides code review guidelines for the Lighthouse project based on analysis of actual PR reviews from the past year. These patterns reflect the team's standards and expectations.

## Overview

Code reviews in Lighthouse emphasize:
- **Correctness** over clever code
- **Clarity** through good documentation and naming
- **Thoroughness** in considering edge cases
- **Safety** through proper error handling and panic avoidance
- **Maintainability** for long-term health

## Critical: Consensus Crate (`consensus/` excluding `types/`)

**The consensus crate requires extra scrutiny** as it implements Ethereum specification logic. Small bugs here can cause consensus failures.

### Requirements

**Use Safe Math Operations**
- ALWAYS use `saturating_*` or `checked_*` arithmetic operations
- Never use standard arithmetic operators that can panic
- Critical that this crate behaves deterministically with NO undefined behavior

```rust
// ❌ NEVER do this in consensus crate
let result = a + b;
let value = epoch * slots_per_epoch;

// ✅ ALWAYS do this
let result = a.saturating_add(b);
let value = epoch.saturating_mul(slots_per_epoch);

// ✅ Or use safe_arith crate
use safe_arith::SafeArith;
let result = a.safe_add(b)?;
```

**Zero Panics**
- NO `.unwrap()`, `.expect()`, array indexing `[i]`, or operations that could panic
- Return `Result` or `Option` instead
- This is even more critical than in other parts of the codebase

```rust
// ❌ NEVER
let item = array[1];
let value = some_option.unwrap();

// ✅ ALWAYS
let item = array.get(1)?;
let value = some_option.ok_or(Error::MissingValue)?;
```

**Deterministic Behavior**
- Code must produce identical results across all platforms
- No platform-specific behavior
- No undefined behavior of any kind

## Panic Avoidance (All Code)

**Panics should be avoided at all costs throughout the codebase.**

### Rules
- NEVER use `.unwrap()` or `.expect()` at runtime
- ALWAYS prefer `array.get(index)?` over `array[index]`
- Return `Result` or `Option` instead of panicking
- Use proper error handling with `Result` types

### Exceptions
**Only acceptable during startup:**
- CLI flag validation
- Configuration parsing
- Initial setup validation

```rust
// ✅ Acceptable during startup
let config_value = matches.get_one::<String>("required-flag")
    .expect("Required flag must be present due to clap validation");

// ✅ If you MUST make runtime assumptions, document thoroughly
let item = array.get(1).expect("Array always has at least 2 elements due to validation in constructor");
// IMPORTANT: Detailed reasoning MUST be provided in nearby comments explaining why this is safe
```

## Documentation & Code Clarity

### Fix Typos and Grammar
- Typos in comments, documentation, and variable names should be corrected
- Grammar matters in documentation

### Variable Naming: Avoid Ambiguous Abbreviations

**Bad:**
```rust
// What do these mean?
let bb = ...;
let bl = ...;
let bbrange_queue = ...;
let blbrange_queue = ...;
```

**Good:**
```rust
// Clear and unambiguous
let beacon_block = ...;
let blob = ...;
let block_brange_queue = ...;
let blob_brange_queue = ...;
```

**Review Pattern:**
- Question confusing variable names
- Suggest clearer alternatives
- Example from actual review: "fix: clarify `bb` vs `bl` variable names"

### Write Clear Comments

- Explain the "why" not just the "what"
- Document non-obvious behavior and edge cases
- Clarify intent when logic is complex

**Good:**
```rust
// Ensure the earliest columns we serve are within the data availability window
if earliest_custodied_slot < column_data_availability_boundary_slot {
    column_data_availability_boundary_slot
} else {
    earliest_custodied_slot
}
```

**Better:**
```rust
// Request blocks only from peers of this specific chain
&self.peers,
// Request columns from all synced peers, even if they are not part of this chain. Since we request by root
// this is safe. We request from peers outside the chain to reduce the chance of getting stuck in the case
// this chain is useful but it has very few peers that agree on the exact same head root.
&synced_column_peers,
```

### Remove Dead Code
- Delete commented-out code rather than leaving it in place
- Don't add `// removed` comments for deleted code
- Trust version control for history

### TODOs Require GitHub Issues

**MANDATORY:** All `TODO` comments must link to a GitHub issue.

```rust
// ❌ NEVER
// TODO: Implement proper validation here

// ✅ ALWAYS
// TODO: Implement proper validation here
// https://github.com/sigp/lighthouse/issues/1234
pub fn my_function(&mut self, _something: &[u8]) -> Result<String, Error> {
    Ok("unimplemented".to_string())
}
```

## Error Handling

### Don't Silently Swallow Errors

Always log errors, even if you handle them gracefully. Consider the appropriate log level.

**Needs Improvement:**
```rust
self.store
    .get_data_column_custody_info()
    .unwrap_or(None)
```

**Better:**
```rust
self.store
    .get_data_column_custody_info()
    .unwrap_or_else(|e| {
        error!(self.log, "Failed to read custody info"; "error" => ?e);
        None
    })
```

### Handle Edge Cases
- Think through what happens at boundaries
- Consider empty states, missing data, concurrent operations
- Document assumptions

**Good review question:**
> "Should we check the returned outcome here? What happens if it returns `Ok(BatchState::Failed)`?"

### Check Return Values

Don't ignore results that might indicate failure.

**Review pattern:**
```rust
// Does this need checking?
batch.download_failed(None)?;

// Consider: What if it returns Failed state?
```

## Performance & Concurrency

### Lock Ordering and Contention

**Critical**: Take great care to avoid deadlocks when working with locks. Seek detailed review for any lock-related changes.

- Document lock ordering requirements
- Keep lock scopes as narrow as possible
- Consider the order of lock acquisition

**Example Discussion from Actual Review:**
> "This introduces a new read of the `store.split` `RwLock` in this function, prior to reading the `canonical_head` `RwLock`. The risk here is... However, using the split over the head may be preferable, as there is more write-contention for the head."

### When to Use try_read

- Use `try_read` when falling back to an alternative is acceptable
- Use blocking `read` when the alternative is more expensive (e.g., state reconstruction)

**From actual review:**
> "There's no deadlock risk. I figure try_read isn't worth it because the alternative is doing state reconstruction. We may as well wait for the split lock, as it will be less work in total."

### Rayon Thread Pool Usage

- Avoid using the rayon global thread pool
- Use scoped rayon pools started by beacon processor for computational intensive tasks
- Prevents CPU oversubscription when beacon processor has fully allocated CPUs

## Async Patterns

### Never Block in Async Context

**❌ Bad:**
```rust
async fn some_handler() {
    let result = expensive_computation(); // blocks async runtime
}
```

**✅ Good:**
```rust
async fn some_handler() {
    let result = tokio::task::spawn_blocking(|| {
        expensive_computation()
    }).await?;
}
```

## Architecture & Design

### Avoid Dependency Bloat

- Consider whether imports add unnecessary dependencies
- Question large imports when only primitives are needed
- Consider feature flags for optional functionality

**Example from review:**
> "These `types` imports add a lot of bloat to this crate... might be worth considering some kind of `core` or `primitives` feature in `types`"

### Schema Migrations

- Database schema changes require migrations
- Don't forget to add migration code when changing stored types
- **Review pattern:** "Needs a schema migration, see my comment"

### Backwards Compatibility

- Consider existing users when changing behavior
- Document breaking changes clearly
- Prefer additive changes when possible

## Review Process Etiquette

### Giving Feedback

#### Claude Comments: Keep Natural and Minimal

**When Claude Code writes review comments:**
- Keep tone natural and conversational, not robotic
- Avoid checklists or structured formatting in follow-up comments

**Example of initial review comment:**
```
Missing test coverage for the None blobs path. The existing test at
`store_tests.rs:2874` still provides blobs. Should add a test passing
None to verify backfill handles this correctly.
```

**Example of follow-up after author addresses comments:**
```
LGTM, thanks!
```
or
```
Thanks for the updates, looks good!
```

**Avoid in follow-ups:**
- Repeating what was fixed, even conversationally (makes it obvious it's AI-generated)
- Checklists of fixed items (✅ Item 1 fixed, ✅ Item 2 fixed...)
- Overly structured or formatted responses

#### Focus on Actionable Issues

**Limit comments to 3-5 key issues.** Prioritize:
1. **Correctness issues** - bugs, race conditions, panics
2. **Missing test coverage** - especially for edge cases
3. **Complex logic needing documentation** - help future maintainers
4. **API design concerns** - before they become locked in

**Avoid commenting on:**
- Minor style issues if the code is otherwise good
- Things that are nice-to-have but not important
- Issues already caught by CI (formatting, linting)

**Example of good prioritization:**
```
✅ Comment on:
- Missing null check that could cause panic
- Complex graffiti truncation logic without explanation
- Edge case with empty user input not tested

❌ Don't comment on:
- Variable could have a slightly better name
- Function could be split into two functions
- Code could be more idiomatic
```

#### Verify Assumptions Before Commenting

**If CI is passing, trust it.** Don't comment about missing types, imports, or dependencies that must exist for tests to pass.

**Check the full diff, not just what you can see:**
- If you see an import but not the definition, assume it's defined elsewhere
- If tests reference new functionality, assume it's wired up correctly
- Focus on logic errors, not integration issues caught by CI

**Bad example:**
```
❌ "Cannot find the GraffitiPolicy type definition. Please add it."
(CI is passing, so it must exist)
```

**Good example:**
```
✅ "Please verify that graffiti_policy was added to ValidatorBlocksQuery
and is properly deserialized from query parameters."
(Asking for verification of completeness, not asserting something is missing)
```

**Real example from PR #7558 review:**

From the initial review:
- **Issue:** Initially commented that `GraffitiPolicy` type was missing
- **Feedback:** "That's not possible because the CI is currently passing"
- **Lesson:** If CI passes, the type must exist somewhere in the codebase or in this PR

**Corrected approach:**
- Remove assertions about missing types/imports when CI passes
- Focus on whether new query parameters are properly wired through the API
- Ask for verification of completeness rather than asserting things are missing

#### Keep Reviews Concise

**Be direct and to the point.** Avoid verbose explanations with multiple sections, checkboxes, or formatting. State the issue clearly and suggest a fix.

**Bad (too verbose):**
```
## Code Review

### ✅ Strengths
1. Proper prerequisite: Correctly builds on #8417
2. Safety compliance: No `.unwrap()` calls
...

### ⚠️ Recommendations
#### 1. Missing test coverage
The existing test still provides blobs...
```

**Good (concise):**
```
Missing test coverage for the None blobs path. The existing test at
`store_tests.rs:2874` still provides blobs. Should add a test passing
None to verify backfill handles this correctly.
```

**When acknowledging good work, keep it brief and general:**
```
This is a really clean refactor. I think you've covered all cases and the tests looks very tidy!
```

**Note**: Focus on qualities like thoroughness, clarity, or tidiness. Don't enumerate the specific technical changes that were made.

#### Use Suggestion Blocks for Simple Fixes
```suggestion
// Fixed typo or simple improvement
```

#### Catch Variable Name Mix-ups

From actual reviews:
> "Discrepancy is here -- I believe they should be flipped"

When you spot variable names that seem swapped or confused, point it out clearly.

#### Link to Relevant Code
- Reference commits, line numbers, and related PRs
- Example: `[679899e](https://github.com/sigp/lighthouse/pull/8247/commits/679899e...)`

#### Frame Feedback Constructively

**Good examples from actual reviews:**
- "Should we check the returned outcome here? What happens if it returns `Ok(BatchState::Failed)`?"
- "I think this should be the `head_epoch`, not the `wall_slot_epoch`... I think we want the cgc change effect to happen from where we start syncing right?"
- "This comment is a little unclear. Maybe something along the lines of..."
- "Great catch. Fixed in ac8adb4"

**Use natural, conversational language:**
- Use questions ("Should we avoid `.expect()` here?") rather than directives ("This must not use `.expect()`")
- Say "we typically avoid..." instead of "coding standards require..."
- Frame as suggestions: "Could we..." or "Would it make sense to..."
- Avoid overly formal or prescriptive tone

**Examples:**
```
❌ Too formal/prescriptive:
"This violates Lighthouse's coding standards which strictly prohibit runtime panics."

✅ Natural and constructive:
"Should we avoid `.expect()` here? This gets called in hot paths and we typically try to avoid runtime panics outside of startup."
```

#### Explain Your Reasoning

Share the reasoning behind suggestions:
> "I was under the assumption that a peer will be requesting columns from us based on our nodes status v2 response. during a cgc change our status v2 will only return the earliest slot in which we've fulfilled our custody requirements for the current cgc..."

#### Request Second Reviews for Sensitive Code

- Security-critical code should have multiple reviewers
- Consensus-critical code needs thorough review
- Example: "I might wait for a 2nd review before merging, seeing as this is potentially quite sensitive."

### Accepting Feedback

#### Acknowledge Good Catches
- "Great catch. Fixed in ac8adb4"
- "nice catch!"
- "Yeah! Good catch!"

#### Explain Decisions
- If not taking a suggestion, explain why
- Share reasoning about trade-offs
- Example: "Good call! It'd also minimise the unnecessary CGC changes that may happen. I like the fact that it reduce the number of edge cases and is more predictable"

#### Defer When Appropriate
- "Mind if I do it in a separate PR?"
- "I've discussed offline and going to make a follow up PR so it's bit easier to review"

## Common Review Patterns

### 1. Fork-Specific Changes (Critical)

When PRs add new fork variants to existing types:
- Verify current production fork code path is unchanged
- Check SSZ compatibility (field order must match existing stored data)
- Verify rollback/error paths handle edge cases (empty collections)
- Don't derive `Decode` on transparent SSZ enums - use `from_ssz_bytes_for_fork`
- Compare with similar types (e.g., BeaconBlock pattern)

### 2. API Design Quality

- Constructor signatures should be consistent (all return `Result` or all return `Self`)
- Avoid `Option` parameters when the value is always required
- Extract duplicated logic (e.g., fork-checking) to single location

### 3. Edge Cases in Error/Rollback Paths

Always check: What happens when collections are empty? Old code may have handled empty as no-op, new code may error.

### 4. Variable Naming Issues
Look for:
- Ambiguous abbreviations (`bb`, `bl`, `bbrange`)
- Swapped variable names
- Inconsistent naming conventions

### 5. Concurrency & Threading
Look for:
- Lock ordering
- Potential deadlocks
- Race conditions
- Atomic operations

### 6. Error Handling
Check:
- Are errors logged?
- Are edge cases handled?
- Are return values checked?
- Is context provided with errors?

### 7. Documentation
Review:
- Are comments clear?
- Are typos fixed?
- Is the "why" explained?
- Are assumptions documented?

### 8. Code Safety
Examine:
- Any `.unwrap()` or `.expect()` calls?
- Array indexing without bounds checking?
- Arithmetic that could overflow?
- Consensus crate using safe math?

## Anti-Patterns to Avoid

### 1. Over-Engineering
- Don't add abstractions until needed
- Keep solutions simple and focused
- "Three similar lines of code is better than a premature abstraction"

### 2. Unnecessary Complexity
- Avoid feature flags for simple changes
- Don't add fallbacks for scenarios that can't happen
- Trust internal code and framework guarantees

### 3. Premature Optimization
- Optimize hot paths based on profiling, not assumptions
- Document performance considerations but don't over-optimize
- Consider the cost-benefit of optimizations

### 4. Hiding Important Information
- Don't use generic variable names when specific ones are clearer
- Don't skip logging just to keep code shorter
- Don't omit error context

## Design Principles

### Simplicity First
Only introduce complexity when it's necessary or makes the code clearer. Question every layer of abstraction:
- Is this `Arc` needed, or is the inner type already `Clone`?
- Is this `Mutex` needed, or can ownership be restructured?
- Is this wrapper type adding value or just indirection?

If you can't articulate why a layer of abstraction exists, it probably shouldn't.

### High Cohesion, Loose Coupling
- **High cohesion**: Group related state and behavior together. If two fields are always set together, used together, and invalid without each other, they belong in a struct.
- **Loose coupling**: Components should have minimal dependencies on each other's internals.

**Example of poor cohesion:**
```rust
struct Service {
    sender: Option<Sender>,     // Always set together
    cache: Option<Cache>,       // Always set together
}
// Caller must check both, easy to get out of sync
```

**Better:**
```rust
struct Service {
    monitor: Option<HeadMonitor>,  // Single Option
}
struct HeadMonitor {
    sender: Sender,
    cache: Cache,
}
// Invariant encoded in types, impossible to have one without the other
```

## Questions to Ask During Review

1. **Correctness**
   - Does this handle all edge cases?
   - What happens when inputs are empty/null/zero?
   - Are there race conditions?
   - Could this panic?

2. **Safety** (especially for consensus crate)
   - Are all arithmetic operations safe?
   - Could this panic in any scenario?
   - Is behavior deterministic across platforms?

3. **Fork-Specific Changes**
   - Is the current production fork unchanged?
   - Is SSZ encoding compatible with stored data?

4. **API Design**
   - Are signatures consistent across similar functions?
   - Is there duplicated logic that should be extracted?

5. **Performance**
   - Could this block important operations?
   - Is there unnecessary duplication?
   - Are locks held longer than needed?

6. **Security**
   - Could this be exploited for DoS?
   - Is user input validated?
   - Are errors revealing sensitive information?

7. **Maintainability**
   - Is the intent clear from the code?
   - Would a new contributor understand this?
   - Is there adequate documentation?
   - Are variable names clear?

8. **Testing**
   - How was this tested?
   - Are there new test cases?
   - What scenarios might break this?

## Examples of Good Reviews

### Example 1: Catching a Bug Early
> "Discrepancy is here -- I believe they should be flipped"
>
> Shows the variable name swap and suggests the fix.

**Why this is good:** Catches a subtle bug that could cause runtime issues. Direct and clear.

### Example 2: Thoughtful Technical Discussion
> "I think this should be the `head_epoch`, not the `wall_slot_epoch` derived from the canonical_head? I think we want the cgc change effect to happen from where we start syncing right? The wall clock epoch can be anywhere based on how much behind the node is"

**Why this is good:** Explains reasoning, considers the broader system behavior, helps the author think through edge cases.

### Example 3: Constructive Wording Suggestion
> "This comment is a little unclear. Maybe something along the lines of:
> ```
> the node has a higher contribution of custody from the number of validators attached compared to the one derived from cli flags
> ```
> Something a little less verbose than that though :P"

**Why this is good:** Provides a concrete alternative, acknowledges it could be improved further, maintains friendly tone.

### Example 4: Acknowledging Good Fixes
> "Good call! It'd also minimise the unnecessary CGC changes that may happen. I like the fact that it reduce the number of edge cases and is more predictable"

**Why this is good:** Acknowledges the improvement, explains additional benefits, reinforces good decisions.

## Tracing and Logging

### Logging Levels

Use appropriate log levels:
- **`crit`**: Critical issues - Lighthouse may not function correctly
- **`error`**: Errors with moderate impact - expect user reports
- **`warn`**: Unexpected but recoverable - user might report if excessive
- **`info`**: High-level status - should not be excessive
- **`debug`**: Developer-focused events and expected errors

### Tracing Spans

**Avoid overuse:**
- Don't add spans to simple getter methods
- Be cautious of span explosion with recursive functions
- Use spans per meaningful computation step, not every function
- Don't use `span.enter()` or `span.entered()` in async tasks

**❌ Bad:**
```rust
#[instrument]
fn get_head_block_root(&self) -> Hash256 {
    self.head_block_root
}
```

**✅ Good:**
```rust
#[instrument(skip(self))]
async fn process_block(&self, block: Block) -> Result<(), Error> {
    // meaningful computation
}
```

## Code Quality Checks

Before approval, verify:

- [ ] **No panics**: No `.unwrap()`, `.expect()`, or array indexing without bounds checks
- [ ] **Consensus safe**: If touching consensus crate, all arithmetic is safe
- [ ] **Errors logged**: Errors aren't silently swallowed
- [ ] **Clear naming**: Variable names are unambiguous
- [ ] **TODOs linked**: All TODOs have GitHub issue links
- [ ] **Tests present**: Non-trivial changes have tests
- [ ] **Documentation**: Complex logic has explanatory comments
- [ ] **Lock safety**: Lock ordering is safe and documented
- [ ] **No blocking**: Async code doesn't block the runtime

## Efficient Review Workflow for Claude

**CRITICAL: Always get user approval before publishing comments.**

When reviewing PRs as Claude Code:

### 0. User Approval Required (ALWAYS)
**Before publishing any PR review:**
1. Perform the full review analysis
2. Prepare the proposed comments
3. **Show the user exactly what will be posted**
4. **Wait for explicit approval** ("ok please publish" or similar)
5. Only then publish to GitHub

**Never publish reviews without user approval.** The user may want to:
- Refine the wording
- Add or remove comments
- Verify assumptions
- Adjust priorities

### 1. Initial Assessment (Quick Pass)
- Read PR description to understand the goal
- Check which files changed and identify critical areas
- Look for consensus crate changes (highest priority)
- Identify complexity hotspots (nested logic, long functions)

### 2. Deep Review (Focused)
**Prioritize reading these in order:**
1. **Test files** - understand what behavior is expected
2. **Core logic changes** - the main implementation
3. **API/interface changes** - public contracts
4. **Documentation** - verify it matches implementation

**Don't spend time on:**
- Generated files (unless they look wrong)
- Trivial test updates (adding `None` parameters)
- Formatting-only changes
- Files with only import additions

### 3. Identify Review Comments
**Ask yourself:**
- Does this introduce a panic path? (highest priority)
- Are edge cases tested?
- Is complex logic documented?
- Could this have a race condition?

**Keep to 3-5 comments maximum.** If you find more issues:
- Group related issues into one comment
- Only comment on the most important ones
- Suggest a follow-up review for minor issues

### 4. Format for Publishing
**Structure each comment:**
```
[Brief description of issue]

[Explanation in 1-2 sentences]

[Optional: code example or suggestion]
```

**Publish as a single review comment** with all issues grouped, rather than individual line comments (unless targeting specific lines).

### 5. What Good Looks Like
- 3-5 concise, actionable comments
- Each comment has clear next steps
- Focus on correctness and safety
- Provide context and reasoning
- Use questions more than directives

**Example of efficient review:**
```
**Review completed - 3 items found**

1. **Query parameter handling** (line 3534)
   Please verify graffiti_policy was added to ValidatorBlocksQuery.

2. **Complex logic documentation** (line 751)
   Consider adding a comment explaining the graffiti truncation strategy.

3. **Edge case test coverage** (line 96)
   Missing test for empty graffiti with AppendClientVersions policy.
```

### 6. Incorporate User Feedback

**After each review, update this document with learnings.**

When the user provides feedback on your review comments:
1. Note what was wrong or could be improved
2. Add examples to this document
3. Update guidelines to prevent similar issues

**Common feedback to incorporate:**
- "That type already exists" → Update "Verify Assumptions" section
- "This comment is too verbose" → Add to "Keep Reviews Concise" examples
- "Focus on X instead of Y" → Update prioritization guidelines
- "Good catch on Z" → Add to successful review patterns

**How to update:**
```bash
# After receiving feedback, immediately update CODE_REVIEW.md
# Add the lesson learned to the appropriate section
# Include specific examples from the interaction
```

**Example update pattern:**
```markdown
#### New Lesson: [Category]

From PR #XXXX review:
- **Issue:** [What went wrong]
- **Feedback:** [What user said]
- **Lesson:** [What to do differently]

**Example:**
❌ Bad: [Old approach]
✅ Good: [Improved approach]
```

### 7. Recent Lessons Learned

#### Lesson: Don't Default to Verbose Structured Reviews (PR #7423)

**Issue:** Despite reading CODE_REVIEW.md guidelines, initially wrote a verbose review with sections, checkboxes, and "Overall Assessment" headers.

**Feedback:** "why is your format a bit off? are you using CODE_REVIEW.md?"

**Lesson:** The ingrained habit of writing structured reviews is strong. Must actively resist the urge to add:
- Summary/Overview sections
- Multiple numbered/bulleted subsections
- ✅ Checkboxes for "Positive Observations"
- "Overall Assessment" or "Verdict" sections
- Verbose explanations with multiple paragraphs

**Correct approach:**
- 3-5 concise comments maximum
- Natural, conversational language
- Direct statements without structure
- Questions instead of directives

❌ Bad:
```
## Code Review Summary

### Key Issues Found

#### 1. Trait Definition Inconsistency (Critical)
...detailed explanation...

### Positive Observations
✅ No unwrap/expect added
✅ Test coverage updated

### Overall Assessment
This is a thorough refactor...
```

✅ Good:
```
The trait definitions use `impl Future + Send + Sync` but implementations use `async fn`. Could we make these consistent?

Have you tested whether this prevents the freeze on affected kernels?
```

#### Lesson: Verify Assumptions Even More Rigorously (PR #7423)

**Issue:** Initially flagged "holding tokio RwLock across await" as a problem, applying sync lock thinking to async locks.

**Feedback:** "are you absolutely sure these are valid? i want to avoid false positives"

**Lesson:** Need even more rigorous verification:
1. **Check if pattern is idiomatic first** - tokio RwLock guards are DESIGNED to be held across await
2. **Understand the technology** - don't assume sync lock rules apply to async locks
3. **CI passing is strong signal** - if all tests pass, be very cautious about claiming issues
4. **Research before commenting** - when unsure, investigate the actual best practices for the technology

**Key insight:** `tokio::sync::RwLock` guards being held across `.await` is **not a bug**, it's the intended usage pattern. Unlike `parking_lot::RwLock` guards which are `!Send` and cause compiler errors, tokio guards are `Send` and designed for async contexts.

#### Lesson: Consider PR Context and Staleness (PR #7423)

**Issue:** Didn't initially consider that PR was old and the original bug reports had stopped.

**Feedback:** "the reason i ask is because this PR was created a long time ago and we have not had any further updates and reports of the bug"

**Lesson:** When reviewing PRs:
1. **Check PR age** - old PRs may be addressing issues that are no longer relevant
2. **Check linked issues** - have the problems continued or stopped?
3. **Ask about merge intent** - stale PRs may not be intended for merge
4. **Suggest validation steps** - for large refactors, recommend testnet deployment

**Good pattern for stale PRs:**
```
This looks like a solid code quality improvement...

Are you planning to merge this? If so, we should deploy to a testnet node soon after to validate under real conditions given the scope of the changes.
```

This acknowledges uncertainty about merge intent while being constructive if merge is planned.

## Deep Review Techniques

### Don't Just Summarize - Find New Issues

When the user shares findings or points you to specific issues, **don't just echo their findings back**. Use their hints as a starting point to dig deeper and find additional issues they haven't identified yet.

**Bad pattern:**
```
User: "I found a bug at line 866"
Claude: "Yes, line 866 has a bug where X happens"
```

**Good pattern:**
```
User: "I found a bug at line 866"
Claude: [Investigates thoroughly, then reports]
"I found 3 additional issues:
1. Similar bug at line 1708
2. The fix at 866 is incomplete because...
3. There's also a related issue in the config parsing..."
```

### Verify Against Specifications

When reviewing changes to spec-defined behavior:

1. **Read the actual spec** - Check `./consensus-specs/` or fetch the relevant spec PR
2. **Compare formulas exactly** - Don't assume the code matches the spec
3. **Check constant values** - Verify BPS values, timeouts, etc. match spec definitions
4. **Look for spec comments** - Comments like "~33%" vs "exactly 1/3" matter

**Example from actual review:**
```
Spec: ATTESTATION_DUE_BPS = 3333 (yields 3999ms, not 4000ms)
Code comment: "one_third_slot_duration"
Issue: Variable name is misleading - it's ~33.33%, not exactly 1/3
```

### Trace Data Flow End-to-End

When new configuration fields are added, trace the complete flow:

1. **Config file** → Does the YAML contain the field?
2. **Config struct** → Does the parsing struct have the field with serde attributes?
3. **apply_to_chain_spec** → Is the field actually read and applied?
4. **Runtime usage** → Is the value used correctly everywhere?

**Common bug pattern: Silent config ignoring**
```yaml
# Config file has:
ATTESTATION_DUE_BPS: 3333
```
```rust
// But Config struct is missing the field - value silently ignored!
pub struct Config {
    // attestation_due_bps is NOT here
}
```

### Check Error Handling Fallbacks

Examine every `.unwrap_or()`, `.unwrap_or_else()`, and `.unwrap_or_default()`:

**Ask:** "If this fallback triggers, does the code behave correctly?"

**Dangerous patterns:**
```rust
// Always triggers "late" condition on error
if delay >= threshold.unwrap_or(delay) { ... }

// Uses zero which may cause division errors or wrong calculations
let timing = get_timing().unwrap_or(Duration::from_secs(0));

// Arbitrary fallback that makes no semantic sense
let threshold = get_threshold().unwrap_or(some_unrelated_value);
```

**Better approach:** Fail loudly at startup rather than silently degrading at runtime.

### Look for Incomplete Migrations

When a PR changes a pattern across the codebase, verify completeness:

1. **Search for old pattern** - `grep` for the thing being replaced
2. **Check all call sites** - Every usage should be updated consistently
3. **Look for TODO comments** - These indicate known incomplete work
4. **Check test files** - Tests often lag behind implementation changes

**Example checklist for "replace X with Y" PRs:**
- [ ] All production code updated
- [ ] All test code updated
- [ ] All config files updated
- [ ] Config parsing updated (not just config files!)
- [ ] Documentation updated
- [ ] No remaining references to old pattern

### Verify Arithmetic and Precision

For timing, financial, or consensus-critical calculations:

1. **Check units** - milliseconds vs seconds vs slots vs epochs
2. **Check truncation** - `as_secs()` truncates milliseconds
3. **Check integer division** - Order of operations matters for precision
4. **Check overflow potential** - Large multiplications before divisions

**Example precision issue:**
```rust
// Truncates sub-second precision
let slots = DELAY_SECONDS / spec.get_slot_duration().as_secs();

// If slot_duration is 5500ms (5.5s), as_secs() returns 5
// This gives wrong number of slots!
```

**Example missing multiplication:**
```rust
// Bug: Missing slots_per_epoch multiplication
let delay = EPOCHS * slot_duration.as_secs();  // Wrong!

// Should be:
let delay = EPOCHS * slots_per_epoch * slot_duration.as_secs();
```

### Check for Semantic Changes

When code is "just refactored," verify behavior is actually unchanged:

1. **Metrics** - Does the metric still measure the same thing?
2. **Log messages** - Do logs still convey the same information?
3. **API responses** - Are response semantics preserved?
4. **Timing** - Are delays/timeouts the same duration?

**Example semantic change (bug):**
```rust
// BEFORE: Measures delay from slot start
let delay = seen_time - slot_start;

// AFTER: Measures delay from attestation due time (4s into slot)
let delay = seen_time - slot_start - attestation_due;

// This changes what the metric means!
// Metric description says "from slot start" but now measures something else
```

### Check for File/Merge Conflicts

Before diving deep into code review:

1. **Check if files still exist on base branch** - Files may have been renamed/moved
2. **Look for recent base branch changes** - PR may need rebase
3. **Verify the PR is up-to-date** - Stale PRs may have conflicts

**Example:**
```
PR adds: validator_custody.rs (790 lines)
Base branch: File was renamed to custody_context.rs in recent PR
Result: PR creates duplicate/conflicting file
```

### Systematic Bug Hunting Checklist

For thorough reviews, systematically check:

**Arithmetic:**
- [ ] Units match (ms vs s vs slots)
- [ ] No precision loss from truncation
- [ ] Correct order of operations
- [ ] No missing multiplications/divisions

**Error Handling:**
- [ ] Fallback values make semantic sense
- [ ] Errors don't cause silent incorrect behavior
- [ ] Critical errors fail loudly vs silently degrading

**Config/Data Flow:**
- [ ] New config fields are actually parsed
- [ ] Default values are backwards-compatible
- [ ] All usages of config are updated

**Consistency:**
- [ ] Same pattern used everywhere (no partial migrations)
- [ ] Variable names match actual semantics
- [ ] Comments match code behavior

**Integration:**
- [ ] No conflicts with recent base branch changes
- [ ] Tests updated to match implementation
- [ ] Documentation matches new behavior

## Conclusion

Good code review is about:
- **Helping** each other write safer, clearer code
- **Learning** from each other's experience and domain knowledge
- **Protecting** the project from bugs, panics, and consensus failures
- **Improving** code quality and maintainability over time

Be thorough but kind. Be specific but constructive. Focus on making the codebase better for everyone.

**Remember:** In consensus-critical code, there's no such thing as being too careful.
