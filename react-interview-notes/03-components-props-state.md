# 03 — Components, Props & State

> **Interview + Understanding Guide** — Roadmap Section 3
>
> Goal: Master props, state as a snapshot, lifting state, derived state, controlled vs uncontrolled components, composition, and how React preserves or resets state.

---

# 1. Function Components

```jsx
function Avatar({ src, size = 48, alt }) {
  return <img src={src} width={size} height={size} alt={alt} />;
}

export default function Profile() {
  return <Avatar src="/me.png" alt="Me" />;
}
```

- A component is a function: **props in → JSX out**.
- Never define components **inside** other components:

```jsx
function Parent() {
  // ❌ New component type every render → child remounts, loses state & focus
  function Child() { return <input />; }
  return <Child />;
}
```

---

# 2. Props ⭐

**Props** are the inputs passed from a parent. They are **read-only**.

```jsx
function Button({ label, variant = "primary", onClick, disabled = false, ...rest }) {
  return (
    <button className={`btn btn-${variant}`} onClick={onClick} disabled={disabled} {...rest}>
      {label}
    </button>
  );
}

<Button label="Save" onClick={save} type="submit" />
```

### Props are immutable

```jsx
function Child(props) {
  props.name = "x"; // ❌ never mutate props
}
```

To "change" a prop, the **parent** changes its state and re-renders with new props.

### `children`

```jsx
function Card({ title, children }) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="card-body">{children}</div>
    </div>
  );
}

<Card title="Profile">
  <Avatar />
  <p>Bio…</p>
</Card>
```

`children` can be anything renderable — elements, strings, arrays, even functions (render props).

### Passing functions & components as props

```jsx
<List items={users} renderItem={(u) => <UserRow user={u} />} />
<Layout sidebar={<Sidebar />} content={<Feed />} />   // "slots"
```

### Default values

Use destructuring defaults (`defaultProps` for function components was removed in React 19).

---

# 3. State ⭐⭐⭐

**State** is component memory: data that changes over time and should trigger a re-render.

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

### Why not a regular variable?

```jsx
function Counter() {
  let count = 0;                       // ❌ reset to 0 every render
  return <button onClick={() => count++}>{count}</button>; // changes don't trigger re-render
}
```

State is (1) **preserved between renders** and (2) **setting it triggers a re-render**.

### State is a snapshot ⭐⭐⭐

Each render has its **own fixed** state values. `setState` doesn't change the current variable — it schedules a new render with the new value.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
    console.log(count); // 0 — still this render's snapshot
  }
  // After click: 1, not 3
}
```

```jsx
// Functional updates read the latest queued value
setCount((c) => c + 1);
setCount((c) => c + 1);
setCount((c) => c + 1); // → 3
```

### Updating objects and arrays immutably

```jsx
const [user, setUser] = useState({ name: "Asha", address: { city: "Pune" } });

setUser({ ...user, name: "Ravi" });
setUser((u) => ({ ...u, address: { ...u.address, city: "Delhi" } }));

const [todos, setTodos] = useState([]);
setTodos((t) => [...t, newTodo]);                                // add
setTodos((t) => t.filter((x) => x.id !== id));                   // remove
setTodos((t) => t.map((x) => (x.id === id ? { ...x, done: !x.done } : x))); // update
```

Mutating and setting the same reference → React bails out → **no re-render**.

### State is isolated and private

Two `<Counter />` instances each have their own state. State belongs to a **position in the tree**, not to the component function.

---

# 4. Props vs State

| | Props | State |
| --- | --- | --- |
| Owned by | Parent | The component itself |
| Mutable? | No (read-only) | Yes, via setter |
| Purpose | Configure a component | Remember changing data |
| Change triggers re-render? | Yes (parent re-renders with new props) | Yes |

---

# 5. Lifting State Up ⭐

When two components need the **same** changing data, move the state to their **closest common parent** and pass it down.

```jsx
function Accordion() {
  const [activeIndex, setActiveIndex] = useState(0); // single source of truth

  return (
    <>
      <Panel title="About" isActive={activeIndex === 0} onShow={() => setActiveIndex(0)}>
        About text…
      </Panel>
      <Panel title="Details" isActive={activeIndex === 1} onShow={() => setActiveIndex(1)}>
        Details text…
      </Panel>
    </>
  );
}

function Panel({ title, children, isActive, onShow }) {
  return (
    <section>
      <h3>{title}</h3>
      {isActive ? <p>{children}</p> : <button onClick={onShow}>Show</button>}
    </section>
  );
}
```

`Panel` is now **controlled** by its parent.

### Single source of truth

Each piece of state should live in **one** place. Don't duplicate the same data in multiple components' state — sync bugs follow.

---

# 6. Choosing the State Structure ⭐

### Principles (from the React docs)

1. **Group related state** — if two values always change together, merge them (`{ x, y }`).
2. **Avoid contradictions** — not `isSending` + `isSent`; use `status: "typing" | "sending" | "sent"`.
3. **Avoid redundant state** — compute from existing state/props.
4. **Avoid duplication** — store the selected **id**, not a copy of the selected object.
5. **Avoid deep nesting** — normalize (`{ byId, allIds }`).

### Derived state — don't store what you can compute ⭐

```jsx
// ❌ Redundant state + effect to sync it
const [items, setItems] = useState([]);
const [total, setTotal] = useState(0);
useEffect(() => setTotal(items.reduce((s, i) => s + i.price, 0)), [items]);

