---
name: rn-state-expert
description: "Expert React Native state guidance — client state (Redux Toolkit / Zustand / Jotai / Context) and server state / data-fetching (TanStack Query / RTK Query / SWR): cache invalidation, refetch races, stale closures, and the RN-specific wiring web apps don't need. Loaded by rn-execute and rn-code-review when work touches state/queries/stores. Reads .claude/rn-profile.md."
---

# React Native State Expert

How it's used: **rn-execute** loads this and *applies* it while writing state / query / store
code; **rn-code-review** uses it as the state *critique* lens. It's routed in when the diff touches
`useQuery`/`useMutation`/`queryClient`, `createSlice`/`configureStore`, `create((set)=>`, `atom(`,
`useStore`, or server-state/cache code.

**Source of truth:** the **official maintainer docs are the primary authority** and win on any
conflict — **TanStack Query** for server state (incl. its React Native page), and **Redux Toolkit /
Zustand / Jotai / SWR** for their respective libraries (see `SOURCES.yaml`). Community sources only
supply structure. Client-state *library choice* is a **project convention, not an official mandate**
(see §6).

**Profile-aware:** read `.claude/rn-profile.md` FIRST. Two fields drive the verdicts below —
- **`data_fetching`** (`tanstack-query` | `rtk-query` | `swr` | `apollo` | `fetch` | `none`) — which
  server-state lens applies (§1–§4).
- **`state`** (`redux-toolkit` | `zustand` | `jotai` | `context` | `mobx` | `none`) — which client-state
  lens applies (§5). Align to what the project already uses; **don't push a library switch.**

Also honor `architecture` (free-form — no canonical RN pattern; don't impose one) and `language`.

---

## 1. SERVER STATE — RN manual wiring (THE centerpiece)

TanStack Query "works out of the box" in React Native, **but two automatic web behaviors are NOT
automatic on native** and must be wired by hand once, at app setup
(tanstack.com/query/v5/docs/framework/react/react-native). Missing this wiring is the #1 RN
data-fetching defect — queries silently go stale because they never refetch.

- **Refetch on reconnect — NOT automatic.** TanStack Query has no built-in network-status detector
  on native. Wire `onlineManager` once with **`@react-native-community/netinfo`** (or
  `expo-network`):
  ```ts
  import NetInfo from '@react-native-community/netinfo';
  import { onlineManager } from '@tanstack/react-query';
  onlineManager.setEventListener((setOnline) =>
    NetInfo.addEventListener((state) => setOnline(!!state.isConnected)),
  );
  ```
  Without it, queries don't know the device went offline/online and won't refetch on reconnect.
- **Refetch on app focus — must use `AppState`, NOT browser focus.** Native has no window-focus
  event. Subscribe to the `AppState` `'change'` event and drive `focusManager`, **guarded by
  `Platform.OS !== 'web'`** so the web build keeps its native focus handling:
  ```ts
  import { AppState, Platform } from 'react-native';
  import { focusManager } from '@tanstack/react-query';
  AppState.addEventListener('change', (status) => {
    if (Platform.OS !== 'web') focusManager.setFocused(status === 'active');
  });
  ```
- **Verify the wiring exists** before trusting `refetchOnReconnect` / `refetchOnWindowFocus`. If the
  app relies on focus/reconnect refetch but never set up `onlineManager`/`focusManager`, that's a bug
  even though nothing errors.
- **`screen` focus ≠ `AppState` focus.** Refetching when a *screen* regains focus (navigating back)
  is a separate concern — drive it from the navigator's focus event (e.g. `useFocusEffect` +
  `refetch`, or `refetchOnMount`), not from `focusManager`.

*(If `data_fetching: rtk-query` or `swr`, the same RN gap applies — these libraries also assume web
focus/online events. RTK Query exposes `setupListeners` (redux-toolkit.js.org/rtk-query) and `focus`
behavior that needs equivalent native wiring; SWR (swr.vercel.app) needs
`revalidateOnFocus`/`revalidateOnReconnect` bridged to `AppState`/NetInfo. Apply the same checklist.)*

## 2. Cache invalidation & query-key design

- **Query keys are the cache identity.** They must be **stable, serializable, and fully describe the
  inputs** — every variable the query depends on (ids, filters, params) belongs *in the key*, not
  closed over. A param read from a closure but absent from the key → stale data served for the wrong
  inputs. Use structured keys (`['todos', { status, page }]`).
- **Invalidate by key after mutations.** A mutation that changes server data must
  `queryClient.invalidateQueries({ queryKey: [...] })` (or update the cache directly) so dependent
  queries refetch. A successful mutation with no invalidation = a screen showing stale data until the
  next cold load.
- **Don't over-invalidate.** Invalidating a broad prefix refetches everything under it — scope the
  key to what actually changed.

## 3. Refetch races, stale data & loading transitions

- **Refetch races / async overwrite.** When inputs change fast (typing a search, switching tabs), an
  older in-flight request can resolve *after* a newer one and overwrite correct state with stale
  results. With a query lib, key the query by the input so each input is its own cache entry and the
  library discards the loser. In a hand-rolled `useEffect` fetch, you **must** guard with an
  `ignore`/`AbortController` cleanup — this is the classic RN stale-closure data bug.
