# 09 — HTTP & Resources 🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 9
>
> Goal: Fetch data the Angular way — `HttpClient` setup (Fetch by default in v22), typed requests, functional interceptors (auth, errors, retries, loading, caching), error handling, and the now-stable signal-based `resource`, `rxResource` and `httpResource`.

---

# 1. Setup ⭐

```ts
// app.config.ts
import { provideHttpClient, withInterceptors, withXsrfConfiguration } from "@angular/common/http";

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, errorInterceptor]),
      // withXhr(),              // v22: opt back into XMLHttpRequest (Fetch is default)
    ),
  ],
};
```

🆕 **v22:** `HttpClient` uses the **Fetch API by default**. `withFetch()` is deprecated (no-op); `withXhr()` restores XHR (e.g. for upload progress in older setups). `reportProgress` is deprecated in favour of `reportUploadProgress` / `reportDownloadProgress`.

Legacy NgModule apps: `HttpClientModule` (deprecated) → `provideHttpClient()`.

---

# 2. Making Requests ⭐

```ts
export interface Product { id: string; name: string; price: number }

@Service()
export class ProductApi {
  private readonly http = inject(HttpClient);
  private readonly base = "/api/products";

  list(params: { page: number; q?: string }) {
    return this.http.get<Product[]>(this.base, {
      params: new HttpParams({ fromObject: { page: params.page, ...(params.q ? { q: params.q } : {}) } }),
    });
  }

  get(id: string) {
    return this.http.get<Product>(`${this.base}/${id}`);
  }

  create(body: Omit<Product, "id">) {
    return this.http.post<Product>(this.base, body);
  }

  update(id: string, body: Partial<Product>) {
    return this.http.patch<Product>(`${this.base}/${id}`, body);
  }

  remove(id: string) {
    return this.http.delete<void>(`${this.base}/${id}`);
  }

  // full response (headers, status)
  listWithTotal() {
    return this.http.get<Product[]>(this.base, { observe: "response" }).pipe(
      map((res) => ({ items: res.body ?? [], total: Number(res.headers.get("X-Total-Count")) })),
    );
  }
}
```

Key facts:
- Returns **cold observables** — **nothing is sent until subscribed**; each subscribe sends a new request.
- Completes after one response → no manual unsubscribe needed (but unsubscribing **cancels** the request).
- `get<T>` is a type **assertion**, not runtime validation (validate with zod etc. if needed).
- Options: `headers`, `params`, `responseType: 'blob' | 'text' | 'arraybuffer'`, `observe: 'events'`, `withCredentials`, `context` (HttpContext tokens for interceptors).

---

# 3. Error Handling ⭐

```ts
this.api.get(id).pipe(
  retry({ count: 2, delay: (_, attempt) => timer(attempt * 500) }),  // backoff
  catchError((err: HttpErrorResponse) => {
    if (err.status === 0) console.error("Network error", err.error);
    else console.error(`Backend ${err.status}`, err.error);
    return throwError(() => new Error("Could not load product"));
  }),
);
```

`HttpErrorResponse` has `status`, `statusText`, `error` (body), `url`. Status `0` = network/CORS failure.

---

# 4. Functional Interceptors ⭐⭐⭐

Interceptors are middleware for every request/response: add headers, handle errors globally, log, cache, show loaders.

```ts
import { HttpInterceptorFn, HttpErrorResponse } from "@angular/common/http";

// 1. Auth header
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthStore).token();
  if (!token || !req.url.startsWith("/api")) return next(req);
  return next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }));  // requests are immutable → clone
};

// 2. Global errors (401 → login)
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  const toast = inject(ToastService);
  return next(req).pipe(
    catchError((err: HttpErrorResponse) => {
      if (err.status === 401) router.navigate(["/login"]);
      else if (err.status >= 500) toast.error("Server error, please retry");
      return throwError(() => err);
    }),
  );
};

// 3. Loading indicator
export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  const loader = inject(LoadingService);
  loader.start();
  return next(req).pipe(finalize(() => loader.stop()));
};

// 4. Per-request opt-out via HttpContext
export const SKIP_LOADER = new HttpContextToken<boolean>(() => false);
// this.http.get(url, { context: new HttpContext().set(SKIP_LOADER, true) })
// in interceptor: if (req.context.get(SKIP_LOADER)) return next(req);
```

