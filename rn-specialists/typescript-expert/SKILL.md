---
name: typescript-expert
description: "Expert TypeScript guidance for React Native — strict mode, discriminated-union state modeling, type-safe navigation params and API boundaries, generics, no-any discipline. Loaded by rn-execute (apply while writing) and rn-code-review (critique) when work changes types/interfaces/generics or tsconfig. Reads .claude/rn-profile.md (language)."
---

# TypeScript Expert

How it's used: **rn-execute** loads this and *applies* it while writing or changing TypeScript
(types, interfaces, generics, tsconfig); **rn-code-review** uses it as the TypeScript *critique* lens.

**Source of truth:** the **official docs are the primary authority** — **reactnative.dev/docs/typescript**
and **docs.expo.dev/guides/typescript** for the RN/Expo setup, the **TypeScript Handbook**
(typescriptlang.org) for the language itself (see `SOURCES.yaml`). They win on any conflict. Community
sources fill in structure only; genuinely contested calls are flagged, not silently picked.

**Profile-aware:** read `.claude/rn-profile.md` FIRST. The **`language`** field sets the rigor bar:
- **`ts-strict`** — full strictness. Every rule below is a hard expectation; flag violations as defects.
- **`ts`** — TypeScript without enforced strict. Apply the rules as strong recommendations and
  *encourage* turning `strict: true` on; raise no-any / discriminated-union points as improvements.
- **`js`** — JavaScript. Keep critique minimal; the one high-value move is to *suggest gradual TS
  adoption* (rename a leaf module to `.ts`, add `tsconfig` with `allowJs`/`checkJs`) — don't litter a
  JS codebase with type demands it hasn't opted into.

Where the profile is silent, prefer the official-docs default — which is **TypeScript, strict**:
TS is the default for new React Native and Expo projects (reactnative.dev/docs/typescript;
docs.expo.dev/guides/typescript).

---

## 1. Strictness & no-any discipline

- **tsconfig at the project root extends the canonical base**: `@react-native/typescript-config` for
  bare RN, `expo/tsconfig.base` for Expo (docs confirm both). Don't reinvent compiler options the base
  already sets — extend it, then add `"strict": true` if the base doesn't (Expo's guide explicitly
  recommends `"strict": true`). Generated files (`expo-env.d.ts`, `*.d.ts` under generated paths) are
  off-limits — never hand-edit.
- **Prefer `strict: true`.** It turns on `noImplicitAny`, `strictNullChecks`, and the rest as a bundle.
  Under `ts-strict`, treat implicit `any` as a defect, not a warning.
- **`unknown` over `any`** for values of unproven shape (API responses, `JSON.parse`, caught errors in
  `catch`). `unknown` forces a narrowing check before use; `any` silently disables the type checker and
  leaks. A lone `any` should be justified in a comment or removed.
- **Non-null assertions (`!`) used to *dodge* the type checker are a smell** — `data!.user!.name` hides
  a real "this can be null" path. Narrow it (guard, optional chaining `?.`, nullish coalescing `??`)
  instead. `!` is acceptable only where invariants the compiler can't see genuinely hold (and say why).
- **Unsafe `as` casts** (`response as User`) assert a shape the compiler never verified — they're a
  runtime-crash waiting to happen. Prefer real narrowing or schema validation (§4). `as const` (literal
  inference) and `as unknown as T` double-casts are different beasts — the latter is a louder smell.
- Verification for a TS change is `tsc --noEmit` (type-check, no build) plus the project's `lint` — read
  these from `verify_command`. A change isn't "done" until `tsc --noEmit` is clean.

## 2. Discriminated-union state modeling (the high-value RN pattern)

- **Model async UI state as a tagged union, not booleans + optionals.** The recurring RN mistake is
  `{ isLoading: boolean; data?: T; error?: Error }` — which admits impossible states
  (`isLoading: true` *and* `data` set, or `error` *and* `data` both present). Replace with a
  **discriminated union** keyed on a literal `status`:

  ```ts
  type RemoteData<T> =
    | { status: 'idle' }
    | { status: 'loading' }
    | { status: 'success'; data: T }
    | { status: 'error'; error: Error };
  ```

  Now `data` only *exists* in the `success` branch and `error` only in `error` — the impossible states
  are **unrepresentable**, and the compiler narrows `data`/`error` for you after a `status` check. This
  is the TS Handbook's "discriminated union": a common property (the *discriminant*) with literal types.
- **Exhaustive `switch` with a `never` default.** Handle every variant and assert exhaustiveness so
  adding a new variant becomes a compile error, not a silent fall-through:

  ```ts
  switch (state.status) {
    case 'idle':    return <Idle />;
    case 'loading': return <Spinner />;
    case 'success': return <View data={state.data} />;
    case 'error':   return <ErrorView error={state.error} />;
    default: {
      const _exhaustive: never = state;   // compile error if a case is missing
      return _exhaustive;
    }
  }
  ```

  (TS Handbook: `never` is assignable to nothing, so an unhandled variant fails to assign.)
