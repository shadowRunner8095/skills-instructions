---
description: "Code review rules for test reliability, determinism, and isolation. Flags smells that introduce flakiness, branching, hidden environment coupling, or non-reproducible behavior."
applyTo: "**/*.tested.stories.ts,**/*.tested.stories.tsx,**/*.tested.stories.js,**/*.tested.stories.jsx,**/*.test.ts,**/*.test.tsx,**/*.test.js,**/*.test.jsx,**/*.spec.ts,**/*.spec.tsx,**/*.spec.js,**/*.spec.jsx"
---

# Test Smells — Reliability & Determinism

## Purpose
A test that fails on CI and passes locally — or vice versa — is worse than no test: it trains the team to ignore failures. Flag any pattern that introduces non-determinism, hidden environment coupling, or branching that makes coverage unprovable.

## Conditional Test Logic
Flag `if`, `switch`, `for`, `while`, `try/catch`, ternaries, and `.forEach`/`.filter` whose bodies contain assertions. Branching means the assertion may never run, and coverage is unprovable.

- Use `it.each([...])` / `test.each([...])` to parameterize.
- Split branches into separate `it` blocks named after the branch.
- Reject:
  ```ts
  it('renders rows', () => {
    for (const row of rows) {
      if (row.visible) expect(screen.getByText(row.label)).toBeVisible();
    }
  });
  ```
- Prefer:
  ```ts
  it.each(rows.filter(r => r.visible))('renders $label', ({ label }) => {
    expect(screen.getByText(label)).toBeVisible();
  });
  ```

## Exception Handling
Flag manual `try/catch` blocks that call `fail()`, `expect(false).toBe(true)`, or rethrow to assert error behavior. The framework already handles this.

- Sync errors: `expect(() => sut(arg)).toThrow(SpecificError)`.
- Async errors: `await expect(sut(arg)).rejects.toThrow(SpecificError)`.
- Always assert the error's *type or message*, never just that *something* threw.
- Reject:
  ```ts
  try {
    sut.compute();
  } catch (e) {
    fail((e as Error).message);
  }
  ```

## Sleepy Test
Flag any wall-clock wait inside a test body: `setTimeout`, `setInterval`, `await sleep(N)`, `await new Promise(r => setTimeout(r, N))`. Sleeps are flaky on slow CI.

- Use `vi.useFakeTimers()` / `jest.useFakeTimers()` and advance time deterministically (`vi.advanceTimersByTime`, `jest.runAllTimers`).
- For async UI: `await waitFor(...)`, `await findByText(...)`, `await screen.findBy*` (these poll with timeout).
- In Storybook play functions: `await waitFor(() => expect(...))` or `await canvas.findByRole(...)`.
- Reject any literal `await new Promise(r => setTimeout(r, 500))` outside fake-timer setup.

## Mystery Guest
Flag tests that read or write the real filesystem, network, database, environment variables, `process.cwd()`, system clock, or other process-wide state.

- Mock with `vi.mock` / `jest.mock`, `msw` for HTTP, `memfs` for fs, `vi.setSystemTime` for clock.
- Inject dependencies via parameter or factory; do not import singletons that touch I/O.
- Reject `fs.readFileSync('./data.json')`, raw `fetch('https://...')`, `new Date()` (without mocked time), or direct `process.env.X` reads in test bodies.
- For Storybook stories: mock with `args: { fetchUsers: fn() }` and configure in `beforeEach`, never call real network.

## Resource Optimism
When a test *must* touch a real resource (rare; flag and request justification), it must verify preconditions — not assume them.

- Assert fixture existence/shape in `beforeAll`; fail fast with a clear message.
- Use `os.tmpdir()` / `tmp` libraries; clean up in `afterEach` / `afterAll`.
- Reject `fs.readFileSync('./fixtures/file.json')` followed directly by use, with no existence check and no cleanup.
- Reject hardcoded absolute paths (`/tmp/...`, `/Users/...`, `C:\\...`).

## Reviewer checklist
- Is every assertion guaranteed to execute on every run?
- Are errors asserted via `.toThrow` / `.rejects.toThrow`?
- Are timers, clock, network, fs, and env mocked?
- Will this test pass identically on Linux CI, Windows, and a teammate's laptop?
- If a fixture is read, is existence verified and cleanup done?
