---
name: rn-tdd
description: "The test-driven-development DISCIPLINE for React Native — red→green→refactor, the prove-it pattern (reproduce a bug with a failing test before fixing), vertical tracer-bullet slices (one test → one impl, never all-tests-then-all-code), test-behavior-through-public-interfaces, state-over-interaction, DAMP tests, real-over-mocks, the test pyramid. Process, not framework mechanics — for Jest matchers, RNTL queries/render/user-event, doubles, and migration see rn-testing-expert. Loaded by rn-execute while writing code (and available to rn-code-review as a TDD-discipline lens). Reads .claude/rn-profile.md (verify_command, test_roots)."
---

# React Native TDD — the discipline

> **Generated skill — original wording, consolidated by `rn-skill-consolidate`** from
> addyosmani/agent-skills (`test-driven-development`) · mattpocock/skills (`engineering/tdd`)
> — both MIT. TypeScript-native rewrite. **Authoritative testing semantics** live in
> [`rn-testing-expert`](../rn-testing-expert/SKILL.md) (Jest + React Native Testing Library docs).
> Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

This skill owns the **process**: how to drive code with tests. It does **not** teach framework
syntax — `expect` matchers, `render`/`screen`/`waitFor`, `user-event`, query priority, doubles,
mock placement, parallelization all live in **`rn-testing-expert`**. Use them together: this
decides *when/why* you write a test; `rn-testing-expert` decides *how* to write it idiomatically.

**How it's used:** `rn-execute` *applies* this while implementing; it's also available to
`rn-code-review` as a TDD-discipline critique lens.

## Step 0 — Load profile
Read `.claude/rn-profile.md`: `verify_command` (how to run the suite), `test_roots`, `source_roots`.
You will **run** tests here, not just write them — know the command before you start. If
`verify_command` is unset, find the targeted run (e.g. `jest path/to/file.test.ts` or
`jest -t "{test name}"`) so you can observe one test going red then green without the full suite.

## When to Use
Implementing any logic/behavior · fixing any bug (the **Prove-It** pattern) · modifying existing
behavior · adding edge-case handling · any change that could break existing behavior.

**When NOT to use:** pure non-logic changes — copy, assets, config, styling-only — say so in the report.

---

## The cycle — RED → GREEN → REFACTOR (run the suite at each step)

```
   RED                  GREEN                 REFACTOR
 write a test      minimum code to       clean up; behavior
 that FAILS   ──→  make it PASS    ──→    unchanged          ──→ (repeat)
     │                   │                     │
 RUN it →           RUN it →              RUN the suite →
 see it fail        see it pass           still green
```

The runs are **not optional**. "Looks right" / "should pass" is not done — **execute the suite and read
the output.**

