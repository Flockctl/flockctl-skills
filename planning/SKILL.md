---
name: planning
description: Decompose features into structured implementation plans (milestone → slice → task hierarchy) stored as markdown files on disk. Use this skill when generating a project plan, breaking down a feature into tasks, creating milestones and slices, or when the user asks to plan, decompose, or scope work. Keywords: plan, план, decompose, milestone, slice, task, scope, breakdown, спланируй, декомпозиция, разбей на задачи.
---

# Plan Generation

Decompose a feature into a structured plan of milestones, slices, and tasks,
written as markdown files on disk. Each task should be executable by an AI
agent with no additional context.

## Planning Methodology

### The Hierarchy

```
Milestone  →  a shippable version (contains 3-10 slices)
  Slice    →  one demoable vertical capability (contains 1-7 tasks)
    Task   →  one context-window-sized unit of work
```

A task must fit in one context window. An agent that runs out of context
mid-task either stops partway or starts guessing — both produce broken work.
If a task feels like it won't fit, split it.

### Before You Plan: Identify Ambiguity

Before creating plan files, analyze the user's description for gray areas.
For each area of ambiguity, make a reasonable decision and document it in the
milestone description. The user can review and override.

Gray areas to check:
- Visual features → layout, density, interactions, empty states
- APIs → response format, error handling, pagination, auth
- Data model → relationships, constraints, cascading behavior
- Content → structure, validation rules, defaults

### Decomposition: Vertical Slices, Not Horizontal Layers

Each slice must be a **vertical cut** through the entire stack — from database to UI.
This is the most important planning principle.

**Good slice:** "User can create and list projects" (DB + API + UI all in one slice)
**Bad slice:** "Create all database models" (horizontal layer — nothing demoable)

Why vertical slices matter:
- Each slice produces a demoable result you can verify
- Independent slices parallelize better
- Failures are contained to one capability, not one layer

### Risk-First Ordering

Order slices within a milestone by risk level: **high → medium → low**.

High-risk slices go first because:
- Unknown unknowns surface early when you still have time to adapt
- The plan can be revised based on what you learn from the risky parts
- If something is going to fail, fail fast

**What makes a slice high risk:**
- New technology or unfamiliar pattern
- Unclear requirements or external dependencies
- Performance-sensitive or security-critical path
- Integration with third-party services

### Task Sizing and Verification

Every task needs a `verify` field — a mechanically checkable outcome.
Not "it should work" but a specific command or observable behavior.