- Apply the same shape to reducer actions, navigation events, and any "one of N kinds" domain value —
  not just loading state. Flag boolean-soup state objects in review.

## 3. Type-safe navigation params

- **No untyped route params.** Route params are an external boundary — typos and missing keys must be
  compile errors, not runtime `undefined`s. Check the profile's `navigation` field:
- **`react-navigation`** — define a **`ParamList`** type mapping each route to its params, and pass it
  to the navigator (`createNativeStackNavigator<RootStackParamList>()`). Type screens via
  `NativeStackScreenProps<RootStackParamList, 'Route'>` (or `route`/`navigation` generics) so
  `route.params` and `navigation.navigate('X', params)` are both checked. Register a global
  `RootParamList` declaration so the untyped `useNavigation()` is typed app-wide.
- **`expo-router`** — enable **typed routes** (`experiments.typedRoutes: true` in app config) so `Href`
  / `<Link href>` are statically validated and bad paths fail to compile. Type params with the
  **`useLocalSearchParams<...>()`** generic (first generic = file-system route, second = query params)
  rather than reading untyped `params`. Don't hand-roll param types that the generated `expo-env.d.ts`
  already provides.
- Either way: params must be serializable — don't smuggle functions or class instances through route
  params just because `any` lets you.

## 4. Type-safe API & data boundaries

- **Validate/parse external data at the boundary — don't assert it.** Network responses,
  `AsyncStorage`, deep-link payloads, and push data arrive as `unknown` truth. Casting
  `(await res.json()) as User` is a *lie to the compiler*; the data is whatever the server actually
  sent. **Parse it** with a runtime schema (zod, io-ts, valibot, …) so the validated value's *static*
  type is *derived from* the schema (`z.infer<typeof UserSchema>`) and the two can't drift.
- Type query/mutation results: with `data_fetching: tanstack-query`, let `useQuery`/`useMutation` carry
  the parsed type through the generics; don't re-`any` the result downstream.
- Caught errors are `unknown` (under strict) — narrow before use (`instanceof Error`, a schema, a type
  guard) rather than `error.message` on an `any`.
- Prefer precise modeling over loose escape hatches: literal unions for enumerable string fields,
  `readonly`/`Readonly<T>` for data you don't mutate, branded types only where a real ID-mixup risk
  justifies the ceremony.

## 5. Generics & component / hook typing

- **Props as a named `type`/`interface`**; destructure with defaults. Type children as `ReactNode`.
- **`React.FC` caveat** — it's optional and falling out of favor: it historically forced an implicit
  `children`, complicates generic components, and hides the return type. Prefer typing props directly
  (`function Card({ title }: CardProps)`); don't *add* `React.FC` to existing code just to "be safe".
- **Generic components/hooks** — when a component or hook is shape-polymorphic (a `List<T>`, a
  `useFetch<T>`), make `T` a real generic threaded from input to output, not `any`. A custom hook's
  **return type should be explicit and stable** (an `as const` tuple or a named return type) so callers
  get precise inference and the contract is reviewable.
- **Event and style typing** — type RN handlers with the real event types
  (`GestureResponderEvent`, `NativeSyntheticEvent<TextInputChangeEventData>`, etc.), not `any`. Type
  styles via the RN style types (`StyleProp<ViewStyle>`, `TextStyle`, `ImageStyle`) — these catch
  typo'd / invalid style keys at compile time. (Per the project's `styling`: NativeWind/Tamagui/etc.
  bring their own typed prop surface — defer to it rather than hand-rolling.)
- Lean on **utility types** (`Partial`, `Pick`, `Omit`, `Record`, `ReturnType`, `Parameters`) to derive
  types from a single source instead of duplicating shapes that then drift.

## Contested / judgment calls

- **`type` vs `interface`** — both are fine; the Handbook's lean is `interface` for object/extension
  shapes, `type` for unions/tuples/mapped types. Match the file's existing convention; don't churn a
  codebase to flip between them.
- **Schema-validation library** — zod is the common default, but io-ts / valibot / typebox are
  legitimate. The *principle* (parse, don't assert) is universal; the library is the project's call.
- **Strictness ratchet under `language: ts`** — pushing toward `strict: true` is the right direction,
  but flag it as an improvement with a migration path, not as a blocking defect, when the project hasn't
  opted in.

## Currency

The official docs are the source of truth (above). TypeScript is the default for new React Native and
Expo projects; `strict` is the canonical target (Expo's guide recommends it explicitly, the bare-RN and
Expo tsconfig bases are the documented extension points). Expo Router typed routes and the
`useLocalSearchParams` generics are current Expo Router API. No version-specific claims beyond what the
linked docs state are included until verified there.
