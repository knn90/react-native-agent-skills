---
name: rn-scout
description: "Fast, token-efficient codebase scouting for a React Native project. Spawns parallel Explore subagents scoped to feature/module folders. Use for file discovery, gathering task context, and locating where a feature lives before edits. Reads .claude/rn-profile.md for repo layout."
argument-hint: "[search-target] [quick|full]"
model: best
effort: xhigh
---

# Scout — React Native

Find things fast using parallel `Explore` subagents. Output is a **map**, not analysis.

## Step 0 — Load profile

Read `.claude/rn-profile.md` for `source_roots`, `test_roots`, `generated_paths`,
`navigation`, `state`, `data_fetching`, `networking`. If absent → tell the user to run
`rn-project-init` first (greenfield repos have nothing to scout).

## When to Use

- A feature spans multiple feature folders / packages / `source_roots`
- "Where is X?", "find Y", "locate the hook/screen/navigator for Z"
- Before cross-cutting edits (shared state, context providers, network operations)
- DRY check — confirm a pattern exists before adding a new one

## When NOT to Use

| Case | Use instead |
|---|---|
| Single file, known path | `Read` |
| One symbol grep | `Grep` |
| Trade-off discussion | `rn-brainstorm` |
| Full implementation plan | `rn-plan` |
| Pattern outside the repo | `rn-research` |

---

## Argument Parsing

- **TARGET** (required) — symbol name, feature area, function, network operation, or NL description.
- **Depth** — `quick` (default, single agent) | `full` (2-3 parallel agents).

If TARGET missing, ask via `AskUserQuestion`.

---

## Workflow

### 1. Estimate scale (cheap probes)

Use profile paths:
```
Glob {source_roots}/**/<term>*.{ts,tsx}
Glob {test_roots}/**/<term>*.{test,spec}.{ts,tsx}
Grep <term> across the narrowed dirs
```

Agent count:
- **1** — < 50 matched files, single feature folder
- **2** — feature touches app code + a workspace package
- **3** — split: app code / workspace packages / tests

Don't spawn agents for trivial single-file lookups (overhead > benefit).

### 2. Spawn parallel `Explore` agents (single message → concurrent)

Each gets a tight scope. Prompt template:

```
Scope: {absolute path under a source_root}
Search target: {TARGET}

Find:
1. Primary implementation files (screen components, hooks, containers, navigators, services)
2. Tests covering them ({test_roots})
3. Direct usages: callers, navigation wiring, context/provider registrations
4. Related models / DTOs / network operations ({networking}/{data_fetching})
5. accessibilityLabel / testID usages (if a UI surface)

Return brief markdown: `file_path:line — one-line description`.
Do NOT paste file contents. Cap at 25 most relevant matches.
```

**Never** scout into `generated_paths` from the profile (e.g. `ios/`, `android/`, `.expo/`, generated types).

### 3. Aggregate

Combine, dedupe, sort by relevance. Note any agent that timed out and continue.

---

## Report Format

```markdown
# Scout Report: <TARGET>

**Branch**: <current>   **Agents**: <N>

## Relevant Files
### App code
- `<path>:<line>` — <desc>
### Workspace packages
- `<path>:<line>` — <desc>
### Tests
- `<path>:<line>` — <desc>

## Patterns Observed
- <state pattern, provider mechanism, navigation wiring actually seen>

## Unresolved Questions
- <gap or ambiguity>
```

If invoked inline (no report needed), output to the user directly.

## Constraints

- **DO NOT** read full files into main context — that's the subagent's job.
- **DO NOT** scout into `generated_paths`.
- **DO NOT** spawn agents for trivial single-file lookups.
- **Verify, don't guess** — every returned path must exist. Cap at top 25.
