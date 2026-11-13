# 02 — JSX, Conditional Rendering & Lists

> **Interview + Understanding Guide** — Roadmap Section 2
>
> Goal: Understand what JSX compiles to, its rules and gotchas, conditional rendering patterns, and — most importantly — how keys work in lists.

---

# 1. What is JSX?

**JSX** is a syntax extension that lets you write HTML-like markup in JavaScript. Browsers don't understand it — a compiler (Babel, SWC, esbuild, TypeScript) transforms it into function calls.

```jsx
const el = <h1 className="title">Hello {name}</h1>;
```

### Classic transform (pre-React 17)

```js
const el = React.createElement("h1", { className: "title" }, "Hello ", name);
// required `import React from "react"` in every file
```

### New JSX transform (React 17+)

```js
import { jsx as _jsx } from "react/jsx-runtime";
const el = _jsx("h1", { className: "title", children: ["Hello ", name] });
// no need to import React for JSX
```

Both produce a **React element** object: `{ type: "h1", props: { className: "title", children: [...] }, key: null }`.

### Lowercase vs Capitalized

```jsx
<div />      // type: "div"   → DOM element
<MyButton /> // type: MyButton → component function
<myButton /> // ❌ treated as an unknown HTML tag "mybutton"
```

Components **must** start with a capital letter.

---

# 2. JSX Rules

1. **Return a single root** — wrap siblings in a parent or a **Fragment**.
2. **Close all tags** — `<img />`, `<input />`, `<br />`.
3. **camelCase attributes** — `className`, `htmlFor`, `onClick`, `tabIndex`, `strokeWidth`.
4. `{}` embeds **JavaScript expressions** (not statements: no `if`, `for` inside).
5. `style` takes an **object**: `style={{ backgroundColor: "red", fontSize: 16 }}` (numbers → `px` for most properties).
6. Comments: `{/* comment */}`.
7. `aria-*` and `data-*` keep their dashes.

```jsx
function Card({ user }) {
  return (
    <>
      {/* Fragment: no extra DOM node */}
      <label htmlFor="email">Email</label>
      <input id="email" className="input" defaultValue={user.email} />
      <div style={{ marginTop: 8 }} data-testid="card" aria-live="polite">
        {user.name.toUpperCase()}
      </div>
    </>
  );
}
```

### Fragments

```jsx
<>...</>                              // short syntax
<Fragment key={item.id}>...</Fragment> // needed when you require a key (lists)
```

### What JSX renders for different values

| Value | Renders |
| --- | --- |
| string / number | text |
| `true`, `false`, `null`, `undefined` | nothing |
| array | each item (needs keys) |
| element | that element |
| plain object | ❌ Error: Objects are not valid as a React child |

```jsx
<p>{0}</p>        // "0" ❗ (numbers render, including 0)
<p>{NaN}</p>      // "NaN"
<p>{false}</p>    // nothing
<p>{user}</p>     // ❌ crash if user is an object
```

### JSX escapes values (XSS protection)

```jsx
const input = "<img src=x onerror=alert(1)>";
<div>{input}</div>; // rendered as TEXT, not HTML ✅
```

Raw HTML requires `dangerouslySetInnerHTML` — see `27-security.md`.

---

# 3. Conditional Rendering ⭐

```jsx
// 1. if / early return
function Status({ loading, error, data }) {
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <List data={data} />;
}

// 2. Ternary
{isLoggedIn ? <Dashboard /> : <Login />}

// 3. Logical AND
{hasUnread && <Badge />}

// 4. Variable
let content = <Empty />;
if (items.length) content = <List items={items} />;
return <section>{content}</section>;

// 5. Object map (many states)
const views = { idle: <Idle />, loading: <Spinner />, done: <Result /> };
return views[status];
```

### ⚠️ The `&&` with 0 trap

```jsx
{messages.length && <Messages />} // renders "0" when empty ❗

// ✅
{messages.length > 0 && <Messages />}
{!!messages.length && <Messages />}
```

`0 && x` evaluates to `0`, and React renders numbers.

### Returning `null`

A component can return `null` to render nothing — it's still mounted (its state and effects live).

```jsx
function Banner({ show }) {
  if (!show) return null;
  return <div className="banner">Sale!</div>;
}
```

### Conditional rendering and state

```jsx
{isEdit ? <Form key="edit" /> : <Form key="new" />}
```

Same component type at the same position → React **keeps state**. Different keys (or different types) → **resets** state. See §5.

---

# 4. Rendering Lists ⭐

```jsx
const todos = [
  { id: 1, text: "Learn JSX" },
  { id: 2, text: "Learn hooks" },
];

function TodoList() {
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  );
}

// Filter + map
{todos.filter((t) => !t.done).map((t) => <TodoItem key={t.id} todo={t} />)}
```

