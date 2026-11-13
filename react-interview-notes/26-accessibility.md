# 26 — Accessibility (a11y)

> **Interview + Understanding Guide** — Roadmap Section 26
>
> Goal: Build React UIs usable by everyone — keyboard, screen reader, low vision, motor and cognitive needs. Know semantic HTML, ARIA (and when not to use it), focus management, accessible forms/modals/menus, and testing.

---

# 1. Why It Matters

- ~1 in 6 people live with a disability; everyone benefits (keyboard power users, bright sunlight, broken trackpad).
- Legal requirements (ADA, EU Accessibility Act 2025, Section 508) — WCAG 2.1/2.2 **AA** is the usual target.
- Better semantics → better SEO and testability (RTL queries by role).

### WCAG principles — POUR

| Principle | Meaning |
| --- | --- |
| **Perceivable** | Text alternatives, captions, sufficient contrast, adaptable layout |
| **Operable** | Keyboard accessible, enough time, no seizure triggers, navigable |
| **Understandable** | Readable, predictable, helps users avoid/correct errors |
| **Robust** | Works with assistive technologies (valid semantics) |

---

# 2. Semantic HTML First ⭐

The **first rule of ARIA**: don't use ARIA if a native element does the job.

```jsx
// ❌ div soup
<div className="btn" onClick={save}>Save</div>

// ✅ native button: focusable, Enter/Space work, announced as "button"
<button type="button" onClick={save}>Save</button>
```

| Need | Use |
| --- | --- |
| Action | `<button>` |
| Navigation | `<a href>` (or router `<Link>`) |
| Page regions | `<header>`, `<nav>`, `<main>`, `<aside>`, `<footer>`, `<section aria-labelledby>` |
| Headings | `<h1>`–`<h6>` in logical order (one `h1` per page) |
| Lists | `<ul>/<ol>/<li>` |
| Tables of data | `<table>`, `<th scope>`, `<caption>` |
| Forms | `<label>`, `<fieldset>`, `<legend>` |
| Disclosure | `<details>/<summary>` |
| Modal | `<dialog>` |

If you must use a `div` as a button, you need `role="button"`, `tabIndex={0}`, Enter **and** Space key handlers… which is why you shouldn't.

---

# 3. Images & Icons

```jsx
<img src="/chart.png" alt="Revenue grew 20% from January to March" />  // meaningful
<img src="/divider.png" alt="" />                                         // decorative → empty alt

<button aria-label="Close dialog">
  <XIcon aria-hidden="true" />                                            // icon-only button
</button>

<svg role="img" aria-labelledby="t1"><title id="t1">Warning</title>…</svg>
```

---

# 4. Forms ⭐

```jsx
function EmailField({ error }) {
  const id = useId();
  return (
    <div>
      <label htmlFor={id}>Email address</label>
      <input
        id={id}
        type="email"
        autoComplete="email"
        required
        aria-invalid={!!error}
        aria-describedby={error ? `${id}-error` : `${id}-hint`}
      />
      <p id={`${id}-hint`}>We'll never share your email.</p>
      {error && <p id={`${id}-error`} role="alert">{error}</p>}
    </div>
  );
}
```

- Every input needs a **label** (visible preferred; `aria-label` if not possible). Placeholders are not labels.
- Group radios/checkboxes with `<fieldset><legend>`.
- Announce errors (`role="alert"` / `aria-live`), associate them with `aria-describedby`, focus the first invalid field on submit.
- Use correct `type` and `autocomplete` for mobile keyboards and autofill.
- Don't disable the submit button without explaining why.

---

# 5. Keyboard Accessibility ⭐

- Everything interactive must be **reachable with Tab** and **operable** with Enter/Space/arrows/Escape.
- **Visible focus indicator** — never `outline: none` without a replacement; use `:focus-visible`.
- Logical **DOM order** = visual order; avoid positive `tabIndex` (`tabIndex={1}`).
- `tabIndex={0}` → add to tab order; `tabIndex={-1}` → focusable programmatically only.
- **Skip link**:

