# 16 — Component Design Patterns 🔥

> **Interview + Understanding Guide** — Roadmap Section 16
>
> Goal: Know the patterns used to build reusable, flexible components and design systems — composition/slots, container/presentational, HOCs, render props, custom hooks, compound components, controlled/uncontrolled props, prop getters, headless components, polymorphic components, provider pattern — and when to choose each.

---

# 1. Composition & Slots ⭐

The foundation of all React patterns: build complex UI by **combining** components and passing UI as props.

```jsx
function Card({ header, footer, children }) {
  return (
    <article className="card">
      {header && <header className="card-header">{header}</header>}
      <div className="card-body">{children}</div>
      {footer && <footer className="card-footer">{footer}</footer>}
    </article>
  );
}

<Card
  header={<h2>Order #123</h2>}
  footer={<Button onClick={pay}>Pay now</Button>}
>
  <OrderSummary />
</Card>
```

Avoids prop explosion (`showHeader`, `headerTitle`, `headerIcon`, …) — let consumers pass elements.

---

# 2. Container / Presentational

Separate **what data / behaviour** from **how it looks**.

```jsx
// Presentational: pure UI, props only, easy to test & reuse (Storybook)
function UserList({ users, onSelect }) {
  return <ul>{users.map((u) => <li key={u.id} onClick={() => onSelect(u)}>{u.name}</li>)}</ul>;
}

// Container: data + logic
function UserListContainer() {
  const { data: users = [] } = useQuery({ queryKey: ["users"], queryFn: fetchUsers });
  const navigate = useNavigate();
  return <UserList users={users} onSelect={(u) => navigate(`/users/${u.id}`)} />;
}
```

Hooks made strict container components less necessary — a **custom hook** (`useUsers`) often plays the container role — but the separation of concerns is still valuable.

---

# 3. Higher-Order Components (HOC)

A function that takes a component and returns an enhanced component.

```jsx
function withAuth(Component) {
  function WithAuth(props) {
    const { user, isLoading } = useAuth();
    if (isLoading) return <Spinner />;
    if (!user) return <Navigate to="/login" replace />;
    return <Component {...props} user={user} />;
  }
  WithAuth.displayName = `withAuth(${Component.displayName || Component.name})`;
  return WithAuth;
}

const ProtectedDashboard = withAuth(Dashboard);
```

**Use cases:** cross-cutting concerns (auth, logging, feature flags, error boundaries, analytics), wrapping third-party components, legacy class code (`connect()` from react-redux, `withRouter`).

**Problems:**
- Wrapper hell in DevTools
- **Prop collisions** (two HOCs injecting `data`)
- Implicit props — hard to know where a prop comes from
- Static methods not copied (need `hoist-non-react-statics`)
- Must not create HOCs inside render (remounts)

Modern alternative: **custom hooks**.

---

# 4. Render Props

A component takes a **function** that returns UI, and calls it with its internal state.

```jsx
function MouseTracker({ render }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return (
    <div onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })} style={{ height: 300 }}>
      {render(pos)}
    </div>
  );
}

<MouseTracker render={({ x, y }) => <p>Mouse at {x}, {y}</p>} />

// Function-as-children variant
<MouseTracker>{({ x, y }) => <Cursor x={x} y={y} />}</MouseTracker>
```

Still useful for: lists with custom item rendering (`renderItem`), virtualized lists, headless libraries (Downshift), giving consumers full control over markup. For pure logic sharing, hooks are cleaner.

---

# 5. Custom Hooks (the modern default)

```jsx
function useMousePosition() {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const move = (e) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener("mousemove", move);
    return () => window.removeEventListener("mousemove", move);
  }, []);
  return pos;
}
```

### HOC vs Render Props vs Hooks ⭐

| | HOC | Render props | Custom hooks |
| --- | --- | --- | --- |
| Shares | Logic + wrapping UI | Logic + rendering control | Logic only |
| Nesting / wrapper hell | ❌ yes | ❌ callback nesting | ✅ flat |
| Prop origin clarity | ❌ implicit | ✅ explicit | ✅ explicit |
| Naming collisions | ❌ possible | ✅ none | ✅ none |
| Works in class components | ✅ | ✅ | ❌ |
| Modern recommendation | Cross-cutting wrappers only | Rendering customization | **Default for logic reuse** |

---

# 6. Compound Components ⭐⭐⭐

A set of components that **work together** and share implicit state through context — like `<select>` and `<option>`. Gives consumers flexible markup with a clean API.

