# 12 — DOM & Browser APIs 🔥🔥

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 12
>
> Goal: Be fluent with DOM manipulation, master the event system (bubbling, capturing, delegation), and know when to use `MutationObserver`, `IntersectionObserver` and `ResizeObserver`.

---

# 1. The DOM Tree ⭐

The **DOM (Document Object Model)** is the browser's in-memory **tree of objects** representing the HTML document. JavaScript reads and changes the page through it.

```html
<html>
  <body>
    <div id="app">
      <h1>Hello</h1>
      <p>World</p>
    </div>
  </body>
</html>
```

```text
document
 └── html
      └── body
           └── div#app
                ├── h1 ── "Hello"   (text node)
                └── p  ── "World"
```

### Node vs Element

| | Node | Element |
| --- | --- | --- |
| What | Any item in the tree | Only tag nodes (`<div>`, `<p>`) |
| Includes | elements, text, comments, document | — |
| Navigation | `childNodes`, `firstChild`, `nextSibling`, `parentNode` | `children`, `firstElementChild`, `nextElementSibling`, `parentElement` |

```js
div.childNodes; // includes whitespace text nodes ❗
div.children;   // only elements ✅
```

### How the browser renders (context)

```text
HTML → DOM ┐
           ├─▶ Render Tree → Layout (reflow) → Paint → Composite
CSS → CSSOM┘
```

DOM changes may trigger **layout** and **paint** — expensive. See `23-javascript-performance.md`.

---

# 2. Selecting Elements

```js
document.getElementById("app");            // single element or null (fastest)
document.querySelector(".card > h2");      // FIRST match for any CSS selector, or null
document.querySelectorAll("li.active");    // STATIC NodeList of all matches
document.getElementsByClassName("item");   // LIVE HTMLCollection
document.getElementsByTagName("li");       // LIVE HTMLCollection

element.querySelector("span");             // search inside an element
element.closest(".card");                  // nearest ancestor (or itself) matching — great for delegation
element.matches(".btn.primary");           // does it match the selector?
```

### Static vs Live collections ⭐

```js
const live = document.getElementsByTagName("li");   // 3 items
const stat = document.querySelectorAll("li");       // 3 items

list.appendChild(document.createElement("li"));

live.length; // 4 — updates automatically
stat.length; // 3 — snapshot
```

- `NodeList` from `querySelectorAll` has `forEach` but not `map/filter` → `Array.from(list)` or `[...list]`.
- `HTMLCollection` has no `forEach`.
- Looping over a **live** collection while removing items skips elements — convert to an array first.

---

# 3. Manipulating the DOM

### Creating and inserting

```js
const li = document.createElement("li");
li.textContent = "New item";
li.className = "item";

list.append(li);             // end (accepts nodes AND strings, multiple)
list.prepend(li);            // start
list.appendChild(li);        // older; one node; returns it
ref.before(li);              // before ref (sibling)
ref.after(li);               // after ref
ref.replaceWith(li);
list.insertAdjacentHTML("beforeend", "<li>Parsed HTML</li>");
// positions: beforebegin | afterbegin | beforeend | afterend
```

> Inserting a node that's **already in the DOM moves it** — a node can only exist in one place. Use `node.cloneNode(true)` to duplicate.

### Removing

```js
li.remove();                 // modern
list.removeChild(li);        // older
list.replaceChildren();      // remove all children
```

### Content: `textContent` vs `innerText` vs `innerHTML` ⭐

| Property | Reads/writes | Notes |
| --- | --- | --- |
| `textContent` | Raw text of all nodes (incl. hidden) | Fast, **safe** (no HTML parsing) |
| `innerText` | Rendered, visible text | Triggers **layout** (slower), respects CSS |
| `innerHTML` | HTML string | Parses HTML → **XSS risk** with user input; destroys existing child nodes & their listeners |

```js
el.innerHTML = `<p>${userInput}</p>`; // ❌ XSS: userInput = "<img src=x onerror=alert(1)>"
el.textContent = userInput;           // ✅ rendered as text
```

### Batch DOM insertions with `DocumentFragment`

```js
const fragment = document.createDocumentFragment();
for (const item of items) {
  const li = document.createElement("li");
  li.textContent = item;
  fragment.append(li);   // in memory — no reflow
}
list.append(fragment);   // ONE insertion → one reflow
```

### Attributes vs Properties

```js
input.getAttribute("value");   // initial HTML attribute
input.value;                   // current live value (property)
input.setAttribute("disabled", "");
input.removeAttribute("disabled");
input.hasAttribute("disabled");
input.disabled = true;         // property — usually preferred
```

