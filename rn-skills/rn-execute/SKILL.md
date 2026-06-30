---
name: rn-execute
description: "ALWAYS activate before implementing ANY feature, plan, or fix in a React Native project. The only implementer. Enforces plan-first, test-first (TDD always), verify-before-claim (runs the project's verify_command), and a mandatory code-review gate. Solo by default; --team N spawns parallel dev agents in isolated worktrees with peer review + merge. Heightened rigor for high_rigor_domains (checkout/auth/payment/PII). Reads .claude/rn-profile.md."
argument-hint: "[task | plan-path | TICKET-ID] [--fast | --auto | --no-test | --team N | --solo]"
---

# Execute — React Native implementation

End-to-end implementation skill (**the only implementer** in the suite). Enforces plan-first
execution, follows the project's architecture, runs `verify_command` before claiming done.
Solo by default; scales to a parallel dev team in isolated worktrees with `--team N`.

**YAGNI + KISS + DRY.** Token efficiency. Concise reports.

## Step 0 — Load profile

Read `.claude/rn-profile.md`: `architecture`, `state`, `navigation`, `data_fetching`,
`networking`, `styling`, `i18n`, `forms`, `feature_flags`, `crash_reporting`, `language`,
`react_compiler`, `verify_command`, `high_rigor_domains`, `generated_paths`, `plans_dir`,
`rules_file`, ticket fields. If missing → run `rn-project-init`.

## When to Use
Before writing **any** implementation code (feature, fix, refactor); translating an
approved plan into code; resolving a ticket end-to-end.

## When NOT to Use
| Case | Use instead |
|---|---|
| Need trade-off discussion | `rn-brainstorm` |
| Need a written plan first | `rn-plan` |
| Just find files | `rn-scout` |
| Library docs | `WebFetch` against the official URL |
| Trivial fix (< 20 lines, single file) | Direct edit |

---

## HARD-GATE: Plan-First
```
Do NOT write implementation code until a plan exists and has been reviewed.
Exceptions:
- --fast — skips research but still does scout → micro-plan → code
- User explicitly says "just code it" / "skip planning" — respect the override
"Simple" tasks are where unexamined assumptions waste the most time.
```

| Tempting | Reality |
|---|---|
| "Too simple to plan" | 30-sec plan prevents 30-min rework |
| "I know how to do this" | Knowing ≠ planning. Write the file paths down. |
| "I'll plan as I go" | That's hoping, not planning |