```jsx
const TabsContext = createContext(null);

function useTabs() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error("Tabs.* must be used inside <Tabs>");
  return ctx;
}

function Tabs({ defaultValue, value: controlledValue, onChange, children }) {
  const [uncontrolled, setUncontrolled] = useState(defaultValue);
  const isControlled = controlledValue !== undefined;
  const value = isControlled ? controlledValue : uncontrolled;

  const setValue = useCallback(
    (v) => {
      if (!isControlled) setUncontrolled(v);
      onChange?.(v);
    },
    [isControlled, onChange],
  );

  const ctx = useMemo(() => ({ value, setValue }), [value, setValue]);
  return <TabsContext.Provider value={ctx}><div className="tabs">{children}</div></TabsContext.Provider>;
}

function TabList({ children }) {
  return <div role="tablist">{children}</div>;
}

function Tab({ value, children }) {
  const { value: active, setValue } = useTabs();
  const selected = active === value;
  return (
    <button
      role="tab"
      id={`tab-${value}`}
      aria-selected={selected}
      aria-controls={`panel-${value}`}
      tabIndex={selected ? 0 : -1}
      onClick={() => setValue(value)}
    >
      {children}
    </button>
  );
}

function TabPanel({ value, children }) {
  const { value: active } = useTabs();
  if (active !== value) return null;
  return (
    <div role="tabpanel" id={`panel-${value}`} aria-labelledby={`tab-${value}`}>
      {children}
    </div>
  );
}

Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Usage — consumer controls structure & order
<Tabs defaultValue="profile">
  <Tabs.List>
    <Tabs.Tab value="profile">Profile</Tabs.Tab>
    <Tabs.Tab value="billing">Billing</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="profile"><Profile /></Tabs.Panel>
  <Tabs.Panel value="billing"><Billing /></Tabs.Panel>
</Tabs>
```

Used by Radix UI, Headless UI, Reach UI, Chakra: `Accordion.Item`, `Menu.Button`, `Dialog.Trigger`, `Select.Option`.

(Older implementations used `React.Children.map` + `cloneElement` to inject props; context is more flexible — works with any nesting depth.)

---

# 7. Controlled & Uncontrolled Props ⭐

Reusable components should support **both**:
- **Uncontrolled**: `defaultValue` — component manages its own state (easy to use).
- **Controlled**: `value` + `onChange` — parent owns the state (full control: reset, sync, validate).

```jsx
function useControllableState({ value, defaultValue, onChange }) {
  const [internal, setInternal] = useState(defaultValue);
  const isControlled = value !== undefined;
  const current = isControlled ? value : internal;

  const setValue = useCallback(
    (next) => {
      const resolved = typeof next === "function" ? next(current) : next;
      if (!isControlled) setInternal(resolved);
      onChange?.(resolved);
    },
    [isControlled, current, onChange],
  );

  return [current, setValue];
}

function Toggle({ checked, defaultChecked = false, onChange }) {
  const [on, setOn] = useControllableState({ value: checked, defaultValue: defaultChecked, onChange });
  return <button role="switch" aria-checked={on} onClick={() => setOn((v) => !v)}>{on ? "On" : "Off"}</button>;
}

<Toggle defaultChecked />                                  // uncontrolled
<Toggle checked={enabled} onChange={setEnabled} />         // controlled
```

Don't switch between controlled and uncontrolled during a component's life.

---

# 8. State Reducer Pattern (inversion of control)

Let consumers **override** internal state transitions.

```jsx
function useToggle({ reducer = toggleReducer } = {}) {
  const [{ on }, dispatch] = useReducer(reducer, { on: false });
  const toggle = () => dispatch({ type: "toggle" });
  return { on, toggle };
}

function toggleReducer(state, action) {
  switch (action.type) {
    case "toggle":
      return { on: !state.on };
    default:
      return state;
  }
}

// Consumer: prevent toggling more than 4 times
const [clicks, setClicks] = useState(0);
const { on, toggle } = useToggle({
  reducer(state, action) {
    if (action.type === "toggle" && clicks >= 4) return state;
    return toggleReducer(state, action);
  },
});
```

Popularized by Kent C. Dodds / Downshift.

---

# 9. Prop Getters

Return functions that produce the props consumers should spread — merging internal handlers with the consumer's own.

```jsx
const callAll = (...fns) => (...args) => fns.forEach((fn) => fn?.(...args));

function useDisclosure() {
  const [open, setOpen] = useState(false);
  const id = useId();

  const getTriggerProps = ({ onClick, ...rest } = {}) => ({
    "aria-expanded": open,
    "aria-controls": id,
    onClick: callAll(onClick, () => setOpen((o) => !o)), // consumer's onClick still runs
    ...rest,
  });

  const getContentProps = (props = {}) => ({ id, hidden: !open, ...props });

  return { open, getTriggerProps, getContentProps };
}

function FAQ() {
  const { getTriggerProps, getContentProps } = useDisclosure();
  return (
    <>
      <button {...getTriggerProps({ onClick: () => track("faq_open") })}>Shipping?</button>
      <div {...getContentProps()}>We ship worldwide.</div>
    </>
  );
}
```

---

# 10. Headless Components ⭐

Provide **behaviour, state and accessibility** without any styling — consumers render all markup.

