# When (and where) to mock — TypeScript

> Reference for [`rn-tdd`](../SKILL.md). TypeScript-native rewrite of mattpocock/skills `tdd/mocking.md` (MIT).
> This is the *decision* (when/where) + design-for-substitutability. The **double taxonomy** (dummy/fake/
> stub/spy/mock), `jest.mock` placement, and module mocking live in `rn-testing-expert §8` — don't duplicate them here.

## Mock only at system boundaries
Substitute the things you don't control or can't make deterministic:
- **Network** — `fetch` / your API client / a GraphQL or query layer (never hit the live network in unit tests)
- **Persistence** — AsyncStorage / MMKV / SQLite / a remote DB (prefer an **in-memory store** over a mock when feasible)
- **Time & randomness** — `new Date()`, timers, `Math.random()`, `crypto.randomUUID()` → inject a fixed clock/value or use fake timers (deterministic tests only)
- **File system / native modules** — sometimes; prefer a fake module over deep mocking

**Don't mock what you own** — your own functions, internal collaborators, value logic. Mocking internals produces
tests coupled to structure that pass while production breaks (see [tests.md](tests.md)). Prefer the real thing:
real > fake (in-memory) > stub > mock.

## Design for substitutability

**1. Inject dependencies — don't construct them inside.**
```ts
// Easy to substitute: the dependency comes in
async function processPayment(order: Order, client: PaymentClient): Promise<Receipt> {
  return client.charge(order.total);
}

// Hard: builds its own concrete dependency — nothing to swap in a test
async function processPayment(order: Order): Promise<Receipt> {
  const client = new StripeClient(Secrets.stripeKey);   // unmockable without module mocking
  return client.charge(order.total);
}
```
Inject via the project's `di` (parameter/default-argument injection, a factory, or context + a test double). A
change that's *hard to test* is a design signal — fix the seam, don't skip the test.

**2. Prefer SDK-style interfaces over one generic `request()`.**
One method per operation → each is independently stubbable with a single fixed return; no conditional logic
inside the double, and a test's dependencies are visible from the interface it stubs.
```ts
// GOOD: one method per operation
interface OrdersApi {
  user(id: UserId): Promise<User>;
  orders(forUserId: UserId): Promise<Order[]>;
  createOrder(draft: OrderDraft): Promise<Order>;
}

// BAD: a single generic entry point forces if/else branching inside every stub
interface GenericApi {
  request(endpoint: Endpoint, options: RequestOptions): Promise<unknown>;
}
```
Benefits: each stub returns one concrete shape · no branching in test setup · the interface documents which
operations a unit actually uses · type-safe per operation.
