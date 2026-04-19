---
name: debugging
description: Systematic debugging workflow — reproduce the bug, isolate the root cause, fix, then verify with a regression test. Use this skill when fixing a bug, debugging an error, investigating why something doesn't work, diagnosing a test failure, or when a task describes broken behavior that needs fixing. Also use when the user reports an error message, unexpected behavior, or a regression. Keywords: debug, отладка, bug, баг, fix, фикс, почини, пофикси, сломалось, не работает, ошибка, error, broken, regression, fails, crash, investigate, диагностика.
---

# Debugging

Find and fix bugs by narrowing down the root cause through evidence, not by
guessing. The tempting shortcut — read the error, guess, patch, hope — fixes
symptoms instead of causes, introduces new bugs, and burns cycles on wrong
hypotheses. The workflow below is slower on trivial bugs but dramatically
faster on anything non-trivial.

## Workflow

### Step 1: Reproduce

Before touching any code, reproduce the bug. A bug you can't reproduce is a bug
you can't verify you've fixed.

Find the project's test runner and build/run commands. Check the Makefile,
package.json, pyproject.toml, Cargo.toml, or CI config.

**For test failures:** run the specific failing test in isolation.
**For runtime errors:** start the app and trigger the error path.
**For build errors:** run the build/compile/typecheck command.

Record the exact error output — you'll need it to verify the fix later.

### Step 2: Isolate

Narrow down WHERE the bug lives. The goal is to go from "something is broken"
to "this specific function/line produces wrong output for this input."

**Reading the stack trace:**
- Start from the bottom (the actual error)
- Work upward to find the first frame in YOUR code (not library code)
- That frame is your starting point

**Binary search for the source:**
- If the bug is in a pipeline (A → B → C → D), check the output of B first
  - If B's output is correct, the bug is in C or D
  - If B's output is wrong, the bug is in A or B
- Add temporary print/log statements at boundaries to check intermediate values
- Remove them after you find the root cause

**Common isolation techniques:**
- Comment out sections to find which one causes the failure
- Test with minimal input to rule out data-dependent issues
- Check if the bug reproduces with a fresh database/state

### Step 3: Identify Root Cause

Once you know WHERE the bug is, understand WHY it happens:

- Read the code carefully — what does it actually do vs what it should do?
- Check recent changes — `git log -5 --oneline <file>` and `git diff` for context
- Consider edge cases — empty collections, null/nil/None values, off-by-one, race conditions
- Check types — is the value actually what the code assumes it is?

**Common root causes:**
| Symptom | Likely Cause |
|---------|-------------|
| Null/undefined reference error | Wrong import/access, missing field, wrong shape |
| Wrong data returned | Query/filter logic error, wrong join, missing condition |
| Works locally, fails in test | Test isolation issue, mock not matching reality |
| Intermittent failure | Race condition, shared mutable state, timing |
| Works for some inputs | Missing edge case handling (null, empty, boundary) |

### Step 4: Write a Regression Test FIRST

Before fixing the code, write a test that captures the broken behavior.
Use whatever test framework and assertion style the project already uses.

The test should:
- Reproduce the exact bug scenario
- Fail before your fix (confirm by running it)
- Pass after your fix

This proves:
1. You're testing the right thing
2. Your fix actually addresses the bug
3. The bug won't quietly regress later

### Step 5: Fix

Make the minimal change that fixes the root cause. "Minimal" means:
- Don't refactor surrounding code
- Don't add defensive checks everywhere "just in case"
- Don't "improve" code that isn't part of the bug
- Fix the cause, not the symptom

**Good fix:** Add null check where the data can actually be null
**Bad fix:** Add null checks to every function in the file "for safety"

### Step 6: Verify

1. Run your regression test — it should pass now
2. Run the full test suite — nothing else should break
3. If the bug was runtime: reproduce the original scenario and confirm it works
4. Remove any temporary print/log statements

## Debugging Specific Scenarios

### API Returns Wrong Status Code / Body
1. Check the route handler — is the right status code set?
2. Check middleware — is something intercepting/modifying the response?
3. Check validation — is the request being rejected before reaching your handler?
4. Use verbose HTTP client output to see exact request/response headers

### Database Query Returns Wrong Data
1. Run the query directly in the DB to see raw results
2. Check joins and WHERE clauses
3. Check if ORM is adding unexpected conditions (soft delete, tenant filter)
4. Check if the data was wrong to begin with (seed/migration issue)

### Test Passes Locally, Fails in CI
1. Check test isolation — does the test depend on execution order?
2. Check for hardcoded paths, ports, or timestamps
3. Check for shared state between tests (DB not cleaned up)
4. Try running tests in a specific order to reproduce

## Anti-Patterns

- **Shotgun debugging** — changing multiple things at once. You won't know which
  change fixed it, and you may introduce new bugs. Change one thing, test, repeat.
- **Fixing the test instead of the code** — if the test fails, the test might be
  right and the code wrong. Understand the intent before changing either.
- **Suppressing errors** — catching/ignoring without handling the error hides
  the bug. Fix the cause, don't silence the symptom.
- **"It works now"** — always understand WHY it works. If you can't explain the
  fix, you haven't fixed it.