**Verification ladder** (prefer earlier, stronger checks over later ones):
1. **Behavioral testing** — `curl http://localhost:52077/api/... | jq .` (proves it actually works end-to-end)
2. **Test execution** — `npm test -- --grep 'task name'` (proves intended behavior in isolation)
3. **Static checks** — `npm run lint`, `npx tsc --noEmit` (proves it's well-formed, not that it works)
4. **Files exist** — use only as a fallback when nothing else applies; real implementation, not stubs

### Dependency Awareness and Parallelism

Tasks without shared file dependencies should not depend on each other — false
dependencies serialize work that could run in parallel. Group tasks into
implicit **waves**: wave N runs in parallel, wave N+1 depends on wave N.

**Good dependency graph:**
```
T1 (models)  T2 (config)     ← Wave 1: parallel
    ↓             ↓
T3 (API)     T4 (CLI)        ← Wave 2: parallel
    ↓             ↓
       T5 (UI)               ← Wave 3
```

**Bad dependency graph (serial chain):**
```
T1 → T2 → T3 → T4 → T5     ← Every task waits for the previous one
```

### Anti-Patterns

Avoid these common planning mistakes:
- **Stub slices** — "Set up project structure" with no real functionality
- **God tasks** — one task that implements an entire feature end-to-end
- **Phantom dependencies** — marking tasks as dependent when they don't share files
- **Missing verification** — "manually test in browser" is not a verify step
- **Scope creep** — adding nice-to-have features that weren't in the user's description
- **Over-engineering** — adding abstractions, factories, or patterns "for future extensibility"

## File Structure

Plans live in `{projectPath}/.flockctl/plan/` with this layout:

```
.flockctl/plan/
├── OVERVIEW.md              ← narrative summary (you write this)
├── 00-milestone-name/
│   ├── milestone.md
│   ├── 00-slice-name/
│   │   ├── slice.md
│   │   ├── 00-task-name.md
│   │   ├── 01-task-name.md
│   ├── 01-slice-name/
│   │   ├── slice.md
│   │   └── ...
├── 01-another-milestone/
│   └── ...
```

### OVERVIEW.md

Write `.flockctl/plan/OVERVIEW.md` as the **one-page narrative** of the plan — what a new teammate (or a future agent coming back to this plan) should read first.

Include:
- **Vision** — what the whole plan achieves in 1–3 sentences
- **Approach** — how the work is decomposed and why (risk-first, vertical slices, what's been explicitly scoped out)
- **Structure** — a compact tree of milestones → slices → tasks (titles only, no status tracking — that lives in each file)
- **Key decisions** — ambiguity calls, trade-offs, what we're NOT building

Status/progress does NOT belong here — per-task status lives in each task's `.md` file. `OVERVIEW.md` is about intent, not state, so it stays fresh across execution.

Update `OVERVIEW.md` only on structural changes (new milestone, slice split, approach pivot). Do not rewrite it on every status flip.

## Slug Format

Directory and file names use: `{NN}-{title-slug}` where NN is zero-padded order (00, 01, 02...) and title-slug is lowercase with hyphens.

Examples: `00-core-api`, `01-user-authentication`, `00-setup-database.md`

## File Formats

### milestone.md
```markdown
---
title: "Milestone Title"
status: pending
order: 0
vision: "What this milestone achieves"
success_criteria:
  - "First criterion"
  - "Second criterion"
created_at: "2026-01-01T00:00:00.000Z"
updated_at: "2026-01-01T00:00:00.000Z"
---

Optional description body. Document any ambiguity decisions here.
```

### slice.md (inside a slice directory)
```markdown
---
title: "Slice Title"
status: pending
order: 0
risk: high
depends:
  - "00-other-slice-slug"
goal: "What this slice delivers"
demo: "How to verify it works (specific steps)"
success_criteria: "Acceptance criteria"
created_at: "2026-01-01T00:00:00.000Z"
updated_at: "2026-01-01T00:00:00.000Z"
---

Optional description.
```

Risk values: `high`, `medium`, `low`
Order slices by risk: high first, then medium, then low.
`depends` references other slice slugs within the same milestone.

### task .md files (inside a slice directory)
```markdown
---
title: "Task Title"
status: pending
order: 0
estimate: "1-2 hours"
files:
  - "src/routes/example.ts"
  - "src/models/example.ts"
verify: "Run tests: npm test -- --grep 'example'"
depends:
  - "00-other-task-slug"
created_at: "2026-01-01T00:00:00.000Z"
updated_at: "2026-01-01T00:00:00.000Z"
---

Detailed task description. Include:
- What to implement and how
- Which existing patterns to follow (reference specific files)
- Edge cases to handle
- What NOT to do (scope boundaries)
```

`depends` references other task slugs within the same slice.

## File-Writing Rules

- Create directories with `mkdir -p` before writing files
- Use ISO timestamps for `created_at` / `updated_at`
- Reference existing code by path so the implementing agent has a concrete
  analogy: "follow the pattern in src/routes/tasks.ts" beats "create a standard router"

---

## Mode: Quick

Generate exactly ONE milestone with 3-6 slices, ordered by risk (high → low).
Focus on the critical path to a working product.

Keep task descriptions concise — just enough for an agent to execute without ambiguity.
Skip deep risk analysis. Every slice still needs a `demo` field showing how to verify it works.

---

## Mode: Deep

Generate 2-5 milestones depending on project scope.

### Additional Analysis Before Planning

Before creating files, analyze:
1. **Risk inventory** — What could go wrong? Unknown technologies, unclear requirements, performance bottlenecks
2. **Proof strategy** — Which slice retires which risk? Map risks to slices explicitly
3. **Integration points** — Where do slices connect? Document hand-off boundaries
4. **What we're NOT building** — Explicitly exclude out-of-scope features

### Additional frontmatter for milestones:
```yaml
key_risks:
  - risk: "Description of risk"
    why_it_matters: "Impact if not addressed"
proof_strategy:
  - risk_or_unknown: "What needs proving"
    retire_in: "Which slice proves this"
    what_will_be_proven: "Success condition"
definition_of_done:
  - "Global done criterion"
```

### Additional frontmatter for slices:
```yaml
proof_level: contract  # contract | integration | operational | final-assembly
integration_closure: "How this integrates with the rest"
observability_impact: "Logging/monitoring changes"
threat_surface: "Security considerations"
```

### Additional frontmatter for tasks:
```yaml
inputs:
  - "Input from other tasks/systems"
expected_output:
  - "What this task produces"
failure_modes:
  - dep_fails: "When dependency X fails"
    task_behavior: "This task does Y"
negative_tests:
  - "Test for invalid input"
  - "Test for error conditions"
```

### Reassessment Guidance

After each milestone completes, reassess the remaining plan:
- Did the completed work reveal new information that changes priorities?
- Should any remaining slices be reordered, split, or dropped?
- Are the original success criteria still the right ones?

Be thorough with risk analysis, verification, and failure modes.
