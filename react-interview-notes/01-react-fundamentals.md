# 01 — React Fundamentals

> **Interview + Understanding Guide** — Roadmap Section 1
>
> Goal: Build the correct mental model of React — what it is, how it renders, what the Virtual DOM really does — so every later topic makes sense.

---

# 1. What is React?

> **React is a JavaScript library for building user interfaces out of components. You describe what the UI should look like for a given state, and React efficiently updates the DOM when that state changes.**

### Core ideas

| Idea | Meaning |
| --- | --- |
| **Declarative** | Describe *what* the UI should be (`UI = f(state)`), not *how* to mutate the DOM step by step |
| **Component-based** | UI is split into reusable, composable functions that return UI |
| **Unidirectional data flow** | Data flows down via props; changes flow up via callbacks |
| **Learn once, write anywhere** | Same model for web (react-dom), native (React Native), servers (RSC) |

### Imperative vs Declarative

```js
// Imperative (vanilla JS): you manage every DOM change
const btn = document.querySelector("#like");
let liked = false;
btn.addEventListener("click", () => {
  liked = !liked;
  btn.textContent = liked ? "Unlike" : "Like";
  btn.classList.toggle("active", liked);
});
```

```jsx
// Declarative (React): describe the UI for each state
function LikeButton() {
  const [liked, setLiked] = useState(false);
  return (
    <button className={liked ? "active" : ""} onClick={() => setLiked(!liked)}>
      {liked ? "Unlike" : "Like"}
    </button>
  );
}
```

As the UI grows, imperative code turns into a tangle of "if this changed, update that". React removes that class of bugs: you only describe the end state.

### Library vs Framework

React is a **library** focused on the view layer. It doesn't ship routing, data fetching, forms or build tooling — you choose them (React Router, TanStack Query, React Hook Form, Vite) or use a **framework built on React** (Next.js, Remix/React Router framework mode, Expo). The React team now recommends starting new apps with a framework.

---

# 2. Elements, Components, Instances ⭐

```jsx
// Component: a function that returns UI
function Greeting({ name }) {
  return <h1>Hello {name}</h1>;
}

// Element: a plain, immutable object describing what to render
const element = <Greeting name="Asha" />;
// ≈ { type: Greeting, props: { name: "Asha" }, key: null, ... }
```

| Term | What it is |
| --- | --- |
| **Component** | A function (or class) that takes props and returns elements |
| **Element** | A lightweight object `{ type, props, key }` — a description, not a DOM node |
| **Instance / Fiber** | React's internal record for a mounted component: holds state, hooks, effects, position in the tree |

Elements are cheap to create and immutable. Creating `<Greeting />` does **not** call `Greeting` — React calls it later when rendering.

---

# 3. The Virtual DOM ⭐⭐⭐

### What it is

The **Virtual DOM** is the tree of React elements (plain JS objects) your components return. On every update React builds a new tree, **compares it with the previous one (reconciliation)**, and applies only the necessary changes to the real DOM.

```text
state change
   ↓
re-run components → new element tree (virtual)
   ↓
diff against previous tree (reconciliation)
   ↓
minimal DOM operations (commit)
```

### What it is NOT (common misconceptions)

- ❌ "The Virtual DOM is faster than the DOM." Hand-written, precise DOM updates are faster. The VDOM's value is letting you write **declarative** code with **good-enough** performance automatically.
- ❌ "React re-renders = DOM updates." A re-render only re-runs your function; the DOM changes only if the output differs.
- ❌ "React diffs the whole DOM." It diffs **its own element trees**, never reads the real DOM to compare.

> Modern React internals use **Fiber** nodes rather than a literal "virtual DOM" copy, but the term is still used in interviews. See `09-rendering-reconciliation-fiber.md`.

---

# 4. Bootstrapping an App

```html
<!-- index.html -->
<div id="root"></div>
<script type="module" src="/src/main.jsx"></script>
```

