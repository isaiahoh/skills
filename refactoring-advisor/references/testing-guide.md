# Testing Guide for Refactoring

Write tests that lock down current behavior before any code changes. The tests are your safety net — run them after every refactoring step. You are not testing correctness — you are preserving observable behavior so refactoring can proceed safely.

## Core Principles

1. **Tests must be fully automatic and self-checking.** Every test verifies its own result via assertions. Never rely on visual inspection or manual checking of console output.
2. **Testing is risk-driven, not exhaustive.** Focus on areas most likely to break. Don't test trivial accessors that just read/write a field. *It is better to write and run incomplete tests than not to run complete tests.*
3. **Fresh fixture per test.** Never share mutable state between tests. Use `beforeEach` / `setUp` / equivalent to create a new fixture for every test. Shared mutable fixtures cause nondeterministic failures that destroy trust in the test suite.

## What to Test

- **Observable outputs**: return values, side effects, state changes — what callers care about
- **Each execution path**: branches, switch cases, loops — focus on paths where bugs are likely
- **Boundaries**: empty collections, zero, negatives, empty strings, single-element collections, threshold edges, wrong types
- **Key calculations**: verify with known inputs and hand-checked expected outputs
- **Error conditions**: bad input behavior must be preserved (errors, exceptions, edge-case returns)
- **Modifications to fixtures**: when setters or mutators trigger derived calculations or side effects, test through the mutation

**Don't test**: internal helpers that will be restructured, performance characteristics, exact internal state. Test through the public interface.

## How to Write

1. **Identify the public interface** — functions/methods callers interact with. These are your test entry points.
2. **Create fixtures** — representative input data that exercises each code path. Use a `beforeEach` / `setUp` block so each test gets a fresh copy.
3. **Write self-checking tests** — follow the **arrange-act-assert** pattern:
   - **Arrange**: set up the fixture (often handled by `beforeEach`)
   - **Act**: exercise the code under test
   - **Assert**: verify the result with an assertion
   - *(Teardown is usually implicit when using fresh fixtures per test)*
4. **Prefer one assertion per test.** Multiple assertions in a single test hide failures — when the first assertion fails, subsequent ones don't run. Group closely related checks only when they verify aspects of the same logical operation.
5. **Probe boundaries** — after happy-path tests, systematically test:
   - Empty collections (what if the list has zero items?)
   - Zero and negative numbers
   - Empty strings and missing/null data
   - Values at exact thresholds (one above, one below)
   - Wrong types or malformed input (e.g., string where number expected)
6. **Verify each test can fail.** After writing a test, temporarily inject a fault into the code (e.g., add `* 2` to a calculation), confirm the test fails, then revert the fault. This guards against tests that pass for the wrong reason.
7. **Run the full suite against unmodified code** — all tests must pass before refactoring begins. If a test fails, fix the test, not the code.

## Characterization Tests (Locking Down Existing Code)

When writing tests for code that already exists and is assumed to be working:

1. Set the expected value to a **placeholder** (e.g., `expect(result).equal("TODO")`)
2. Run the test — it will fail and show you the **actual value**
3. Replace the placeholder with the actual value
4. **Inject a fault** into the code, confirm the test now fails
5. Revert the fault — the test passes again

You're not verifying correctness — you're locking down current behavior so you'll know if refactoring changes it.

## Test Naming and Organization

- **Name tests descriptively**: `test_<unit>_<scenario>` or `"<unit> <scenario>"` — the name should tell you what broke when the test fails.
- **Group tests by unit**: one `describe` / test class per public function or logical unit.
- **Standard fixture, visible variations**: use `beforeEach` for the common case. When a test needs a different fixture, create it within the test and call it out clearly (e.g., a separate `describe` block for "no producers" or "empty input").

## Framework Selection

Use whatever is standard for the language: pytest (Python), Jest/Vitest (JS/TS), JUnit 5 (Java), xUnit (C#), RSpec (Ruby), built-in `testing` (Go), Catch2 (C++). If the codebase already has a test setup, use that.

## Test File Placement

Match the project's existing conventions. If there are no conventions:
- Place test files adjacent to source files (`foo.test.js` next to `foo.js`), or
- Mirror the source tree under a `tests/` directory

## Handling Side Effects and External Dependencies

When the code under test performs I/O, network calls, or database access:
- **Prefer testing through the public interface** at the highest stable level you can.
- If external calls make tests slow or flaky, stub/mock only at the boundary (the I/O call itself), not internal logic.
- If the code is too tangled to test without mocking internals, note this as a smell — it often indicates Feature Envy or a need for Extract Function to separate pure logic from side effects.

## When to Stop

The right measure is subjective confidence: *How confident are you that if someone introduces a defect, some test will fail?* If you can refactor and be pretty sure you haven't introduced a bug because tests come back green, you have good enough tests.

Signs of over-testing: you spend more time changing tests than the code under test, and the tests feel like they're slowing you down. This is rare compared to under-testing, but watch for it.
