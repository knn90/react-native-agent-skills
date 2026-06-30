---
name: rn-research
description: "Strategic technical research for a React Native project (TypeScript / React / React Native / navigation / state / data fetching / security / accessibility). Use for library evaluation, framework pattern questions, migration questions, CVE checks. Grounded first in what already works in the codebase. Outputs a sourced report. Reads .claude/rn-profile.md."
argument-hint: "[topic | TICKET-ID]"
---

# Research — React Native

Strategic research producing an actionable, sourced report grounded in current React/React
Native best practices **and** what already works in this codebase.

**Principles:** YAGNI + KISS + DRY. Honest, brutal, concise.
**Bias toward what already works in the repo.** Don't recommend a library the project
doesn't use unless you can show the existing pattern is genuinely insufficient.

## Step 0 — Load profile

Read `.claude/rn-profile.md`: `rn_version`, `expo_sdk`, `new_architecture`, `architecture`,
`networking`, `data_fetching`, `navigation`, `state`, `reports_dir`,
`ticket_system`/`ticket_pattern`/`ticket_fetch`. If missing → run `rn-project-init`.

## When to Use

- Evaluating a library/SDK for adoption
- Framework patterns (state, navigation, performance) for an unfamiliar problem
- Migration questions (data-fetching layer upgrade, New Architecture adoption, React Navigation→Expo Router)
- Security best practices (secure storage, biometrics, SSL pinning, deep-link validation)
- Accessibility patterns beyond what the repo already documents
- Recent CVEs / breaking changes for an existing dependency

## When NOT to Use

| Case | Use instead |
|---|---|
| Pattern already exists in repo | `rn-scout` |
| API doc for a known package | `WebFetch` against the official URL |
| Trade-off discussion of approaches | `rn-brainstorm` |
| Step-by-step debug reasoning | `rn-sequential-thinking` |

---

## Argument Parsing

- **TICKET-ID** (matches `ticket_pattern`) → fetch via `ticket_fetch` for context.
- **Free-form topic** → research as-is. If missing, ask via `AskUserQuestion`.

---

## Process

### Phase 1 — Scope
- Key terms; recency window (< 12 months unless historical); in/out of scope.
- Source ranking: react.dev > reactnative.dev > docs.expo.dev > maintainer docs > major eng blogs > recent SO.

### Phase 2 — Codebase reality check (FIRST)
Before any web search:
- Is there already a working pattern? → `rn-scout` / `Grep`.
- What's pinned? → `package.json`, lockfile (`package-lock.json` / `yarn.lock` / `pnpm-lock.yaml`).
- What's documented? → `rules_file` + `docs_root`.
- **If the codebase already solves it → say so and stop.** That's a successful outcome.

### Phase 3 — External research (≤ 5 search calls)
Preferred order: react.dev → reactnative.dev → docs.expo.dev → maintainer docs
(TanStack Query / React Navigation / Shopify FlashList / Callstack RNTL / Detox / Maestro,
e.g. the lib for `data_fetching` or `navigation`) → reputable RN eng blogs → recent
GitHub Discussions/SO.

Query patterns: `<topic> react native <rn_version>`, `<topic> expo sdk <expo_sdk>`,
`<topic> <data-fetching-lib> <version>`, `<topic> CVE <year>`.

### Phase 4 — Cross-reference & validate
- 2+ independent sources per claim; flag conflicts.
- Reject APIs above `rn_version` / `expo_sdk` unless the user agrees to bump the version floor.
- Discard anything older than the framework major version in use.

### Phase 5 — Report
Save to `{reports_dir}/research-{YYMMDD-HHMM}-{TICKET|slug}.md`.
`{YYMMDD-HHMM}` MUST come from `bash -c 'date +%y%m%d-%H%M'`, not model memory.
Create `{reports_dir}` if absent.

---

## Report Template

```markdown
# Research: <topic>

**Ticket**: <id | n/a>   **Date**: <YYYY-MM-DD>   **Branch**: <current>
**Versions**: RN <rn_version>, Expo SDK <expo_sdk>, <data-fetching-lib <Z> if any>  ← from repo

## Executive Summary
<3-5 bullets — findings + recommendation>

## Codebase Context
- Existing pattern: <what's there> (path:line)
- Why current approach falls short: <reason | "n/a — greenfield">

## Findings
### Best Practices (current consensus)
### Security / Privacy  (secure storage, SSL pinning, biometrics, deep-link validation — flag affected pinned versions)
### Performance Insights  (numbers > vibes)
### Comparative Analysis (if multiple options)
| Option | Effort | Risk | Maintenance | Fit |

## Recommendation
**Chosen**: …   **Rationale**: …

## Implementation Sketch
- Affected modules / state shape / data fetching / networking / i18n impact

## Sources
1. [Title](url) — checked YYYY-MM-DD

## Open Questions
- <unresolved — owner: PM / BE / Design>
```

## Constraints

- **DO NOT** implement — research only. **DO NOT** exceed 5 web searches.
- **DO NOT** recommend a library the project doesn't need (YAGNI).
- **MUST** cite sources with check dates; save under `{reports_dir}`.
- **Skip if already known** — codebase + `rules_file` answer it → say so and stop.
- **Sacrifice grammar for concision** — bullet > paragraph.