- The `key` goes on the **outermost element returned by `map`**, not inside the child component.
- `key` is **not a prop** — `TodoItem` can't read `props.key`. Pass `id` separately if needed.

---

# 5. Keys ⭐⭐⭐

### Why keys exist

When a list changes, React must match old items with new ones to decide what to **keep, move, insert, or remove**. Keys give items a **stable identity** across renders.

```text
Old: [A(key=1), B(key=2), C(key=3)]
New: [D(key=4), A(key=1), B(key=2), C(key=3)]

With keys → React sees A, B, C unchanged; inserts D at the start. ✅
Without (index) keys → React thinks item 0 changed A→D, 1 changed B→A, ... and appends C. ❌ (more work, wrong state)
```

### Rules

- **Unique among siblings** (not globally).
- **Stable** — same item → same key every render.
- Don't generate keys during render (`Math.random()`, `crypto.randomUUID()` inside `map`) → new key every render → **remount every time** (lost state, focus, slow).

### Why array index is a bad key ⭐⭐⭐

Fine **only** if the list is static (never reordered, filtered, or inserted into) and items have no state.

```jsx
function List() {
  const [items, setItems] = useState(["Apple", "Banana"]);
  return (
    <>
      <button onClick={() => setItems(["Cherry", ...items])}>Add to top</button>
      {items.map((item, index) => (
        <div key={index}>
          {item} <input placeholder="note" />
        </div>
      ))}
    </>
  );
}
```

Type "fruit note" into Apple's input, then click **Add to top**:
- Keys `0, 1` now map to `Cherry, Apple` → React reuses the DOM `<input>` at index 0 → your note now appears next to **Cherry** ❗
- With `key={item}` (or an id), the note stays with Apple ✅

Problems with index keys: state/inputs attached to the wrong item, broken animations, unnecessary re-renders.

### Where do good keys come from?

- Database IDs from the API
- Generate an ID **when creating** the item (`crypto.randomUUID()` in the add handler) and store it
- A naturally unique field (email, slug)

### Using `key` to reset state ⭐

Changing a component's key forces React to **unmount and remount** it with fresh state.

```jsx
// Reset the form when switching users
<ProfileForm key={userId} user={user} />
```

This is the idiomatic way to "reset all state when a prop changes" — no effect needed.

---

# 6. Common Gotchas

```jsx
// ❌ Calling the handler during render
<button onClick={handleClick()}>   // runs on every render
// ✅
<button onClick={handleClick}>
<button onClick={() => handleClick(id)}>

// ❌ class instead of className (works with a warning, but wrong)
// ❌ for instead of htmlFor

// ❌ Inline object literal as default prop → new reference each render
function List({ items = [] }) {} // new [] every render (matters for memo/deps)
const EMPTY = [];
function List2({ items = EMPTY }) {} // ✅ stable

// Boolean attributes
<input disabled />          // disabled={true}
<input disabled={false} />  // not disabled

// Spreading props
<Button {...props} />       // convenient, but can pass unknown attributes to DOM
```

---

# 7. Output Questions 🧪

```jsx
// Q1
function A() { const count = 0; return <div>{count && <span>Items</span>}</div>; }

// Q2
function B() { return <div>{[1, 2, 3]}</div>; }

// Q3
function C() { return <div>{true}{null}{undefined}{"x"}</div>; }

// Q4
function D() { return <div>{{ a: 1 }}</div>; }

// Q5
function E() { return <p>{"<b>hi</b>"}</p>; }
```

<details><summary>Answers</summary>

1. `<div>0</div>`
2. `<div>123</div>` (arrays render each item; React warns about missing keys only for elements)
3. `<div>x</div>`
4. Error: Objects are not valid as a React child
5. Literal text `<b>hi</b>` (escaped)

</details>

---

# 8. Interview Answers

**Why do we need keys? Why not index?**

> "Keys give list items a stable identity so React can match items between renders and know which were added, removed or moved. With index keys, inserting or reordering shifts every index, so React associates the wrong state and DOM nodes with items — inputs keep the wrong text, for example — and does extra work. Use a stable unique id; index is only OK for static lists without state."

**What does JSX compile to?**

> "Function calls that create React elements — `jsx()` from `react/jsx-runtime` with the new transform, or `React.createElement` in the classic one. The result is a plain object with `type`, `props` and `key`."

---

# 9. Key Takeaways

- JSX → `jsx()` / `createElement` → element objects.
- Capitalized = component; lowercase = DOM tag.
- `{}` takes expressions; JSX escapes strings automatically.
- `0 && <X />` renders `0`.
- Keys: unique among siblings, stable, not index (unless static), not random.
- Change `key` to reset a component's state.

## Readiness Checklist

- [ ] Write JSX and its compiled output side by side
- [ ] List 5 conditional rendering patterns and the `&&` trap
- [ ] Demonstrate the index-key input bug and fix it
- [ ] Use `key` to reset a form
