---
name: react-expert
description: "Expert React / React Native component guidance — hooks rules, render performance, React-Compiler-aware memoization, list performance (FlatList/FlashList), accessibility. Loaded by rn-execute (apply while writing) and rn-code-review (critique) when work touches components/hooks. Reads .claude/rn-profile.md."
---

# React Expert

How it's used: **rn-execute** loads this and *applies* it while writing React / React Native
components and hooks; **rn-code-review** uses it as the component/hooks *critique* lens.

**Source of truth:** the **official docs are the primary authority** — **react.dev** for React,
hooks, and the React Compiler; **reactnative.dev** for RN primitives, list performance, and
accessibility (see `SOURCES.yaml`). They win on any conflict. Community sources fill in detail;
genuinely contested calls are flagged, not silently picked.

**Profile-aware:** read `.claude/rn-profile.md` FIRST. Two fields change the verdicts below:
- **`react_compiler`** (`enabled` | `disabled`) — flips the memoization stance (§2).
- **`new_architecture`** (`true` | `false`) — gates FlashList v2 recommendations (§3).

Also honor `rn_version`, `architecture` (free-form — RN has no canonical folder pattern; don't
impose one), and `language`. Where the profile is silent, prefer the official-docs default.

---

## 1. Hooks correctness (these ARE bugs)

- **Rules of Hooks** (react.dev/reference/rules/rules-of-hooks): call Hooks only at the **top
  level** of a component or a custom Hook — never inside conditions, loops, nested functions,
  `try`/`catch`, or after an early `return`. Only call them from React function components or
  custom Hooks (names starting with `use`). A conditionally-called Hook is a defect, full stop.
- **exhaustive-deps** (react.dev/reference/eslint-plugin-react-hooks/lints/exhaustive-deps): an
  effect / memo / callback dependency array must include **every reactive value** it reads (props,
  state, context, and values derived from them). Omitting one → **stale closures** reading old
  values. **Never** silence the lint with `// eslint-disable-next-line` — that's a real bug
  smuggled past the linter. Fix the cause instead: move the value inside the effect, lift it to a
  ref, extract a pure function, or remove the dependency by restructuring
  (react.dev/learn/removing-effect-dependencies). **Treat an exhaustive-deps suppression as a
  compilation error.**
- **Effects** model *synchronization with an external system*, not "run some code after render."
  If an effect only computes a value from props/state, derive it during render instead. If it only
  responds to a user event, that logic belongs in the event handler. Every effect that subscribes /
  opens / connects needs a matching **cleanup**.
- **Custom Hooks** share *logic*, not *state* — each call site gets its own independent state.
  Name them `useX`; keep them pure (no side effects at call time outside effects).

## 2. Render performance + memoization stance (PROFILE-DRIVEN)

**Memoization is a performance optimization, NEVER a correctness mechanism.** Both
react.dev/reference/react/useMemo and /useCallback say it plainly: *"If your code doesn't work
without it, find the underlying problem and fix it first."* A component that breaks without a memo
has a real bug (usually a missing key, an unstable identity, or state that should be derived) —
fix *that*. **Do NOT report a missing `useMemo` / `useCallback` / `React.memo` as a bug.**

- **`react_compiler: enabled`** (default in Expo SDK 54+; the React Compiler reached **v1.0 stable
  on 2025-10-07** and auto-memoizes at build time for React *and* React Native —
  react.dev/learn/react-compiler/introduction, react.dev/blog/2025/10/07/react-compiler-1):
  - For **new** code, rely on the compiler — don't hand-add `useMemo`/`useCallback`/`React.memo`.
  - Treat any remaining manual memo as an **escape hatch**. The most common legitimate use is
    **stabilizing a value/function used as another Hook's dependency** (e.g. an effect dep).
  - For **existing** memoization: leave it in place. Don't strip it blindly — test before removing,
    since some was added to satisfy a real dependency-identity need.
- **`react_compiler: disabled`** — manual memo can matter, but only in the narrow cases below.

**`useMemo` is only valuable in three cases** (react.dev/reference/react/useMemo):
  1. the calculation is **noticeably slow** *and* its dependencies rarely change,
  2. the value is **passed to a `React.memo`-wrapped child** (preserves its identity), or
  3. the value is **used as another Hook's dependency**.
  Outside these there's **no benefit** — *but no significant harm either*. **Blanket memoization is
  a valid team style**; the only downside is readability. **Do NOT flag blanket memoization as an
  error** — at most note it as a style preference.

**`useCallback` is valuable only when** the function is (1) passed to a `React.memo`-wrapped child,
or (2) used as another Hook's dependency (react.dev/reference/react/useCallback).

