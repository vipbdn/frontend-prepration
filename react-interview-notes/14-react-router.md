# 14 — React Router

> **Interview + Understanding Guide** — Roadmap Section 14
>
> Goal: Understand client-side routing, and use React Router (v6.4+/v7 data APIs) for nested routes, layouts, params, search params, navigation, protected routes, loaders/actions, error handling and lazy loading.

---

# 1. Client-Side Routing

In an SPA, navigating **doesn't reload the page**. The router:
1. Intercepts link clicks (`preventDefault`)
2. Updates the URL with the **History API** (`history.pushState`)
3. Renders the components matching the new URL
4. Listens to `popstate` for back/forward

```text
Server routing: click → HTTP request → full HTML page
Client routing: click → pushState → React renders new route (no reload)
```

Server must return `index.html` for all app routes (SPA fallback), otherwise refreshing `/users/5` gives a 404.

### `BrowserRouter` vs `HashRouter`

| BrowserRouter | HashRouter |
| --- | --- |
| Clean URLs `/users/5` | `/#/users/5` |
| Needs server fallback config | Works on any static host |
| SEO-friendly | Hash not sent to server |

> React Router v7 merged with Remix: it can be used as a **library** (below) or a **framework** (SSR, file routes, type-safe loaders). Same core APIs.

---

# 2. Data Router Setup (recommended, v6.4+)

```jsx
import { createBrowserRouter, RouterProvider } from "react-router-dom"; // v7: from "react-router"

const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,
    errorElement: <RootError />,
    children: [
      { index: true, element: <Home /> },                     // "/"
      { path: "about", element: <About /> },                   // "/about"
      {
        path: "users",
        element: <UsersLayout />,
        children: [
          { index: true, element: <UsersList />, loader: usersLoader },
          { path: ":userId", element: <UserDetail />, loader: userLoader, action: userAction },
        ],
      },
      {
        path: "admin",
        lazy: () => import("./routes/admin"), // code-split route module
      },
      { path: "*", element: <NotFound /> },
    ],
  },
]);

createRoot(root).render(<RouterProvider router={router} />);
```

### Declarative (component) style — still common

```jsx
<BrowserRouter>
  <Routes>
    <Route path="/" element={<RootLayout />}>
      <Route index element={<Home />} />
      <Route path="users/:userId" element={<UserDetail />} />
      <Route path="*" element={<NotFound />} />
    </Route>
  </Routes>
</BrowserRouter>
```

(Loaders/actions require the data router.)

---

# 3. Nested Routes & Layouts ⭐

```jsx
function RootLayout() {
  return (
    <>
      <Navbar />
      <main>
        <Outlet /> {/* child route renders here */}
      </main>
      <Footer />
    </>
  );
}
```

- Child paths are **relative** to the parent.
- `index` route renders at the parent's exact path.
- Layout routes without `path` group routes under a shared UI (e.g. auth layout).
- Nested routes → nested UI → layouts don't remount when switching children.

---

# 4. Links & Navigation

```jsx
import { Link, NavLink, useNavigate, Navigate } from "react-router-dom";

<Link to="/users/5">Profile</Link>
<Link to="edit">Edit</Link>                    {/* relative */}
<Link to="/search?q=react" state={{ from: "home" }}>Search</Link>

<NavLink to="/inbox" className={({ isActive }) => (isActive ? "active" : "")}>Inbox</NavLink>

function SaveButton() {
  const navigate = useNavigate();
  const save = async () => {
    await api.save();
    navigate("/dashboard", { replace: true }); // replace: no history entry
    // navigate(-1) → back
  };
}

// Declarative redirect during render
if (!user) return <Navigate to="/login" replace />;
```

Use `<Link>` (renders `<a>`) for navigation — accessible, supports open-in-new-tab. Use `navigate()` after actions (submit, login).

---

# 5. Params, Search Params, Location ⭐

```jsx
import { useParams, useSearchParams, useLocation } from "react-router-dom";

// Route: /users/:userId/posts/:postId
function Post() {
  const { userId, postId } = useParams(); // always strings
}

// /products?category=books&page=2
function Products() {
  const [searchParams, setSearchParams] = useSearchParams();
  const category = searchParams.get("category") ?? "all";
  const page = Number(searchParams.get("page") ?? 1);

  const goToPage = (p) =>
    setSearchParams((prev) => {
      prev.set("page", String(p));
      return prev;
    });
}

function Analytics() {
  const location = useLocation(); // { pathname, search, hash, state, key }
  useEffect(() => trackPageView(location.pathname), [location.pathname]);
}
```

**Put filters, sorting, pagination and tabs in search params** — shareable, bookmarkable, survives refresh, back button works.

---

# 6. Protected Routes ⭐

```jsx
function RequireAuth({ children }) {
  const { user, isLoading } = useAuth();
  const location = useLocation();

  if (isLoading) return <FullPageSpinner />;
  if (!user) {
    return <Navigate to="/login" replace state={{ from: location }} />; // remember where they were going
  }
  return children;
}

// Layout-route version
function ProtectedLayout() {
  const { user } = useAuth();
  return user ? <Outlet /> : <Navigate to="/login" replace />;
}

// Router
{ element: <ProtectedLayout />, children: [{ path: "dashboard", element: <Dashboard /> }] }

// After login
const from = location.state?.from?.pathname ?? "/dashboard";
navigate(from, { replace: true });
```

### Role-based