- **`isLoading` vs `isFetching` vs `isPending`.** `isLoading`/`isPending` = no cached data yet (first
  load → show a spinner); `isFetching` = a background refetch with data already on screen (keep the
  stale data, maybe a subtle indicator). Don't blank the screen on every background refetch.
- **Cover every transition.** loading → loaded / **empty** / **error**, plus refetch-while-showing-data.
  A missing empty or error branch is a state-transition hole, not a cosmetic gap.

## 4. Mutations, optimistic updates & dependent/parallel queries

- **Optimistic update needs rollback.** If you write the cache before the server confirms, snapshot
  the previous value in `onMutate`, **roll back in `onError`**, and reconcile in `onSettled`
  (invalidate or refetch). An optimistic update with no rollback path leaves the UI lying after a
  failed request.
- **Dependent queries** — gate a query that needs another's result with `enabled: !!dependency`
  rather than nesting fetches; don't fire it with an `undefined` param.
- **Parallel queries** — independent reads run in parallel (`useQueries` for a dynamic set); don't
  serialize them in an effect chain.
- **Pagination / infinite** — use the library's `useInfiniteQuery` (cursor/`getNextPageParam`); don't
  accumulate pages in client state by hand (re-introduces the stale/race bugs the lib already solves).

## 5. CLIENT STATE — library-agnostic correctness (drive by profile `state`)

These hold **whatever** `state` library the project uses; apply the matching per-library note below.

- **Don't store server state in a client store.** Cache/server data (anything fetched from an API)
  belongs in the **`data_fetching`** layer (TanStack Query / RTK Query / SWR), *not* hand-mirrored
  into Redux/Zustand/Jotai/Context. Duplicating it re-creates the invalidation, refetch-race, and
  staleness problems the query layer exists to solve. Client stores are for **client-owned** state
  (UI toggles, form/wizard state, session, preferences).
- **Selector stability & stale closures.** A selector returning a **new object/array each call**
  (`(s) => ({ a: s.a, b: s.b })`) makes the component re-render every store update — use stable
  selectors, select primitives, or the library's shallow-equality helper. Selectors and effects that
  close over store values can read **stale** snapshots; subscribe/select the value rather than
  capturing it once.
- **Single source of truth.** Don't mirror store/server state into local `useState` — derive during
  render. Mirrored state drifts out of sync.

**Per-library notes (apply the one matching `state`):**
- **`redux-toolkit`** (redux-toolkit.js.org) — use `createSlice` + `configureStore`; reducers are
  "mutating" via Immer but stay immutable underneath. Use `createSelector` for derived/memoized
  reads. For *server* state prefer **RTK Query** (`createApi`) over hand-written thunks. RTK requires
  wrapping the app in a `<Provider>`.
- **`zustand`** (zustand.docs.pmnd.rs) — "Zustand and Redux both use an immutable state model;
  Redux requires wrapping the app in context providers, **Zustand does not**"
  (zustand.docs.pmnd.rs comparison). Subscribe with **narrow selectors** + `useShallow` for object
  picks to avoid over-rendering; never select the whole store.
- **`jotai`** (jotai.org) — atomic, bottom-up; state is split into `atom`s, components subscribe to
  the specific atoms they read (fine-grained re-renders by design). Derive with read-only/derived
  atoms instead of recomputing in components.
- **`context`** — built-in, **fine for low-frequency or global config** (theme, auth, locale). But
  every consumer re-renders on **any** value change, and a new object/array `value=` each render
  re-renders all consumers — memoize the value and split contexts by update cadence. Don't use Context
  as a high-frequency app store; that's what the dedicated libraries are for.

## 6. Don't mandate a library / don't store server state in a client store

- **Client-state library choice is a project convention, not an official mandate.** react.dev has no
  opinion on Redux vs Zustand vs Jotai vs Context. Present trade-offs **neutrally**, align to the
  project's **`state`** field, and **don't propose a switch** for a small change. Only raise a library
  change if the profile is `none` *and* the work genuinely needs shared cross-component state.
- **The one hard line:** server/cache state → the **`data_fetching`** layer; client-owned state →
  the client store. Crossing that line is the recurring bug, regardless of which libraries are chosen.

## What NOT to flag

- A project's **choice** of client-state library (or of Context for global config). Convention, not a
  bug — align to `state`, don't push a switch.
- **Manual memo / selector memoization** that exists to stabilize a selector or store subscription —
  that's a sanctioned use, not redundant (see react-expert for the memo stance).
- Using the `data_fetching` library's **own** pagination/optimistic/dependent-query primitives
  instead of hand-rolling — that's correct, not over-engineering.

## Currency

Maintainer docs are the source of truth (above). Baseline consulted: **TanStack Query v5** (incl. its
React Native page), **Redux Toolkit / RTK Query**, **Zustand**, **Jotai**, **SWR**. Version-specific
claims are gated to the profile's `data_fetching` / `state`; anything not verified against the
official docs is omitted rather than guessed.
