---
name: react-testing-library-best-practices
description: Best practices and common mistakes for React Testing Library tests. Use when writing, reviewing, or debugging RTL tests with Jest — covers query priority, async patterns, user interactions, and anti-patterns to avoid.
---

# React Testing Library Best Practices

Distilled from Kent C. Dodds — [Common Mistakes with React Testing Library](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library).

Use this skill whenever writing, reviewing, or refactoring tests that use `@testing-library/react`. The goal of every rule below is the same: **tests should resemble how users use the software**.

## Quick Reference (Decision Table)

| Situation | Use |
|---|---|
| Querying for an element that should exist | `screen.getBy*` |
| Querying for an element that should NOT exist | `screen.queryBy*` (only here) |
| Querying for an element that will appear async | `await screen.findBy*` |
| First-choice query type | `*ByRole` with `{ name: ... }` |
| Simulating user interaction | `userEvent` (not `fireEvent`) |
| Asserting an element property | `@testing-library/jest-dom` matchers (`toBeDisabled`, `toBeInTheDocument`, ...) |
| Waiting for async assertion | `await waitFor(() => expect(...))` — single assertion, no side-effects |

## The 15 Mistakes

### 1. Not using the Testing Library ESLint plugins

Install both — they catch most of the issues below automatically:

```sh
yarn add -D eslint-plugin-testing-library eslint-plugin-jest-dom
```

`create-react-app` ships with `eslint-plugin-testing-library` enabled.

---

### 2. Naming the `render()` return value `wrapper`

`wrapper` is Enzyme legacy. The return value of `render()` doesn't wrap anything; it's a bag of utilities you usually shouldn't reach for.

```js
// ❌
const wrapper = render(<Example prop="1" />)
wrapper.rerender(<Example prop="2" />)

// ✅
const { rerender } = render(<Example prop="1" />)
rerender(<Example prop="2" />)
```

If you really need the bag, name it `view`.

---

### 3. Calling `cleanup()` manually

Modern test frameworks auto-cleanup between tests. Manual `cleanup` is dead code.

```js
// ❌
import { render, screen, cleanup } from '@testing-library/react'
afterEach(cleanup)

// ✅
import { render, screen } from '@testing-library/react'
```

---

### 4. Destructuring queries from `render()` instead of using `screen`

`screen` removes maintenance burden (no destructure list to keep updated) and gives you autocomplete.

```js
// ❌
const { getByRole } = render(<Example />)
const errorMessageNode = getByRole('alert')

// ✅
render(<Example />)
const errorMessageNode = screen.getByRole('alert')
```

`screen.debug()` is also strictly better than `wrapper.debug()`.

---

### 5. Asserting on raw DOM properties instead of jest-dom matchers

jest-dom matchers produce dramatically better failure messages.

```js
const button = screen.getByRole('button', { name: /disabled button/i })

// ❌ — error: "Expected: true, Received: false"
expect(button.disabled).toBe(true)

// ✅ — error: "Received element is not disabled: <button />"
expect(button).toBeDisabled()
```

Common jest-dom matchers: `toBeInTheDocument`, `toBeDisabled`, `toBeChecked`, `toHaveTextContent`, `toHaveValue`, `toHaveAttribute`, `toHaveClass`, `toBeVisible`.

---

### 6. Wrapping things in `act()` unnecessarily

`render` and `fireEvent` are already wrapped in `act`. Manual `act()` calls just hide the real warning the framework is trying to give you.

```js
// ❌
act(() => {
  render(<Example />)
})
const input = screen.getByRole('textbox', { name: /choose a fruit/i })
act(() => {
  fireEvent.keyDown(input, { key: 'ArrowDown' })
})

// ✅
render(<Example />)
const input = screen.getByRole('textbox', { name: /choose a fruit/i })
fireEvent.keyDown(input, { key: 'ArrowDown' })
```

When you see an `act()` warning, fix the underlying async issue (usually missing `await`), don't suppress it.

---

### 7. Using the wrong query

