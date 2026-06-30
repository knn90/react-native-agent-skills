---
name: rn-solid-expert
description: "Expert SOLID + clean-decoupling review for React Native / TypeScript — the 5 principles by name (SRP, OCP, LSP, ISP, DIP), dependency inversion + composition-root DI, decoupling patterns (Decorator / Composite / Adapter / Facade), and framework isolation (keep React Native / navigation / storage / native-module APIs out of the domain). Consolidated by rn-skill-consolidate. Loaded by rn-code-review (critique) and rn-execute (apply) when a change adds/alters types, interfaces, services, hooks/containers, or DI/composition wiring. Delegates *which architecture pattern* to the architecture lens. Reads .claude/rn-profile.md."
---

# SOLID / Decoupling Expert (React Native / TypeScript)

> **Generated skill — original wording, consolidated by `rn-skill-consolidate`.** Authority: **Robert C.
> Martin's SOLID + Clean Architecture** (canonical, general SW engineering) translated to **TypeScript's
> structural typing + React composition**. Worked patterns: **refactoring.guru** (Decorator / Composite /
> Adapter / Facade). Also references community frontend-engineering skills (cite/learn-from). Last
> consolidated **2026-06-30**. Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

How it's used: **rn-code-review** routes structural/architectural slices here; **rn-execute** applies
this while writing. **Scope = principles, not pattern choice** — *which* architecture (feature-folders /
container-presentational / clean-ish domain-data-ui) is decided by the architecture lens; this skill
judges the SOLID/decoupling *beneath* any pattern.

**Source of truth:** SOLID is **general** software engineering (Robert C. Martin) — React/RN take no
official app-architecture stance. The idiomatic translation is **interface-driven design + composition**
(TS structural interfaces, small types, dependency passed via props/context/constructor params/factory
functions). Gate to the project's `architecture` + conventions in `rules_file`.

**North star (YAGNI):** SOLID serves *change*, not ceremony. An abstraction earns its keep only with
≥2 implementations **or** a real test seam. Don't add an interface with one implementer and no test — that's speculative generality.

---

## The five principles (rule · TS/React idiom · smell → fix)

**1. SRP — Single Responsibility.** One reason to change / one concern a unit answers to. *Smell:* a
hook/container that fetches **and** parses **and** caches **and** formats; a `Manager`/`Helper`/`utils`
doing everything. *Fix:* one use-case/service per responsibility; the hook/container only orchestrates.

**2. OCP — Open/Closed.** Add behavior by adding a type, not by editing existing code. *TS:* an
interface + a new implementer / injected strategy. *Smell:* a `switch (kind) { … }` you must edit for
every new case. *Fix:* polymorphism / strategy map / a registry.

**3. LSP — Liskov Substitution.** An implementer must be usable everywhere the abstraction is, with no
surprises. *TS:* prefer composition + interfaces over deep class inheritance. *Smell:* an implementer
that `throw new Error("unsupported")`s or no-ops part of the interface. *Fix:* split the interface (→ ISP).

**4. ISP — Interface Segregation.** Many small, client-focused interfaces over one fat one. *TS:*
`FeedLoader`, `ImageCache` — compose with intersection types (`A & B`). *Smell:* a god interface with
15 methods; implementers stubbing half of them. *Fix:* split by what each client actually uses.

**5. DIP — Dependency Inversion.** Policy depends on **abstractions it owns**, not on concrete
details. *TS:* the **domain declares** `interface FeedLoader`; the networking/storage module
*implements* it; the composition root wires them. *Smell:* a hook/service that `import`s `fetch`/
`AsyncStorage`/a native module or `new`s a concrete client inline. *Fix:* invert — depend on a domain
interface, inject the impl.

## Dependency injection & the composition root
- **Inject at the edge.** The **composition root** (app entry, a provider tree, or a `makeX`/`*Composer`
  factory) is the *only* place that knows concrete types and assembles the graph. Everything else takes
  abstractions — via constructor params, factory args, props, or a typed React context.
- *Smells:* module-level singletons / direct imports reached from policy code; **over-injection**
  (≥4–5 deps ⇒ the unit does too much → split it, or hide a subsystem behind a Facade).
- Default-param or context injection gives testability without ceremony (no live `fetch`,
  `AsyncStorage`, `Date.now()`, or `navigation` reached from inside).

## Decoupling patterns (the decoupling toolkit)
- **Decorator** — add a cross-cutting concern (logging, analytics, retry, caching) with the *same*
  interface in and out, without touching the decorated unit.
- **Composite** — combine implementations behind one interface (e.g. remote-**with-fallback-to**-cache `FeedLoader`).
- **Adapter** — bridge a concrete/SDK type (a native module, an HTTP client) to the domain's interface, keeping the SDK at the edge.
- **Facade** — hide a multi-step subsystem behind a simple interface for the composition root.

## Framework isolation
- The **domain is pure TypeScript** — no `react-native` imports, no navigation / storage / native-module
  APIs (`AsyncStorage`, `react-navigation`, `fetch`, `Platform`, `NativeModules`) in domain types.
  Frameworks are *replaceable details* behind interfaces the domain owns; map DTO → domain at the
  boundary. *Smell:* a `react-native` import in a model; a wire/`AsyncStorage` DTO leaking into the
  domain or into a component. (Pairs with the testing skill's mockability + state skill's render rules.)

## Interface-driven design (the TS translation)
- Small interfaces, plain types, intersection/union composition, and structural typing for shared shape —
  this is how SOLID lands idiomatically in TypeScript. Prefer `interface`/`type` seams and factory
  functions over inheritance. But heed the north star: **don't over-abstract.**

## Review checklist (what this skill flags)
Per changed/added unit: single responsibility? depends on abstractions or concretes? a framework
leaking into the domain? interface the right size (ISP) / honestly substitutable (LSP)? extended vs
modified (OCP)? wired at the composition root, not imported as a singleton? Output each finding as
`{ severity, file:line, principle, problem, fix }`. Severity: **Critical** (framework leak into domain,
concrete dependency that blocks testing a high-value path) · **Important** (God unit, fat interface,
over-injection, OCP switch-edit) · **Nit** (naming, minor seam). Be specific; don't invent abstraction
needs (flag *speculative* abstractions too).

## Boundaries (avoid duplication)
- **Pattern selection** (feature-folders vs container/presentational vs clean-ish) → the architecture lens, not here.
- **Render correctness / stale closures / effect deps** → `rn-state-expert`. **React composition / state ownership** →
  `react-expert`. **Test seams / doubles** → `rn-testing-expert`. **Type-level modeling** → `typescript-expert`.
  This skill is the principle-level "is it decoupled and substitutable?" lens that sits beneath all of them.