// ✅ Compute during render
const total = items.reduce((s, i) => s + i.price, 0);
// expensive? → useMemo
```

```jsx
// ❌ Copy of a prop in state (stops updating when the prop changes)
function Message({ color }) {
  const [c, setC] = useState(color);
}
// ✅ Use the prop directly — or name it `initialColor` if you really want only the first value
```

### Store IDs, not copies

```jsx
// ❌ selectedItem becomes stale when items update
const [selectedItem, setSelectedItem] = useState(items[0]);
// ✅
const [selectedId, setSelectedId] = useState(items[0].id);
const selectedItem = items.find((i) => i.id === selectedId);
```

---

# 7. Controlled vs Uncontrolled Components ⭐⭐⭐

### Form inputs

```jsx
// Controlled: React state is the source of truth
function Controlled() {
  const [name, setName] = useState("");
  return <input value={name} onChange={(e) => setName(e.target.value)} />;
}

// Uncontrolled: the DOM holds the value; read it when needed
function Uncontrolled() {
  const inputRef = useRef(null);
  const submit = () => console.log(inputRef.current.value);
  return (
    <>
      <input ref={inputRef} defaultValue="Asha" />
      <button onClick={submit}>Submit</button>
    </>
  );
}
```

| Controlled | Uncontrolled |
| --- | --- |
| `value` + `onChange` | `defaultValue` + ref / `FormData` |
| Instant validation, formatting, conditional UI | Less code, fewer re-renders |
| Re-renders on every keystroke | Value read on submit |
| Easy to reset/prefill from state | File inputs are always uncontrolled |

⚠️ `value` without `onChange` → read-only input + warning. Switching between `undefined` and a string value → "changing an uncontrolled input to be controlled" warning (initialize with `""`).

### Components in general

A component is **controlled** when its important state is driven by props from the parent; **uncontrolled** when it manages it internally. Good reusable components often support **both** (see `16-component-patterns.md` §controlled props).

---

# 8. Preserving and Resetting State ⭐⭐⭐

React keeps state **as long as the same component type renders at the same position in the tree**.

```jsx
function App() {
  const [isFancy, setIsFancy] = useState(false);
  return isFancy ? <Counter isFancy /> : <Counter />; // same type, same position → state preserved
}
```

```jsx
// Different types at the same position → state reset
{isPaused ? <p>Paused</p> : <Counter />}

// Same type but different key → state reset
{isPlayerA ? <Counter key="A" /> : <Counter key="B" />}
```

Reset form on selection change:

```jsx
<ContactForm key={selectedContactId} contact={selectedContact} />
```

---

# 9. Composition vs Inheritance

React favours **composition**. Specialize components by passing props/children, not by extending classes.

```jsx
function Dialog({ title, children, footer }) {
  return (
    <div role="dialog" aria-label={title}>
      <h2>{title}</h2>
      {children}
      <footer>{footer}</footer>
    </div>
  );
}

function ConfirmDialog({ onConfirm }) {
  return (
    <Dialog title="Are you sure?" footer={<button onClick={onConfirm}>Yes</button>}>
      This can't be undone.
    </Dialog>
  );
}
```

Composition also **avoids prop drilling** and **re-renders** (passing elements as `children` — see `15-performance-optimization.md`).

---

# 10. Output Questions 🧪

### Q1

```jsx
function App() {
  const [n, setN] = useState(0);
  return (
    <button onClick={() => { setN(n + 1); setN(n + 1); }}>{n}</button>
  );
}
```
After one click?

<details><summary>Answer</summary>`1`</details>

### Q2

```jsx
function App() {
  const [n, setN] = useState(0);
  return (
    <button onClick={() => { setN(n + 5); setN((p) => p + 1); }}>{n}</button>
  );
}
```

<details><summary>Answer</summary>`6` — replace with 5, then updater 5 → 6.</details>

### Q3

```jsx
function App() {
  const [show, setShow] = useState(true);
  return (
    <>
      {show ? <Counter /> : <Counter />}
      <button onClick={() => setShow(!show)}>toggle</button>
    </>
  );
}
```
Counter is at 3. Click toggle. Value?

<details><summary>Answer</summary>`3` — same type, same position → state preserved.</details>

### Q4

```jsx
function App() {
  const [user, setUser] = useState({ name: "A" });
  const rename = () => { user.name = "B"; setUser(user); };
  return <button onClick={rename}>{user.name}</button>;
}
```

<details><summary>Answer</summary>Still shows `A` — same object reference, React bails out.</details>

---

# 11. Interview Answers

**Props vs state**

> "Props are inputs passed from the parent and are read-only. State is data a component owns and changes over time; updating it with the setter triggers a re-render. When multiple components need the same state, I lift it to their closest common parent."

**Controlled vs uncontrolled**

> "A controlled input gets its value from React state and updates it through `onChange`, so React is the single source of truth — good for validation and dynamic UI. An uncontrolled input keeps its value in the DOM and I read it with a ref or `FormData`, which is simpler and causes fewer re-renders."

**Why is state a snapshot?**

> "Each render gets its own constant state values. Calling the setter schedules a re-render with the new value; it doesn't change the variable in the current render. So multiple `setCount(count + 1)` calls in one handler all use the same value — I use functional updates when the next state depends on the previous one."

---

# 12. Key Takeaways

- Components: pure functions, never defined inside other components.
- Props are read-only; `children` enables composition.
- State is a snapshot; use functional updates for sequential changes.
- Update objects/arrays immutably.
- Lift shared state; keep a single source of truth.
- Don't store derived or duplicated state; store IDs.
- Controlled (value+onChange) vs uncontrolled (defaultValue+ref/FormData).
- State lives at a tree position; reset with a different `key` or type.

## Readiness Checklist

- [ ] Explain state as snapshot with the triple-setState example
- [ ] Lift state for an accordion/tabs
- [ ] Refactor redundant state into derived values
- [ ] Explain controlled vs uncontrolled with trade-offs
- [ ] Explain when state is preserved vs reset
