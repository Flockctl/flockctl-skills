---
name: testing
description: Systematic test writing and execution — what to test, how to structure tests, and common pitfalls. Use this skill when writing tests for new code, adding test coverage, creating regression tests, verifying a feature works, or when a task's verify step involves running tests. Also use when reviewing whether existing tests are sufficient. Keywords: test, тест, тесты, write test, напиши тест, покрой тестами, coverage, покрытие, verify, проверь, assertion, тестирование, unit test, integration test.
---

# Testing

Write tests that catch real bugs — not tests that merely exist to increase
coverage numbers.

## Core Principle: Test Behavior, Not Implementation

A good test answers "does this code do the right thing from the caller's
perspective?" A bad test answers "does this code call these internal methods
in this order?" Implementation-detail tests break on every refactor even when
behavior is unchanged, creating noise that trains the team to ignore test
failures. Behavior tests break only when actual functionality changes — which
is exactly when you want to know.

## Before Writing Tests: Find the Project's Patterns

Every project has established testing conventions. Before writing any test:

1. **Find existing test files** — search for `test`, `spec`, `_test` files
2. **Read 2-3 tests** — understand the assertion style, setup/teardown patterns,
   how fixtures/factories work, what helpers exist
3. **Follow the same patterns** — use the same test framework, assertion library,
   mocking approach, directory structure, and naming conventions

This matters because inconsistent test style makes the test suite harder to
maintain than no tests at all.

## What to Test

### For API Endpoints / Request Handlers

Every endpoint needs tests for:

1. **Happy path** — correct input produces correct output and status
2. **Validation** — bad input produces the correct error response
3. **Not found** — missing resource returns appropriate error
4. **Edge cases** — empty lists, maximum values, special characters

### For Service / Business Logic

Focus on the function's contract — given these inputs, expect these outputs:

1. **Normal inputs** — typical valid data
2. **Boundary values** — 0, 1, max, empty string, empty collection
3. **Error conditions** — null/nil/None, invalid types, missing fields
4. **State transitions** — pending → running → completed

### For Data Transformations

Test the transformation at the boundary — input shape in, output shape out.
Don't test intermediate steps — they're implementation details.

## Test Structure: Arrange-Act-Assert

Every test follows three phases. Keep them visually distinct:

```
// Arrange — set up the preconditions
(create objects, set state, prepare input)

// Act — execute the behavior under test
(call the function, send the request)

// Assert — verify the outcome
(check return value, check side effects, check state)
```

**One act per test.** If you find yourself having multiple "Act" sections,
you're testing multiple behaviors — split into separate tests.

## Test Naming

Test names should read as specifications. A failing test's name should tell you
what broke without reading the code:

**Good names:**
- "creates a record with default status when status is not provided"
- "returns not-found error when resource does not exist"
- "excludes disabled items from resolved list"

**Bad names:**
- "test create" — create what? With what inputs?
- "works correctly" — what does "correctly" mean?
- "handles edge case" — which edge case?

## Boundary Value Analysis

For any numeric or string input, test these boundaries:

| Type | Test Values |
|------|------------|
| Number (0..100) | -1, 0, 1, 50, 99, 100, 101 |
| String (required) | `""`, `" "`, `"a"`, very long string |
| Collection | empty, one element, many elements, null/nil |
| Optional field | present, missing, null, empty |
| Pagination | page 0, page 1, last page, beyond last page |
| Date/time | past, now, future, epoch, invalid format |

You don't need ALL of these for every input — pick the ones relevant to the
function's behavior. If the function clamps to 0..100, test -1, 0, 100, 101.
If it just stores the value, test only the required/missing boundary.

## Test Isolation

Each test must be independent — running it alone or in any order should produce
the same result.

**Database tests:**
- Use transactions or cleanup in setup/teardown hooks
- Each test creates its own data — never rely on data from another test
- Use unique identifiers to avoid collisions

**File system tests:**
- Create temp directories in setup
- Clean up in teardown
- Check if the project has test helpers or fixture utilities

**Time-dependent tests:**
- Mock the clock/time source
- Never use sleep/delay — it makes tests slow and flaky

## When to Use Integration vs Unit Tests

**Unit tests** — for pure logic with no I/O:
- Data transformations, calculations, validation logic
- Fast, deterministic, easy to debug

**Integration tests** — for code that crosses boundaries:
- HTTP request → handler → database → response
- Service functions that query the database
- Workflows that span multiple modules

Most projects benefit from a **diamond shape**: many integration tests at the API level,
some unit tests for complex logic, few end-to-end tests for critical paths.

## Anti-Patterns

**Testing the mock, not the code.**
If you mock a function and then assert that the mock returns what you told it to,
you're testing the mock framework, not your code.

**Assertions without meaning.**
Asserting that a response "is defined" or "is not null" passes for almost any
response — even completely wrong ones. Assert on specific values.

**Testing implementation details.**
Asserting that an internal method was called with specific arguments breaks when
you refactor internals. Assert on the observable output instead.

**Snapshot abuse.**
Snapshot tests of large objects catch everything but signal nothing — any change
triggers a failure, so developers learn to blindly update snapshots.

**No assertion at all.**
A test with no assertion always passes. This is worse than no test because it
creates false confidence. Every test must assert something specific.

**Overly DRY test code.**
Tests should be readable in isolation. If understanding a test requires reading
5 helper functions and 3 fixtures, the test is too abstract. Some repetition
in tests is OK — clarity matters more than DRY.
