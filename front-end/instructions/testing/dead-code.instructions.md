---
description: "Code review rules for dead code and debugging residue in tests. Flags content that runs without asserting, pretends to run but does not, or pollutes CI output with leftover debugging."
applyTo: "**/*.tested.stories.ts,**/*.tested.stories.tsx,**/*.tested.stories.js,**/*.tested.stories.jsx,**/*.test.ts,**/*.test.tsx,**/*.test.js,**/*.test.jsx,**/*.spec.ts,**/*.spec.tsx,**/*.spec.js,**/*.spec.jsx"
---

# Test Smells — Dead Code & Debugging Residue

## Purpose
Test files attract debugging artifacts — `console.log` lines, commented-out blocks, `.skip` markers, tautological assertions — that survive merges and silently degrade the suite. Flag anything that runs but adds no signal, or that doesn't run but pretends to.

## Empty Test
Flag any `it` / `test` / Storybook `play` whose body is empty, contains only comments, or contains only setup with no assertion or action.

- Either delete the test or restore its body. An empty test is worse than no test: it reports green and gives false coverage.
- Reject:
  ```ts
  it('parses credentials', () => {
    // const creds = parse(SAMPLE);
    // expect(creds.user).toBe('user@example.com');
  });
  ```
- A single `it.todo('description')` is acceptable as a placeholder *only* if the PR explicitly tracks it (linked issue or TODO comment). Otherwise flag.
- Empty Storybook `play` functions: reject `play: async () => {}`.

## Ignored Test
Flag `.skip`, `xit`, `xdescribe`, `it.skip`, `describe.skip`, `test.skip`, or comment-out of entire blocks. Skipped tests rot.

- Either fix the test, delete it, or convert to `it.todo` with a linked issue reference in a comment.
- Conditional skips (`it.skipIf(env)`) require an inline comment explaining the condition.
- Reject `it.skip('flaky for now', ...)` without a tracking ticket reference.
- Reject `.only` (`it.only`, `describe.only`, `fit`, `fdescribe`) left in committed code — it silently disables every other test.
- Reject `// @ts-ignore` / `// @ts-expect-error` placed *just to suppress a failing assertion type* — fix the type or fix the test.

## Redundant Print
Flag `console.log`, `console.debug`, `console.info`, `console.dir`, `console.trace`, `process.stdout.write` inside test bodies, hooks, or Storybook `play`. Prints are debugging leftovers; they pollute CI output and never assert anything.

- Remove all `console.*` calls from tests.
- If logging is genuinely needed for diagnostics (rare), use the assertion's message argument or `expect.fail(message)`.
- Reject:
  ```ts
  const result = transformer.transform(input);
  console.log('result =', result);
  expect(result).toEqual(expected);
  ```
- Tests that *assert on* `console.error` calls (e.g. `expect(spy).toHaveBeenCalled()`) are fine — that's a real assertion, not a print.

## Redundant Assertion
Flag assertions that are tautologically true or false: `expect(true).toBe(true)`, `expect(x).toBe(x)`, `expect(1).toEqual(1)`, or weaker assertions left in front of stronger ones.

- Every assertion must be capable of failing if the SUT regresses.
- Replace `toBeDefined` / `toBeTruthy` placeholders with the real expectation once the value is known.
- Reject:
  ```ts
  expect(true).toBe(true);
  expect(user).toBe(user);
  expect(result).toBeTruthy(); // redundant if next line is stricter
  expect(result).toEqual(EXPECTED);
  ```
- A standalone `expect(value).toBeDefined()` is acceptable only when *existence* is the actual contract being tested (e.g. `process.env.X` guard).

## Reviewer checklist
- Does every block have at least one assertion that can fail?
- Are there any commented-out test bodies?
- Is there any `.skip` / `.only` / `xit` / `fdescribe` left in?
- Are there any `console.*` calls outside of explicit spy assertions?
- Are there any tautological `expect(x).toBe(x)` patterns?
- A passing test that asserts nothing real is a regression waiting to happen — flag it as severely as a failing test.
