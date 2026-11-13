# 25 — Testing React

> **Interview + Understanding Guide** — Roadmap Section 25
>
> Goal: Know the testing pyramid/trophy, write behaviour-focused tests with React Testing Library + Vitest/Jest, mock network calls with MSW, test hooks, async UI, context/providers, and know where E2E (Playwright/Cypress) and visual tests fit.

---

# 1. What to Test & Why

### Testing Trophy (Kent C. Dodds)

```text
        /\        E2E            — few: critical user journeys (login, checkout)
       /  \
      /    \      Integration    — MOST: components working together with real-ish data
     /      \
    /________\    Unit           — pure functions, hooks, reducers, utils
    ‾‾‾‾‾‾‾‾‾‾    Static         — TypeScript, ESLint (catch typos/types for free)
```

> **"The more your tests resemble the way your software is used, the more confidence they can give you."**

Test **behaviour**, not **implementation details**:
- ✅ "When I click Add, the item appears in the list"
- ❌ "setItems was called with …", "state.count === 1", "component has class X"

Implementation-detail tests break on refactors and miss real bugs.

---

# 2. Tooling

| Tool | Role |
| --- | --- |
| **Vitest** / Jest | Test runner, assertions, mocks, fake timers (Vitest = fast, Vite-native, Jest-compatible API) |
| **React Testing Library (RTL)** | Render components and query the DOM like a user |
| **@testing-library/user-event** | Realistic user interactions (typing, clicking, tab) |
| **@testing-library/jest-dom** | DOM matchers (`toBeInTheDocument`, `toBeDisabled`, `toHaveValue`) |
| **jsdom** / happy-dom | Simulated browser environment |
| **MSW** (Mock Service Worker) | Mock network at the request level |
| **Playwright** / Cypress | End-to-end tests in real browsers |
| **Storybook** + Chromatic | Component development, visual regression, interaction tests |
| **axe** (jest-axe, @axe-core/playwright) | Automated accessibility checks |

---

# 3. React Testing Library Basics ⭐

```jsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Counter } from "./Counter";

test("increments when clicking the button", async () => {
  const user = userEvent.setup();
  render(<Counter />);

  expect(screen.getByText("Count: 0")).toBeInTheDocument();
  await user.click(screen.getByRole("button", { name: /increment/i }));
  expect(screen.getByText("Count: 1")).toBeInTheDocument();
});
```

### Query priority ⭐ (use the most accessible query available)

1. **`getByRole`** (with `name`) — buttons, links, headings, textboxes, checkboxes… ✅ best
2. `getByLabelText` — form fields
3. `getByPlaceholderText`
4. `getByText` — non-interactive text
5. `getByDisplayValue`
6. `getByAltText`, `getByTitle`
7. `getByTestId` — last resort (`data-testid`)

If you can't find an element by role/label, it's often an **accessibility bug**.

### Query variants

| Variant | No match | Multiple matches | Async? | Use |
| --- | --- | --- | --- | --- |
| `getBy…` | throws | throws | no | Element should be there now |
| `queryBy…` | returns `null` | throws | no | Assert something is **not** present |
| `findBy…` | rejects (after timeout) | rejects | ✅ returns Promise | Element appears **asynchronously** |
| `getAllBy` / `queryAllBy` / `findAllBy` | — | arrays | | Multiple elements |

```jsx
expect(screen.queryByRole("alert")).not.toBeInTheDocument();
expect(await screen.findByText("Welcome, Asha")).toBeInTheDocument();
```

### `userEvent` vs `fireEvent`

`userEvent` simulates full interactions (focus, keydown, keypress, input, keyup, click sequence); `fireEvent` dispatches a single DOM event. Prefer **userEvent**.

```jsx
await user.type(screen.getByLabelText(/email/i), "a@b.com");
await user.selectOptions(screen.getByRole("combobox"), "admin");
await user.keyboard("{Enter}");
await user.tab();
await user.upload(fileInput, new File(["x"], "a.png", { type: "image/png" }));
```

### Scoping

