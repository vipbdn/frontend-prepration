# 17 — Refs, Portals & DOM

> **Interview + Understanding Guide** — Roadmap Section 17
>
> Goal: Use refs correctly (DOM refs, forwarding, callback refs, ref cleanup, imperative handles), manage focus, and render modals/tooltips with portals while understanding how events bubble through them.

---

# 1. DOM Refs Recap

```jsx
function Form() {
  const inputRef = useRef(null);
  return (
    <>
      <input ref={inputRef} />
      <button onClick={() => inputRef.current.focus()}>Focus</button>
    </>
  );
}
```

- `ref.current` is `null` during the first render; set **before** layout effects run (commit phase); reset to `null` on unmount.
- Access refs in **effects and handlers**, not during render.
- Refs are an **escape hatch** — use for focus, selection, scroll, media, measurements, integrating non-React libraries. Don't use them to change what React manages (e.g. `ref.current.textContent = …` on React-rendered content).

---

# 2. Forwarding Refs to Child Components ⭐

By default, a ref on a custom component doesn't reach its DOM node.

### React 19: `ref` is a regular prop

```jsx
function TextInput({ label, ref, ...props }) {
  return (
    <label>
      {label}
      <input ref={ref} {...props} />
    </label>
  );
}

function Login() {
  const emailRef = useRef(null);
  useEffect(() => emailRef.current.focus(), []);
  return <TextInput label="Email" ref={emailRef} />;
}
```

### React ≤18: `forwardRef`

```jsx
const TextInput = forwardRef(function TextInput({ label, ...props }, ref) {
  return <input ref={ref} {...props} />;
});
```

`forwardRef` still works in React 19 but will be deprecated over time. Design-system components should always forward refs (so consumers can focus/measure/position them).

---

# 3. Callback Refs ⭐

Pass a **function** as `ref`; React calls it with the DOM node on attach (and `null` on detach in React ≤18).

```jsx
function MeasureOnMount() {
  const [height, setHeight] = useState(0);
  const measuredRef = useCallback((node) => {
    if (node !== null) setHeight(node.getBoundingClientRect().height);
  }, []);
  return <div ref={measuredRef}>Height: {height}px</div>;
}
```