Follow the [priority order](https://testing-library.com/docs/queries/about/#priority):

1. **Accessible to everyone:** `getByRole`, `getByLabelText`, `getByPlaceholderText`, `getByText`, `getByDisplayValue`
2. **Semantic queries:** `getByAltText`, `getByTitle`
3. **Test IDs (last resort):** `getByTestId`

```js
// ❌
// DOM: <label>Username</label><input data-testid="username" />
screen.getByTestId('username')

// ✅
// DOM: <label for="username">Username</label><input id="username" type="text" />
screen.getByRole('textbox', { name: /username/i })
```

#### 7a. Don't reach for `container.querySelector`

```js
// ❌
const { container } = render(<Example />)
const button = container.querySelector('.btn-primary')

// ✅
render(<Example />)
const button = screen.getByRole('button', { name: /click me/i })
```

`container` queries bypass accessibility, are brittle to refactors, and are hard to read.

#### 7b. Query by actual text content

Test IDs hide whether your i18n / content is correct. Querying by text means content changes show up as test failures — which is **good**: they force a deliberate decision.

```js
// ❌
screen.getByTestId('submit-button')

// ✅
screen.getByRole('button', { name: /submit/i })
```

#### 7c. Prefer `*ByRole` even when text-matching feels easier

`*ByRole` handles split text across child elements (it matches accessible name) and gives you a roles table on failure.

```js
// DOM: <button><span>Hello</span> <span>World</span></button>

// ❌ — fails because text is split across spans
screen.getByText(/hello world/i)

// ✅
screen.getByRole('button', { name: /hello world/i })
```

---

### 8. Adding `aria-*` and `role` attributes incorrectly

If you're reaching for ARIA on a native element, you almost always have the wrong native element.

```jsx
// ❌ — button already has role="button"
<button role="button">Click me</button>

// ✅
<button>Click me</button>
```

Note: `<input />` has no implicit role until you add `type` — write `<input type="text" />`.

Only add ARIA when you're building a custom widget that has no semantic HTML equivalent.

---

### 9. Using `fireEvent` instead of `@testing-library/user-event`

`fireEvent` fires one event. Real users fire sequences (`keyDown` → `keyPress` → `keyUp`, focus changes, hover, etc.). `userEvent` reproduces the sequence.

```js
// ❌
fireEvent.change(input, { target: { value: 'hello world' } })

// ✅
const user = userEvent.setup()
await user.type(input, 'hello world')
```

In v14+, always call `userEvent.setup()` once per test and `await` every interaction.

---

### 10. Using `query*` for anything except non-existence

`query*` returns `null` silently — you lose helpful "elements found in the DOM" error messages.

```js
// ❌ — if alert is missing, error is just "expected null to be in document"
expect(screen.queryByRole('alert')).toBeInTheDocument()

// ✅ — assert existence with get*, non-existence with query*
expect(screen.getByRole('alert')).toBeInTheDocument()
expect(screen.queryByRole('alert')).not.toBeInTheDocument()
```

Rule: `query*` exists for **one** purpose — proving an element is *not* there.

---

### 11. Wrapping `getBy*` in `waitFor` when you should use `findBy*`

`findBy*` = `waitFor` + `getBy*`, with better error messages and less code.

```js
// ❌
const submitButton = await waitFor(() =>
  screen.getByRole('button', { name: /submit/i })
)

// ✅
const submitButton = await screen.findByRole('button', { name: /submit/i })
```

---

### 12. Passing an empty callback to `waitFor`

`waitFor(() => {})` only waits for the next event-loop tick. Tests that pass on this are accidentally correct and will flake when async behavior shifts.

```js
// ❌
await waitFor(() => {})
expect(window.fetch).toHaveBeenCalledWith('foo')

// ✅
await waitFor(() => expect(window.fetch).toHaveBeenCalledWith('foo'))
```

---

### 13. Multiple assertions inside one `waitFor`

If the second assertion is the one that fails, you wait for the full timeout (default 1s) before learning that. Move stable assertions out of `waitFor`.

```js
// ❌
await waitFor(() => {
  expect(window.fetch).toHaveBeenCalledWith('foo')
  expect(window.fetch).toHaveBeenCalledTimes(1)
})

// ✅
await waitFor(() => expect(window.fetch).toHaveBeenCalledWith('foo'))
expect(window.fetch).toHaveBeenCalledTimes(1)
```

---

### 14. Performing side-effects inside `waitFor`

`waitFor` retries its callback. Side-effects (clicks, key events, mutations) get executed multiple times at non-deterministic intervals.

```js
// ❌
await waitFor(() => {
  fireEvent.keyDown(input, { key: 'ArrowDown' })
  expect(screen.getAllByRole('listitem')).toHaveLength(3)
})

// ✅
fireEvent.keyDown(input, { key: 'ArrowDown' })
await waitFor(() => {
  expect(screen.getAllByRole('listitem')).toHaveLength(3)
})
```

The `waitFor` callback should contain **only** assertions.

---

### 15. Using `getBy*` as an implicit assertion

`screen.getByRole('alert')` does throw if the alert is missing — but a reader can't tell whether the line is intentional or leftover. Spell out the assertion.

```js
// ❌
screen.getByRole('alert', { name: /error/i })

// ✅
expect(screen.getByRole('alert', { name: /error/i })).toBeInTheDocument()
```

## Reviewer Checklist

When reviewing or writing an RTL test, walk this list:

- [ ] No `wrapper`; no `cleanup()`; queries use `screen`
- [ ] `*ByRole` with accessible name is the default query type
- [ ] No `container.querySelector`, no `getByTestId` unless nothing else works
- [ ] `userEvent` (not `fireEvent`) for interactions; every interaction `await`-ed
- [ ] `getBy*` for existence, `queryBy*` only for non-existence, `findBy*` for async
- [ ] No manual `act()` wrapping `render` / `fireEvent` / `userEvent`
- [ ] `waitFor` callbacks have one assertion, no side-effects
- [ ] Assertions use jest-dom matchers, not raw DOM property reads
- [ ] No drive-by ARIA on native elements

## Source

Kent C. Dodds, *Common Mistakes with React Testing Library* — https://kentcdodds.com/blog/common-mistakes-with-react-testing-library
