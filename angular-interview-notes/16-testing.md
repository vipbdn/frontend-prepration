# 16 — Testing 🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 16
>
> Goal: Test Angular apps confidently — Vitest (default since v21), TestBed, component tests with signals, services, HTTP, routing, forms, component harnesses, async utilities, and E2E with Playwright/Cypress.

---

# 1. Testing Landscape ⭐

| Layer | Tooling | What |
| --- | --- | --- |
| Unit | **Vitest** (default runner since v21), Jasmine/Karma (legacy), Jest | Pure functions, services, pipes, signals |
| Component / integration | **TestBed** + Vitest, Angular Testing Library, CDK harnesses | Rendering, inputs/outputs, DOM interaction |
| E2E | **Playwright**, Cypress | Real browser user flows |

Testing pyramid: many unit tests, a solid layer of component/integration tests, a few E2E for critical flows. Test **behaviour, not implementation**.

```bash
ng test                       # Vitest via @angular/build:unit-test
ng g @schematics/angular:migrate-karma-to-vitest   # v22 migration helper (Karma → Vitest)
```

🆕 v22: `zone.js/plugins/vitest-patch` lets `fakeAsync` work with Vitest in zone-based apps; `TestBed.getLastFixture()` returns the most recently created fixture.

---

# 2. Testing Pure Logic & Signals

```ts
import { describe, it, expect } from "vitest";

describe("TruncatePipe", () => {
  const pipe = new TruncatePipe();
  it("truncates long text", () => expect(pipe.transform("hello world", 5)).toBe("hello…"));
  it("handles null", () => expect(pipe.transform(null)).toBe(""));
});
```

```ts
describe("CartStore", () => {
  let store: CartStore;
  beforeEach(() => {
    TestBed.configureTestingModule({});
    store = TestBed.inject(CartStore);
  });

  it("computes totals", () => {
    store.add({ id: "1", name: "Pen", price: 10 });
    store.add({ id: "1", name: "Pen", price: 10 });
    expect(store.count()).toBe(2);
    expect(store.total()).toBe(20);
  });
});
```

Effects run during change detection; in tests call `TestBed.tick()` (v20+, formerly `flushEffects`) to flush them.

---

# 3. Component Tests with TestBed ⭐⭐

```ts
import { ComponentFixture, TestBed } from "@angular/core/testing";

describe("Counter", () => {
  let fixture: ComponentFixture<Counter>;
  let el: HTMLElement;

  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [Counter] }).compileComponents();  // standalone → imports
    fixture = TestBed.createComponent(Counter);
    el = fixture.nativeElement;
    await fixture.whenStable();          // zoneless-friendly: wait for CD
  });

  it("increments on click", async () => {
    el.querySelector("button")!.click();
    await fixture.whenStable();          // or fixture.detectChanges()
    expect(el.querySelector("span")!.textContent).toContain("1");
  });
});
```

### Inputs & outputs

```ts
it("renders user and emits select", async () => {
  const fixture = TestBed.createComponent(UserCard);
  fixture.componentRef.setInput("user", { id: 1, name: "Asha" });   // works for signal inputs
  await fixture.whenStable();
  expect(fixture.nativeElement.textContent).toContain("Asha");

  const spy = vi.fn();
  fixture.componentInstance.selected.subscribe(spy);                 // output() supports subscribe
  fixture.nativeElement.querySelector("button").click();
  expect(spy).toHaveBeenCalledWith(1);
});
```

### Mocking dependencies

```ts
await TestBed.configureTestingModule({
  imports: [ProductList],
  providers: [
    { provide: ProductApi, useValue: { list: vi.fn(() => of([{ id: "1", name: "Pen", price: 10 }])) } },
  ],
}).compileComponents();
```

Override child components: `TestBed.overrideComponent(Parent, { remove: { imports: [HeavyChild] }, add: { imports: [FakeChild] } })`.

---

# 4. Angular Testing Library (popular alternative)

```ts
import { render, screen } from "@testing-library/angular";
import userEvent from "@testing-library/user-event";

it("adds a todo", async () => {
  await render(TodoApp);
  await userEvent.type(screen.getByRole("textbox", { name: /new todo/i }), "Write tests{Enter}");
  expect(screen.getByText("Write tests")).toBeTruthy();
});
```

Queries by role/label → tests mirror how users interact, resilient to refactors.

---

# 5. Component Harnesses (CDK) ⭐

Harnesses provide a stable API for interacting with components (especially Angular Material) without depending on internal DOM:

```ts
import { TestbedHarnessEnvironment } from "@angular/cdk/testing/testbed";
import { MatButtonHarness } from "@angular/material/button/testing";
import { MatInputHarness } from "@angular/material/input/testing";

it("submits the form", async () => {
  const fixture = TestBed.createComponent(LoginForm);
  const loader = TestbedHarnessEnvironment.loader(fixture);

  const email = await loader.getHarness(MatInputHarness.with({ selector: "[formControlName=email]" }));
  await email.setValue("a@b.com");
  const submit = await loader.getHarness(MatButtonHarness.with({ text: "Login" }));
  await submit.click();
  expect(fixture.componentInstance.submitted()).toBe(true);
});
```