```jsx
import { within } from "@testing-library/react";
const row = screen.getByRole("row", { name: /asha/i });
await user.click(within(row).getByRole("button", { name: /delete/i }));
```

---

# 4. Testing Forms

```jsx
test("shows validation errors and submits valid data", async () => {
  const user = userEvent.setup();
  const onSubmit = vi.fn();
  render(<LoginForm onSubmit={onSubmit} />);

  await user.click(screen.getByRole("button", { name: /log in/i }));
  expect(await screen.findByText(/email is required/i)).toBeInTheDocument();
  expect(onSubmit).not.toHaveBeenCalled();

  await user.type(screen.getByLabelText(/email/i), "asha@example.com");
  await user.type(screen.getByLabelText(/password/i), "supersecret");
  await user.click(screen.getByRole("button", { name: /log in/i }));

  expect(onSubmit).toHaveBeenCalledWith({ email: "asha@example.com", password: "supersecret" });
});
```

---

# 5. Async UI & Network with MSW ⭐

Mock the **network**, not your fetch functions — components run their real data code.

```js
// mocks/handlers.js
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/users/:id", ({ params }) =>
    HttpResponse.json({ id: params.id, name: "Asha" }),
  ),
];

// test/setup.js
import { setupServer } from "msw/node";
export const server = setupServer(...handlers);
beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

```jsx
test("loads and shows the user", async () => {
  renderWithProviders(<UserProfile userId="1" />);
  expect(screen.getByText(/loading/i)).toBeInTheDocument();
  expect(await screen.findByRole("heading", { name: "Asha" })).toBeInTheDocument();
});

test("shows an error when the request fails", async () => {
  server.use(http.get("/api/users/:id", () => new HttpResponse(null, { status: 500 })));
  renderWithProviders(<UserProfile userId="1" />);
  expect(await screen.findByRole("alert")).toHaveTextContent(/something went wrong/i);
});
```

MSW handlers can also power Storybook and local development.

### `waitFor`

```jsx
await waitFor(() => expect(mockSave).toHaveBeenCalledTimes(1));
```

Prefer `findBy*` for elements; use `waitFor` for non-DOM assertions. Don't put side effects inside `waitFor`.

---

# 6. Testing With Providers

```jsx
// test/utils.jsx
export function renderWithProviders(ui, { route = "/", user = null } = {}) {
  const queryClient = new QueryClient({ defaultOptions: { queries: { retry: false } } }); // fresh per test
  return render(
    <QueryClientProvider client={queryClient}>
      <AuthContext.Provider value={{ user, login: vi.fn(), logout: vi.fn() }}>
        <MemoryRouter initialEntries={[route]}>{ui}</MemoryRouter>
      </AuthContext.Provider>
    </QueryClientProvider>,
  );
}
```

For Redux: create a **real store** per test with `configureStore` and optional `preloadedState` — don't mock `useSelector`.

---

# 7. Testing Custom Hooks

```jsx
import { renderHook, act } from "@testing-library/react";

test("useCounter", () => {
  const { result } = renderHook(() => useCounter(5));
  act(() => result.current.increment());
  expect(result.current.count).toBe(6);
});

