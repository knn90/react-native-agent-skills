---
name: rn-project-init
description: "One-time bootstrap for the rn-* skill suite. Creates .claude/rn-profile.md — the single source of project-specific facts every other rn-* skill reads. Greenfield repos → prescriptive (decide conventions up front). Existing apps → descriptive (detect + record). Use before first running rn-scout/plan/execute/code-review/resolve on a project, or when the profile is missing/stale."
argument-hint: "[--greenfield | --detect]"
---

# Project Init — rn-* suite bootstrap

Produce `.claude/rn-profile.md`: the one file that makes the generic `rn-*` skills
concrete for THIS project. Run once per project (re-run to refresh a stale profile).

**Principles:** YAGNI + KISS + DRY. The profile holds facts, not opinions — keep it short.

---

## Two modes

| Repo state | Mode | Profile is… |
|---|---|---|
| Empty / no source files yet | **greenfield** (prescriptive) | a record of decisions you make up front |
| Has `package.json` with `react-native`/`expo` + source | **detect** (descriptive) | a record of reality read from the code |

Auto-detect the mode:
```
Read package.json   → look for react-native / expo in dependencies
Glob **/*.{ts,tsx,js,jsx}   (any real source beyond config/scaffold?)
```
- `package.json` with `react-native`/`expo` + real source → `--detect`.
- No `package.json`, or scaffold only → `--greenfield`.
- `$ARGUMENTS` may force `--greenfield` / `--detect`.

If a profile already exists, show it and ask (via `AskUserQuestion`): keep / refresh / overwrite.

---

## Mode A — Detect (existing app)

Read the codebase, fill the profile from evidence. **Verify every value — never guess.**

Primary evidence is `package.json` (+ its lockfile), the Expo/native config, and a few
dotfiles. Read these first, then map dependencies to fields:

| Profile field | How to detect |
|---|---|
| `flavor` | `expo` in deps → **expo**; `expo` + `expo-dev-client` → **expo-dev-client**; `react-native` without `expo` → **bare** |
| `expo_sdk` | `expo` version in `package.json` (major, e.g. `54`); no `expo` → **none** |
| `rn_version` | `react-native` version in `package.json` (e.g. `0.81`) — replaces iOS `min_ios` |
| `package_manager` | lockfile: `package-lock.json`→npm · `yarn.lock`→yarn · `pnpm-lock.yaml`→pnpm · `bun.lockb`→bun |
| `new_architecture` | `app.json`/`app.config.*` `newArchEnabled`, else `android/gradle.properties` `newArchEnabled` / iOS `Podfile.properties.json`; **RN 0.76+ defaults true** when unset |
| `react_compiler` | `babel.config.js`/`.babelrc` plugins contain `babel-plugin-react-compiler` or `react-compiler` → **enabled**; **Expo SDK 54+ enables it by default** (note this even if absent) |
| `language` | `tsconfig.json` present + `"strict": true` → **ts-strict** · `tsconfig.json` present → **ts** · absent → **js** |
| `monorepo` | `pnpm-workspace.yaml` or `workspaces` in `package.json` → **pm-workspaces** · `nx.json` → **nx** · `turbo.json` → **turborepo** · else **none** |
| `navigation` | `@react-navigation/*` → **react-navigation** · `expo-router` → **expo-router** · else **none** |
| `state` | `@reduxjs/toolkit`→**redux-toolkit** · `zustand`→**zustand** · `jotai`→**jotai** · `mobx`→**mobx** · else **context** |
| `data_fetching` | `@tanstack/react-query`→**tanstack-query** · `@reduxjs/toolkit` (RTK Query)→**rtk-query** · `swr`→**swr** · `@apollo/client`→**apollo** · else **fetch** |
| `styling` | `nativewind` · `styled-components` · `tamagui` · `@shopify/restyle`→**restyle** · `react-native-unistyles`→**unistyles** · else **stylesheet** |
| `networking` | `axios`→**axios** · `ky`→**ky** · `@apollo/client`→**apollo-graphql** · else **fetch** |
| `i18n` | `i18next`→**i18next** · `@lingui/*`→**lingui** · `expo-localization`→**expo-localization** · else **none** |
| `forms` | `react-hook-form`→**react-hook-form** · `formik`→**formik** · else **none** |
| `crash_reporting` | `@sentry/react-native`→**sentry** · `@react-native-firebase/crashlytics`→**crashlytics** · `@bugsnag/*`→**bugsnag** · else **none** |
| `feature_flags` | `statsig`→**statsig** · `launchdarkly`→**launchdarkly** · `@react-native-firebase/remote-config`→**firebase-remoteconfig** · `expo-updates` channels→**expo-updates-channels** · else **none** |
| `source_roots` | dirs with real source: `app/` (Expo Router) · `src/` · package `src/` under workspaces |
| `test_roots` | `__tests__/` · `*.test.tsx`/`*.spec.tsx` globs · `e2e/` (Detox/Maestro) |
| `generated_paths` | `ios/` + `android/` (under Expo CNG these are GENERATED via prebuild — treat as generated) · `.expo/` · `*.gen.ts` · `ios/Pods/` |
| `e2e` | `detox` in deps → **detox** · `@maestro`/`.maestro/`/`maestro` flows → **maestro** · else **none** |
| `verify_command` | propose from `package.json` `scripts` (`typecheck`/`lint`/`test`), else default `tsc --noEmit && eslint . && jest` |
| `build_tool` | `eas.json` → **eas** · `fastlane/` → **fastlane** · else **native** |
| `rules_file` | `CLAUDE.md` else `docs/architecture.md` else **none** |
| `ticket_system`/`ticket_pattern` | branch names + `git log` (e.g. `ABC-123`), or ask |
| `high_rigor_domains` | grep source for `checkout`/`payment`/`auth`/`profile`; default `[auth, PII]` if none |
| `default_base_branch` | `git symbolic-ref refs/remotes/origin/HEAD` (repo default), else `main` |
| `pr_tool` | `gh` if the GitHub CLI is installed + authed, else **none** |
| `architecture` | FREE-FORM — infer from layout: feature folders under `src/`/`app/` → "feature-folders + hooks"; co-located screens/containers → "container/presentational"; `domain/`+`data/`+`ui/` → "clean-ish (domain/data/ui)". RN has no canonical pattern — describe what you see, don't impose one. |

