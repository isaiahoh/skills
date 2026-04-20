---
name: refactoring-advisor
description: >
  A refactoring advisor grounded in Martin Fowler's Refactoring (2nd edition). Use this skill whenever
  the user asks to refactor code, review code for quality issues, clean up a codebase, improve code
  structure, reduce technical debt, or identify code smells. Also trigger when the user mentions
  "bad smells," "code smells," "refactoring," "code review," "tech debt," "clean up this code,"
  "improve this code," "restructure," or "make this code easier to change." Trigger liberally —
  even casual requests like "this code is messy, help me fix it" or "review my module" warrant
  this skill. Works with any programming language.
---

# Refactoring Advisor

Systematic refactoring workflow: **detect smells → build safety net → plan refactorings → apply on confirmation**.

## Guiding Principles

1. **Tests first.** Before refactoring, lock down current behavior with self-checking tests. Run them after every step. Never refactor on a red bar (failing tests).
2. **Small steps.** Each refactoring is a tiny behavior-preserving change. If something breaks, you only have a small change to examine. If code is broken for more than a few minutes, the step was too large.
3. **Behavior-preserving.** Refactoring changes internal structure without changing observable behavior. Don't fix bugs you discover during refactoring — note them and fix them separately. Wear one hat at a time: either refactoring or adding functionality, never both.
4. **Risk-driven.** Focus effort where bugs are most likely. Don't refactor everything — concentrate on the code that's hardest to understand or most likely to change.

## Workflow

### Phase 1: Read the Code

Before spawning subagents, read the code yourself to understand:
- **Language and ecosystem** (determines test framework, idioms, file conventions)
- **Structure** (single file? module? class hierarchy? how do the pieces connect?)
- **What it does** (business logic, data transformations, I/O boundaries)
- **Rough complexity** (how much code, how tangled, how many public entry points)
- **Existing tests** (are there any? what framework? what do they cover?)

This understanding informs how you brief the subagents. Don't delegate blindly — you need enough context to evaluate their output and build the refactoring plan.

### Phase 2: Parallel Analysis

Spawn **two subagents in the same turn**:

**Subagent 1 — Smell Detector**

Brief it with:
- The code to analyze (file paths or inline content)
- The path to `references/smell-catalog.md` (it must read this file)
- The language and any structural context it needs (e.g., "this is a single class with 4 public methods" or "these 3 files form a module")

The subagent must:
1. Read `references/smell-catalog.md` to understand all 24 smell categories
2. Scan the code systematically against each smell
3. For each smell found, report:
   - **Smell name** (from the catalog)
   - **Location** (file, function/method, line numbers or code snippet)
   - **Severity** (high / medium / low — based on how much it hinders understanding or invites bugs)
   - **Recommended technique(s)** (from the catalog's "Apply" guidance)
   - **Brief rationale** (why this is a problem here, not just that it matches a pattern)
4. Return the report organized by severity (high first), not by smell category

**Subagent 2 — Test Scaffolder**

Brief it with:
- The code to test (file paths or inline content)
- The path to `references/testing-guide.md` (it must read this file)
- The language, any existing test setup, and the test framework to use
- The public interface entry points you identified in Phase 1

The subagent must:
1. Read `references/testing-guide.md` to understand the testing approach
2. Identify the public interface — the functions/methods callers interact with
3. Write a test suite that locks down current behavior:
   - **Fresh fixture per test** (use `beforeEach` / `setUp` / equivalent — never share mutable state)
   - **Arrange-act-assert** structure in every test
   - **One logical assertion per test** as a default (exceptions for closely related checks on the same operation)
   - **Happy path first**, then boundary conditions (empty collections, zero, negatives, empty strings, wrong types)
   - **Test mutations**: if setters/mutators trigger derived calculations, test through the mutation
   - **Descriptive test names** that identify what broke when they fail
4. Use the **characterization test workflow** for existing code: placeholder → run → capture actual → inject fault → verify failure → revert
5. Write tests to a file following the project's conventions (adjacent to source, or under `tests/`)
6. **Run the tests against the unmodified code** and confirm they all pass. If any test fails, fix the test (not the code). Report the test run results.
7. Return: the test file(s) written, the run results, and a brief summary of what's covered

### Phase 3: Refactoring Plan

Once both subagents complete, **you** (the orchestrator) synthesize their output:

1. **Read the smell report.** Understand each finding — don't just pass it through.
2. **Read the test suite.** Verify it covers the areas that will be affected by the proposed refactorings. If coverage is insufficient for a planned change, note that additional tests are needed before that step.
3. **Look up technique mechanics.** For each recommended technique, read its file from `references/techniques/`. Each file has step-by-step mechanics and worked examples. **Only read the files you need.** File naming convention: `techniques/technique_name.md` (e.g., "Extract Function" → `techniques/extract_function.md`).
4. **Build an ordered refactoring plan.** For each step:
   - Which smell it addresses
   - Which technique to apply
   - What specifically changes (cite concrete code: function names, line numbers, variable names)
   - Why this step comes before the next (dependencies between refactorings)
   - Any additional tests needed before this step
5. **Order by dependency and risk.** Start with refactorings that enable later ones. Do safe, mechanical steps (renames, extract function) before structural changes (move function, replace conditional with polymorphism). If two changes are independent, the higher-severity smell goes first.

Present the plan to the user. Include:
- A brief summary of what was found and the overall goal
- The ordered steps with enough detail that the user can evaluate each one
- Any trade-offs or judgment calls you made (e.g., "I'm leaving X alone because it's low-severity and touching it risks Y")

### Phase 4: Apply on Confirmation

If the user confirms:
- Apply each step **one at a time**, following the technique's mechanics from the reference file
- **Run the full test suite after every step.** This is non-negotiable. Small steps compose safely only if validated.
- If a test fails: **stop, revert the step, and investigate.** Don't push forward with failing tests. Diagnose whether the failure is a genuine behavior change (step was wrong) or a test that was too tightly coupled to internal structure (update the test).
- Show results after each significant step — what changed and that tests still pass
- If you discover new smells while refactoring, note them but don't address them in the current step. Finish the planned refactoring first.

If the user only wants a review (no code changes), stop after Phase 3.

## Reference Files

- **`references/smell-catalog.md`** — 24 code smells: how to spot each, which techniques to apply
- **`references/testing-guide.md`** — How to write a refactoring safety net: fixture isolation, characterization tests, boundary probing, when to stop
- **`references/refactoring-catalog.md`** — Index of all 66 techniques with file paths
- **`references/techniques/`** — 66 files, one per refactoring technique, each with step-by-step mechanics and worked examples. Read only the ones the smell report points to.
