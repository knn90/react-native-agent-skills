---
name: rn-code-review
description: "Adversarial, multi-lens code review for a React Native project (TypeScript / React / React Native). Resolves a precise scope, checks spec compliance, runs a general + architecture + simplification pass, routes change-typed slices to installed specialist skills (react-expert / rn-state-expert / rn-navigation-expert / rn-testing-expert / typescript-expert / rn-solid-expert), then an adversarial red-team, and synthesizes high-confidence findings. Reads .claude/rn-profile.md. Use for PR / commit / pending-diff review."
argument-hint: "[#PR | COMMIT | --pending | codebase]"
model: best
effort: xhigh
---

# Code Review — React Native

Adversarial, profile-driven review for React Native (TypeScript / React / React Native).
**Precision over recall** — a focused list of real, high-confidence findings beats a wall of nits.

**Principles:** YAGNI + KISS + DRY. Technical correctness over social comfort. **Honest, brutal, concise.**

> **Generated/maintained by `rn-skill-consolidate`** — distilled from the repo's prior reviewer +
> community review skills (review-swarm, review-and-simplify-changes patterns), with
> **react.dev + reactnative.dev + docs.expo.dev + the project's `rules_file` as the source of
> truth**. Last consolidated **2026-06-30**. See `SOURCES.yaml`.

## Step 0 — Load profile
Read `.claude/rn-profile.md`: `architecture`, `language`, `state`, `data_fetching`, `navigation`,
`networking`, `styling`, `i18n`, `react_compiler`, `crash_reporting`, `verify_command`,
`high_rigor_domains`, `generated_paths`, **`specialists`**, `rules_file`, `test_roots`. Missing → run `rn-project-init`.

---

## Phase 0 — Resolve scope (do this FIRST; everything keys off it)

Build ONE canonical `scope` and **print a banner before any finding** — if the scope is wrong, every
finding is suspect.

| Input | Mode | Diff |
|---|---|---|
| `#123` / PR URL | PR | `gh pr diff 123` (base via `gh pr view --json baseRefName`) |
| `abc1234` (7+ hex) | Commit | `git show <sha>` |
| `--pending` | Pending | `git diff` + `git diff --cached` |
| *(no args, recent edits)* | Default | files edited this session |
| `codebase` | Codebase | full scan |

- **Base-branch fallback:** `gh pr view --json baseRefName` → `git rev-parse origin/HEAD` → `origin/main`|`origin/master`.
- **Buckets:** `modified` (review in full, all severities) · `tests-for-modified` (coverage → main report) · `related/context` (read for context only) · `deleted` (spec reasoning only).
- **Exclude always:** `generated_paths`, `node_modules/`, `ios/`+`android/` (generated under Expo CNG), `.expo/`, `dist/`/`.metro-cache/`, `*.gen.ts`/generated GraphQL/codegen output, lockfiles unless the change IS the dep bump.
- **Adjacent quarantine:** issues in context-only files go in an "Adjacent observations" section and **do NOT count** toward the verdict.
- **Banner:** `Scope: PR #123 · base: main · modified: 7 · tests: 2 · HIGH-RIGOR: yes`.
- **Adaptive depth:** tiny diff (≤2 files, ≤30 lines, not HIGH-RIGOR) → review locally, skip the parallel fan-out.

**HIGH-RIGOR:** diff touches any domain listed in the profile's `high_rigor_domains` → adversarial pass + correctness audit are **mandatory**; log `[HIGH-RIGOR]`. (The project defines its own domains via the profile; the skill assumes none.)

---

## Stage 1 — Spec compliance (skip if no plan/spec)
Read the plan (`{plans_dir}/{slug}/plan.md`) and/or the PR/issue (`gh pr view --json title,body,closingIssuesReferences`). Emit a **requirement-coverage table**:

| Requirement / AC | Status | Where |
|---|---|---|
| <criterion> | ✅ met / ⚠️ partial / ❌ missing | `file:line` |