- **Attributes** = what's written in HTML (always strings).
- **Properties** = live JS object state (typed: booleans, numbers).
- `value` attribute = default; `value` property = what the user typed.

### `data-*` attributes

```html
<button data-user-id="42" data-action="delete">Delete</button>
```

```js
btn.dataset.userId;   // "42"  (kebab-case → camelCase, always strings)
btn.dataset.action;   // "delete"
btn.dataset.status = "done"; // adds data-status="done"
```

Perfect for event delegation (store ids on elements).

### Classes and styles

```js
el.classList.add("active", "visible");
el.classList.remove("hidden");
el.classList.toggle("open");          // add/remove
el.classList.toggle("open", isOpen);  // force
el.classList.contains("active");
el.classList.replace("old", "new");

el.style.backgroundColor = "red";     // inline style (camelCase)
el.style.setProperty("--theme", "dark"); // CSS variable
getComputedStyle(el).width;           // final computed value (triggers layout)
```

Prefer toggling **classes** over setting many inline styles.

---

# 4. Events ⭐⭐⭐

## 4.1 Event listeners

```js
function handleClick(event) {
  console.log(event.type, event.target);
}

button.addEventListener("click", handleClick);
button.removeEventListener("click", handleClick); // must be the SAME function reference
```

```js
// ❌ Can't be removed — a different function each time
button.addEventListener("click", () => handleClick());
button.removeEventListener("click", () => handleClick());
```

### Options

```js
el.addEventListener("click", handler, {
  once: true,      // auto-remove after first call
  passive: true,   // promise not to call preventDefault → smoother scrolling (touch/wheel)
  capture: true,   // listen in capture phase
  signal: controller.signal, // remove via AbortController
});

// Remove MANY listeners at once
const controller = new AbortController();
window.addEventListener("resize", onResize, { signal: controller.signal });
window.addEventListener("scroll", onScroll, { signal: controller.signal });
controller.abort(); // removes both
```

`addEventListener` vs `onclick`: `onclick = fn` allows only **one** handler (overwrites); `addEventListener` allows many and has options.

## 4.2 The event object

```js
el.addEventListener("click", (e) => {
  e.type;            // "click"
  e.target;          // element that was actually clicked (deepest)
  e.currentTarget;   // element the listener is attached to
  e.clientX, e.clientY;
  e.key;             // keyboard events
  e.preventDefault();
  e.stopPropagation();
});
```

### `target` vs `currentTarget` ⭐⭐⭐

```html
<ul id="list">
  <li><span>Item</span></li>
</ul>
```

```js
list.addEventListener("click", (e) => {
  console.log(e.target);        // <span> — what was clicked
  console.log(e.currentTarget); // <ul>   — where the listener is
});
```

## 4.3 Event propagation: capturing → target → bubbling ⭐⭐⭐

When you click an element, the event travels in **three phases**:

```text
            window
              │   ▲
   1. CAPTURE │   │ 3. BUBBLE
              ▼   │
           document
              │   ▲
             html
              │   ▲
             body
              │   ▲
            div#parent
              │   ▲
              ▼   │
        2. TARGET (button)
```

1. **Capturing phase** — from `window` down to the target's parent
2. **Target phase** — at the clicked element
3. **Bubbling phase** — from the target back up to `window`

By default listeners run in the **bubbling** phase. Pass `{ capture: true }` (or `true`) to run during capture.

```js
parent.addEventListener("click", () => console.log("parent bubble"));
parent.addEventListener("click", () => console.log("parent capture"), true);
child.addEventListener("click", () => console.log("child"));

// Click child:
// parent capture
// child
// parent bubble
```

### Events that don't bubble

`focus`, `blur` (use `focusin`/`focusout` which do), `mouseenter`, `mouseleave` (use `mouseover`/`mouseout`), `load`, `scroll` on elements (scroll on `document` reaches `window`).

## 4.4 `stopPropagation` vs `stopImmediatePropagation` vs `preventDefault` ⭐⭐⭐

| Method | Effect |
| --- | --- |
| `preventDefault()` | Cancels the browser's **default action** (link navigation, form submit, checkbox toggle, context menu). Propagation continues. |
| `stopPropagation()` | Stops the event moving to **other elements** (further up/down). Other listeners on the **same** element still run. |
| `stopImmediatePropagation()` | Stops propagation **and** remaining listeners on the same element. |

```js
form.addEventListener("submit", (e) => {
  e.preventDefault(); // stop page reload — handle with JS
  submitWithFetch(new FormData(form));
});

link.addEventListener("click", (e) => e.preventDefault()); // don't navigate
```