```jsx
function useCombobox({ items, onSelect }) {
  const [inputValue, setInputValue] = useState("");
  const [open, setOpen] = useState(false);
  const [highlighted, setHighlighted] = useState(-1);

  const filtered = useMemo(
    () => items.filter((i) => i.toLowerCase().includes(inputValue.toLowerCase())),
    [items, inputValue],
  );

  const onKeyDown = (e) => {
    if (e.key === "ArrowDown") { e.preventDefault(); setOpen(true); setHighlighted((h) => Math.min(h + 1, filtered.length - 1)); }
    if (e.key === "ArrowUp") { e.preventDefault(); setHighlighted((h) => Math.max(h - 1, 0)); }
    if (e.key === "Enter" && highlighted >= 0) { onSelect(filtered[highlighted]); setInputValue(filtered[highlighted]); setOpen(false); }
    if (e.key === "Escape") setOpen(false);
  };

  return {
    open, filtered, highlighted,
    getInputProps: () => ({
      value: inputValue, role: "combobox", "aria-expanded": open, "aria-autocomplete": "list",
      onChange: (e) => { setInputValue(e.target.value); setOpen(true); setHighlighted(-1); },
      onKeyDown,
    }),
    getItemProps: (index) => ({
      role: "option", "aria-selected": index === highlighted,
      onMouseEnter: () => setHighlighted(index),
      onMouseDown: (e) => { e.preventDefault(); onSelect(filtered[index]); setInputValue(filtered[index]); setOpen(false); },
    }),
  };
}
```

Libraries: **Radix UI**, **React Aria**, **Headless UI**, **Downshift**, **TanStack Table/Form/Virtual**. Teams pair them with their own styles (e.g. **shadcn/ui** = Radix + Tailwind copied into your codebase).

---

# 11. Polymorphic Components (`as` prop)

```tsx
type ButtonProps<C extends React.ElementType> = {
  as?: C;
  variant?: "primary" | "ghost";
} & Omit<React.ComponentPropsWithoutRef<C>, "as">;

function Button<C extends React.ElementType = "button">({ as, variant = "primary", ...rest }: ButtonProps<C>) {
  const Component = as || "button";
  return <Component className={`btn btn-${variant}`} {...rest} />;
}

<Button onClick={save}>Save</Button>
<Button as="a" href="/docs">Docs</Button>          // renders <a>, href is type-checked
<Button as={Link} to="/home">Home</Button>          // router Link
```

Alternative: **`asChild`** pattern (Radix Slot) — merge props into the single child element.

---

# 12. Provider Pattern

Wrap a subtree in a provider exposing shared state/services (theme, auth, toasts, query client, i18n). Covered in `10-context-api.md`.

---

# 13. Other Patterns Worth Naming

| Pattern | Idea |
| --- | --- |
| **Layout components** | `<Stack gap>`, `<Grid>` encapsulate spacing/layout |
| **Error boundary wrappers** | `<ErrorBoundary fallback>` around features |
| **Feature flags** | `<Feature name="x">` or `useFlag("x")` |
| **Portals** | Modals/tooltips outside overflow containers (`17-refs-portals-dom.md`) |
| **Optimistic UI** | Update UI immediately, reconcile later |
| **Container queries / responsive components** | Adapt to container size |

---

# 14. Interview Answers

**What are compound components?**

> "A group of components that share implicit state through context and are used together, like `<Tabs>`, `<Tabs.List>`, `<Tabs.Tab>` and `<Tabs.Panel>`. The parent owns the state; children read it from context. Consumers get full control over markup and order with a clean declarative API — that's how Radix and Headless UI are built."

**HOC vs render props vs hooks?**

> "All three share logic. HOCs wrap a component and inject props, which causes wrapper hell and unclear prop origins. Render props pass state into a function the consumer renders — flexible but nests. Custom hooks share logic with no extra components and explicit inputs and outputs, so they're my default. I still use render props when consumers need rendering control, and HOCs for cross-cutting wrappers."

**How do you design a reusable component?**

> "Start from composition: accept `children` and slots instead of many config props. Support both controlled and uncontrolled usage. Forward refs and spread remaining props onto the root element, make it accessible by default, keep styling overridable, and for complex widgets separate behaviour into a headless hook so the visuals can vary."

---

# 15. Key Takeaways

- Composition and slots beat configuration props.
- Container/presentational → mostly replaced by custom hooks, still a good separation.
- HOCs: cross-cutting wrappers; watch prop collisions.
- Render props: rendering control; hooks: logic reuse default.
- Compound components via context for flexible, declarative APIs.
- Support controlled + uncontrolled; state reducer and prop getters for inversion of control.
- Headless + polymorphic components power modern design systems.

## Readiness Checklist

- [ ] Build `Tabs` / `Accordion` as compound components (controlled + uncontrolled)
- [ ] Write a `withX` HOC with displayName
- [ ] Convert a render-prop component to a hook
- [ ] Implement `useControllableState`
- [ ] Build a headless combobox hook with prop getters
- [ ] Type a polymorphic `as` component