**Expensive-calculation heuristic** (react.dev/reference/react/useMemo): don't guess — *measure*
with `console.time('x') … console.timeEnd('x')`. Memoize only if it's reliably **~1ms+**. Per the
docs: *"unless you're creating or looping over thousands of objects, it's probably not expensive."*

**Identity & keys (these affect correctness, so they're fair game):**
- Stable, domain-meaningful **`key`** on list items — never the array index for dynamic/reorderable
  lists (causes state to attach to the wrong row). `key` resets a component's state when it changes.
- Don't create new object/array/function literals inline *as props to a `React.memo` child* — it
  defeats the memo. (With the compiler on, this is handled for you.)
- Lift expensive *derivations* out of the render path; don't recompute formatted/sorted/filtered
  data on every render when the inputs didn't change.

## 3. List performance (RN)

For long/virtualized lists use a **virtualized** list, not `.map()` inside a `ScrollView`
(`ScrollView` renders every child up front).

- **`FlatList`** is a thin Virtual List over `VirtualizedList`
  (reactnative.dev/docs/optimizing-flatlist-configuration, /docs/performance, /docs/virtualizedlist):
  - **`windowSize`** default **21** (≈ 10 screens each side of the viewport), **`maxToRenderPerBatch`**
    default **10**. Tuning these is a three-way tradeoff: **blank space on fast scroll** vs **JS-thread
    responsiveness** vs **memory**. Lower `windowSize` → less memory but more blanking; higher
    `maxToRenderPerBatch` → fewer blanks but choppier scrolling. Don't change defaults without a
    measured reason.
  - **Wrap each row component in `React.memo`** so unrelated state changes don't re-render every row.
  - **Hoist `renderItem` out of the parent body and wrap it in `useCallback`.** This is a
    **legitimate manual-memo case even when `react_compiler: enabled`** — a stable `renderItem`
    identity keeps `FlatList` from treating the list as changed. (Don't penalize this as "redundant
    memo.")
  - Keep item components **light** — avoid heavy nesting, inline image decoding, or per-row
    expensive work.
  - Provide **`getItemLayout`** for **fixed-height** rows — it skips async layout measurement and
    makes scroll-to-index and initial render cheaper.
  - Set a real **`keyExtractor`** (stable id); avoid anonymous arrow props that change every render.
  - reactnative.dev itself names **FlashList** and **Legend List** as performance alternatives when
    `FlatList` isn't enough.
- **FlashList v2** (shopify.github.io/flash-list/docs, github.com/Shopify/flash-list) — recommend
  **only when `new_architecture: true`**. v2 is **New-Architecture-only** (use v1 for old arch),
  **removes `estimatedItemSize`** (no longer needed), and recycles views via a configurable pool.
  If `new_architecture` is false/unset, do **not** suggest FlashList v2 — stay on a tuned `FlatList`
  (or FlashList v1).

## 4. Accessibility (RN)

Per reactnative.dev/docs/accessibility:

- Mark interactive/meaningful elements **`accessible`**, and give them an **`accessibilityLabel`**
  (what it is) and an **`accessibilityRole`** (`button`, `link`, `header`, `image`, `adjustable`, …)
  so the screen reader announces them correctly.
- Reflect dynamic status with **`accessibilityState`** (`disabled`, `selected`, `checked`,
  `busy`, `expanded`) and `accessibilityValue` for sliders/progress — never convey state by color
  or position alone.
- Ensure adequate **hit targets** for touchable controls (use `hitSlop` when the visual is small);
  tiny tap areas fail accessibility and usability.
- Verify the **screen-reader reading order** is logical; group related content with
  `accessibilityElementsHidden` / `importantForAccessibility` and grouping views where the default
  order is wrong. Hide purely decorative elements from the accessibility tree.
- Prefer real touchable primitives (`Pressable`, `Button`, `TouchableOpacity`) over a bare `View`
  with an `onPress` — the touchables come with the right roles/traits for free.

## What NOT to flag

- A **missing** `useMemo` / `useCallback` / `React.memo`. Memo is performance, not correctness —
  absence is not a bug. If something breaks without it, the bug is elsewhere; point at *that*.
- **Blanket/defensive memoization** (when `react_compiler: disabled`). It's a valid team style with
  no meaningful runtime cost — at most a readability note, never an error.
- **Manual `useCallback` on a `renderItem`** (or a list/effect-dependency value) even when the
  compiler is on — that's a sanctioned escape hatch, not redundant work.
- **Existing** memoization under `react_compiler: enabled` — don't mass-delete it; test first.

## Currency

react.dev and reactnative.dev are the source of truth (above). Baseline consulted: **React 19**,
**React Compiler 1.0** (stable 2025-10-07, default in Expo SDK 54+), **React Native 0.81**. Version-
specific claims are gated to the profile's `rn_version` / `new_architecture` / `react_compiler`;
anything not verified against the official docs is omitted rather than guessed.
