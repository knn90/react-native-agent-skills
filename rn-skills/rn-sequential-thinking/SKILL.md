---
name: rn-sequential-thinking
description: "Step-by-step analysis with revision/branching for complex React Native problems — multi-layer data-flow bugs, cache/normalisation issues, navigation routing tangles, state-transition holes, async render/effect-dependency races, money/PII correctness audits. Internal reasoning aid, not a planner. Reads .claude/rn-profile.md for context."
argument-hint: "[problem to analyse step-by-step]"
---

# Sequential Thinking — React Native

Structured problem-solving via reflective thought sequences with dynamic adjustment.

## Step 0 — Load profile (light)
Skim `.claude/rn-profile.md` for `architecture`, `state`, `navigation`,
`data_fetching`, `networking`, `high_rigor_domains` so the patterns below map onto this app's vocabulary.

## When to Apply
At least one of:
- **Multi-step data flow** — service → … → screen; trace where state diverges.
- **Cache / normalisation** (if `data_fetching` caches) — query returns right data, UI shows stale.
- **Navigation routing** — push/pop/modal ordering, retained callbacks, race after `await`.
- **State transitions** (`state`) — loading→loaded→error edges; empty vs error vs initial; pagination handoff.
- **Async render correctness** — stale closures, effect dependency gaps, set-state-after-unmount, double-fire effects.
- **Hypothesis-driven debugging** — symptom is N layers from cause.
- **`high_rigor_domains` correctness audits** — wrong logic ships real-money / PII bugs.

## When NOT to Use
| Case | Use instead |
|---|---|
| Simple one-step answer | Just answer |
| Brutal trade-off comparison | `rn-brainstorm` |
| Plan + phases | `rn-plan` |
| Bug needs log/CI investigation | Direct `Bash`/`Read` |

---

## Core Process

1. **Loose estimate** — `Thought 1/N: [framing]`. Adjust N dynamically.
2. **Structure each thought** — build on prior context, one aspect each, state assumptions/
   uncertainties, signal what the next thought addresses.
3. **Dynamic adjustment** — Expand (more complexity), Contract (simpler), Revise, Branch.
4. **Revision**
   ```
   Thought 5/8 [REVISION of Thought 2]: <corrected understanding>
   - Original: … - Why revised: … - Impact: …
   ```
5. **Branching**
   ```
   Thought 4/7 [BRANCH A from 2]: <approach A>
   Thought 4/7 [BRANCH B from 2]: <approach B>
   ```
   Compare, converge with rationale.
6. **Hypothesis & verification**
   ```
   Thought 6/9 [HYPOTHESIS]: <cause/solution>
   Thought 7/9 [VERIFICATION]: <checked file:line — found …>
   ```
   Verification means reading actual code (path:line), not abstract reasoning.
7. **Complete only when ready** — `Thought N/N [FINAL]`.

## Modes
- **Explicit** (visible chain) — user asks for the trace; complexity warrants interception;
  `high_rigor_domains` correctness reasoning that needs an audit trail.
- **Implicit** (internal) — routine multi-step reasoning where visibility just adds noise.

---

## Reusable Patterns (adapt names to `state`/`navigation`/`data_fetching`)

### State-transition audit
```
1: Define expected sequence (initial → loading → loaded/empty/error)
2: Locate the state holder — hook / store / reducer (path:line)
3: Trace each emit point — does each branch exit cleanly?
4: Pagination — cursor handoff between pages
5 [HYPOTHESIS]: skipped state / lost cursor / double-emit
6 [VERIFICATION]: read tests — is this edge covered?
N [FINAL]: root cause + fix + test gap
```

### Cache / normalisation debug  (only if data_fetching caches)
```
1: Identify query key / fragment + cache key policy
2: What writes that key? (mutation, manual cache update, invalidation)
3: Subscribers — which component/hook owns the lifecycle?
4 [HYPOTHESIS]: stale entity after partial update
5 [VERIFICATION]: read the query hook + key fn (never edit generated_paths)
N [FINAL]: cache fix + invalidation strategy
```

### Async render / effect race
```
1: Identify async boundaries (effect / event handler created, awaited, cancelled)
2: Render vs effect — any state write after unmount or off a stale render?
3: Cancellation — does the in-flight request honour an AbortController / cleanup?
4: Re-entrancy — can the effect re-fire while a previous await is pending? (deps, double-mount)
5 [HYPOTHESIS]: e.g. cursor advanced while previous fetch in-flight; stale closure captured old state
6 [VERIFICATION]: read the hook + service; confirm cleanup / dependency array
N [FINAL]: race surface + cleanup/dependency fix
```

### Navigation routing knot
```
1: Diagram screens + desired transitions
2: Locate the navigator / route config — what callbacks pass to children?
3: When captured? stale closure over old params / state?
4: Presentation order — modal over screen? navigate during a transition animation?
5 [HYPOTHESIS]: missed dismiss / animation timing / param mismatch
6 [VERIFICATION]: trace one path end-to-end
N [FINAL]: routing fix + invariant a test must protect
```

### Money / PII correctness audit  (high_rigor_domains)
```
1: What value flows here? (integer cents? number? string from backend?)
2: Precision boundaries — where formatting/rounding happens (floating-point math)
3: Sign / direction — refund vs charge, credit vs debit
4: Edge cases — zero, negative, very large, multi-currency
5: PII — what's logged? crosses analytics / crash reporting?
6 [HYPOTHESIS]: precision loss / leaked PII / wrong sign
7 [VERIFICATION]: run the math on paper + grep log emissions on the value
N [FINAL]: pass/fail per case + remediation
```

## Constraints
- **DO NOT** implement during thinking — just reason.
- **DO NOT** treat this as plan generation — use `rn-plan`.
- **DO NOT** dump the full chain when only a result was asked — collapse to conclusion + key reasoning.
- **Verify, don't speculate** — a claim about code is verified by the next thought (path:line).
- **Stop when done** — don't pad to hit a count.
