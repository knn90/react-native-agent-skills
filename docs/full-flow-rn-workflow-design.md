# Full-Flow React Native Workflow — Design & Implementation Plan

**Status:** v1 building (2026-06-30) — decisions locked (see §6)
**Goal:** One entry point that takes a *ticket or free-form context* and drives it end-to-end:
**scout → plan → grill → implement → verify → review → PR**, project-agnostic (profile-driven).
**Decision taken:** fork-and-translate the proven `ios-agent-skills` suite for React Native — a new
standalone repo, NOT a generalization of the iOS suite. iOS is the structural reference.

---

## 1. What we port (the iOS suite is the reference)

`ios-agent-skills` already implements the whole flow, profile-driven (`.claude/ios-profile.md`). We
re-create it for RN, reading `.claude/rn-profile.md`. The orchestration spine is ~platform-neutral and
ports nearly verbatim; the real work is the RN profile schema + the re-derived specialist set.

| Stage | iOS skill | RN skill |
|---|---|---|
| Bootstrap | `ios-project-init` | `rn-project-init` |
| Orchestrate | `ios-resolve` | `rn-resolve` |
| Scout | `ios-scout` | `rn-scout` |
| Research / ideate / reason | `ios-research`, `ios-brainstorm`, `ios-sequential-thinking` | `rn-research`, `rn-brainstorm`, `rn-sequential-thinking` |
| Plan / harden | `ios-plan`, `ios-grill` | `rn-plan`, `rn-grill` |
| Implement | `ios-execute` | `rn-execute` |
| Review | `ios-code-review` | `rn-code-review` |
| Specialists | swiftui / concurrency / testing / solid / tdd | react / state / navigation / testing / typescript / solid / tdd |
| Maintenance | `ios-skill-consolidate` | `rn-skill-consolidate` (not shipped) |

---

## 2. Research-driven design changes (vs. a naive iOS→RN translation)

Grounded in a deep-research pass (official docs primary). The points where RN genuinely diverges:

| # | Change | Evidence |
|---|---|---|
| 1 | New profile fields `react_compiler` + `new_architecture` | React Compiler v1.0 stable (2025-10-07, default Expo SDK 54); review must branch on it |
| 2 | `react-expert` rule: **memo = performance-only, never correctness**; don't flag blanket memo as error | react.dev useMemo/useCallback |
| 3 | `architecture` is **free-form**, not an enum | react.dev: "no opinions on folders" — RN has no MVVM-C/TCA canon |
| 4 | `flavor` reframed around **CNG/prebuild**, not a managed-vs-bare binary | Expo CNG: ios/android are generated, gitignored artifacts |
| 5 | `monorepo` = PM-workspaces axis (yarn/pnpm/npm/bun) + optional nx/turborepo | Expo monorepo support roots in PM workspaces |
| 6 | List-perf reviewer gates FlashList v2 on `new_architecture` | FlashList v2 is New-Architecture-only |
| 7 | Server-state checklist = RN manual wiring (onlineManager/focusManager/Platform.OS) | TanStack Query RN docs |
| 8 | `verify_command` default = `tsc --noEmit && eslint . && jest` | Jest official default runner; tsc/eslint canonical |

Source-of-truth hierarchy (standing): **react.dev > reactnative.dev > docs.expo.dev > maintainer docs > community.**

---

## 3. Design

### 3.1 Core skills — `rn-skills/` (10)
`rn-resolve` is the front door, chaining the rest with approval + verify gates (identical pipeline to
`ios-resolve`). `rn-execute` is the only implementer (plan-first, TDD-always, verify-before-claim, a
mandatory review gate; `--team N` spawns worktree dev agents + peer review + merge). `rn-code-review`
routes change-typed slices to specialists, then runs general + architecture + adversarial passes.

### 3.2 Profile schema — `rn-profile.template.md`
The single contract. RN-specific additions over the iOS profile: `flavor`, `new_architecture`,
`react_compiler`, `expo_sdk`, `rn_version` (replaces `min_ios`), `package_manager`, `monorepo`,
`language`, `navigation`, `state`, `data_fetching`, `styling`, `e2e`, `build_tool`. `architecture` is
free-form. `verify_command` is the single source of "done".

### 3.3 Specialists — `rn-specialists/` (7) — re-derived, NOT a 1:1 iOS translation

