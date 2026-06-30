---
name: rn-navigation-expert
description: "Expert React Native navigation guidance — React Navigation (static API, typed routes, deep linking, nav-state correctness) and Expo Router (file-based routing, typed routes, linking). Loaded by rn-execute and rn-code-review when work touches navigators/routes/deep-links. Reads .claude/rn-profile.md (navigation, flavor)."
---

# RN Navigation Expert

How it's used: **rn-execute** loads this and *applies* it while writing navigators, routes, and
deep links; **rn-code-review** uses it as the navigation *critique* lens. It is loaded when a change
touches navigators, screens/routes, the `app/` directory, `linking` config, route params, or
deep-link handling.

**Source of truth:** the **maintainer docs are the primary authority** — **docs.expo.dev/router**
for Expo Router and **reactnavigation.org** for React Navigation (see `SOURCES.yaml`). They win on
any conflict. Note Expo Router is *built on* React Navigation, so the param/typing/linking concepts
below carry across both. Community sources fill in detail; contested calls are flagged, not silently
picked.

**Profile-aware:** read `.claude/rn-profile.md` FIRST. Two fields scope the verdicts below:
- **`navigation`** (`expo-router` | `react-navigation` | `none`) — selects which library section
  applies. If `none`, skip this skill.
- **`flavor`** (`expo` | `expo-dev-client` | `bare`) — Expo Router is the standard for Expo-flavored
  apps and Expo officially recommends it for new universal apps (docs.expo.dev/router/introduction);
  React Navigation works for both Expo and bare. **Match the library the project already uses — do
  not push a switch** between them; review what's there.

---

## 1. Typed routes (no `any` on params)

- **Every route's params are typed** — no `any`, no untyped `route.params`. A missing or `any` param
  type is a finding.
- **React Navigation:** the `ParamList` type alias maps route name → params (e.g.
  `type RootStackParamList = { Profile: { userId: string }; Home: undefined }`). It **must be a
  `type` alias, not an `interface`** (per reactnavigation.org/docs/typescript). `undefined` = no
  params; `| undefined` = optional params. Register the root list via **module augmentation of
  `RootParamList`** so `useNavigation`/`Link` are typed everywhere — bare `useNavigation<...>()`
  annotation is *not* statically verified and is the weaker form.
  - **Static API:** derive params with `StaticParamList<typeof RootStack>` instead of hand-writing
    the list; nested dynamic navigators wrap with `NavigatorScreenParams<NestedParamList>`. Prefer
    this when the project uses `createStaticNavigation` / `createXNavigator`.
- **Expo Router:** enable **typed routes** (statically-typed `Link href` + params) so you can only
  link to routes that exist. Read params via `useLocalSearchParams<{ id: string }>()` (or
  `useGlobalSearchParams`) with an explicit type — remember **all URL params arrive as strings**;
  coerce/validate, don't assume `number`/`boolean`.
- List → detail: pass an **ID/lightweight value** as the param, not the whole object/view — refetch
  detail from the ID. Serializable params only.

## 2. Deep linking (URL → screen mapping + validation)

- **React Navigation:** deep links go through the **`linking`** prop on the navigation container /
  `createStaticNavigation`, with **`prefixes`** (schemes + universal-link domains) and **`config`**
  mapping paths → screens. Don't hand-roll deep-link handling via a `ref` — the docs call that
  error-prone. Custom cold-start/foreground handling uses `getInitialURL` / `subscribe`, not ad-hoc
  listeners.
- **Expo Router:** deep links derive from the **file system** — a route in `app/` is automatically a
  deep link, so every screen is linkable. Native scheme/universal-link config lives in app config
  (`scheme`, associated domains); review that the path you expect maps to the file you expect.
- **Validate every param coming from a URL.** A deep link is untrusted input: parse, type-check, and
  guard (missing/garbage `id`, out-of-range, unknown enum) before using it to fetch or navigate.
  Map unknown/invalid URLs to a not-found/fallback route — never crash or silently no-op.
- Verify the **whole path resolves**, including nested navigators/segments, not just the leaf.

## 3. Nav-state correctness

- **Navigation state is the single source of truth — do not duplicate it into component state.**
  Mirroring the active route/params into `useState` (then syncing with effects) is a finding; read
  from the navigator (`route.params`, `useLocalSearchParams`, `useNavigationState`) instead.
- **Nested-navigator param leakage:** params belong to the route that declares them. Don't reach into
  a parent/sibling navigator's params or assume a nested screen sees the parent's params — pass them
  explicitly (`NavigatorScreenParams` / nested `href`). Flag screens reading params they don't own.
- **Auth-gated routes / redirects:** gate protected routes declaratively. React Navigation static API
  uses the screen **`if`** callback (render auth vs app screens conditionally); Expo Router uses a
  guard layout / `<Redirect href=...>` in the protected group. The gate must live in the navigator,
  not as an effect that pushes after the protected screen already mounted (flash + race).
- **Header / screen `options` must be pure** — derive from props/params, no side effects, no
  navigation calls, no state mutation inside the `options` function. Reset the nav stack/`path` on
  account change / logout so a stale back-stack can't leak the previous user's screens.

## 4. Lifecycle (focus / blur / back)

- **`useFocusEffect`** (not a bare `useEffect`) for work that must run when a screen **gains focus**
  and stop when it blurs — and it **must return a cleanup** that tears down subscriptions/timers, or
  they leak across focus cycles. Wrap its callback in `useCallback` to avoid re-running every render.
- Use `useIsFocused` / the `focus`/`blur` events to pause expensive work (polling, video, sensors)
  on blur; don't keep them running on background screens in the stack.
- **Back handling:** prefer the navigator's back behavior; only intercept hardware/gesture back
  (`beforeRemove` event / Android `BackHandler`) when you must confirm unsaved changes — and always
  remove the listener on cleanup. Don't block back without an escape.

## 5. Per-library note (profile `navigation` + `flavor`)

- **`navigation: expo-router`** (typical for `flavor: expo` / `expo-dev-client`): review file-based
  routes under `app/`, `Link` / `useRouter().push|replace` / `useLocalSearchParams`, typed-routes
  enabled, layout-based grouping/guards. Expo recommends it for new universal Expo apps — but if the
  project already uses React Navigation, review that, don't migrate.
- **`navigation: react-navigation`** (works for `expo` and `bare`): review navigator setup (static
  `createStaticNavigation` / `createXNavigator`, or the dynamic component API), `ParamList` typing,
  the `linking` prop, and `navigation`/`route` usage. The **static API is optional** — it simplifies
  config and makes TypeScript + deep linking easier; the dynamic API is still fully valid. Don't flag
  a correct dynamic-API navigator as wrong for not using the static API.
- Where the profile is silent or `none`, do not impose a library.

## Currency

Maintainer docs are the source of truth (above): **reactnavigation.org** (React Navigation v7+,
static API) and **docs.expo.dev/router** (Expo Router). Expo Router is built on React Navigation, so
typing/linking concepts are shared. Community references are structure/checklist only and are
reconciled against the maintainer docs.
