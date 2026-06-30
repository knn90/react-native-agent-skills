# rn-* skill suite

A **project-agnostic React Native engineering workflow** for Claude Code. Point it at any RN app (Expo
or bare), fill in **one** file (`.claude/rn-profile.md`), and you get a full **ticket → PR** pipeline
plus specialist reviewers (used automatically once installed). The skill bodies hardcode **no** app
names, paths, architectures, or commands — every project-specific fact lives in the profile and is read
at runtime.

```
rn-project-init ──→ .claude/rn-profile.md            # run ONCE per project

rn-resolve  ── one command: ticket/context → scout → plan → grill → execute → review → PR
      │
      ├─ rn-scout ─────────┐
      ├─ rn-research ───────┤
      ├─ rn-brainstorm ─────┼─→ rn-plan ─→ rn-grill ─→ rn-execute ─→ rn-code-review
      └─ rn-sequential-thinking  (reasoning aid; plugs in anywhere)
                                          │ apply           │ route
                                          └─── rn-specialists ───┘
                          (React/Hooks · State · Navigation · Testing · TypeScript — official-docs-primary)
```

## Setup (once per project)

1. **Install the skills** — recommended: install as a Claude Code **plugin**:
   ```
   /plugin marketplace add knn90/react-native-agent-skills
   /plugin install rn-skills@react-native-agent-skills
   ```
   Skills are then invocable as `rn-skills:rn-resolve`, `rn-skills:rn-scout`, … and are auto-selected by
   description. Update with `/plugin marketplace update react-native-agent-skills`; remove with
   `/plugin uninstall rn-skills@react-native-agent-skills`. (The maintenance-only `rn-skill-consolidate/`
   is intentionally not part of the plugin.)

   <details><summary>Manual install (no plugin)</summary>

   ```bash
   cp -R rn-skills/*       <project>/.claude/skills/    # the core workflow
   cp -R rn-specialists/*  <project>/.claude/skills/    # optional specialists you want
   ```
   (Or into `~/.claude/skills/`.) **Don't** ship `rn-skill-consolidate/` — it's repo-maintenance.
   </details>

   Newly installed skills appear after the Claude Code session is restarted/reloaded.
2. **Create the profile** — run **`rn-project-init`**. It *detects* an existing app's flavor (Expo vs
   bare), New Architecture / React Compiler status, navigation/state/data libraries, and paths — or sets
   conventions for a greenfield app — and writes `.claude/rn-profile.md`. (Or copy `rn-profile.template.md`
   and fill it in by hand.)
3. **Specialists work automatically** once installed — `rn-code-review` routes change-typed slices to them
   and `rn-execute` applies them while writing. The profile's `specialists:` field is an **optional
   override**:
   ```yaml
   # specialists: [react-expert, rn-testing-expert]   # restrict to just these
   # specialists: none                                 # turn specialist routing off
   ```
   Leave it unset to auto-use every specialist you installed.

## How to use it

### The whole loop, one command

```
rn-resolve ABC-123
rn-resolve "add pull-to-refresh to the orders list"
rn-resolve ABC-123 --team 2          # parallel dev team
rn-resolve ABC-123 --solo --no-pr    # single agent, stop before the PR
```

`rn-resolve` is the front door. It runs:

```
fetch ticket/context → create branch → rn-scout → rn-plan → rn-grill (harden) ──(you approve)──►
    rn-execute (TDD; solo or --team N) → rn-code-review → /commit → open PR
```

You approve the plan **before** any code is written, and nothing is pushed without you.

### Or à la carte

```
rn-scout useOrders                      # where does this live?
rn-research "offline sync options"      # sourced technical research
rn-brainstorm "offline support"         # brutal trade-off analysis → decision report
rn-plan ABC-123                         # phased plan (you approve)
rn-grill <plan-dir>                     # stress-test a plan: grill open decisions, harden it
rn-execute <plan-path> --team 2         # implement (TDD; parallel worktree team)
rn-code-review #42                      # adversarial, multi-lens review of a PR
```

## The skills

**Core workflow — `rn-skills/`**

