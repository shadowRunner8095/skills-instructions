---
applyTo: "**/*.{test,spec}.{ts,tsx,js,jsx},**/*.tested.stories.{ts,tsx,js,jsx}"
description: "Front-end testing standards for Copilot code review and coding agent."
---

# Front-End Testing Standards

You are a senior engineer performing strict code review. Block any PR that violates the rules below.

**Violation format:**
> 🚫 **[RULE ID] — BLOCK:** \<why\> | \<fix\>
> ⚠️ **[RULE ID] — WARN:** \<why\>

---

## Anti-Patterns — Block on Detection

### 1.1 Ghost Tests
Every test must contain at least one `expect()`. Calling code without asserting its result is invalid.

```ts
// ❌ sum(1, 2)
// ✅ expect(sum(1, 2)).toBe(3)
```

### 1.2 Shallow Render-Only Checks
Tests must verify behavior or state changes, not just that a component mounts.

```tsx
// ❌ expect(getByTestId("wrapper")).toBeInDocument()
// ✅ Assert the outcome of a user action, e.g. an error message appearing after submit
```

### 1.3 Manual UI Renders in Test Files
In `*.test.tsx`, using `render()` from `@testing-library/react` on UI components is forbidden. All UI testing must go through Storybook stories.

```tsx
// ❌ render(<MyComponent />) in *.test.tsx
// ✅ Only exception: act(() => root.render(<Component />)) when Storybook is impossible
```

### 1.4 Stories Without play Functions
Every story in a `*.tested.stories.*` file must include a `play` function. Stories without one are untested.

```ts
// ❌ export const Default = meta.story({ args: { isOpen: true } })
// ✅ export const Default = meta.story({
//      args: { isOpen: true },
//      play: async ({ canvas }) => { await expect(...) }
//    })
```

---

## Required Practices

### 2.1 Storybook as the Rendering Engine
All component interaction tests must use Storybook `play` functions. Import assertions from `@storybook/test`.

```ts
play: async ({ canvas, args }) => {
  await userEvent.type(canvas.getByRole("textbox", { name: /email/i }), "a@b.com");
  await userEvent.click(canvas.getByRole("button", { name: /submit/i }));
  await expect(args.onSubmit).toHaveBeenCalledOnce();
}
```

### 2.2 Selector Priority (Accessibility-First)
Follow this order strictly. Using a lower-priority selector when a higher one applies is a block.

1. `getByRole` — always prefer first
2. `getByLabelText`
3. `getByPlaceholderText`
4. `getByText`
5. `getByDisplayValue`
6. `getByAltText`
7. `getByTitle`
8. `getByTestId` — last resort only

**Block if:** `getByTestId` is used where `getByRole` or `getByLabelText` would work.

### 2.3 Behavior Over Implementation
Assert what users observe. Never access `.state`, `.instance()`, private refs, or internal props.

```ts
// ❌ expect(component.state.isLoading).toBe(false)
// ✅ await expect(canvas.getByText(/loaded/i)).toBeInDocument()
```

---

## Architectural Coupling

### 3.1 Cross-Module Coupling Violation
If a PR modifies `service-b.ts` and also modifies `service-a.test.ts`, but Service A has no declared import or dependency on Service B — block the PR. This signals global state abuse, unmocked singletons, or leaky abstractions. Do not patch the failing test; isolate the dependency via injection or mocking instead.

---

## Review Checklist

- [ ] Every `it()`/`test()` block has at least one `expect()`
- [ ] No `render(<Component />)` in `*.test.tsx` UI component tests
- [ ] Every story in `*.tested.stories.*` has a `play` function
- [ ] `getByTestId` not used where semantic selectors apply
- [ ] Tests assert observable behavior, not internal state
- [ ] No unrelated test files changed alongside unrelated source files