```jsx
<a href="#main" className="skip-link">Skip to main content</a>
<main id="main" tabIndex={-1}>…</main>
```

### Roving tabindex (tabs, toolbars, menus)

Only one item in the group is tabbable (`tabIndex=0`); arrow keys move focus within.

```jsx
function Toolbar({ items }) {
  const [active, setActive] = useState(0);
  const refs = useRef([]);
  const onKeyDown = (e) => {
    let next = active;
    if (e.key === "ArrowRight") next = (active + 1) % items.length;
    if (e.key === "ArrowLeft") next = (active - 1 + items.length) % items.length;
    if (next !== active) {
      e.preventDefault();
      setActive(next);
      refs.current[next].focus();
    }
  };
  return (
    <div role="toolbar" aria-label="Formatting" onKeyDown={onKeyDown}>
      {items.map((item, i) => (
        <button key={item} ref={(el) => (refs.current[i] = el)} tabIndex={i === active ? 0 : -1}>
          {item}
        </button>
      ))}
    </div>
  );
}
```

---

# 6. Focus Management in SPAs ⭐

| Situation | What to do |
| --- | --- |
| Modal opens | Move focus into the modal (first focusable or heading) |
| Modal open | **Trap** focus inside; Escape closes |
| Modal closes | **Return focus** to the trigger |
| Route change | Move focus to the new page's `<h1>` / main and announce the title |
| Item deleted | Move focus to a sensible neighbour |
| Form error | Focus the first invalid field / error summary |
| Content loads | Announce via live region instead of stealing focus |

### Accessible modal (key parts)

```jsx
function Modal({ open, onClose, title, children }) {
  const dialogRef = useRef(null);
  const triggerRef = useRef(null);

  useEffect(() => {
    if (!open) return;
    triggerRef.current = document.activeElement;
    dialogRef.current?.focus();

    const onKeyDown = (e) => {
      if (e.key === "Escape") onClose();
      if (e.key === "Tab") trapFocus(e, dialogRef.current);
    };
    document.addEventListener("keydown", onKeyDown);
    document.body.style.overflow = "hidden";       // scroll lock
    return () => {
      document.removeEventListener("keydown", onKeyDown);
      document.body.style.overflow = "";
      triggerRef.current?.focus();                  // restore focus
    };
  }, [open, onClose]);

  if (!open) return null;
  return createPortal(
    <div className="backdrop" onClick={onClose}>
      <div
        ref={dialogRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="dialog-title"
        tabIndex={-1}
        onClick={(e) => e.stopPropagation()}
      >
        <h2 id="dialog-title">{title}</h2>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    document.body,
  );
}

function trapFocus(e, container) {
  const focusables = container.querySelectorAll(
    'a[href], button:not([disabled]), input:not([disabled]), select, textarea, [tabindex]:not([tabindex="-1"])',
  );
  if (!focusables.length) return;
  const first = focusables[0];
  const last = focusables[focusables.length - 1];
  if (e.shiftKey && document.activeElement === first) { e.preventDefault(); last.focus(); }
  else if (!e.shiftKey && document.activeElement === last) { e.preventDefault(); first.focus(); }
}
```

Also make background content inert (`inert` attribute on the app root) — or use native `<dialog>.showModal()` / Radix Dialog / React Aria which handle all of this.

---

# 7. ARIA Essentials

### Roles, states, properties

```jsx
<div role="tablist">…</div>
<button aria-expanded={open} aria-controls="menu-1">Options</button>
<ul id="menu-1" role="menu" hidden={!open}>…</ul>
<button aria-pressed={bold}>B</button>
<div role="progressbar" aria-valuenow={60} aria-valuemin={0} aria-valuemax={100} />
<nav aria-label="Breadcrumb">…</nav>
<a href="/cart" aria-current="page">Cart</a>
<div aria-busy={loading}>…</div>
```

### Live regions — announce dynamic changes

```jsx
<div aria-live="polite" role="status">{items.length} results found</div>   // waits for pause
<div role="alert">Payment failed</div>                                       // assertive, interrupts
```

The live region element must **exist before** the content changes (render it empty, then update text).