1. **RED — write one failing test, run it, watch it fail.** A test that passes the first time proves
   nothing (it isn't exercising new behavior, or the assert is wrong). Confirm it fails **for the
   reason you expect** (the missing behavior) — not a typo, missing import, or compile error.
   ```ts
   // RED — cartTotal doesn't exist yet, so this fails to compile/run. That's the point.
   describe("cartTotal", () => {
     it("totals an empty cart to zero", () => {
       expect(cartTotal([])).toBe(0);
     });
   });
   ```
   Run it (`verify_command` or a targeted `jest -t`) and see RED before writing any production code.

2. **GREEN — minimum code to pass, then run.** Don't over-build; just satisfy this test. Run the suite
   and confirm it now passes (and nothing else broke).

3. **REFACTOR — tidy with tests green, re-run after each step.** Extract duplication, deepen modules,
   improve names. **Never refactor while RED** — get to green first. Run the suite after every refactor
   step to prove behavior is unchanged. (Candidates: [`references/refactoring.md`](references/refactoring.md).)

## Vertical slices, not horizontal — the cardinal anti-pattern
**Do NOT write all the tests first, then all the implementation.** That "horizontal" split (RED = all
tests, GREEN = all code) produces bad tests: written in bulk they verify *imagined* behavior and the
*shape* of data, drift insensitive to real breakage, and lock you into a structure before you understand
the code.

Go **vertical** — tracer bullets. One test → one minimal impl → repeat. Each cycle uses what you
learned from the last.
```
WRONG (horizontal):  RED: test1..test5     then  GREEN: impl1..impl5
RIGHT (vertical):    test1→impl1 · test2→impl2 · test3→impl3 · …
```

## The Prove-It pattern (bug fixes)
When a bug is reported, **do not start by fixing it.** First write a test that **reproduces** it, run it,
and watch it **fail** — that failing run confirms you've actually found the bug. Then fix; the test goes
green and stays as a regression guard. Finally run the **full suite** for no regressions.
```
bug report → write reproduction test → RUN: fails (bug confirmed)
           → implement fix → RUN: passes → RUN full suite: no regressions
```

## Plan before you write tests
- Confirm the **public interface** the change needs, and **which behaviors matter most** — you can't
  test everything; focus on critical paths and complex logic, not every edge.
- List **behaviors** to test (observable outcomes), not implementation steps. If a profile `CONTEXT`/
  rules file names the domain vocabulary, use it so test names match the domain.
- For a non-trivial change, get the behavior list approved before coding (this is the same approval the
  `rn-execute` plan gate covers — don't re-ask if it already happened).

## Writing good tests (TypeScript)
- **Test behavior through the public interface — not implementation.** Assert on the *outcome* (state
  the user/caller sees), not which internal functions were called. A test that breaks when you rename a
  private helper, with behavior unchanged, was testing the wrong thing. (Good/bad examples:
  [`references/tests.md`](references/tests.md); for components: query the rendered output the user sees,
  not internal hook state — see `rn-testing-expert §12`.)
  ```ts
  // Good — observable outcome
  expect(sortByDateDescending(notes)[0].createdAt).toBeGreaterThan(
    sortByDateDescending(notes).at(-1)!.createdAt,
  );
  // Bad — couples to internals; breaks on refactor though behavior is identical
  expect(dbSpy.lastQuery).toBe("ORDER BY created_at DESC");
  ```
- **DAMP over DRY in tests.** Each test reads like a self-contained specification; some duplication is
  fine if it makes the test independently understandable. Don't hide the inputs behind shared helpers.
- **Prefer real implementations over doubles:** real > fake (in-memory) > stub > mock. Use a double only
  when the real thing is slow, non-deterministic, or has side effects you can't control (network, clock,
  random). Over-mocking gives tests that pass while production breaks. (When/where to mock + design-for-
  substitutability: [`references/mocking.md`](references/mocking.md); double taxonomy + `jest.mock`
  placement: `rn-testing-expert §8`.)
- **Arrange-Act-Assert** structure; **one concept per test** (`rejects empty title`, `trims whitespace`
  — not one `validates correctly`); **descriptive names** that read as a spec, via `it("…")`.
- **Deterministic only** — no `new Date()`/`Math.random()` in tests; inject a fixed clock/value (or use
  `jest.useFakeTimers()` — see `rn-testing-expert §9`).

## The test pyramid
Most tests small and fast; fewer as you go up.
```
   E2E / UI (~5%)     full flows on a device/emulator (Detox / Maestro) — critical paths only
 Integration (~15%)   crosses a boundary (network, storage) with a test seam
   Unit (~80%)        pure logic + component render, isolated, milliseconds
```
| Size | Constraints | Speed |
|---|---|---|
| Small | one process, no I/O / network / disk | ms |
| Medium | localhost only, test DB ok, no external services | seconds |
| Large | external services, device/emulator (Detox/Maestro) | minutes |

**The Beyoncé rule:** if you liked it, you should have put a test on it. A refactor or migration isn't
responsible for catching your bug — your tests are.

## Anti-patterns
| Anti-pattern | Fix |
|---|---|
| Testing implementation details | Test inputs → outputs / rendered state, not internal structure |
| Flaky (timing / order-dependent) | Deterministic asserts, isolated state, `waitFor` over `setTimeout`/`sleep` (see `rn-testing-expert §9`) |
| Mocking everything | real > fake > stub > mock; mock only at uncontrollable boundaries |
| Snapshot abuse | sparingly; review every change |
| Bug fix without a reproduction test | Prove-It: failing test first |
| Asserting "all pass" without running | run the suite, read the output |

## Rationalizations (all false)
"I'll test after it works" (you won't; after-the-fact tests test implementation) · "too simple to test"
(simple code complicates; the test documents intent) · "tests slow me down" (now; they speed up every
later change) · "I tested it manually" (doesn't persist) · "it's just a prototype" (prototypes ship).

## Red flags — stop
Code with no corresponding test · a test that passed on its **first** run (suspect) · "all tests pass"
when none were actually run · a bug fix with no reproduction test · test names that don't describe
behavior · `it.skip`/`xit`/disabling tests to make the suite green · re-running the same command on
unchanged code as reassurance (run after a *change*, not for comfort).

## Per-cycle checklist
```
[ ] One behavior, one test — vertical slice (not all-tests-then-all-code)
[ ] Test describes behavior via the public interface; survives an internal refactor
[ ] RED observed: ran it, saw it fail for the expected reason
[ ] GREEN: minimal code; ran it, saw it pass — no speculative features
[ ] Refactored only while green; re-ran the suite after each step
[ ] Bug fix? a reproduction test failed before the fix
[ ] Full suite run before claiming done — no regressions
```

## References
- [`references/tests.md`](references/tests.md) — good vs bad tests; behavior-through-the-interface examples.
- [`references/mocking.md`](references/mocking.md) — when/where to mock (boundaries only) + design-for-substitutability (DI, SDK-style interfaces).
- [`references/refactoring.md`](references/refactoring.md) — refactor candidates for the REFACTOR step.
- `rn-testing-expert` — framework mechanics: Jest matchers, RNTL `render`/`screen`/`waitFor`/`user-event`, query priority, doubles taxonomy, `jest.mock` placement, fake timers/flakiness.