## Argument Parsing
- **Plan path** (`{plans_dir}/.../plan.md`/`phase-*.md`) → execute plan mode.
- **TICKET-ID** → `ticket_fetch` → matching plan, else `rn-plan TICKET-ID`.
- **Free-form** → interactive.
- **Flags:** `--fast` (scout→micro-plan→code), `--auto` (auto-approve gates, sparingly),
  `--no-test` (skip the final verify *run* only — you'll run it yourself; TDD still drives the code; record in report),
  `--team N` (parallel dev team in worktrees — see Phase 3), `--solo` (force single-agent),
  `--devs N` (alias for `--team N`).

If no args, ask via `AskUserQuestion` (what to implement + mode).

---

## Process Flow
```
1. Parse args
2. Resolve plan (load OR rn-plan creates OR --fast micro-plan)
3. Plan Review Gate — user approval (skip only with --auto)
4. Domain rigor flag — touches high_rigor_domains? → HIGH-RIGOR
5. Implementation — test-first (TDD); solo (direct edits / one subagent) OR --team N (worktree dev team)
6. Verification — {verify_command} (MANDATORY)
7. Code Review — rn-code-review --pending (MANDATORY)
8. Finalise — update plan status, ask before commit
```

## Phase 1 — Load context
Resolve the plan (plan path → Read; ticket → fetch + find/create plan; free-form → trivial
inline plan or `rn-plan`). Load `rules_file` + relevant `docs_root`.

**Domain rigor flag:** if the task touches any `high_rigor_domains` → mark **HIGH-RIGOR**:
- adversarial review **mandatory** at the review gate
- a precise decimal/integer-cents representation for monetary values, never JS floating-point `number`
- feature flag default-off + gradual rollout (if `feature_flags` != none)
- no PII in logs / analytics / `crash_reporting` breadcrumbs

## Phase 2 — Plan Review Gate
User approval required (skipped only with `--auto`). Present: files to change (path:line),
phases, test strategy (accessibility labels / test IDs?), feature flag, HIGH-RIGOR flag.
`AskUserQuestion`: Approved → proceed · Revise → regenerate · Abort → stop.
**Do NOT start implementing without approval.**

## Phase 3 — Implementation

**Pick the execution path by plan size** (its complexity / phase count):

| Plan size | Path | How |
|---|---|---|
| trivial / small | **direct** | Edit/Write yourself |
| 1 feature, 5+ files | **delegated** | one `general-purpose` Agent with the plan + `rules_file` |
| large / parallelizable, or `--team N` | **team** | N dev agents in isolated worktrees + peer review + merge → **`references/team-execution.md`** |

**Auto dev count** (when none of `--team N` / `--devs N` / `--solo` is given): read the plan's
`complexity` field (set by rn-plan) → **LOW = solo · MEDIUM = 2 · HIGH = 3**. `--solo` forces 1; an explicit
`--team N` / `--devs N` always wins. For team runs, **load and follow
`references/team-execution.md`** (spawn → context → build → peer review → merge → validate);
it reuses the same profile + discipline below. The discipline applies to **every** path —
your own edits *and* the dev agents' work.

**Apply specialists (on by default).** When the code touches a domain with an installed
specialist skill — React/JSX → `react-expert`, state/render correctness → `rn-state-expert`,
navigation → `rn-navigation-expert`, types → `typescript-expert`, tests → `rn-testing-expert` —
**load its `SKILL.md` and follow it** as you write. No profile entry needed; the profile's
`specialists:` can restrict the set or `none` to disable. (For `--team`, each dev does this for its slice.)

**Test-First (TDD) — always.** **Load and follow `rn-tdd`** (the TDD-discipline skill — the canonical
home for the cycle, the prove-it pattern, and vertical slicing). Drive every unit of behavior with a
Jest + React Native Testing Library test:
**RED** (write a failing test) → **GREEN** (minimum code to make it pass) → **REFACTOR** (tidy,
tests stay green). **Run the suite at each step** — actually execute it (via `verify_command` or a
targeted `jest <path>` / `-t <name>` run) and read the output: see RED fail for the expected reason, then see GREEN
pass. "Should pass" is not done. Never write implementation before a test that fails without it. This holds in
**solo and team** mode, **greenfield and existing** code (greenfield: the first test also
establishes the test pattern). The only exemptions are pure non-logic changes — copy, assets,
config, formatting — and you must say so in the report. If a change is hard to test, that's a
design signal: fix the seam (inject the dependency, split the component/hook), don't skip the test.

**Discipline (enforce `rules_file` + profile conventions while editing):**
- **State** — use `{state}`. No ad-hoc one-off local state when a shared store/context exists.
- **Data fetching** — use `{data_fetching}` (e.g. TanStack Query) for server state; don't hand-roll
  fetch-in-effect when the project has a data layer.
- **Strings** — if `i18n` != none, every user-facing string via the i18n mechanism.
  Never hardcode. Never edit generated translation files.
- **Navigation** — follow `{navigation}` (e.g. navigate via the navigator/router, not by mutating
  global state; screens receive params, don't reach into the nav tree).
- **Components/hooks** — extract logic into testable hooks; keep components thin. Follow `{styling}`.
- **Effects** — correct dependency arrays; never suppress `exhaustive-deps` (restructure instead).
  If `react_compiler` is `enabled`, treat `useMemo`/`useCallback`/`React.memo` as performance escape
  hatches, not correctness tools.
- **Accessibility** — every UI surface used in tests gets an `accessibilityLabel` / `testID`.
- **Types** — honor `{language}` (`ts-strict` → no `any`, no non-null `!` to dodge the checker).
- **Generated code** — never edit anything under `generated_paths` (incl. `ios/`, `android/`,
  `.expo/` under Expo CNG).
- **Comments** — WHY-only. No history, no "added for ticket X", no paraphrasing the code.

**Commit cadence — one concern per commit, via `/commit`** (which extracts the ticket id
from the branch). Commit at each milestone (phase done, self-contained refactor, new
components/hooks, new store/context wiring, new translation keys, new network operation, tests,
snapshot updates). Run relevant tests before each commit — never commit a broken state mid-feature.
Always use `/commit`, not raw `git commit`.

**Tracking:** multi-phase work → `TaskCreate`/`TaskUpdate`; mark `in_progress` before
starting, `completed` when done, don't batch. Update plan checkboxes inline with commit shas.

## Phase 4 — Verification Gate (MANDATORY)
**Iron law: no completion claim without fresh verification.**

Run the profile's command:
```
{verify_command}
```
(typical RN stack: `tsc --noEmit && eslint . && jest` — typecheck + lint + unit.)
Read the actual output. Claim done only on success.

- **`verify_command` unset/empty** → run a **typecheck-only** check (e.g. `tsc --noEmit`)
  and state explicitly in the report: "verify_command not configured — typecheck-only, tests
  not run." Don't pretend tests passed.
- **UI work** → also exercise the change on a device/simulator (`/run` or `/verify` skill) —
  typecheck verifies code, not feature correctness.
- User said "I'll test myself" → respect, but **say so explicitly** and skip running it.

**Red flags — stop if you think** "should pass" / "probably fine" / "seems to work" → run
it, read the output, *then* claim.

## Phase 5 — Code Review Gate (MANDATORY)
Run `rn-code-review --pending`. **Team mode:** the work is committed on `{SLUG}/integrate`, so
review the merged diff instead — `rn-code-review` on `git diff {BASE}...{SLUG}/integrate`
(`--pending` can't see committed merges). For HIGH-RIGOR tasks, adversarial review is mandatory
(the review skill runs it automatically). Resolve all **Critical** before proceeding;
**Important** → fix or document deferral with a ticket.

## Phase 6 — Finalise (only after 4 + 5 pass)
- Update `plan.md` frontmatter `status: completed`; tick remaining checkboxes.
- Update docs if implementation changed architecture/conventions (light edits OK; rewrites
  need approval).
- **Plan lifecycle:** the plan stays on the branch during dev. If your PR flow removes it
  before opening the PR, that's the PR step's job — not here. Mention `rn-plan archive`
  if worth keeping as a decision record.
- **Final commit:** most work is already committed via granular `/commit`. If the tree is
  dirty (checkbox/status/doc tweaks), `/commit` once more. **Do NOT push** without explicit
  instruction.

Final report (solo; team runs add peer-review/merge lines per `references/team-execution.md`):
```
✓ Execute complete: <task slug>
- Plan: {plans_dir}/{slug}/plan.md → status: completed
- Files changed: N (paths…)
- Tests: <n> added, all passing   (or "typecheck-only — verify_command unset")
- Verify: ✅ PASSED / typecheck-only / user-tested
- Review: passed (X critical resolved, Y important deferred)
- Commit: <sha or pending user action>
- Open follow-ups: <list or none>
```

## Greenfield note
First feature establishes the pattern — there's no prior art to match. Be deliberate about
the conventions you set (they're what every later run inherits). Once a reference feature
and a real `verify_command` exist, full rigor resumes.

## Constraints
- **DO NOT** start coding without an approved plan. **DO NOT** auto-commit/push without approval.
- **DO NOT** write implementation before a failing test (TDD) — except pure non-logic changes (state which).
- **DO NOT** skip the verify gate (Phase 4) or the review gate (Phase 5).
- **DO NOT** edit `generated_paths`. **MUST** follow `rules_file`.
- **Correctness for money** — precise decimal/integer-cents only in `high_rigor_domains`, never float. **No PII in logs.**
- **Sacrifice grammar for concision** in reports.