```jsx
// main.jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

- `createRoot` (React 18+) enables **concurrent features** and automatic batching. The legacy `ReactDOM.render` was removed in React 19.
- SSR apps use `hydrateRoot` instead (see `21-ssr-hydration-server-components.md`).

### `StrictMode` (development only)

Helps find bugs by:
1. **Rendering components twice** → exposes impure render logic
2. **Running effects twice** (mount → unmount → mount) → exposes missing cleanups
3. Warning about deprecated APIs

No effect in production builds.

---

# 5. Render and Commit ⭐⭐⭐

Every screen update goes through three steps:

```text
1. TRIGGER   initial render (root.render) or a state update (setState)
2. RENDER    React calls your components to compute the new UI  (pure, no DOM changes)
3. COMMIT    React applies the differences to the DOM, then runs effects
```

```jsx
function Clock({ time }) {
  return (
    <>
      <h1>{time}</h1>
      <input />   {/* typed text survives re-renders: React only updates <h1>'s text */}
    </>
  );
}
```

Key consequences:
- **Rendering must be pure** — same props/state → same JSX, no side effects (React may render more than once, pause or discard renders).
- The DOM is updated **only where output changed**.
- Effects (`useEffect`) run **after** the commit.

---

# 6. Components Must Be Pure ⭐

```jsx
// ❌ Impure: mutates a variable outside render
let guests = 0;
function Cup() {
  guests++;
  return <h2>Tea cup for guest #{guests}</h2>; // different output each render; StrictMode doubles it
}

// ✅ Pure: output depends only on props
function Cup({ guest }) {
  return <h2>Tea cup for guest #{guest}</h2>;
}
```

Allowed during render: creating and mutating **local** variables/objects created in that render.
Not allowed: changing props, state, global variables, the DOM, making requests, setting timers — those go in **event handlers** or **effects**.

---

# 7. Unidirectional Data Flow

```text
        App (state: user)
       /                \
  Header(user)       Profile(user, onUpdate)
                          │
                    onUpdate(newUser) ──▶ App.setUser
```

- Parents pass data **down** via props.
- Children request changes by calling **callbacks** passed down.
- Makes data flow predictable and debuggable (compare two-way binding in AngularJS).

---

# 8. Why React? (Interview question)

- **Declarative** model → fewer UI-state bugs
- **Component reusability** & composition
- **Huge ecosystem** & community, mature tooling
- **React Native** for mobile with the same model
- **Concurrent rendering**, Suspense, Server Components for modern UX/perf
- Strong hiring market and long-term backing (Meta + community)

**Trade-offs / criticisms:** only a view library (many decisions), easy to cause unnecessary re-renders, JSX learning curve, rapid ecosystem churn, bundle size vs smaller alternatives (Preact, Svelte, Solid with fine-grained reactivity).

---

# 9. React vs Other Approaches

| | React | Vue | Angular | Svelte / Solid |
| --- | --- | --- | --- | --- |
| Type | Library | Progressive framework | Full framework | Compiler / fine-grained reactivity |
| Templating | JSX | Templates (+JSX) | Templates | Templates / JSX |
| Reactivity | Re-render + diff (pull) | Proxy-based tracking | Zone.js / Signals | Compile-time / signals (push) |
| State | Hooks, external libs | Reactive refs | Services, RxJS, signals | Stores / signals |

At 4 YOE, be ready to explain **why React re-renders whole subtrees** while signal-based frameworks update exact DOM nodes, and how React mitigates it (memoization, React Compiler).

---

# 10. Interview Answers

**What is React and why use it?**

> "React is a library for building UIs from components. You declare what the UI should look like for a given state, and React handles updating the DOM. That makes complex UIs easier to reason about, components reusable, and it has a big ecosystem and React Native for mobile."

**What is the Virtual DOM?**

> "It's the in-memory tree of React elements that components return. When state changes, React re-renders to produce a new tree, diffs it against the previous one, and commits only the changes to the real DOM. Its main benefit is enabling declarative code with efficient updates — it isn't inherently faster than precise manual DOM manipulation."

**What happens when state changes?**

> "The update triggers a render: React calls the component and its children to get new JSX. In the commit phase, it compares the result with the previous output and applies the minimal DOM changes, then runs effects."

---

# 11. Key Takeaways

- React = declarative, component-based UI library; `UI = f(state)`.
- Element = description object; component = function returning elements.
- Virtual DOM = element tree used for diffing; value is the declarative model.
- Trigger → Render (pure) → Commit (DOM + effects).
- StrictMode double-renders and double-runs effects in dev to expose bugs.
- Data flows down; events flow up.

## Readiness Checklist

- [ ] Explain declarative vs imperative with an example
- [ ] Explain element vs component vs instance
- [ ] Explain Virtual DOM and its misconceptions
- [ ] Describe trigger/render/commit
- [ ] Explain what StrictMode does