- Order matters: requests flow through interceptors **in the order listed**, responses in **reverse**.
- `HttpRequest` / `HttpHeaders` are **immutable** → `req.clone({...})`.
- Class-based interceptors (`HttpInterceptor` + `HTTP_INTERCEPTORS` multi provider + `withInterceptorsFromDi()`) are legacy.

### Token refresh (common question)

```ts
export const refreshInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  return next(req).pipe(
    catchError((err: HttpErrorResponse) => {
      if (err.status !== 401 || req.url.includes("/auth/refresh")) return throwError(() => err);
      return auth.refresh().pipe(          // refresh() should be shared (shareReplay) so parallel 401s refresh once
        switchMap((token) => next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }))),
      );
    }),
  );
};
```

---

# 5. Resources — Signal-Based Async Data ⭐⭐⭐ (stable in v22)

Resources bring async data into the signal graph: they **react to signal params**, expose `value`, `status`, `isLoading`, `error`, and **cancel stale requests**.

## 5.1 `httpResource` (most common)

```ts
import { httpResource } from "@angular/common/http";

@Component({
  selector: "app-user-profile",
  template: `
    @if (user.isLoading()) {
      <app-spinner />
    } @else if (user.error()) {
      <p>Failed to load. <button (click)="user.reload()">Retry</button></p>
    } @else if (user.hasValue()) {
      <h2>{{ user.value().name }}</h2>
    }
  `,
})
export class UserProfile {
  readonly id = input.required<string>();

  // re-fetches whenever id() changes; previous request is aborted
  protected readonly user = httpResource<User>(() => `/api/users/${this.id()}`);
}
```

Request object form + parsing:

```ts
protected readonly products = httpResource<Product[]>(
  () => ({
    url: "/api/products",
    params: { q: this.query(), page: this.page() },
    headers: { "X-Client": "web" },
  }),
  { defaultValue: [], parse: (raw) => productListSchema.parse(raw) },  // e.g. zod validation
);
```

Return `undefined` from the request function to **skip** the request (status `idle`):

```ts
protected readonly details = httpResource<Details>(() => (this.selectedId() ? `/api/details/${this.selectedId()}` : undefined));
```

`httpResource` goes through `HttpClient`, so **interceptors apply**. Variants: `httpResource.text()`, `.blob()`, `.arrayBuffer()`.

## 5.2 `resource` — any Promise-based loader

```ts
import { resource } from "@angular/core";

protected readonly weather = resource({
  params: () => ({ city: this.city() }),                   // reactive params
  loader: async ({ params, abortSignal }) => {
    const res = await fetch(`/api/weather?city=${params.city}`, { signal: abortSignal });
    if (!res.ok) throw new Error("Failed");
    return (await res.json()) as Weather;
  },
});
```

## 5.3 `rxResource` — Observable-based loader

```ts
import { rxResource } from "@angular/core/rxjs-interop";

protected readonly orders = rxResource({
  params: () => this.customerId(),
  stream: ({ params }) => this.orderApi.byCustomer(params),
});
```

## 5.4 Resource API

| Member | Meaning |
| --- | --- |
| `value()` | Current value (throws if in error state — guard with `hasValue()`) |
| `hasValue()` | Type guard: value available |
| `status()` | `'idle' \| 'loading' \| 'reloading' \| 'resolved' \| 'error' \| 'local'` |
| `isLoading()` | Loading or reloading |
| `error()` | Error, if any |
| `reload()` | Re-run loader with same params |
| `set()` / `update()` | Local override (status `local`) — optimistic UI |

## 5.5 v22 additions 🆕

