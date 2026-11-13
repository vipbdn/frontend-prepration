# 10 — Context API

> **Interview + Understanding Guide** — Roadmap Section 10
>
> Goal: Use Context correctly, understand its re-render behaviour and performance pitfalls, and know when to use Context vs props, composition, or a state library.

---

# 1. The Problem: Prop Drilling

```jsx
<App user={user}>
  <Layout user={user}>
    <Sidebar user={user}>
      <Avatar user={user} /> {/* only Avatar needs it */}
```

Passing props through components that don't use them is **prop drilling**: noisy, fragile, hard to refactor.

---

# 2. Context Basics ⭐

```jsx
import { createContext, useContext, useState, useMemo } from "react";

// 1. Create (default value used only when there's NO provider above)
const ThemeContext = createContext("light");

// 2. Provide
function App() {
  const [theme, setTheme] = useState("light");
  return (
    <ThemeContext.Provider value={theme}>   {/* React 19: <ThemeContext value={theme}> also works */}
      <Page />
      <button onClick={() => setTheme((t) => (t === "light" ? "dark" : "light"))}>Toggle</button>
    </ThemeContext.Provider>
  );
}

// 3. Consume
function Button() {
  const theme = useContext(ThemeContext);
  return <button className={`btn-${theme}`}>OK</button>;
}
```

- `useContext` reads from the **nearest** provider above.
- Providers can be **nested/overridden** for subtrees.
- React 19: `use(ThemeContext)` also works (and can be conditional).

---

# 3. Production Pattern: Provider + Custom Hook ⭐

```jsx
const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const login = useCallback(async (credentials) => {
    const u = await api.login(credentials);
    setUser(u);
  }, []);

  const logout = useCallback(() => {
    api.logout();
    setUser(null);
  }, []);

  // Memoize so consumers don't re-render on every AuthProvider render
  const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (ctx === null) throw new Error("useAuth must be used within <AuthProvider>");
  return ctx;
}
```

Benefits: encapsulated state, typed API, helpful error when used outside the provider.

---

# 4. Context Re-render Behaviour ⭐⭐⭐

1. When the provider's `value` changes (`Object.is`), **every component that consumes that context re-renders**.
2. `React.memo` does **not** stop context-triggered re-renders.
3. Consumers re-render even if they only use a part of the value that didn't change.
4. There's **no built-in selector** — `useContext(Ctx).user.name` still subscribes to the whole value.

### Pitfall 1: New object every render

```jsx
function Provider({ children }) {
  const [user, setUser] = useState(null);
  return (
    <AuthContext.Provider value={{ user, setUser }}> {/* ❌ new object each render */}
      {children}
    </AuthContext.Provider>
  );
}
```

If `Provider` re-renders for any reason (its parent re-rendered), all consumers re-render. **Fix:** `useMemo` the value.

### Pitfall 2: One big context

```jsx
<AppContext.Provider value={{ user, theme, cart, notifications, … }}>
```

Changing `cart` re-renders everything that reads `theme`. **Fix:** split contexts by concern and update frequency.

### Pitfall 3: Frequently changing values

Mouse position, scroll, form input, timers in context → re-renders the whole consumer tree constantly. **Fix:** keep fast-changing state local, or use a store with selectors (Zustand, Redux, Jotai, `useSyncExternalStore`).

---

# 5. Optimization Techniques ⭐⭐⭐

### 5.1 Memoize the value

```jsx
const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);
```

### 5.2 Split state and dispatch contexts

```jsx
const CartStateContext = createContext(null);
const CartDispatchContext = createContext(null);

function CartProvider({ children }) {
  const [state, dispatch] = useReducer(cartReducer, initialCart);
  return (
    <CartStateContext.Provider value={state}>
      <CartDispatchContext.Provider value={dispatch}>{children}</CartDispatchContext.Provider>
    </CartStateContext.Provider>
  );
}

// "Add to cart" buttons only need dispatch → never re-render when the cart changes
function AddToCart({ product }) {
  const dispatch = useContext(CartDispatchContext);
  return <button onClick={() => dispatch({ type: "added", product })}>Add</button>;
}
```

### 5.3 Split by domain / update frequency

`ThemeContext`, `AuthContext`, `CartContext`, `ToastContext` — not one `AppContext`.

### 5.4 Memoize expensive children below consumers

```jsx
function Dashboard() {
  const { theme } = useTheme();
  return useMemo(() => <ExpensiveChart theme={theme} />, [theme]);
}
```

### 5.5 Children pattern for the provider

```jsx
// ✅ children are created by App; when CartProvider's state changes,
//    only consumers re-render — not the whole children tree.
<CartProvider>
  <Shop />
</CartProvider>
```