Flag **scope creep** (unrelated refactors bundled in) and **missing work** (`TODO`, `throw new Error("unimplemented")`, empty bodies, stubbed mocks / `jest.fn()` left in prod). No spec available → mark "not assessed" (don't invent one). **Stage 1 FAIL → stop, report, ask.**

---

## Stage 2 — Multi-lens review (run lenses in parallel for non-trivial diffs)

### 2.0 — Specialist routing (FIRST — this is the point)
For each domain the diff touches, route it to the matching specialist below **if that specialist skill
is installed** in this project — **on by default, no profile entry needed**. Spawn a **read-only
`general-purpose` Agent** that loads the specialist's `SKILL.md` and reviews **only its slice** of the
diff; the general lens (2.1) then skips that domain. The profile's `specialists:` is an **optional
override**: if set, restrict routing to that list; `specialists: none` turns routing off.

| Diff signal | Route to (if installed) | Reviews |
|---|---|---|
| `useState`/`useEffect`/`useMemo`/`useCallback`, `.map(` rendering in JSX, `FlatList`/`FlashList`/`SectionList`, `React.memo`, component files | **react-expert** | render correctness, effect deps & stale closures, list keys/perf, memoization stance |
| `useQuery`/`useMutation`/`queryClient`, `createSlice`/`configureStore`, `create((set)=>`, `atom(`, `useStore`, server-state/cache | **rn-state-expert** | server-state & cache, store shape, selectors, transition holes, stale data |
| `createStackNavigator`/`createNativeStackNavigator`, `<Stack.Screen`, `app/` route files, `useRouter`/`Link`/`useLocalSearchParams`, deep-link/`linking` config | **rn-navigation-expert** | param typing, route structure, deep-link handling, back-stack correctness |
| `*.test.tsx`/`*.test.ts`, `render(`, `screen.`, `waitFor`, `act(`, `jest.mock`, files under `test_roots`, `e2e/`, `*.maestro.yaml` | **rn-testing-expert** | RNTL / Jest idioms, async testing, coverage, flakiness, e2e (Detox/Maestro) |
| new/changed `interface`/`type`, generics, `as ` casts, discriminated unions, `tsconfig` strictness | **typescript-expert** | type soundness, unsafe casts, generics, exhaustiveness, strict-mode gaps |
| new/changed types/services/hooks/DI/composition wiring; large modules; cross-layer changes | **rn-solid-expert** | SOLID (SRP/OCP/LSP/ISP/DIP), decoupling (Decorator/Composite/Adapter), composition-root DI, framework isolation |
| *(future specialists, e.g. networking/persistence)* | **rn-`<domain>`-expert** | its domain |

Specialist agent prompt:
```
Load .claude/skills/<specialist>/SKILL.md and apply it. Review ONLY the <domain> aspects of this diff:
<the relevant hunks>. Skip what a general reviewer covers (naming, layering, i18n).
Output each finding as { severity, file:line, problem, fix, confidence: high|med|low }. Be specific; cite the rule.
```
No specialists installed / none match → skip 2.0 (the general lens still covers the floor).

### 2.1 — General RN lens (always; the floor when a specialist isn't installed)
Spawn a `general-purpose` Agent (fill `{…}` from profile). **Authoritative: `rules_file` + react.dev / reactnative.dev / docs.expo.dev** — read them.

```
Review this React Native (TypeScript / React) diff with senior rigor. Flag { severity, file:line, problem, fix, confidence }.

RENDER & EFFECTS (highest-value bugs — unless react-expert handled it):
- stale closures: effects/callbacks closing over stale state/props; missing or wrong effect dependencies (exhaustive-deps).
  NEVER suppress exhaustive-deps — restructure (move into the effect, use a ref, or a functional updater) instead.
- re-render correctness & key prop misuse in lists: index-as-key on reorderable/dynamic lists, unstable keys, missing keys.
- MEMOIZATION IS PERFORMANCE-ONLY, NEVER CORRECTNESS — do NOT flag a missing useMemo/useCallback/React.memo as a bug,
  and do NOT flag blanket memoization as an error (react.dev: "no significant harm"). When react_compiler: enabled,
  manual memo is an escape hatch, not the norm — never demand it.
- async state races / missing cleanup & cancellation (effect returns no teardown; setState after unmount); stale server-state.
ACCESSIBILITY: accessibilityLabel / accessibilityRole / accessibilityState, hit-target size, screen-reader (VoiceOver/TalkBack)
  order (unless react-expert handled it).
PLATFORM DIVERGENCE: Platform.OS branches, `.ios.`/`.android.` files — both paths handled, no one-platform-only regressions.
LIST PERFORMANCE: keys, getItemLayout, heavy renderItem / inline closures in the row (unless react-expert handled it).
MEMORY & LIFETIME: leaked listeners/subscriptions/timers, unbounded caches, AbortController never aborted, animation handles not cancelled.
SECURITY & PRIVACY: token leakage; secure storage (Keychain/Keystore via expo-secure-store / react-native-keychain, not AsyncStorage)
  for credentials; PII in logs/analytics/{crash_reporting}/console.
MONEY (if a money/finance domain is configured in high_rigor_domains): integer minor units or a decimal lib, not float; rounding;
  sign errors; currency parsing.
HYGIENE: comments WHY-only; no dead/commented code; no back-compat shims for code this diff removed.

PROFILE-GATED (run only those that apply):
- state != none → uses {state}; transition holes (loading→loaded/empty/error) covered.
- data_fetching != none → server-state via {data_fetching}; cache keys/invalidation correct; no fetch-in-effect that {data_fetching} should own.
- navigation coordinator → nav actions flow from the navigator/router; screens don't reach into unrelated stacks.
- networking != none → DTO→domain mapping at the boundary; UI doesn't import generated/transport types.
- i18n != none → no hardcoded user-facing strings; new keys added; generated catalogs untouched.
- styling != none → uses {styling}; no inline style objects rebuilt every render on hot paths where it matters.
- Always: never edit generated_paths.
```

### 2.2 — Architecture lens (profile-gated)
Check against the project's **free-form `architecture`**: **dependency direction** (UI → hooks/containers → use-case/repo; models independent), **single source of truth** (no `useState` mirroring store/server state), **business logic out of components** (extract to hooks/use-cases; no God component / massive reducer), **presentation isolation** (no view/JSX imports in domain logic; navigation as typed params/values), **stale-async overwrite / missing cancellation**. Align to the *existing* pattern — **don't propose an architecture switch for a small change**. (When `rn-solid-expert` is installed, 2.0 already covered the SOLID/decoupling depth; this lens then focuses on pattern-fit + the project's `architecture`, not re-deriving the principles.)

### 2.3 — Reuse & simplification lens
Flag both directions: **over-engineering** (duplicated logic that an existing hook/util covers; redundant/derived state stored instead of computed during render; prop sprawl; needless abstraction/indirection; stringly-typed where a union/enum exists) **and over-simplification** (distinct concerns collapsed into one unclear unit). Only findings that materially improve maintainability/correctness/cost — never churn for style. (If the repo has a `simplify` skill, this lens can defer the *applying* to it; here it only flags.)

---

## Stage 3 — Adversarial red-team (always-on except trivial; HIGH-RIGOR never skips)
Try to **break** the change across four lenses (parallel agents for large diffs). Define regression relative to the **stated intent** (what should change vs what must stay unchanged):

1. **Intent & regression** — behavior outside stated scope; broken edge/fallback paths; contract drift between callers & callees; adjacent flows that should have changed together but didn't.
2. **Security & privacy** — authn/authz gaps, unsafe input, injection, secret/token exposure, risky defaults, trust of unverified data (incl. deep-link params), PII leakage.
3. **Performance & reliability** — duplicate work / redundant requests; new work on hot paths (startup/render/list scroll); leaks, retry storms, subscription drift; ordering/race/cancellation brittleness; heavy work on the JS thread.
4. **Contracts & coverage** — API/schema/type/flag mismatches; migration & back-compat fallout; missing/weak tests for changed behavior; **detectability** (would a future regression even be observable — logs/metrics/assertions?).

Each finding: `{ severity, file:line, problem, exploit/scenario, fix, confidence }`.

---

## Synthesis (precision over recall)
Treat all lens output as raw input, then:
- **Dedup** across lenses; **drop** weak/speculative/style-only items and anything that conflicts with the stated intent.
- "May be wrong but intent unclear" → a **question**, not a finding.
- **Normalize:** `{ file:line, category (spec|render|state|navigation|testing|typescript|architecture|security|perf|contracts|simplification|hygiene), severity, why it matters, fix, confidence }`.
- **Order:** high-severity high-confidence first. If nothing material, **say so** — don't manufacture feedback.

## Agent-loop feedback (self-improving rules)
Group findings by rule. A rule firing **≥2×** across the diff = a recurring pattern → propose a one-line **directive** for the project's `rules_file` (e.g. "Never suppress exhaustive-deps — restructure the effect instead"). Especially useful for AI-generated diffs — it turns repeated misses into standing rules. (Propose only; don't edit `rules_file` without approval.)

## Verification gate
**Iron law: no "review passed" without fresh verification.** Tests/build → run `{verify_command}` (unset → typecheck/build-only, say so). Bug-fix → the original reproducer no longer reproduces. Stop if you catch yourself thinking "should pass" — run it.

## Severity model
- **Critical** — crash · data race / async overwrite of correct state · state corruption · float precision loss in money · credential leak · PII in logs → **must fix before merge**.
- **Important** — state-transition hole · layer/dependency violation · memory/listener leak · missing exhaustive-deps that causes a stale-closure bug · missing test for changed behavior · missing accessibility props on tested UI · missing i18n key → **fix before merge or file a ticket**.
- **Nit** — style, naming, API cleanliness → author's call.

## Report format
```markdown
Scope: <banner>
# Code Review — <mode>   HIGH-RIGOR: yes/no   Verdict: APPROVED / CHANGES REQUESTED / BLOCKED

## Summary   Critical: N · Important: N · Nit: N
## Stage 1 — Spec       <coverage table> · scope-creep: <…>
## Findings (by severity)   [Sev] category — file:line — problem → fix (confidence)
   - Specialist findings grouped under their sub-heading (react-expert / rn-state-expert / …)
## Agent-loop feedback   <recurring pattern → proposed rules_file directive>  (or none)
## Adjacent observations (out of scope, not counted)
## Critical open items · Recommended follow-ups
```

## Constraints
- **Read-only review** — find and recommend; do NOT apply fixes (that's `rn-execute` / a `simplify` skill).
- **DO NOT** approve while a Critical remains, or skip Stage 3 (unless trivial & non-HIGH-RIGOR).
- **DO NOT** flag a missing memo as a bug, or blanket memoization as an error — memoization is performance-only.
- **DO NOT** rely on memory — read the actual diff + `rules_file`. Specialists win on their domain.
- **MUST** print the scope banner, record adjudication reasoning, and carry confidence end-to-end.