- **Stable** `resource`, `rxResource`, `httpResource`.
- **`chain()`** — dependent resources: read another resource's value inside params; if the parent isn't ready, a `ResourceDependencyError` keeps the child pending instead of firing a bad request.
- **SSR caching**: give a resource an `id` so the server result is transferred and reused during hydration (no double fetch).
- `debounced()` returns a resource for debounced signal values (experimental).

## 5.6 When to use what ⭐

| Need | Use |
| --- | --- |
| Read data driven by signal params, show loading/error | **`httpResource`** |
| Non-HttpClient async (fetch, IndexedDB, SDK) | `resource` |
| Existing observable APIs/RxJS pipelines | `rxResource` or `toSignal` |
| Mutations (POST/PUT/DELETE) | **`HttpClient` methods** — resources are for **reading** |
| Complex streams (websockets, polling, debouncing chains) | RxJS |

---

# 6. Caching Patterns

```ts
@Service()
export class CategoryApi {
  private readonly http = inject(HttpClient);
  readonly categories$ = this.http.get<Category[]>("/api/categories").pipe(shareReplay(1));  // fetched once
}
```

Interceptor cache (GET only, Map<url, HttpResponse>), SSR `withHttpTransferCache` (on by default with `provideClientHydration`), or libraries like TanStack Query for Angular.

---

# 7. Testing HTTP

```ts
TestBed.configureTestingModule({
  providers: [provideHttpClient(), provideHttpClientTesting()],
});
const api = TestBed.inject(ProductApi);
const httpMock = TestBed.inject(HttpTestingController);

api.get("1").subscribe((p) => expect(p.name).toBe("Pen"));
const req = httpMock.expectOne("/api/products/1");
expect(req.request.method).toBe("GET");
req.flush({ id: "1", name: "Pen", price: 10 });
httpMock.verify();       // no unexpected requests
```

---

# 8. Interview Answers

**How do you make HTTP calls in Angular?**

> "With `HttpClient`, provided via `provideHttpClient()`. Methods return cold, typed observables that complete after one response — nothing is sent until you subscribe, and unsubscribing cancels the request. I wrap calls in a service, handle errors with `catchError` and `retry`, and use functional interceptors for cross-cutting concerns. In v22 HttpClient uses the Fetch API by default. For reading data in components I increasingly use `httpResource`, which is signal-based and stable in v22."

**What are interceptors? Give examples.**

> "Functions that sit in the HTTP pipeline and can inspect or transform every request and response. Typical uses: attaching auth tokens, refreshing expired tokens on 401, global error handling and redirects, loading indicators, logging, caching and adding headers. Requests are immutable so you clone them. They run in the order provided for requests and reverse for responses."

**What is `httpResource` / `resource`?**

> "Signal-based async primitives. You give them a reactive request or params function and they fetch whenever those signals change, cancel stale requests, and expose `value`, `isLoading`, `error`, `status` and `reload` as signals. `httpResource` goes through HttpClient so interceptors apply; `resource` takes any promise loader and `rxResource` an observable. They're for reading data — mutations still use HttpClient methods."

**How do you prevent duplicate HTTP requests?**

> "Remember HttpClient observables are cold: each subscription sends a request. I subscribe once — via a single async pipe, `toSignal` or a resource — or share with `shareReplay`. For in-flight cancellation use `switchMap`, and for double-submit prevention `exhaustMap` or disabling the button. With SSR, the HTTP transfer cache avoids refetching on hydration."

---

# 9. Key Takeaways

- `provideHttpClient(withInterceptors([...]))`; Fetch default in v22, `withXhr()` to opt out.
- HttpClient observables are cold, complete once, cancel on unsubscribe.
- Functional interceptors: clone requests, order matters.
- `httpResource` / `resource` / `rxResource` stable in v22 for signal-driven reads; HttpClient for mutations.
- Test with `provideHttpClientTesting` + `HttpTestingController`.

## Readiness Checklist

- [ ] Write auth, error and loading interceptors
- [ ] Load data with `httpResource` driven by an input
- [ ] Explain resource status values and `hasValue()`
- [ ] Implement token refresh with a shared refresh call
- [ ] Test a service with HttpTestingController