| Specialist | Confidence | Covers |
|---|---|---|
| `react-expert` | HIGH | hooks rules, exhaustive-deps, render perf, **React-Compiler-aware memo**, list perf (FlatList/FlashList), a11y |
| `rn-state-expert` | server=HIGH / client=convention | client state (Redux/Zustand/Jotai/Context) + server state (TanStack Query RN wiring) |
| `rn-navigation-expert` | MED | React Navigation (static API, typed routes, deep linking) vs Expo Router |
| `rn-testing-expert` | MED | Jest + RNTL (v14 async act), native-module mocking, Detox vs Maestro |
| `typescript-expert` | MED | strict mode, discriminated-union state, type-safe nav params + API |
| `rn-solid-expert` | port | SOLID + decoupling; framework-isolation = keep RN/nav/storage out of domain |
| `rn-tdd` | port | red→green→refactor, prove-it bug repro, vertical slices |

**Deferred (flagged, not pre-built):** `rn-perf-expert` (Hermes/JSI/bridge/startup) — folded into
react-expert for now; `rn-native-modules-expert` (TurboModules/Fabric/JSI bridging) — bare-RN only.

---

## 4. End-to-end data flow

```
TICKET-ID / context
      │
      ▼
rn-resolve ──INIT──► branch + plans_dir/{slug}/_status.md
      ├─► rn-scout ───────────► scope.md
      ├─► rn-plan ────────────► plan.md ──[APPROVAL GATE]
      ├─► rn-grill ───────────► hardened plan + decisions log
      ├─► rn-execute --team N ► worktree dev-1..N ──peer review──► integrate
      │        └─ VERIFY: verify_command (tsc + eslint + jest) ◄──┘
      ├─► rn-code-review ─────► verdict (Critical must clear; specialists route by change type)
      ├─► /commit ────────────► granular commits
      └─► gh pr create ───────► PR url ──► (best-effort ticket transition)
```

---

## 5. Implementation plan (phases)

| Phase | Deliverable | Verify |
|---|---|---|
| 1. Foundation | manifests, `rn-profile.template.md`, README, docs/ | template parses; plugin/marketplace valid |
| 2. Core spine | port scout, sequential-thinking, brainstorm, research, resolve, plan, grill | no Swift/Xcode terms; profile reads resolve |
| 3. Init + review | `rn-project-init` (detect RN stack), `rn-code-review` (routing §3.3 + general lens), `rn-execute` (verify/TDD) | routing covers each specialist; gates intact |
| 4. Specialists | author react-expert + rn-state-expert (HIGH research); source nav/testing/typescript; port solid/tdd | each cites an official primary source |
| 5. Consolidate | `rn-skill-consolidate` + per-specialist SOURCES.yaml | rebuild one specialist end-to-end |
| 6. Verify + ship | grep for iOS leftovers; README accurate; commit + push | clean tree; suite installs |

---

## 6. Decisions (locked 2026-06-30)

1. **Separate repo** `react-native-agent-skills` (fork-and-translate, not a generalization).
2. **Flavor-agnostic** — Expo (CNG/dev-client) + bare RN encoded as profile fields.
3. **Full parity** with ios-skills in v1: all 10 core + 7 specialists + consolidate.
4. **Specialist set re-derived, not mirrored** — React Compiler reshapes the components reviewer; no
   concurrency analog; navigation + typescript become first-class specialists.
5. **Source-of-truth** — react.dev > reactnative.dev > Expo/maintainer docs > community.

---

## 7. Risks

- **Two parallel spines drift** (iOS + RN). Mitigate: keep spine edits minimal; reconsider a shared
  core only if duplication actually bites.
- **Specialist staleness** — React/RN moves fast (React Compiler landed Oct 2025). `rn-skill-consolidate`
  + pinned SOURCES.yaml are the freshness guard; official docs win on conflict.
- **Community reference repos predate React Compiler** (VoltAgent lists Recoil). They're `primary:false`
  structure references only — reconcile content against official docs.
- **Open domains** (navigation, client-state libs, testing, TS) were not deep-verified in research —
  Phase 4 runs a per-specialist sourcing pass before authoring.

---

## Appendix — GitHub reference repos (community tier, pinned)

Mined per-specialist for structure; reconciled against official docs. License-checked; `primary:false`.

| Repo | ★ | License | Pinned | Role |
|---|---|---|---|---|
| VoltAgent/awesome-claude-code-subagents | 22.6k | MIT | `c193ad45419c` | RN/React/TS/test/a11y subagents |
| addyosmani/agent-skills | 68k | MIT | `aba7c4e9695c` | TDD, frontend-ui, devtools-testing |
| rohitg00/awesome-claude-code-toolkit | 2.2k | Apache-2.0 | `ebdf1d596d2c` | react/mobile/ts agents, testing cmds |
| hesreallyhim/awesome-claude-code | 47.6k | none | `29755104e1f1` | discovery anchor (cite-only) |
| ComposioHQ/awesome-claude-skills | 66k | none | `92568c1edaff` | discovery anchor (cite-only) |