### 5.6 Context selectors via external store

```jsx
// Put a stable store in context; subscribe to slices with useSyncExternalStore
const StoreContext = createContext(null);

function useStoreSelector(selector) {
  const store = useContext(StoreContext); // store object never changes → no context re-renders
  return useSyncExternalStore(store.subscribe, () => selector(store.getState()));
}
```

This is how Redux/Zustand/`use-context-selector` avoid broad re-renders.

---

# 6. When to Use Context ⭐

### ✅ Good fits — "global-ish, changes rarely"

- Theme, locale/i18n, current user/auth, feature flags
- Dependency injection (API client, analytics, router)
- Compound component internals (Tabs ↔ Tab communication)
- Design-system configuration

### ❌ Poor fits

- Frequently updating state (inputs, animations, real-time data)
- Large complex app state with many consumers → Redux Toolkit / Zustand
- **Server data** → TanStack Query / RTK Query (Context isn't a cache)
- Avoiding 1–2 levels of props → just pass props

### Context is not state management

Context is a **dependency injection / transport** mechanism. State management = `useState`/`useReducer` (inside the provider) + Context to deliver it. It has no selectors, middleware, devtools or caching.

---

# 7. Alternatives to Context for Prop Drilling

### Composition: pass components, not data

```jsx
// ❌ drilling user through Layout and Header
<Layout user={user} />

// ✅ the parent that has the data renders the leaf
<Layout header={<Header avatar={<Avatar user={user} />} />}>
  <Feed />
</Layout>
```

> "Before you use context, try passing props or `children`." — React docs

### Comparison

| Approach | Use when |
| --- | --- |
| Props | Few levels, explicit data flow |
| Composition (`children`/slots) | Intermediate components don't need the data |
| Context | Many distant consumers, rarely changing data |
| Zustand / Redux / Jotai | Frequently updated, shared, complex client state; need selectors/devtools |
| TanStack Query | Server state |
| URL (search params) | Shareable UI state (filters, pagination, tabs) |

---

# 8. Testing Components That Use Context

```jsx
function renderWithProviders(ui, { user = null } = {}) {
  return render(
    <AuthContext.Provider value={{ user, login: vi.fn(), logout: vi.fn() }}>
      <ThemeProvider>{ui}</ThemeProvider>
    </AuthContext.Provider>,
  );
}
```

---

# 9. Output / Behaviour Questions 🧪

### Q1

```jsx
const Ctx = createContext();
const Child = memo(function Child() {
  const v = useContext(Ctx);
  console.log("child");
  return v.count;
});
function App() {
  const [count, setCount] = useState(0);
  const [other, setOther] = useState(0);
  return (
    <Ctx.Provider value={{ count }}>
      <Child />
      <button onClick={() => setOther(other + 1)}>other</button>
    </Ctx.Provider>
  );
}
```
Clicking "other" — does Child re-render?

<details><summary>Answer</summary>

**Yes.** `App` re-renders → new `{ count }` object → context value changed → consumer re-renders despite `memo`. Fix: `useMemo(() => ({ count }), [count])`.

</details>

### Q2

```jsx
const Ctx = createContext("default");
function A() { return <B />; }
function B() { return useContext(Ctx); }
function App() { return <A />; }
```
What does B render?

<details><summary>Answer</summary>`"default"` — no provider above, so the default value is used.</details>

---

# 10. Interview Answers

**What problem does Context solve, and what are its drawbacks?**

> "It passes data deep into the tree without prop drilling — great for theme, auth or locale. The drawback is that every consumer re-renders whenever the provider value changes, there are no selectors, and `memo` doesn't help. So I memoize the value, split contexts by concern — often separating state and dispatch — and keep frequently changing or complex state in a store like Zustand or Redux, and server data in TanStack Query."

**Is Context a replacement for Redux?**

> "Not really. Context is a transport mechanism; combined with `useReducer` it can handle simple global state. Redux Toolkit adds selectors that limit re-renders, middleware, devtools with time travel, and established patterns — useful for large, frequently changing shared state. Many apps today need neither for server data because TanStack Query handles that."

---

# 11. Key Takeaways

- Context = dependency injection to avoid prop drilling.
- Provider + custom hook + error when missing.
- All consumers re-render on value change; `memo` doesn't block it.
- Memoize the value, split contexts (state/dispatch, by domain), keep fast-changing state out.
- Try props/composition first; use stores for complex/high-frequency state; Query libs for server state.

## Readiness Checklist

- [ ] Build Auth/Theme providers with custom hooks
- [ ] Demonstrate & fix the new-object-value re-render
- [ ] Split a cart context into state + dispatch
- [ ] Explain Context vs Redux vs Zustand vs composition