```jsx
function RequireRole({ role, children }) {
  const { user } = useAuth();
  if (!user) return <Navigate to="/login" replace />;
  if (!user.roles.includes(role)) return <Navigate to="/403" replace />;
  return children;
}
```

⚠️ Client-side guards are **UX only** — the API must enforce authorization.

### Loader-based protection (data router)

```js
export async function dashboardLoader() {
  const user = await getCurrentUser();
  if (!user) throw redirect("/login");
  return getDashboardData(user.id);
}
```

Runs **before** rendering → no flash of protected content.

---

# 7. Loaders & Actions (data APIs) ⭐

### Loaders — fetch before render

```jsx
export async function userLoader({ params, request }) {
  const res = await fetch(`/api/users/${params.userId}`, { signal: request.signal });
  if (res.status === 404) throw new Response("Not Found", { status: 404 });
  return res.json();
}

function UserDetail() {
  const user = useLoaderData();
  return <h1>{user.name}</h1>;
}
```

Benefits: fetching starts **in parallel with code loading**, no render→fetch waterfalls, request cancellation built in, errors handled by `errorElement`.

### Actions — handle mutations (forms)

```jsx
export async function userAction({ request, params }) {
  const formData = await request.formData();
  const errors = validate(formData);
  if (errors) return { errors };                   // read with useActionData
  await api.updateUser(params.userId, Object.fromEntries(formData));
  return redirect(`/users/${params.userId}`);       // loaders revalidate automatically
}

function EditUser() {
  const actionData = useActionData();
  const navigation = useNavigation();               // { state: "idle" | "loading" | "submitting" }
  return (
    <Form method="post">
      <input name="name" />
      {actionData?.errors?.name && <p>{actionData.errors.name}</p>}
      <button disabled={navigation.state === "submitting"}>Save</button>
    </Form>
  );
}
```

- `<Form>` submits to the route action without a page reload.
- After an action, **all loaders revalidate** → UI stays in sync.
- `useFetcher()` for mutations **without navigation** (like buttons, inline edits).
- `defer` / streaming (v6) or returning promises (v7) + `<Await>` for slow data.

### Loaders + TanStack Query

Use loaders to **prefetch into the query cache**, and `useQuery` in components for caching/background refresh:

```js
export const loader = (queryClient) => async ({ params }) =>
  queryClient.ensureQueryData({ queryKey: ["user", params.userId], queryFn: () => fetchUser(params.userId) });
```

---

# 8. Error Handling

```jsx
import { useRouteError, isRouteErrorResponse } from "react-router-dom";

function RootError() {
  const error = useRouteError();
  if (isRouteErrorResponse(error)) {
    return <h1>{error.status} {error.statusText}</h1>; // thrown Response (404 etc.)
  }
  return <h1>Something went wrong: {error.message}</h1>;
}
```

`errorElement` catches errors from loaders, actions, and rendering in that route subtree — nested routes can have their own, so one broken widget doesn't break the whole app.

---

# 9. Lazy Loading Routes ⭐

```jsx
// Data router: lazy route module (component + loader + action)
{ path: "admin", lazy: () => import("./routes/admin") }
// admin.jsx exports: Component, loader, action, ErrorBoundary

// Classic: React.lazy + Suspense
const Settings = lazy(() => import("./pages/Settings"));
<Route path="settings" element={<Suspense fallback={<Spinner />}><Settings /></Suspense>} />
```

Route-based splitting is the highest-impact code splitting.

---

# 10. Other Features

| Feature | API |
| --- | --- |
| Scroll restoration | `<ScrollRestoration />` |
| Block navigation (unsaved changes) | `useBlocker` |
| Pending UI | `useNavigation().state` |
| Route matching info | `useMatch`, `useMatches` (breadcrumbs via `handle`) |
| Relative routing | `to=".."` |
| Optimistic UI | `fetcher.formData` |

---

# 11. Interview Answers

**How does client-side routing work?**

> "The router intercepts link clicks, updates the URL with the History API instead of requesting a new page, and renders the components matching that URL. It listens to `popstate` for back and forward. The server must serve `index.html` for all routes so deep links and refreshes work."

**How do you implement protected routes?**

> "A guard component or layout route checks auth state — rendering a loader while auth is resolving, then either the `Outlet` or a `<Navigate to='/login' replace>` that stores the original location in state so we can redirect back after login. With data routers I prefer checking in the loader and throwing `redirect`, so protected content never flashes. The real enforcement is on the API."

**What are loaders and actions?**

> "Loaders fetch a route's data before it renders, in parallel with lazy-loaded code, avoiding fetch-on-render waterfalls. Actions handle form submissions for a route; after an action, loaders revalidate so the UI stays consistent. Errors in either go to the route's `errorElement`."

---

# 12. Key Takeaways

- SPA routing = History API + render matching routes; server fallback needed.
- Data router (`createBrowserRouter`) unlocks loaders, actions, errorElement, lazy.
- Nested routes + `<Outlet>` = persistent layouts.
- `useParams` (strings), `useSearchParams` (put UI state in the URL), `useNavigate`, `<Navigate>`.
- Protected routes: guard/layout or loader `redirect`; server enforces.
- Lazy-load routes for code splitting.

## Readiness Checklist

- [ ] Build nested layouts with index routes and 404
- [ ] Implement protected + role-based routes with redirect-back
- [ ] Sync filters/pagination to search params
- [ ] Use loader + action + useNavigation for an edit form
- [ ] Lazy-load a route and add an error element