Use cases:
- Run code when a node **appears** (conditionally rendered elements — a `useEffect` with `[]` wouldn't re-run)
- Refs for **dynamic lists** (a Map of refs)
- Combining multiple refs

### React 19: ref cleanup functions

```jsx
<div
  ref={(node) => {
    const observer = new ResizeObserver(() => console.log(node.offsetWidth));
    observer.observe(node);
    return () => observer.disconnect(); // called on detach instead of calling ref(null)
  }}
/>
```

### Refs for a list of items

```jsx
function CatList({ cats }) {
  const itemsRef = useRef(new Map());

  const scrollTo = (id) => itemsRef.current.get(id)?.scrollIntoView({ behavior: "smooth", block: "center" });

  return (
    <ul>
      {cats.map((cat) => (
        <li
          key={cat.id}
          ref={(node) => {
            if (node) itemsRef.current.set(cat.id, node);
            else itemsRef.current.delete(cat.id);
          }}
        >
          {cat.name}
        </li>
      ))}
    </ul>
  );
}
```

(You can't call `useRef` inside a loop — rules of hooks.)

### Merging refs (own + forwarded)

```jsx
function mergeRefs(...refs) {
  return (node) => {
    refs.forEach((ref) => {
      if (typeof ref === "function") ref(node);
      else if (ref) ref.current = node;
    });
  };
}

function Input({ ref, ...props }) {
  const localRef = useRef(null);
  useAutoFocus(localRef);
  return <input ref={mergeRefs(localRef, ref)} {...props} />;
}
```

---

# 4. `useImperativeHandle` (recap)

Expose a limited API instead of the DOM node:

```jsx
function SearchInput({ ref }) {
  const inputRef = useRef(null);
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    clear: () => { inputRef.current.value = ""; },
  }), []);
  return <input ref={inputRef} />;
}
```

See `07-advanced-hooks.md` §6.

---

# 5. Focus Management ⭐

Critical for accessibility (`26-accessibility.md`):

```jsx
// Focus first invalid field on submit
function onSubmitError(errors) {
  const first = Object.keys(errors)[0];
  formRef.current.querySelector(`[name="${first}"]`)?.focus();
}

// Restore focus when a modal closes
function useRestoreFocus(isOpen) {
  const previouslyFocused = useRef(null);
  useEffect(() => {
    if (isOpen) {
      previouslyFocused.current = document.activeElement;
      return () => previouslyFocused.current?.focus();
    }
  }, [isOpen]);
}
```

`autoFocus` prop works on mount; for dynamic cases use refs + effects.

### Scrolling

```jsx
messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
listRef.current.scrollTop = listRef.current.scrollHeight;
```

When you need to scroll right after a state update that adds DOM, either use an effect on the data, or `flushSync(() => setItems(...))` and then scroll.

---

# 6. Integrating Non-React Libraries

```jsx
function MapView({ center, zoom }) {
  const containerRef = useRef(null);
  const mapRef = useRef(null);

  // Create once
  useEffect(() => {
    mapRef.current = new maplibregl.Map({ container: containerRef.current, center, zoom });
    return () => mapRef.current.remove(); // destroy on unmount
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  // Sync props → imperative API
  useEffect(() => {
    mapRef.current?.flyTo({ center, zoom });
  }, [center, zoom]);

  return <div ref={containerRef} style={{ height: 400 }} />;
}
```

Pattern: **ref to a container React never touches**, create the instance in an effect, sync props in separate effects, destroy in cleanup. Same for charts (Chart.js, D3), editors, video players.

---

# 7. Portals ⭐

Render children into a **different DOM node** outside the parent hierarchy while keeping them in the **React tree**.

```jsx
import { createPortal } from "react-dom";

function Modal({ isOpen, onClose, title, children }) {
  if (!isOpen) return null;
  return createPortal(
    <div className="backdrop" onClick={onClose}>
      <div
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        className="modal"
        onClick={(e) => e.stopPropagation()} // don't close when clicking inside
      >
        <h2 id="modal-title">{title}</h2>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    document.body, // or document.getElementById("modal-root")
  );
}
```

### Why portals?

Parent styles can clip or trap overlays:
- `overflow: hidden` / `overflow: auto` containers
- `z-index` stacking contexts
- `transform` / `filter` (create containing blocks for `position: fixed`)

Use for: **modals, dialogs, tooltips, popovers, dropdown menus, toasts, full-screen overlays**.

### Portal behaviour ⭐

| Aspect | Behaves like |
| --- | --- |
| DOM position | Inside the target container (e.g. `body`) |
| **Context** | Still receives context from React parents ✅ |
| **Events** | **Bubble through the React tree**, not the DOM tree ❗ |
| State / lifecycle | Owned by the React parent |

```jsx
<div onClick={() => console.log("parent clicked")}>
  <Modal isOpen>…click inside modal…</Modal>
</div>
// Clicking inside the modal logs "parent clicked" — even though in the DOM the modal is in <body>.
```

Gotchas:
- `useOnClickOutside` checking `ref.current.contains(e.target)` treats portal content as "outside" in the DOM → include the portal node.
- Need focus trapping, `Escape` to close, scroll locking, `aria-modal`, returning focus — or use the native **`<dialog>`** element (`showModal()` gives top-layer rendering, focus trapping basics and Esc) or a library (Radix Dialog).

### Native `<dialog>` alternative

```jsx
function NativeModal({ open, onClose, children }) {
  const ref = useRef(null);
  useEffect(() => {
    const dialog = ref.current;
    if (open && !dialog.open) dialog.showModal(); // renders in the top layer — no portal needed
    if (!open && dialog.open) dialog.close();
  }, [open]);
  return (
    <dialog ref={ref} onClose={onClose}>
      {children}
    </dialog>
  );
}
```

---

# 8. Interview Answers

**When do you use refs?**

> "For things outside React's declarative model: focusing inputs, scrolling, measuring layout, playing media, and integrating imperative libraries like maps or charts. Also to hold mutable values that shouldn't trigger renders, like timer IDs. I don't read refs during render or use them to mutate DOM that React controls."

**What are portals and how do events behave?**

> "`createPortal` renders children into a different DOM node, like `document.body`, which avoids clipping by `overflow` or `z-index` stacking contexts — ideal for modals and tooltips. The portal stays in the React tree, so it still gets context, and events bubble through React ancestors rather than DOM ancestors."

**How do you pass a ref to a custom component?**

> "In React 19, `ref` is just a prop on function components, so I forward it to the inner element. In earlier versions I'd wrap the component with `forwardRef`. If I want to expose only certain methods, I use `useImperativeHandle`."

---

# 9. Key Takeaways

- Refs: escape hatch for DOM/imperative APIs; use in effects/handlers, not render.
- React 19: `ref` as a prop; `forwardRef` for older versions.
- Callback refs for dynamic nodes/lists; React 19 ref cleanup functions; merge refs when needed.
- `useImperativeHandle` to expose a minimal API.
- Integrate libraries: container ref + create/sync/destroy effects.
- Portals: different DOM location, same React tree (context + event bubbling).

## Readiness Checklist

- [ ] Forward a ref in React 18 and React 19 styles
- [ ] Manage a Map of refs for scroll-to-item
- [ ] Build an accessible modal with a portal (Esc, focus, backdrop)
- [ ] Explain portal event bubbling and the click-outside gotcha
- [ ] Wrap a non-React library in a component
