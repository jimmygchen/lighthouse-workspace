# Code Review Task

You are reviewing code for the Lighthouse project.

## Required Reading

1. **Read** `CODE_REVIEW.md` in the workspace root - Comprehensive review guidelines
2. **Read** `lighthouse/CLAUDE.md` - Lighthouse-specific coding standards

## Your Task

Perform a thorough code review focusing on:

1. **Consensus Crate Safety** (if applicable)
   - Safe math operations (saturating_*, checked_*)
   - Zero panics
   - Deterministic behavior

2. **General Code Safety**
   - No `.unwrap()` or `.expect()` at runtime
   - No array indexing without bounds checks
   - Proper error handling

3. **Code Clarity**
   - Clear variable names (avoid ambiguous abbreviations)
   - Well-documented complex logic
   - TODOs linked to GitHub issues

4. **Error Handling**
   - Errors are logged, not silently swallowed
   - Edge cases are handled
   - Return values are checked

5. **Concurrency & Performance**
   - Lock ordering is safe
   - No blocking in async context
   - Proper use of rayon thread pools

6. **Architecture**
   - No unnecessary dependencies
   - Schema migrations if needed
   - Backwards compatibility considered

Provide specific, constructive feedback with line references where possible.
