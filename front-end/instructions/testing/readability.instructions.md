---
description: "Code review rules for test readability and self-documentation. Flags smells that make a test hard to understand or whose failure message gives no useful signal."
applyTo: "**/*.tested.stories.ts,**/*.tested.stories.tsx,**/*.tested.stories.js,**/*.tested.stories.jsx,**/*.test.ts,**/*.test.tsx,**/*.test.js,**/*.test.jsx,**/*.spec.ts,**/*.spec.tsx,**/*.spec.js,**/*.spec.jsx"
---

# Test Smells — Readability & Self-Documentation

## Purpose
A test must communicate intent at a glance. When it fails, the test name and assertion message alone should tell the reviewer what broke and why. Flag any test where the *what*, the *why*, or the *expected value* is opaque to a reader who hasn't read the SUT.

## Assertion Roulette
Flag `it`/`test`/`play` blocks with **more than one assertion** where a failure cannot be uniquely attributed.

- Prefer one logical assertion per test, derived from the test name.
- When multiple assertions are unavoidable, use a labeled message: `expect(value, 'returns admin role').toBe('admin')` (Vitest) or `expect.soft(...)`.
- For object shape, prefer `toEqual` / `toMatchObject` over multiple property assertions.
- Reject sequences like:
  ```ts
  expect(user.id).toBe(1);
  expect(user.name).toBe('Paul');
  expect(user.role).toBe('admin');
  ```
  unless the test name covers the full shape, or rewrite as `expect(user).toMatchObject({ id: 1, name: 'Paul', role: 'admin' })`.

## Magic Number Test
Flag numeric, string, or boolean literals in assertions whose meaning isn't obvious from context.

- Extract literals into typed `const` with explanatory names.
- Use enums or `as const` for domain values.
- Reject:
  ```ts
  expect(calc.compute(15.5)).toEqual({ hour: 15, minute: 30 });
  ```
- Prefer:
  ```ts
  const HALF_PAST_THREE_PM_DECIMAL = 15.5;
  const HALF_PAST_THREE_PM = { hour: 15, minute: 30 } as const;
  expect(calc.compute(HALF_PAST_THREE_PM_DECIMAL)).toEqual(HALF_PAST_THREE_PM);
  ```

## Sensitive Equality
Flag assertions that compare via `.toString()`, `JSON.stringify`, template strings, or default coercion. They break silently when the SUT adds a field, implements `toJSON`, or changes key order.

- Compare structured data with `toEqual` / `toMatchObject` / property assertions.
- For DOM nodes use `@testing-library` matchers (`toHaveTextContent`, `toHaveAttribute`, `toBeVisible`).
- Reject:
  ```ts
  expect(user.toString()).toBe('User(1, Paul)');
  expect(JSON.stringify(result)).toBe(snapshot);
  ```
- `toMatchSnapshot` is acceptable only for stable, small, intentional shapes — flag snapshots over arbitrary class instances or large trees.

## Unknown Test
Flag any `it` / `test` / Storybook `play` whose body contains no assertion. A test with no `expect` passes when the SUT is broken.

- Every block must end in at least one `expect(...)`, `await expect(...).rejects/resolves...`, or Storybook `await expect(canvas...)`.
- Tests that only check absence of throws must use `expect(() => fn()).not.toThrow()` explicitly.
- Reject:
  ```ts
  it('hits the API', async () => {
    await client.getPOICategories(16);
  });
  ```
- In Storybook play functions, reject `play` blocks that only call `userEvent.*` with no `await expect(...)` at the end.

## Default Test
Flag tests created from a scaffolder template that were never specialized.

- Reject names like `test1`, `example`, `addition_isCorrect`, `should work`, `it works`, file names like `Example.test.ts`.
- Test names must follow `it('does X when Y', ...)` or Given/When/Then. The name should make the assertion line redundant.
- Reject test files that import nothing from the SUT or contain only the framework's example assertion (`expect(2 + 2).toBe(4)`).

## Reviewer checklist
- Can the failure message alone identify what broke?
- Is every literal a named constant or self-evident?
- Are comparisons structural, not stringified?
- Does every block end in `expect`?
- Does the test name describe behavior, not the function called?