| Skill | Role |
|---|---|
| `rn-project-init` | One-time: writes `.claude/rn-profile.md` (detect existing app / set greenfield conventions). |
| `rn-resolve` | **The front door.** ticket/context → scout → plan → grill → execute → review → PR. Orchestrates; never implements directly. |
| `rn-scout` | Fast, token-efficient parallel code discovery — returns a *map*, not analysis. |
| `rn-research` | Sourced technical research grounded in the codebase. |
| `rn-brainstorm` | Brutally honest trade-off analysis → decision report. |
| `rn-sequential-thinking` | Step-by-step reasoning aid; plugs in anywhere. |
| `rn-plan` | Phased implementation plan, with an approval gate. |
| `rn-grill` | Stress-tests the plan **before** approval: interrogates open decisions one at a time, folds answers back into `plan.md`. Auto-skips when the plan is unambiguous. |
| `rn-execute` | **The only implementer.** plan → code → verify → review. **TDD always**; solo, or `--team N` (parallel worktree devs + peer review + merge). |
| `rn-code-review` | Multi-lens adversarial review; **routes change-typed slices to the specialists**; precision-over-recall findings. |

**Specialists — `rn-specialists/` (optional add-ons; auto-used once installed)**

Deep domain reviewers, consolidated from curated sources with **React / React Native / library official docs as the source of truth**. `rn-code-review` triggers them per change type; `rn-execute` applies them while writing.

| Specialist | Domain |
|---|---|
| `react-expert` | React/RN components — hooks, render perf, React-Compiler-aware memo, list perf (FlatList/FlashList), accessibility |
| `rn-state-expert` | State — client (Redux/Zustand/Jotai/Context) + server (TanStack Query / RTK Query) |
| `rn-navigation-expert` | Navigation — React Navigation vs Expo Router, typed routes, deep linking |
| `rn-testing-expert` | Testing — Jest + React Native Testing Library; Detox / Maestro e2e |
| `typescript-expert` | TypeScript — strict mode, discriminated-union state, type-safe nav/api |
| `rn-solid-expert` | SOLID + decoupling; keep RN/navigation/storage out of the domain |
| `rn-tdd` | TDD discipline — red→green→refactor, prove-it bug repro, vertical slices |

**Maintenance — `rn-skill-consolidate/` (repo-only; not shipped)**

Rebuilds any specialist (or a core skill like `rn-code-review`) from its `SOURCES.yaml`: a staleness audit + **web discovery** for newer/better sources + fan-out extraction + synthesis. Keeps skills from rotting; official docs always win on conflict.

## The profile — the one file that matters

`.claude/rn-profile.md` is the single source of every project-specific fact: `source_roots`, `flavor`,
`new_architecture`, `react_compiler`, `architecture`, `navigation`, `state`, `data_fetching`,
`verify_command`, `high_rigor_domains`, `specialists`, `ticket_fetch`, `default_base_branch`, `pr_tool`,
… Every skill reads it first, so the skill bodies stay generic. Start from `rn-profile.template.md`.

## Layout

```
react-native-agent-skills/
├── .claude-plugin/              # plugin + marketplace manifests (the one-command install)
├── README.md
├── rn-profile.template.md       # copy to <project>/.claude/rn-profile.md and fill in
├── rn-skills/                   # core workflow (10 skills)       — shipped by the plugin
├── rn-specialists/              # optional specialist reviewers   — shipped by the plugin
├── rn-skill-consolidate/        # repo-maintenance (rebuilds skills from sources) — NOT shipped
└── docs/                        # design notes
```

## Core principles (every skill)

- **Profile-driven** — read `.claude/rn-profile.md` first; never hardcode project facts.
- **Plan-first** — no implementation before an approved plan (hard gate in `rn-execute`).
- **TDD always** — `rn-execute` writes a failing test before the code.
- **Verify before claiming** — run `verify_command` (typecheck + lint + unit), read the output, *then* claim done (empty → build-only, stated).
- **Official docs = source of truth** — specialists and review defer to react.dev / reactnative.dev / Expo / maintainer docs over community sources.
- **HIGH-RIGOR escalation** — diffs touching the profile's `high_rigor_domains` get a mandatory adversarial pass + correctness audit.
- **Precision over recall** — reviews surface a focused list of real findings, not a wall of nits.
