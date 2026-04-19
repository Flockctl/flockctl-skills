---
name: implementation
description: Systematic approach to implementing features and code changes — analyze existing patterns first, then implement by analogy, then verify. Use this skill whenever executing a plan task, implementing a feature, writing new code, adding an endpoint, creating a component, or modifying existing functionality. Also use when the task prompt describes what to build but not how. Keywords: implement, реализуй, сделай, добавь, создай, build, create, add feature, new endpoint, new component, write code, напиши код, фича, feature.
---

# Implementation

Write correct, minimal code that fulfills the task by following existing
patterns in the codebase.

## Core Principle: Pattern-First

Understand the codebase before writing anything. Most bad AI-written code
"works in isolation" but ignores the conventions, utilities, and patterns
already established in the project — resulting in duplicated helpers,
inconsistent error handling, and drift that someone else has to clean up later.

## Workflow

### Step 1: Analyze the Task

Read the task description and identify:
- **What** needs to be built (the deliverable)
- **Where** it fits in the codebase (which layer, module, package)
- **What exists** that's similar (find the closest analogy)

### Step 2: Find Patterns

Before writing a single line, search the codebase for analogous implementations.
This is language-agnostic — the principle is the same whether the project is Python,
TypeScript, Go, Rust, or anything else:

- Adding an endpoint/handler? Read an existing one that does something similar.
- Adding a model/schema? Read the existing schema definitions and migrations.
- Adding a UI component? Find the most similar existing component.
- Adding a service/module? Find one with similar responsibilities.

**What to look for in existing code:**
- Import/dependency patterns — what utilities, helpers, types are used
- Error handling — how are errors raised, caught, and returned
- Validation — how is input validated (schema library, manual checks, decorators)
- Naming conventions — how are variables, functions, files, classes named
- File organization — where does this type of code live in the project structure
- Configuration — how are settings, env vars, and feature flags accessed

Use `grep`, `find`, or the project's search tools to locate patterns quickly.
Searching before coding takes seconds; fixing a mismatched implementation takes hours.

### Step 3: Implement by Analogy

Write code that follows the patterns you found. This means:
- Use the same utilities (don't reinvent a helper if one exists)
- Use the same error types (don't use raw strings if the project has error classes)
- Use the same validation approach (don't hand-roll if there's a schema library)
- Use the same file structure (don't put handlers in the models directory)
- Use the same naming conventions (casing, prefixes, suffixes)

### Step 4: Verify

Run the verification command from the task's `verify` field. If no verify field,
discover the project's verification tools and run them:
1. Type/lint checks — the project's static analysis tool
2. Tests — the project's test runner
3. Build — confirm the project still builds
4. For API changes — hit the endpoint manually and verify the response

Look at the project's Makefile, package.json, pyproject.toml, Cargo.toml, or
CI config to find the right commands.

## Anti-Patterns

These are the most common mistakes. Each one wastes time and creates bugs:

**Writing code without reading the codebase first.**
Even if you "know" how to write a REST endpoint, this project may do it differently.
Always check how the project does it before implementing.

**Creating new utilities that already exist.**
Search before creating. A quick grep takes seconds.
Creating a duplicate leads to divergence and maintenance burden.

**Adding features that weren't requested.**
The task says "add a name field"? Add a name field. Don't also add validation,
a character counter, an auto-suggest dropdown, and a migration rollback script.
Scope creep is the #1 way to introduce bugs in unrelated areas.

**Over-abstracting for "future extensibility".**
Don't create an abstract factory when you need one function.
You can't predict the future. Write simple code today;
refactor when a real pattern emerges.

**Ignoring error handling patterns.**
If every handler in the project catches errors with a specific helper, do the same.
Don't invent a new error handling approach.

## When You're Stuck

If the task is ambiguous:
1. Re-read the task description — the answer is often there
2. Look at the parent slice's `goal` and `demo` fields for intent
3. Make the simplest reasonable decision and document it in a code comment
4. Don't block on perfection — a working implementation that can be refined is
   better than no implementation

If you hit a technical blocker:
1. Check if the dependency/tool/API you need is already available in the project
2. Look for workarounds in similar code
3. Implement a minimal version that satisfies the verify step
4. Note the limitation in a comment for follow-up
