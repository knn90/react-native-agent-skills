# Good vs bad tests (TypeScript)

> Reference for [`rn-tdd`](../SKILL.md). TypeScript-native rewrite of mattpocock/skills `tdd/tests.md` (MIT).
> The *one rule*: test observable behavior through the **public interface**, never internal structure.

## Good — integration-style, behavior through the public API
Exercises real code paths the way a caller would; survives internal refactors; the name says **what**, not how.

```ts
it("lets a user check out with a valid cart", async () => {
  const cart = new Cart();
  cart.add(product);
  const result = await checkout(cart, paymentMethod);
  expect(result.status).toBe("confirmed");        // outcome the caller cares about
});
```
Characteristics: behavior callers care about · public API only · one logical assertion · survives refactors · describes WHAT.

## Bad — coupled to implementation
Breaks when you refactor even though behavior is unchanged — the signal it was testing *how*, not *what*.

```ts
// BAD: asserts an internal interaction, not an outcome
it("calls paymentService.process during checkout", async () => {
  const process = jest.fn();
  await checkout(cart, { process });
  expect(process).toHaveBeenCalledTimes(1);        // breaks on any refactor; proves nothing about the result
});
```
Red flags: mocking internal collaborators · testing private functions · asserting call counts/order · name describes HOW · verifying through a back channel instead of the interface.

## Verify through the interface, not a back channel
```ts
// BAD: reaches past the interface into the store
it("writes a row when a user is created", async () => {
  await createUser({ name: "Alice" });
  const rows = await db.query("SELECT * FROM users WHERE name = ?", ["Alice"]);
  expect(rows).not.toHaveLength(0);                // couples the test to the schema
});

// GOOD: round-trips through the public API
it("makes a created user retrievable", async () => {
  const created = await createUser({ name: "Alice" });
  const fetched = await userById(created.id);
  expect(fetched.name).toBe("Alice");
});
```

For a screen/component, the "interface" is what the **user sees and does** — query the rendered output
and drive interactions, don't reach into hook state or props:
```ts
// GOOD: behavior through the rendered interface
it("shows the greeting after submit", async () => {
  render(<Greeter />);
  await userEvent.type(screen.getByPlaceholderText("Name"), "Alice");
  await userEvent.press(screen.getByRole("button", { name: "Submit" }));
  expect(await screen.findByText("Hello, Alice")).toBeOnTheScreen();
});
```
(see `rn-testing-expert §12`). For matchers, `render`/`screen`/`waitFor`, query priority, and
`user-event`, see `rn-testing-expert §3–5`.
