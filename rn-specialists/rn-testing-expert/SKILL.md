---
name: rn-testing-expert
description: "Expert React Native testing guidance — Jest + React Native Testing Library (queries, user-event, async act/waitFor, renderHook, provider wrappers), mocking native modules, snapshots, and e2e (Detox vs Maestro): when to use each, flakiness, parallelization. Loaded by rn-execute (apply while writing tests) and rn-code-review (critique) when work touches tests. Reads .claude/rn-profile.md."
---

# React Native Testing Expert (Jest + RNTL, Detox/Maestro e2e)

How it's used: **rn-execute** *applies* this while writing tests; **rn-code-review** uses it as the
testing *critique* lens — both when work touches tests.

**Source of truth:** the **official docs are the primary authority** (see `SOURCES.yaml`) —
[jestjs.io](https://jestjs.io/docs/tutorial-react-native) for the runner, **React Native Testing
Library** ([oss.callstack.com/react-native-testing-library](https://oss.callstack.com/react-native-testing-library))
for component/hook testing, and for e2e the **maintainer docs** of each tool —
[wix.github.io/Detox](https://wix.github.io/Detox) and [maestro.dev/docs](https://docs.maestro.dev).
They win on any conflict. Community sources supply structure and mistake patterns, not semantics.

**Profile-aware:** read `.claude/rn-profile.md` FIRST. Honor:
- **`test_roots`** — where unit/component tests live; scope reviews and new tests there.
- **`e2e`** (`detox` | `maestro` | `none`) — picks the e2e stack in §6. If `none`, don't push e2e; if set, critique against *that* tool's idioms, not the other.
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
- **Parameterized cases:** use **`it.each`/`test.each`** for the same behavior across many inputs — each row is its own reported case (failure names the input), far better than an in-test `for` loop or copy-paste. Keep expectations as concrete literals, not values derived from the code under test.

```tsx
const user = userEvent.setup()
render(<LoginForm onSubmit={onSubmit} />)
await user.type(screen.getByLabelText('Email'), 'a@b.co')
await user.press(screen.getByRole('button', { name: 'Sign in' }))
expect(onSubmit).toHaveBeenCalledWith({ email: 'a@b.co' })
```

## 2. Testing custom hooks & rendering with providers

- **Custom hooks → `renderHook`** (from RNTL), not a fake host component. Read state via `result.current`; wrap any call that updates state in **`await act(async () => …)`**; use `rerender`/`unmount` to exercise dependency changes and cleanup. Assert on the hook's *returned* values/handlers, not its internals. [callstack RNTL]

```tsx
const { result } = renderHook(() => useCounter(10))
await act(async () => result.current.increment())
expect(result.current.count).toBe(11)
```

- **Components/hooks that need context** (navigation, query client, a store, theme, SafeArea) — pass a **`wrapper`** to `render`/`renderHook` instead of mounting the whole app. Build one reusable `renderWithProviders` helper (a fresh `QueryClient` **per test** — no shared cache; an in-memory/real store; `NavigationContainer` or the Expo Router test harness as the project's `navigation` dictates).
- Missing-provider failures surface as "could not find <X> context" — that's a setup gap, not a component bug; fix the wrapper, don't mock the hook away.

## 3. Mocking native modules

- **Native modules don't exist in the Jest node env** — calls throw or no-op. Mock them. Prefer a **library's own jest preset/setup** where it ships one; otherwise `jest.mock(...)`.
- Common, official mocks to wire in **`jest.setup.js`** (referenced via `setupFiles`/`setupFilesAfterEach`):
  - **`@react-native-async-storage/async-storage`** → the package's documented jest mock.
  - **`react-native-reanimated`** → its bundled mock (`require('react-native-reanimated/mock')` / setup entry).
  - **NetInfo** (`@react-native-community/netinfo`) → its jest mock.
- For your own native/TurboModule wrappers, `jest.mock('../path')` returning a typed fake; assert on the wrapper's behavior, not the native bridge.
- **Timers:** use `jest.useFakeTimers()` + `jest.advanceTimersByTime(...)` for time-dependent logic — but pair fake timers with RNTL's async helpers carefully (`waitFor` under fake timers needs the timers advanced). Don't `sleep` real time in tests.
- Keep mocks **minimal and deterministic** — mock only the boundary you don't own; over-mocking hides real integration bugs.

## 4. Snapshot testing — sparingly

- Snapshots verify *unintended* output changes; they are **not** a substitute for behavior assertions. A passing snapshot proves nothing about correctness — only that output didn't change.
- Keep them **small and intentional** (`toMatchSnapshot()` on a focused subtree, or inline `toMatchInlineSnapshot()` so the expected value is reviewable in the test). **Avoid huge full-tree snapshots** — they're noisy, get rubber-stamped on update (`-u`), and rot. Never snapshot non-deterministic output (dates, ids, animated values).
- A changed snapshot in review is a prompt to *read the diff and decide*, not to blindly re-record.

## 5. Test design

- **Behavior over implementation; public interface only.** Render and drive the component as a user would; never assert on internal state or call private methods. Tests should survive a refactor that preserves behavior.
- **State-over-interaction.** Prefer asserting the resulting *state/output* (what's rendered, what the user sees) over asserting that a particular function was called. Reserve spy/interaction assertions for genuine contracts (a callback prop, an analytics call) — they're brittle otherwise.
- **DAMP, not DRY.** Tests should be **D**escriptive **A**nd **M**eaningful **P**hrases — readable top-to-bottom with their setup inline; a little duplication beats a clever shared helper that hides what's under test. Use `fixture`-style builders with sensible defaults so each test sets only what matters.
- **Test pyramid.** Many fast unit/component tests (Jest + RNTL), fewer integration tests, **very few** e2e — e2e is slow and the most flake-prone, so cover only critical user journeys there, not edge cases.

## 6. E2E — Detox vs Maestro (profile-driven)

Choose per the profile's **`e2e`** field — these are **two valid, different approaches**; pick the one the project uses and critique against *its* idioms. Present neutrally; don't import a vendor "verdict."

- **Detox** ([wix.github.io/Detox]) — **gray-box** e2e: has access to the app's internals, so it **synchronizes with the app** (waits for it to be idle) rather than relying on arbitrary sleeps. Tests are **authored in JavaScript/TypeScript** alongside the codebase; aims for "maximum velocity and zero flakiness" on real devices/simulators. More setup, but co-located JS tests and tight sync suit JS-heavy teams.
- **Maestro** ([maestro.dev/docs]) — **black-box** e2e: drives the app from the outside via **declarative YAML flows**; "the simplest and most effective framework for painless mobile UI automation." Minimal/zero IDE setup, fast to start, with built-in waiting that tolerates UI timing. Lower barrier; flows are language-agnostic, not co-located with app code.
- **Stable e2e regardless of tool:** anchor selectors on accessibility identifiers / `testID`, not pixel positions or brittle text; rely on the framework's built-in synchronization/wait rather than fixed sleeps; keep flows scoped to one journey; seed deterministic data and reset app state between flows.

## 7. Flakiness, coverage & parallelization

- **Jest runs test files in parallel across workers by default** — so **no shared mutable module state, no order dependence** between files. Reset between tests: `clearMocks`/`resetMocks` (or `jest.clearAllMocks()` in `afterEach`), restore real timers, clear AsyncStorage/NetInfo mocks, fresh `QueryClient` per test.
- **Flakiness checklist:** no `setTimeout`/sleep-based waiting (use `waitFor`/`findBy*`); no un-awaited async (`act`/`user-event`/`findBy*` are all async — missing `await` is the #1 RNTL flake); no order reliance or unreset globals/singletons; deterministic data and a fixed clock (fake timers, injected dates) — no `Date.now()`/random in assertions; mock the network, never hit it.
- **Coverage** is a gap-finder, not a goal — `jest --coverage`; enforce a sane `coverageThreshold` if the project wants a gate, but treat 100% as a smell (it rewards testing trivial getters). Cover behavior and branches that matter, not lines for their own sake.
- **E2e is the flakiest tier** — keep it small (pyramid), lean on the tool's synchronization, retry only as a stopgap (with a tracked TODO), and run it on demand, not in `verify_command`.

## 8. Common mistakes / anti-patterns

- Missing `await` on `act` / `user-event` / `waitFor` / `findBy*` (v14 async) — the top flake source.
- `*ByTestId` where a role/label/text query would work; querying by implementation detail.
- `getBy*` to assert *absence* (it throws) — use `queryBy*` for "not present."
- Mounting the whole app (or mocking the hook away) instead of a `wrapper` with the needed providers; sharing one `QueryClient`/store across tests so cache bleeds between them.
- Testing a custom hook via a throwaway host component instead of `renderHook`; reading hook internals instead of its returned API.
- Huge full-tree `toMatchSnapshot` rubber-stamped with `-u`; snapshotting non-deterministic output.
- Arbitrary `setTimeout`/sleep instead of `waitFor`/`findBy*`; unmanaged fake timers stalling `waitFor`.
- `fireEvent` where `user-event` would exercise the real interaction path.
- Forgetting to mock a native module (AsyncStorage/Reanimated/NetInfo) → cryptic env errors; or over-mocking until the test proves nothing.
- Asserting on internal state/props instead of rendered behavior; spy-heavy interaction tests that break on refactor.
- Putting e2e in the unit gate; treating Detox and Maestro as interchangeable in one suite instead of following the profile's `e2e`.
- Citing vendor "X vs Y" comparison blogs as neutral verdicts — ground the choice in each tool's own maintainer docs and the profile.