You can write custom harnesses (`ComponentHarness`) for your own components; same harness works in Playwright/Protractor environments too.

---

# 6. Testing Services with HTTP ⭐

```ts
describe("ProductApi", () => {
  let api: ProductApi;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting()] });
    api = TestBed.inject(ProductApi);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => http.verify());

  it("GETs a product", () => {
    let result: Product | undefined;
    api.get("1").subscribe((p) => (result = p));
    const req = http.expectOne("/api/products/1");
    expect(req.request.method).toBe("GET");
    req.flush({ id: "1", name: "Pen", price: 10 });
    expect(result?.name).toBe("Pen");
  });

  it("handles errors", () => {
    let error: unknown;
    api.get("x").subscribe({ error: (e) => (error = e) });
    http.expectOne("/api/products/x").flush("Not found", { status: 404, statusText: "Not Found" });
    expect(error).toBeTruthy();
  });
});
```

Testing an interceptor: provide `provideHttpClient(withInterceptors([authInterceptor]))` and assert `req.request.headers.get("Authorization")`.

`httpResource` tests: flush requests with `HttpTestingController`, then `await TestBed.inject(ApplicationRef).whenStable()` and assert `resource.value()`.

---

# 7. Async Testing

| Tool | Use |
| --- | --- |
| `async/await` + `fixture.whenStable()` | Default, works zoneless |
| `vi.useFakeTimers()` + `vi.advanceTimersByTime(ms)` | Debounce, timers (Vitest) |
| `fakeAsync` + `tick(ms)` + `flush()` | Zone-based fake time (needs zone; Vitest patch in v22) |
| `waitForAsync` | Legacy zone helper |

```ts
it("debounces search", async () => {
  vi.useFakeTimers();
  const fixture = TestBed.createComponent(UserSearch);
  const input: HTMLInputElement = fixture.nativeElement.querySelector("input");
  input.value = "an"; input.dispatchEvent(new Event("input"));
  vi.advanceTimersByTime(300);
  await fixture.whenStable();
  expect(apiMock.search).toHaveBeenCalledWith("an");
  vi.useRealTimers();
});
```

---

# 8. Testing Routing & Guards

```ts
import { RouterTestingHarness } from "@angular/router/testing";

it("navigates to product detail", async () => {
  TestBed.configureTestingModule({ providers: [provideRouter([{ path: "products/:id", component: ProductDetail }], withComponentInputBinding())] });
  const harness = await RouterTestingHarness.create();
  const component = await harness.navigateByUrl("/products/42", ProductDetail);
  expect(component.id()).toBe("42");
});

it("authGuard redirects when logged out", () => {
  TestBed.configureTestingModule({ providers: [{ provide: AuthStore, useValue: { isLoggedIn: () => false } }] });
  const result = TestBed.runInInjectionContext(() => authGuard({} as any, { url: "/settings" } as any));
  expect(result instanceof UrlTree).toBe(true);
});
```

---

# 9. E2E with Playwright

```ts
import { test, expect } from "@playwright/test";

test("checkout flow", async ({ page }) => {
  await page.goto("/products");
  await page.getByRole("button", { name: "Add to cart" }).first().click();
  await page.getByRole("link", { name: /cart/i }).click();
  await expect(page.getByText("Total")).toBeVisible();
});
```

`ng e2e` can be wired to Playwright/Cypress via community builders.

---

# 10. Interview Answers

**How do you test Angular components?**

> "With TestBed: I import the standalone component, provide mocks for its dependencies, create a fixture, set inputs with `componentRef.setInput`, interact through the DOM, wait with `whenStable` or `detectChanges`, and assert on rendered output and emitted outputs. I prefer user-centric queries — Angular Testing Library or CDK harnesses for Material — so tests survive refactors. Since v21 the default runner is Vitest."

**How do you test HTTP code?**

> "Provide `provideHttpClient()` and `provideHttpClientTesting()`, inject `HttpTestingController`, call the service, `expectOne` the URL, assert method and body, `flush` a response or an error, and call `verify()` afterwards to ensure no unexpected requests."

**What are component harnesses?**

> "A CDK testing API that wraps a component's DOM behind methods like `setValue` or `click`, so tests don't depend on internal markup. Angular Material ships harnesses for every component, and the same harness works in unit and E2E environments."

**How do you test async code like debounce?**

> "With fake timers — Vitest's `useFakeTimers` and `advanceTimersByTime`, or `fakeAsync` with `tick` in zone-based setups — then wait for stability and assert. For promises and HTTP, `await fixture.whenStable()`."

---

# 11. Key Takeaways

- Vitest default (v21+); migration schematic in v22.
- TestBed + `setInput` + `whenStable`; mock providers.
- Harnesses / Testing Library for resilient tests.
- `HttpTestingController` for HTTP; `RouterTestingHarness` for routes; `runInInjectionContext` for guards.
- Fake timers for debounce; Playwright for E2E.

## Readiness Checklist

- [ ] Test a component with signal inputs & outputs
- [ ] Test a service + interceptor with HttpTestingController
- [ ] Test a debounce with fake timers
- [ ] Use a Material harness
- [ ] Write one Playwright flow
