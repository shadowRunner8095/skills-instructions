---
description: "Code review rules for test scope, granularity, and fixture hygiene. Flags smells where a test tries to verify too much, too little, or shares fixtures that couple unrelated cases together."
applyTo: "**/*.tested.stories.ts,**/*.tested.stories.tsx,**/*.tested.stories.js,**/*.tested.stories.jsx,**/*.test.ts,**/*.test.tsx,**/*.test.js,**/*.test.jsx,**/*.spec.ts,**/*.spec.tsx,**/*.spec.js,**/*.spec.jsx"
---

# Test Smells — Scope & Design

## Purpose
Each test should answer **one** well-formed question about the SUT. Tests that try to verify everything at once become unmaintainable; tests that share over-broad fixtures couple unrelated cases together. Flag granularity, scope, and fixture-coupling problems.

## Eager Test
Flag tests that exercise more than one production method/component behavior in a single block. Eager tests are hard to name, hard to fail-diagnose, and creep into integration tests.

- Each `it` should target one method, one branch, or one user-facing behavior.
- Use the test name as a contract: if the name needs *and*, split it.
- Reject:
  ```ts
  it('parses GPS sentence', () => {
    const s = new NmeaSentence('$GPGSA,...');
    expect(s.getLatestPdop()).toBe('2.5');
    expect(s.getLatestHdop()).toBe('1.3');
    expect(s.getLatestVdop()).toBe('2.1');
  });
  ```
- Prefer one `it.each` over the three values, or three named `it` blocks.

## Lazy Test
Flag clusters of tests that all call the same production function in trivially different ways. They likely belong in a single parameterized test.

- Consolidate with `it.each([{ input, expected, label }])`.
- When cases share setup but diverge in assertion, group under one `describe('Cryptographer.decrypt', ...)`.
- Reject six near-identical `it('decrypts X', () => expect(decrypt(X)).toBe(...))` blocks differing only in literals.

## Duplicate Assert
Flag identical (same matcher, same arguments) `expect` calls within a single `it` block. They add no signal and usually indicate copy-paste during debugging.

- One assertion per condition. If a value must be checked under different states, restructure with `describe`/`beforeEach` or parameterize.
- Reject:
  ```ts
  expect(isValid('Fritz-box')).toBe(true);
  // ... unrelated lines ...
  expect(isValid('Fritz-box')).toBe(true);
  ```

## General Fixture
Flag `beforeEach`/`beforeAll` blocks that initialize fields, mocks, or DOM state that some test bodies in the file never use. Over-broad fixtures slow tests, hide dependencies, and tempt developers to silently couple new tests to unrelated state.

- Move setup into the smallest `describe` that needs it.
- Prefer factory functions (`function makeUser(overrides?: Partial<User>) { ... }`) called per-test.
- For Storybook stories, prefer per-story `beforeEach` and `args` over global decorators that mock more than the story uses.
- Reject a top-level `beforeEach` creating 6 mocks when half the `it` blocks need only 2.

## Constructor Initialization (TS variant: module-level mutable state)
TS test files have no constructors, but the equivalent smell is **module-level mutable state**: `let user = makeUser()` at file scope, mutated across tests, never reset. Test order becomes load-bearing.

- Initialize mutable state inside `beforeEach` so each test starts clean.
- Prefer `const` factories called per-test over shared `let` bindings.
- Reject:
  ```ts
  const user = new User('Paul'); // top-level
  it('updates name', () => { user.name = 'X'; expect(user.name).toBe('X'); });
  it('keeps original', () => { expect(user.name).toBe('Paul'); }); // order-dependent!
  ```
- A top-level `const` of an immutable value (`const FROZEN = Object.freeze({...})`) is fine.

## Reviewer checklist
- Does the test name describe exactly one behavior?
- Are near-identical tests collapsed into `it.each`?
- Is every fixture used by every test in its scope?
- Does any test depend on order via shared mutable state?
- Are factories used instead of file-level `let` bindings?
