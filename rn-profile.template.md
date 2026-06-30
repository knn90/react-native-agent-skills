---
# rn-profile.md — single source of project-specific facts for the rn-* skill suite.
# Every rn-* skill reads this FIRST. Copy this file to `.claude/rn-profile.md`
# at the project root and fill it in (or let `rn-project-init` generate it).
#
# Rule of thumb: if a value differs between two React Native apps, it belongs HERE, not in a skill body.

app_name:                 # e.g. Acme
state: existing           # existing | greenfield

# ── Repo layout ────────────────────────────────────────────────
source_roots: []          # e.g. [app/, src/, packages/*/src]
test_roots: []            # e.g. [__tests__/, src/**/*.test.tsx, e2e/]
generated_paths: []       # NEVER edit — e.g. [ios/, android/, .expo/, *.gen.ts, ios/Pods/]
                          #   (under Expo CNG the native ios/ + android/ dirs are GENERATED — treat as generated)
docs_root:                # docs/ | none
plans_dir:                # docs/plans | .claude/plans
reports_dir:              # docs/reports | .claude/reports
rules_file:               # CLAUDE.md | docs/architecture.md | none  (always/never rules)

# ── Flavor & platform (RN-specific; no iOS analog) ─────────────
flavor:                   # expo | expo-dev-client | bare
                          #   expo          = managed, runs in Expo Go / dev build, native dirs generated via prebuild (CNG)
                          #   expo-dev-client = custom dev client + Expo tooling
                          #   bare          = `react-native` CLI, hand-maintained ios/ + android/
new_architecture:         # true | false   (Fabric / TurboModules. Gates FlashList v2, etc. RN 0.76+ default = true)
react_compiler:           # enabled | disabled
                          #   enabled = build-time auto-memoization (default in Expo SDK 54+). Changes the memo review stance.
rn_version:               # e.g. 0.81      (replaces iOS min_ios)
expo_sdk:                 # e.g. 54 | none (none = bare RN with no Expo)
package_manager:          # npm | yarn | pnpm | bun
monorepo:                 # pm-workspaces | nx | turborepo | none   (Expo monorepo support roots in PM workspaces)

# ── Architecture (FREE-FORM — RN has NO canonical pattern; react.dev has no opinion on folders) ──
architecture:             # free text, e.g. "feature-folders + hooks" | "container/presentational" | "clean-ish (domain/data/ui)"
language:                 # ts-strict | ts | js     (ts-strict is canonical for new RN/Expo projects)

# ── Stack ──────────────────────────────────────────────────────
navigation:               # expo-router | react-navigation | none
state:                    # redux-toolkit | zustand | jotai | context | mobx | none
data_fetching:            # tanstack-query | rtk-query | swr | apollo | fetch | none
styling:                  # stylesheet | nativewind | styled-components | tamagui | unistyles | restyle

# ── Integrations (set to `none` if unused — skills skip the related checks) ──
networking:               # axios | fetch | apollo-graphql | ky | none
i18n:                     # i18next | lingui | expo-localization | none
forms:                    # react-hook-form | formik | none
feature_flags:            # statsig | launchdarkly | expo-updates-channels | firebase-remoteconfig | none
crash_reporting:          # sentry | crashlytics | bugsnag | none
ticket_system:            # Jira | GitHub | Linear | none
ticket_pattern:           # regex, e.g. ABC-\d+   (used to detect a ticket id in args/branch)
ticket_fetch:             # MCP tool or CLI to fetch a ticket | none

# ── Workflow / PR (used by rn-resolve + rn-execute --team) ─────
default_base_branch: main # branch-from point + PR target; overridable with --base
pr_tool:                  # gh | none   (none → rn-resolve prints manual PR steps)

# ── Verification ───────────────────────────────────────────────
# Whatever this project actually runs to prove a change is correct.
# Typical RN stack: typecheck + lint + unit. Free-form.
#   tsc --noEmit && eslint . && jest
#   yarn typecheck && yarn lint && yarn test
#   npx expo lint && jest
# Empty/unset → skills fall back to BUILD-ONLY and say so explicitly in the report.
verify_command: |

build_tool:               # eas | fastlane | native   (EAS Build spans both Expo and bare RN)
e2e:                      # detox | maestro | none     (optional; slow — run on demand, not every verify)

# ── Rigor ──────────────────────────────────────────────────────
# Domains that force adversarial review + correctness audit every time.
high_rigor_domains: [checkout, payment, auth, PII, money]

# ── Specialist skills (optional add-ons) ───────────────────────
# ON BY DEFAULT: any installed rn-*-expert / react-expert / typescript-expert is auto-used —
# rn-code-review routes to it per change type, rn-execute applies it while writing.
# This field is an OPTIONAL OVERRIDE.
specialists:              # unset = auto-use all installed · [react-expert, rn-testing-expert] = restrict · none = off
---

# Project Notes (free text)

Anything a skill should know that doesn't fit a field above — naming conventions,
forbidden patterns, "always do X / never do Y" rules specific to this app.