> ⚠️ Avoid `stopPropagation` unless necessary — it breaks delegation, analytics and "click outside to close" logic that listen at the document level.

`return false` in an `onclick` attribute handler = `preventDefault` only; in `addEventListener` it does nothing. (In jQuery it did both.)

## 4.5 Event Delegation ⭐⭐⭐

Instead of attaching a listener to every child, attach **one** listener to a common ancestor and use `event.target` to figure out which child was clicked. Works because events **bubble**.

```html
<ul id="todo-list">
  <li data-id="1">Learn JS <button class="delete">✕</button></li>
  <li data-id="2">Learn React <button class="delete">✕</button></li>
  <!-- items added dynamically later -->
</ul>
```

```js
const list = document.getElementById("todo-list");

list.addEventListener("click", (e) => {
  const deleteBtn = e.target.closest(".delete"); // handles clicks on nested elements too
  if (!deleteBtn || !list.contains(deleteBtn)) return;

  const li = deleteBtn.closest("li");
  deleteTodo(li.dataset.id);
  li.remove();
});
```

### Benefits

- **One listener** instead of N → less memory, faster setup
- Works for **dynamically added** elements automatically
- No need to add/remove listeners when items change (fewer leaks)

### Why `closest()` and not just `e.target`

If the button contains an icon `<svg>`, `e.target` may be the `<svg>` or `<path>`. `closest(".delete")` walks up to the actual button.

### Limitations

- Events that don't bubble (`focus`, `mouseenter`) → use bubbling alternatives
- A child calling `stopPropagation()` blocks delegation

> **React uses delegation internally**: since React 17 it attaches listeners at the **root container**, not on each element.

## 4.6 Custom Events

```js
const event = new CustomEvent("cart:updated", {
  detail: { count: 3 },   // payload
  bubbles: true,          // let it bubble
});
cartIcon.dispatchEvent(event);

document.addEventListener("cart:updated", (e) => {
  console.log(e.detail.count); // 3
});
```

Useful for decoupled communication between widgets / micro-frontends / web components.

`dispatchEvent` runs listeners **synchronously** (unlike real user events, which are queued as tasks).

---

# 5. Browser Observers

Observers replace inefficient polling and scroll/resize handlers. Their callbacks are batched and run off the hot path.

## 5.1 `IntersectionObserver` ⭐ (most used)

Notifies when an element **enters or leaves the viewport** (or another container).

```js
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        const img = entry.target;
        img.src = img.dataset.src;    // lazy load image
        observer.unobserve(img);      // stop watching once loaded
      }
    });
  },
  {
    root: null,          // viewport
    rootMargin: "200px", // start loading 200px before visible
    threshold: 0.1,      // 10% visible
  },
);

document.querySelectorAll("img[data-src]").forEach((img) => observer.observe(img));
```

**Use cases:** lazy-loading images, infinite scroll (observe a sentinel at the list end), analytics "ad viewed", animate on scroll, pausing off-screen videos.

```js
// Infinite scroll
const sentinel = document.querySelector("#load-more-sentinel");
new IntersectionObserver(([entry]) => {
  if (entry.isIntersecting) loadNextPage();
}).observe(sentinel);
```

## 5.2 `ResizeObserver`

Notifies when an **element's size** changes (not just the window).

```js
const ro = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const { width } = entry.contentRect;
    entry.target.classList.toggle("compact", width < 400);
  }
});
ro.observe(card);
```

**Use cases:** responsive components (container-based), charts that redraw on resize, textarea auto-grow. (CSS container queries now cover many styling cases.)

## 5.3 `MutationObserver`

Notifies when the **DOM changes** — children added/removed, attributes changed, text changed.

```js
const mo = new MutationObserver((mutations) => {
  for (const m of mutations) {
    if (m.type === "childList") console.log("added:", m.addedNodes);
    if (m.type === "attributes") console.log(m.attributeName, "changed");
  }
});

mo.observe(container, {
  childList: true,   // direct children added/removed
  subtree: true,     // the whole subtree
  attributes: true,
  characterData: true,
});

mo.disconnect(); // stop
```

- Callbacks are **microtasks** — batched after DOM changes, before paint.
- **Use cases:** reacting to third-party scripts changing the DOM, browser extensions, auto-initializing widgets on inserted elements, undo/redo tracking.

### Always disconnect

```js
// React
useEffect(() => {
  const observer = new IntersectionObserver(cb);
  observer.observe(ref.current);
  return () => observer.disconnect(); // prevent leaks
}, []);
```

---

