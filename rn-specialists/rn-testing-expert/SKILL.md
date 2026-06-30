---
name: rn-testing-expert
description: "Expert React Native testing guidance — Jest + React Native Testing Library (queries, user-event, async act/waitFor), mocking native modules, and e2e (Detox vs Maestro): when to use each, flakiness, parallelization. Loaded by rn-execute (apply while writing tests) and rn-code-review (critique) when work touches tests. Reads .claude/rn-profile.md."
---

# React Native Testing Expert (Jest + RNTL, Detox/Maestro e2e)

How it's used: **rn-execute** *applies* this while writing tests; **rn-code-review** uses it as the
testing *critique* lens — both when work touches tests.

**Source of truth:** the **official docs are the primary authority** (see `SOURCES.yaml`) —
[jestjs.io](https://jestjs.io/docs/tutorial-react-native) for the runner, **React Native Testing
Library** ([oss.callstack.com/react-native-testing-library](https://oss.callstack.com/react-native-testing-library))
for component testing, and for e2e the **maintainer docs** of each tool —
[wix.github.io/Detox](https://wix.github.io/Detox) and [maestro.dev/docs](https://docs.maestro.dev).
They win on any conflict. Community sources supply structure and mistake patterns, not semantics.

**Profile-aware:** read `.claude/rn-profile.md` FIRST. Honor:
- **`test_roots`** — where unit/component tests live; scope reviews and new tests there.
- **`e2e`** (`detox` | `maestro` | `none`) — picks the e2e stack in §4. If `none`, don't push e2e; if set, critique against *that* tool's idioms, not the other.
- **`verify_command`** — the unit gate (typically `tsc --noEmit && eslint . && jest`). E2e is slow — run on demand, never as part of `verify_command`.

---

## 1. Unit / component — Jest + RNTL

- **Jest is RN's official default runner** — included by default since **RN 0.38** via the `react-native` preset (`preset: 'react-native'` in `jest.config.js`); a node env that mimics an RN app. Don't reinvent the runner config. [jestjs.io]
- **Test behavior through the public interface**, the way a user perceives the component — render, interact, assert on what's on screen. Don't reach into state, props, or instance internals.
- **Query priority (RNTL): accessibility-first.** Prefer queries by **role** (`*ByRole`), **label/text** (`*ByLabelText`, `*ByText`, `*ByPlaceholderText`, `*ByDisplayValue`) over `*ByTestId`. Use `testID` only as the last resort when nothing user-facing identifies the node — it couples tests to implementation. [callstack RNTL: queries]
- **Variant semantics:** `getBy*` (throws if 0 or >1 — assert presence), `queryBy*` (returns `null` — assert *absence*, never for presence), `findBy*` (async, retries — for elements that appear later). `*AllBy*` for collections.
- **async `act` (v14):** RNTL v14's `act()` is **async by default and always returns a Promise — always `await act(...)`**; v14 requires **React 19+**. You rarely call `act` directly: **`render`, `rerender`, and `fireEvent` are already wrapped in `act`**. For state that settles asynchronously, use **`await waitFor(...)`** or **`findBy*`** — never an arbitrary `setTimeout`/sleep. [callstack RNTL: understanding-act]
- **Prefer `user-event` over `fireEvent`** for realistic interaction — it dispatches the full event sequence a real user triggers (press in/out, focus, change), so it catches bugs `fireEvent` misses. `user-event` APIs are **async — `await user.press(...)` / `await user.type(...)`**; set up with `const user = userEvent.setup()`. Reserve `fireEvent` for low-level/edge cases. [callstack RNTL: user-event]
- Keep assertions on **observable output** (rendered text, accessibility state, callbacks fired) — `toHaveTextContent`, `toBeOnTheScreen`, `toHaveAccessibilityState`, jest matchers for spies.

## 2. Mocking native modules

- **Native modules don't exist in the Jest node env** — calls throw or no-op. Mock them. Prefer a **library's own jest preset/setup** where it ships one; otherwise `jest.mock(...)`.
- Common, official mocks to wire in **`jest.setup.js`** (referenced via `setupFiles`/`setupFilesAfterEnv`):
  - **`@react-native-async-storage/async-storage`** → the package's documented jest mock.
  - **`react-native-reanimated`** → its bundled mock (`require('react-native-reanimated/mock')` / setup entry).
  - **NetInfo** (`@react-native-community/netinfo`) → its jest mock.
- For your own native/TurboModule wrappers, `jest.mock('../path')` returning a typed fake; assert on the wrapper's behavior, not the native bridge.
- **Timers:** use `jest.useFakeTimers()` + `jest.advanceTimersByTime(...)` for time-dependent logic — but pair fake timers with RNTL's async helpers carefully (`waitFor` under fake timers needs the timers advanced). Don't `sleep` real time in tests.
- Keep mocks **minimal and deterministic** — mock only the boundary you don't own; over-mocking hides real integration bugs.

## 3. Test design

- **Behavior over implementation; public interface only.** Render and drive the component as a user would; never assert on internal state or call private methods. Tests should survive a refactor that preserves behavior.
- **State-over-interaction.** Prefer asserting the resulting *state/output* (what's rendered, what the user sees) over asserting that a particular function was called. Reserve spy/interaction assertions for genuine contracts (a callback prop, an analytics call) — they're brittle otherwise.
- **DAMP, not DRY.** Tests should be **D**escriptive **A**nd **M**eaningful **P**hrases — readable top-to-bottom with their setup inline; a little duplication beats a clever shared helper that hides what's under test. Use `fixture`-style builders with sensible defaults so each test sets only what matters.
- **Test pyramid.** Many fast unit/component tests (Jest + RNTL), fewer integration tests, **very few** e2e — e2e is slow and the most flake-prone, so cover only critical user journeys there, not edge cases.

## 4. E2E — Detox vs Maestro (profile-driven)

Choose per the profile's **`e2e`** field — these are **two valid, different approaches**; pick the one the project uses and critique against *its* idioms. Present neutrally; don't import a vendor "verdict."

- **Detox** ([wix.github.io/Detox]) — **gray-box** e2e: has access to the app's internals, so it **synchronizes with the app** (waits for it to be idle) rather than relying on arbitrary sleeps. Tests are **authored in JavaScript/TypeScript** alongside the codebase; aims for "maximum velocity and zero flakiness" on real devices/simulators. More setup, but co-located JS tests and tight sync suit JS-heavy teams.
- **Maestro** ([maestro.dev/docs]) — **black-box** e2e: drives the app from the outside via **declarative YAML flows**; "the simplest and most effective framework for painless mobile UI automation." Minimal/zero IDE setup, fast to start, with built-in waiting that tolerates UI timing. Lower barrier; flows are language-agnostic, not co-located with app code.
- **Stable e2e regardless of tool:** anchor selectors on accessibility identifiers / `testID`, not pixel positions or brittle text; rely on the framework's built-in synchronization/wait rather than fixed sleeps; keep flows scoped to one journey; seed deterministic data and reset app state between flows.

## 5. Flakiness & parallelization

- **Jest runs test files in parallel across workers by default** — so **no shared mutable module state, no order dependence** between files. Reset between tests: `clearMocks`/`resetMocks` (or `jest.clearAllMocks()` in `afterEach`), restore real timers, clear AsyncStorage/NetInfo mocks.
- **Flakiness checklist:** no `setTimeout`/sleep-based waiting (use `waitFor`/`findBy*`); no un-awaited async (`act`/`user-event`/`findBy*` are all async — missing `await` is the #1 RNTL flake); no order reliance or unreset globals/singletons; deterministic data and a fixed clock (fake timers, injected dates) — no `Date.now()`/random in assertions; mock the network, never hit it.
- **E2e is the flakiest tier** — keep it small (pyramid), lean on the tool's synchronization, retry only as a stopgap (with a tracked TODO), and run it on demand, not in `verify_command`.

## 6. Common mistakes / anti-patterns

- Missing `await` on `act` / `user-event` / `waitFor` / `findBy*` (v14 async) — the top flake source.
- `*ByTestId` where a role/label/text query would work; querying by implementation detail.
- `getBy*` to assert *absence* (it throws) — use `queryBy*` for "not present."
- Arbitrary `setTimeout`/sleep instead of `waitFor`/`findBy*`; unmanaged fake timers stalling `waitFor`.
- `fireEvent` where `user-event` would exercise the real interaction path.
- Forgetting to mock a native module (AsyncStorage/Reanimated/NetInfo) → cryptic env errors; or over-mocking until the test proves nothing.
- Asserting on internal state/props instead of rendered behavior; spy-heavy interaction tests that break on refactor.
- Putting e2e in the unit gate; treating Detox and Maestro as interchangeable in one suite instead of following the profile's `e2e`.
- Citing vendor "X vs Y" comparison blogs as neutral verdicts — ground the choice in each tool's own maintainer docs and the profile.