### Visually hidden (screen-reader-only) text

```css
.sr-only {
  position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px;
  overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; border: 0;
}
```

```jsx
<button><TrashIcon aria-hidden="true" /><span className="sr-only">Delete item</span></button>
```

### Common ARIA mistakes

- Adding roles that duplicate native semantics (`<button role="button">`)
- `aria-hidden="true"` on focusable elements
- `aria-label` on non-interactive `div`s (often ignored)
- Using ARIA to describe visuals instead of fixing semantics

Follow the **WAI-ARIA Authoring Practices (APG)** patterns for tabs, combobox, menu, listbox, tree, dialog.

---

# 8. Visual Accessibility

- **Color contrast**: 4.5:1 normal text, 3:1 large text & UI components (WCAG AA).
- Don't convey information by **color alone** (add icons/text for errors).
- Support **zoom** to 200% and text resize; use `rem`, avoid fixed heights.
- Respect **`prefers-reduced-motion`**:

```css
@media (prefers-reduced-motion: reduce) {
  * { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
}
```

- Touch targets ≥ 24×24 px (WCAG 2.2), ideally 44×44.

---

# 9. Route Changes in SPAs

Screen readers don't notice client-side navigations. On route change:

```jsx
function RouteAnnouncer() {
  const location = useLocation();
  const [message, setMessage] = useState("");
  useEffect(() => {
    setMessage(`Navigated to ${document.title}`);
    document.querySelector("h1")?.focus(); // h1 with tabIndex={-1}
  }, [location.pathname]);
  return <div aria-live="assertive" className="sr-only">{message}</div>;
}
```

(Next.js includes a route announcer.)

---

# 10. Testing Accessibility

| Method | Tools |
| --- | --- |
| Linting | `eslint-plugin-jsx-a11y` |
| Unit/integration | RTL role queries; `jest-axe` / `vitest-axe` |
| E2E | `@axe-core/playwright` |
| Browser | Lighthouse, axe DevTools, WAVE, Chrome Accessibility tree |
| **Manual** (essential) | Keyboard-only walkthrough; screen readers (NVDA/JAWS on Windows, VoiceOver on macOS/iOS, TalkBack on Android); zoom 200% |

```jsx
import { axe } from "vitest-axe";
test("form has no a11y violations", async () => {
  const { container } = render(<SignupForm />);
  expect(await axe(container)).toHaveNoViolations();
});
```

Automated tools catch roughly 30–40% of issues — manual testing is required.

---

# 11. Interview Answers

**How do you make a React app accessible?**

> "Start with semantic HTML — real buttons, links, labels, headings and landmarks — and only add ARIA where native elements can't express the pattern, following the ARIA Authoring Practices. Ensure full keyboard support with visible focus, manage focus on modals and route changes, announce dynamic updates with live regions, check contrast and reduced motion. I lint with jsx-a11y, test with RTL role queries and axe, and do manual keyboard and screen-reader passes. For complex widgets I use accessible primitives like Radix or React Aria."

**How do you build an accessible modal?**

> "`role='dialog'`, `aria-modal`, and `aria-labelledby` pointing to the title; move focus into it on open, trap Tab inside, close on Escape, make the background inert and lock scroll, and return focus to the trigger on close. Or use the native `<dialog>` with `showModal()`, which gives top-layer rendering and Escape handling."

---

# 12. Key Takeaways

- Semantic HTML first; ARIA only to fill gaps.
- Labels for every input; errors linked with `aria-describedby` and announced.
- Keyboard: tab order, visible focus, Enter/Space/Escape/arrows, skip links, roving tabindex.
- Focus management for modals, route changes, deletions, errors.
- Live regions for async updates.
- Contrast, not-color-only, zoom, reduced motion, target size.
- Lint + axe + RTL roles + manual screen-reader testing.

## Readiness Checklist

- [ ] Refactor a div-button into an accessible button
- [ ] Build an accessible form with errors
- [ ] Build a modal with focus trap & return
- [ ] Implement accessible tabs (roles + roving tabindex)
- [ ] Add a route announcer
- [ ] Run axe and a keyboard-only test