Use `Explore`/`Grep`/`Glob` (or invoke `rn-scout` if the repo is large). Confirm the
detected `flavor`, `architecture`, `verify_command`, and `high_rigor_domains` with the
user before writing — these drive the most downstream behaviour. Where any value is
ambiguous (e.g. mixed lockfiles, both navigators present, `newArchEnabled` unset),
confirm via `AskUserQuestion` rather than guessing.

---

## Mode B — Greenfield (new app)

Nothing to detect. **Decide** the conventions via `AskUserQuestion`, then record them.
This is the highest-leverage moment in the project's life — treat the foundational
choices (flavor, navigation, state) with real rigor (a multi-year commitment), not a
quick menu pick.

Ask (bundle into 2-4 `AskUserQuestion` groups):

1. **Flavor + platform** — `expo` (managed, native dirs via prebuild/CNG) · `expo-dev-client`
   (custom dev client + Expo tooling) · `bare` (`react-native` CLI, hand-maintained
   `ios/` + `android/`). Set `expo_sdk` (or `none`), target `rn_version`, and
   `new_architecture` (true for RN 0.76+ unless a dep forces old arch).
2. **Language** — **ts-strict** (canonical for new RN/Expo) · ts · js. Recommend ts-strict
   and push back on plain js for a multi-year app.
3. **Navigation + architecture** — `expo-router` (file-based) vs `react-navigation`;
   `architecture` is free text (feature-folders + hooks is the boring proven default —
   recommend it for small teams, push back on novelty). *(This is `rn-brainstorm` applied
   to foundations — borrow its brutal-honesty stance.)*
4. **State + data-fetching + styling** — state: zustand/redux-toolkit/jotai/context/mobx ·
   data: tanstack-query/rtk-query/swr/apollo/fetch · styling: stylesheet/nativewind/
   styled-components/tamagui/unistyles/restyle. `none`/defaults are valid early.
5. **react_compiler** — enabled (default on Expo SDK 54+) vs disabled. Note: memo becomes
   build-time + automatic when enabled.
6. **Integrations** — networking · i18n · forms · crash_reporting · feature_flags. Pick or
   `none` (skills skip the related checks when `none`).
7. **Verify command** — default `tsc --noEmit && eslint . && jest`; or `npx expo lint && jest`;
   or wire to `package.json` scripts. May be left **empty** → build-only. Also `build_tool`
   (eas/fastlane/native) and `e2e` (detox/maestro/none).
8. **Ticket system** + pattern, or `none`.
9. **HIGH-RIGOR domains** — which of checkout/payment/auth/PII/money this app will have.

Do **NOT** generate a verify wrapper script. The verify mechanism is just the
`verify_command` string — keep it as configuration, not a committed artifact.

Optionally (ask first) create the docs scaffold the other skills expect:
`{plans_dir}` and `{reports_dir}`, and a starter `rules_file` (CLAUDE.md) capturing the
always/never rules implied by the chosen stack.

---

## Output

Write `.claude/rn-profile.md` from `rn-profile.template.md`, filled in. Then:

```
✓ Profile written: .claude/rn-profile.md
- Mode: detect | greenfield
- Flavor: <expo | expo-dev-client | bare>   RN: <rn_version>   Expo SDK: <expo_sdk | none>
- New arch: <true|false>   React Compiler: <enabled|disabled>   Language: <ts-strict|ts|js>
- Stack: nav <…>   state <…>   data <…>   styling <…>
- Verify: <command or "build-only (unset)">
- HIGH-RIGOR domains: <…>
- Specialists available: <list or none>

Next: the rn-* skills are now live for this project.
- Find code            → rn-scout
- Plan a feature       → rn-plan
- Implement            → rn-execute
- Resolve end-to-end   → rn-resolve <ticket | "description">  (scout→plan→execute→review→PR)
```

## Greenfield follow-through

For the first 1-2 features there's no prior art to DRY against — that's expected. The
other skills relax their "match existing patterns" checks until ≥1 reference feature
exists, then resume normal rigor. Once `verify_command` points at a real test setup, the
verification gate goes from build-only to full.

## Constraints

- **DO NOT** create a verify script — verify is a profile string only.
- **DO NOT** guess detected values — confirm flavor/architecture/verify/rigor with the user.
- **DO NOT** invoke implementation skills — this only writes the profile (+ optional docs scaffold).
- **MUST** keep the profile minimal — facts that vary between apps, nothing else.
- **MUST** use only the fields in `rn-profile.template.md` — add nothing beyond the template.