test("hook with providers", () => {
  const wrapper = ({ children }) => <QueryClientProvider client={new QueryClient()}>{children}</QueryClientProvider>;
  const { result } = renderHook(() => useUser("1"), { wrapper });
});
```

Prefer testing hooks **through the components that use them** unless the hook is a reusable library piece.

---

# 8. Timers

```jsx
test("debounced search", async () => {
  vi.useFakeTimers({ shouldAdvanceTime: true });
  const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
  render(<Search />);

  await user.type(screen.getByRole("searchbox"), "react");
  expect(fetchSpy).not.toHaveBeenCalled();

  await act(() => vi.advanceTimersByTimeAsync(300));
  expect(fetchSpy).toHaveBeenCalledWith("react");
  vi.useRealTimers();
});
```

---

# 9. Mocking

```js
vi.mock("./analytics", () => ({ track: vi.fn() }));          // module mock
const spy = vi.spyOn(window, "confirm").mockReturnValue(true); // spy
vi.fn().mockResolvedValue({ ok: true });                        // async mock
```

Mock at the **edges** (network, time, randomness, browser APIs like `IntersectionObserver`, `matchMedia` which jsdom lacks). Don't mock child components or React internals unless necessary.

---

# 10. `act()`

React needs updates wrapped in `act()` so effects and state updates flush before assertions. RTL's `render`, `userEvent`, and `findBy`/`waitFor` already wrap in `act`. The warning "An update … was not wrapped in act(...)" usually means an **async update happened after the test finished** → await the UI change (`findBy`) or the pending promise.

(React 19: import `act` from `react`, not `react-dom/test-utils`.)

---

# 11. Snapshot Tests

```jsx
expect(container).toMatchSnapshot();
```

Easy to write, easy to rubber-stamp. Use **sparingly** for small stable output (serialized config, error messages). Prefer explicit assertions; use visual regression tools for UI appearance.

---

# 12. E2E Testing (Playwright)

```ts
import { test, expect } from "@playwright/test";

test("user can check out", async ({ page }) => {
  await page.goto("/");
  await page.getByRole("link", { name: "Shoes" }).click();
  await page.getByRole("button", { name: "Add to cart" }).first().click();
  await page.getByRole("link", { name: /cart \(1\)/i }).click();
  await page.getByRole("button", { name: "Checkout" }).click();
  await expect(page.getByRole("heading", { name: "Order confirmed" })).toBeVisible();
});
```

- Real browsers (Chromium, Firefox, WebKit), auto-waiting, parallel, trace viewer, network mocking.
- Keep E2E for **critical paths**; run in CI against preview deployments.
- Cypress: similar goals, runs in-browser, great DX; Playwright is faster/multi-browser and now very popular.

---

# 13. Testing Strategy for a Real App (interview answer)

1. **Static**: TypeScript strict, ESLint (incl. jsx-a11y, react-hooks).
2. **Unit**: utils, reducers, selectors, complex hooks, validation schemas.
3. **Integration (most)**: features rendered with providers + MSW — forms, lists, error states, permissions.
4. **E2E**: 5–20 critical journeys (auth, checkout, onboarding).
5. **Visual/a11y**: Storybook + Chromatic, axe checks.
6. **CI**: run on every PR; coverage as a signal, not a target (~70–80% meaningful coverage).

---

# 14. Interview Answers

**How do you test React components?**

> "With React Testing Library and Vitest or Jest, testing behaviour the way a user would: render the component, find elements by role and accessible name, interact with `userEvent`, and assert on what's visible. I mock the network with MSW instead of mocking fetch functions, render with real providers through a custom render helper, and keep E2E tests in Playwright for critical flows."

**getBy vs queryBy vs findBy?**

> "`getBy` throws if the element isn't there — for things that should exist now. `queryBy` returns null — for asserting absence. `findBy` returns a promise and retries until the element appears — for async UI."

**What does 'testing implementation details' mean?**

> "Testing internals the user can't see — state variables, which functions were called internally, component instance methods, CSS class names. Those tests break on refactors without catching real bugs. Testing observable behaviour gives more confidence."

---

# 15. Key Takeaways

- Testing trophy: most value in integration tests; behaviour over implementation.
- RTL queries: role > label > text > testId; get/query/find semantics.
- `userEvent.setup()` for interactions; `findBy` for async.
- MSW mocks the network; custom render with providers; fresh QueryClient per test.
- `renderHook` for reusable hooks; fake timers for debounce/throttle.
- Snapshots sparingly; Playwright for critical E2E; axe for a11y.

## Readiness Checklist

- [ ] Write a form test with validation & submission
- [ ] Set up MSW and test loading/success/error
- [ ] Write a `renderWithProviders` helper
- [ ] Test a debounced input with fake timers
- [ ] Write one Playwright E2E for login
- [ ] Explain your testing strategy for a production app