# 6. Other Useful Browser APIs (quick reference)

| API | Use |
| --- | --- |
| `fetch` + `AbortController` | Network requests, cancellation |
| `requestAnimationFrame` | Animations synced to frames |
| `requestIdleCallback` | Low-priority background work |
| `history.pushState` / `popstate` | SPA routing |
| `URL` / `URLSearchParams` | Parse/build URLs & query strings |
| `navigator.clipboard` | Copy/paste (async, needs permission/user gesture) |
| `Intl` (`NumberFormat`, `DateTimeFormat`) | Locale formatting |
| `matchMedia` | JS media queries (dark mode, breakpoints) |
| `navigator.onLine` + `online`/`offline` events | Connectivity |
| `document.visibilityState` + `visibilitychange` | Pause work in hidden tabs |
| `BroadcastChannel` | Messaging between tabs |

```js
const params = new URLSearchParams(location.search);
params.get("q");
params.set("page", "2");
history.pushState({}, "", `?${params}`);
```

---

# 7. Output / Behaviour Questions 🧪

### Q1

```html
<div id="outer"><button id="btn">Click</button></div>
```

```js
outer.addEventListener("click", () => console.log("outer"));
btn.addEventListener("click", (e) => { console.log("btn 1"); e.stopPropagation(); });
btn.addEventListener("click", () => console.log("btn 2"));
// user clicks button
```

<details><summary>Answer</summary>

`btn 1`, `btn 2` — `stopPropagation` stops `outer`, but other listeners on the same element still run. With `stopImmediatePropagation`, only `btn 1`.

</details>

### Q2

```js
outer.addEventListener("click", () => console.log("outer capture"), true);
outer.addEventListener("click", () => console.log("outer bubble"));
btn.addEventListener("click", () => console.log("btn"));
// click button
```

<details><summary>Answer</summary>

`outer capture`, `btn`, `outer bubble`

</details>

### Q3

```js
const lis = document.getElementsByTagName("li"); // 3 items
for (let i = 0; i < lis.length; i++) {
  lis[i].remove();
}
console.log(lis.length);
```

<details><summary>Answer</summary>

`1` — the collection is **live**; removing shifts indexes, so every other element is skipped. Fix: `[...lis].forEach((li) => li.remove())`.

</details>

### Q4

```js
btn.addEventListener("click", () => console.log("A"));
btn.addEventListener("click", () => console.log("A"));
function h() { console.log("B"); }
btn.addEventListener("click", h);
btn.addEventListener("click", h);
// click
```

<details><summary>Answer</summary>

`A A B` — two different arrow functions are two listeners; adding the **same** function reference twice (same type/capture) is ignored.

</details>

---

# 8. Interview Answers

**Event bubbling vs capturing**

> "When an event happens, it first travels from the window down to the target — the capturing phase — then fires on the target, then bubbles back up through the ancestors. Listeners run in the bubbling phase by default; passing `capture: true` makes them run on the way down."

**What is event delegation and why use it?**

> "Instead of attaching listeners to many child elements, I attach one listener to a common parent and use `event.target` — usually with `closest()` — to find which child was interacted with. It works because events bubble. It uses less memory, automatically handles elements added later, and avoids adding and removing listeners. React itself uses delegation at the root."

**preventDefault vs stopPropagation**

> "`preventDefault` cancels the browser's default behaviour, like submitting a form or following a link, but the event still propagates. `stopPropagation` stops the event from reaching other elements but doesn't cancel the default action."

---

# 9. Key Takeaways

- DOM = tree of nodes; `children` (elements) vs `childNodes` (all nodes).
- `querySelectorAll` → static NodeList; `getElementsBy*` → live collections.
- Use `textContent` for user data; `innerHTML` is an XSS risk.
- Batch inserts with `DocumentFragment`; toggle classes instead of many inline styles.
- Events: capture → target → bubble; `target` vs `currentTarget`.
- `preventDefault` ≠ `stopPropagation`; `stopImmediatePropagation` also stops same-element listeners.
- Delegation = one parent listener + `closest()`; works for dynamic children.
- Observers: Intersection (visibility), Resize (element size), Mutation (DOM changes, microtasks). Always disconnect.

## Readiness Checklist

- [ ] Draw the 3 event phases
- [ ] Implement a delegated todo list (add/delete/toggle) with one listener
- [ ] Explain target vs currentTarget, preventDefault vs stopPropagation
- [ ] Lazy-load images and build infinite scroll with IntersectionObserver
- [ ] Explain live vs static collections
- [ ] Remove listeners correctly (same reference / AbortController)
