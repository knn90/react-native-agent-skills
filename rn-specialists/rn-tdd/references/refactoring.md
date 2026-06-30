# Refactor candidates — TypeScript

> Reference for [`rn-tdd`](../SKILL.md). TypeScript-native rewrite of mattpocock/skills `tdd/refactoring.md` (MIT).
> The REFACTOR step of the cycle — **only while green**, re-run the suite after each step.

After a test goes green, scan the new (and surrounding) code for these smells:

| Smell | Fix |
|---|---|
| **Duplication** | Extract a function / module; lift shared setup |
| **Long function** | Break into small helpers — keep tests on the **public** interface, not the helpers |
| **Shallow module** (big interface, thin body) | Combine or **deepen** — hide complexity behind a small interface |
| **Feature envy** (logic reaching into another type's data) | Move the logic to where the data lives |
| **Primitive obsession** (`string`/`number` carrying domain meaning) | Introduce a typed value — a branded type, a `type Status = …` union, a typed id |
| **Existing code the new code exposes as wrong** | Fix it now while you have tests + context |

## Rules
- **Never refactor while RED.** Get to green first, then improve with the suite as your safety net.
- **Re-run tests after every refactor step** — behavior must stay identical; the tests prove it.
- **Don't change behavior here.** New behavior needs a new failing test first (back to RED) — refactoring is
  structure-only.
- **Test through the public interface**, so extracting helpers/splitting modules doesn't touch a single test
  (if it does, the tests were coupled to implementation — see [tests.md](tests.md)).
- For SOLID/value-object/deep-module judgement calls, defer to `rn-solid-expert`; for component/hook-specific
  restructuring, `react-expert`.

Watch for what new code **reveals** about old code — the cheapest time to fix an adjacent design problem is
when you already have it loaded and under test. (Still: keep the change scoped — note larger cleanups rather
than sprawling.)
