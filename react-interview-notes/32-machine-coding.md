# 32 — Machine Coding Round 🔥🔥🔥

> **Interview + Understanding Guide** — Roadmap Section 32
>
> Goal: Build common UI components under time pressure (45–90 min) with clean state design, accessibility, edge cases and good communication. Complete reference solutions for 14 problems + specs for more.
>
> **Practise:** set a timer, build without looking, then compare.

---

# 0. How to Approach a Machine Coding Round

```text
1. CLARIFY (3–5 min)   Requirements, must-haves vs nice-to-haves, data source (mock/API), styling expectations
2. DESIGN (3–5 min)    Component tree, state (what/where), data shape, events
3. BUILD CORE (25–40)  Working happy path first — no styling rabbit holes
4. EDGE CASES (5–10)   Empty, loading, error, boundaries, rapid clicks, keyboard
5. POLISH (rest)       Accessibility, styling, small optimizations, cleanup
6. EXPLAIN             Trade-offs, what you'd add with more time (tests, perf, a11y)
```

**What interviewers score:** working solution, state modelling, component decomposition, code readability, edge cases, accessibility, React best practices (keys, immutability, effects cleanup), communication.

**Habits:** talk while coding; keep state minimal and derived; name things well; handle loading/error/empty; use semantic elements; clean up timers/listeners; don't over-engineer.

---

# 1. Todo App (CRUD + filter + persistence) ⭐

Requirements: add, toggle, edit, delete, filter (all/active/completed), clear completed, count, persist to localStorage.

```jsx
import { useEffect, useMemo, useReducer, useState } from "react";

const FILTERS = { all: () => true, active: (t) => !t.done, completed: (t) => t.done };

function todosReducer(todos, action) {
  switch (action.type) {
    case "added":
      return [...todos, { id: crypto.randomUUID(), text: action.text, done: false }];
    case "toggled":
      return todos.map((t) => (t.id === action.id ? { ...t, done: !t.done } : t));
    case "edited":
      return todos.map((t) => (t.id === action.id ? { ...t, text: action.text } : t));
    case "deleted":
      return todos.filter((t) => t.id !== action.id);
    case "cleared_completed":
      return todos.filter((t) => !t.done);
    default:
      throw new Error(`Unknown action ${action.type}`);
  }
}

function loadTodos() {
  try {
    return JSON.parse(localStorage.getItem("todos")) ?? [];
  } catch {
    return [];
  }
}

export default function TodoApp() {
  const [todos, dispatch] = useReducer(todosReducer, undefined, loadTodos);
  const [text, setText] = useState("");
  const [filter, setFilter] = useState("all");

  useEffect(() => {
    localStorage.setItem("todos", JSON.stringify(todos));
  }, [todos]);

  const visible = useMemo(() => todos.filter(FILTERS[filter]), [todos, filter]);
  const remaining = todos.filter((t) => !t.done).length;

  function handleSubmit(e) {
    e.preventDefault();
    const trimmed = text.trim();
    if (!trimmed) return;
    dispatch({ type: "added", text: trimmed });
    setText("");
  }

  return (
    <section>
      <h1>Todos</h1>
      <form onSubmit={handleSubmit}>
        <label htmlFor="new-todo" className="sr-only">New todo</label>
        <input id="new-todo" value={text} onChange={(e) => setText(e.target.value)} placeholder="What needs doing?" />
        <button type="submit">Add</button>
      </form>

      {visible.length === 0 ? (
        <p>{todos.length ? "Nothing here." : "No todos yet."}</p>
      ) : (
        <ul>
          {visible.map((todo) => (
            <TodoItem key={todo.id} todo={todo} dispatch={dispatch} />
          ))}
        </ul>
      )}

      <footer>
        <span>{remaining} item{remaining === 1 ? "" : "s"} left</span>
        <div role="group" aria-label="Filter todos">
          {Object.keys(FILTERS).map((f) => (
            <button key={f} aria-pressed={filter === f} onClick={() => setFilter(f)}>{f}</button>
          ))}
        </div>
        <button onClick={() => dispatch({ type: "cleared_completed" })} disabled={remaining === todos.length}>
          Clear completed
        </button>
      </footer>
    </section>
  );
}

function TodoItem({ todo, dispatch }) {
  const [isEditing, setIsEditing] = useState(false);
  const [draft, setDraft] = useState(todo.text);

  function save() {
    const trimmed = draft.trim();
    if (trimmed) dispatch({ type: "edited", id: todo.id, text: trimmed });
    else setDraft(todo.text);
    setIsEditing(false);
  }

  return (
    <li>
      <input
        type="checkbox"
        checked={todo.done}
        onChange={() => dispatch({ type: "toggled", id: todo.id })}
        aria-label={`Mark "${todo.text}" as ${todo.done ? "not done" : "done"}`}
      />
      {isEditing ? (
        <input
          autoFocus
          value={draft}
          onChange={(e) => setDraft(e.target.value)}
          onBlur={save}
          onKeyDown={(e) => {
            if (e.key === "Enter") save();
            if (e.key === "Escape") { setDraft(todo.text); setIsEditing(false); }
          }}
        />
      ) : (
        <span onDoubleClick={() => setIsEditing(true)} style={{ textDecoration: todo.done ? "line-through" : "none" }}>
          {todo.text}
        </span>
      )}
      <button onClick={() => setIsEditing(true)} aria-label={`Edit "${todo.text}"`}>✎</button>
      <button onClick={() => dispatch({ type: "deleted", id: todo.id })} aria-label={`Delete "${todo.text}"`}>✕</button>
    </li>
  );
}
```

Talking points: reducer for many update types, lazy init from storage, derived `visible`/`remaining`, stable ids, Escape to cancel edit.

---

# 2. Autocomplete / Typeahead ⭐⭐⭐ (most asked)

Requirements: debounce input, fetch suggestions, cancel stale requests, loading/error/empty states, keyboard navigation (↑ ↓ Enter Esc), highlight match, click to select, accessible combobox, cache results.

```jsx
import { useEffect, useId, useRef, useState } from "react";

function useDebounce(value, delay) {
  const [v, setV] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setV(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return v;
}

const cache = new Map(); // simple in-memory cache: query → results

export function Autocomplete({ fetchSuggestions, onSelect, minChars = 2 }) {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [status, setStatus] = useState("idle"); // idle | loading | error | done
  const [open, setOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);
  const debounced = useDebounce(query.trim(), 300);
  const listId = useId();

  useEffect(() => {
    if (debounced.length < minChars) {
      setResults([]);
      setStatus("idle");
      return;
    }
    if (cache.has(debounced)) {
      setResults(cache.get(debounced));
      setStatus("done");
      return;
    }
    const controller = new AbortController();
    setStatus("loading");
    fetchSuggestions(debounced, controller.signal)
      .then((data) => {
        cache.set(debounced, data);
        setResults(data);
        setStatus("done");
        setActiveIndex(-1);
      })
      .catch((err) => {
        if (err.name !== "AbortError") setStatus("error");
      });
    return () => controller.abort(); // stale request cancelled → no race condition
  }, [debounced, fetchSuggestions, minChars]);

  function select(item) {
    setQuery(item.label);
    setOpen(false);
    setActiveIndex(-1);
    onSelect?.(item);
  }

  function handleKeyDown(e) {
    if (!open && (e.key === "ArrowDown" || e.key === "ArrowUp")) setOpen(true);
    switch (e.key) {
      case "ArrowDown":
        e.preventDefault();
        setActiveIndex((i) => (results.length ? (i + 1) % results.length : -1));
        break;
      case "ArrowUp":
        e.preventDefault();
        setActiveIndex((i) => (results.length ? (i - 1 + results.length) % results.length : -1));
        break;
      case "Enter":
        if (open && activeIndex >= 0) {
          e.preventDefault();
          select(results[activeIndex]);
        }
        break;
      case "Escape":
        setOpen(false);
        break;
    }
  }

  const showList = open && debounced.length >= minChars;

  return (
    <div className="autocomplete">
      <input
        type="text"
        role="combobox"
        aria-expanded={showList}
        aria-controls={listId}
        aria-autocomplete="list"
        aria-activedescendant={activeIndex >= 0 ? `${listId}-${activeIndex}` : undefined}
        value={query}
        onChange={(e) => { setQuery(e.target.value); setOpen(true); }}
        onKeyDown={handleKeyDown}
        onBlur={() => setOpen(false)}
        onFocus={() => query && setOpen(true)}
        placeholder="Search…"
      />
      {showList && (
        <ul id={listId} role="listbox" className="suggestions">
          {status === "loading" && <li aria-live="polite">Loading…</li>}
          {status === "error" && <li role="alert">Something went wrong.</li>}
          {status === "done" && results.length === 0 && <li>No results</li>}
          {status === "done" &&
            results.map((item, i) => (
              <li
                key={item.id}
                id={`${listId}-${i}`}
                role="option"
                aria-selected={i === activeIndex}
                className={i === activeIndex ? "active" : ""}
                onMouseEnter={() => setActiveIndex(i)}
                onMouseDown={(e) => { e.preventDefault(); select(item); }} // mousedown fires before input blur
              >
                <Highlight text={item.label} query={debounced} />
              </li>
            ))}
        </ul>
      )}
    </div>
  );
}

function Highlight({ text, query }) {
  if (!query) return text;
  const idx = text.toLowerCase().indexOf(query.toLowerCase());
  if (idx === -1) return text;
  return (
    <>
      {text.slice(0, idx)}
      <mark>{text.slice(idx, idx + query.length)}</mark>
      {text.slice(idx + query.length)}
    </>
  );
}

// usage
const fetchCountries = (q, signal) =>
  fetch(`/api/countries?q=${encodeURIComponent(q)}`, { signal })
    .then((r) => { if (!r.ok) throw new Error(r.status); return r.json(); });

<Autocomplete fetchSuggestions={fetchCountries} onSelect={(c) => console.log(c)} />;
```

Talking points: debounce vs throttle; AbortController for races; cache; `onMouseDown` + `preventDefault` to beat blur; `aria-activedescendant`; highlighting without `dangerouslySetInnerHTML`; `fetchSuggestions` should be stable (module-level or `useCallback`) to avoid refetching; scale-up: virtualize long lists, TanStack Query for caching.

---

# 3. Infinite Scroll ⭐

```jsx
function InfiniteFeed() {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [retryKey, setRetryKey] = useState(0); // bump to re-run the fetch for the same page
  const sentinelRef = useRef(null);

  useEffect(() => {
    let ignore = false;
    setLoading(true);
    setError(null);
    fetch(`/api/posts?page=${page}&limit=20`)
      .then((r) => r.json())
      .then((data) => {
        if (ignore) return;
        setItems((prev) => [...prev, ...data.items]);
        setHasMore(data.hasMore);
      })
      .catch((e) => !ignore && setError(e))
      .finally(() => !ignore && setLoading(false));
    return () => { ignore = true; };
  }, [page, retryKey]);

  useEffect(() => {
    const node = sentinelRef.current;
    if (!node || !hasMore || loading || error) return;
    const observer = new IntersectionObserver(
      ([entry]) => { if (entry.isIntersecting) setPage((p) => p + 1); },
      { rootMargin: "300px" }, // prefetch before reaching the bottom
    );
    observer.observe(node);
    return () => observer.disconnect();
  }, [hasMore, loading, error]);

  return (
    <div>
      {items.map((item) => <article key={item.id}>{item.title}</article>)}
      {loading && <p>Loading…</p>}
      {error && <button onClick={() => setRetryKey((k) => k + 1)}>Retry</button>}
      {!hasMore && <p>You've reached the end.</p>}
      <div ref={sentinelRef} aria-hidden="true" />
    </div>
  );
}
```

Talking points: IntersectionObserver vs scroll listeners; prevent duplicate loads (guard on `loading`); dedupe items; virtualization for thousands of items; "Load more" button fallback for accessibility; with TanStack Query use `useInfiniteQuery`.

---

# 4. Pagination ⭐

```jsx
function getPageNumbers(current, total, siblings = 1) {
  const pages = new Set([1, total, current]);
  for (let i = 1; i <= siblings; i++) {
    pages.add(current - i);
    pages.add(current + i);
  }
  const sorted = [...pages].filter((p) => p >= 1 && p <= total).sort((a, b) => a - b);
  const result = [];
  sorted.forEach((p, i) => {
    if (i > 0 && p - sorted[i - 1] > 1) result.push("…");
    result.push(p);
  });
  return result; // e.g. [1, "…", 4, 5, 6, "…", 20]
}

function Pagination({ page, totalPages, onPageChange }) {
  if (totalPages <= 1) return null;
  return (
    <nav aria-label="Pagination">
      <button onClick={() => onPageChange(page - 1)} disabled={page === 1}>Previous</button>
      {getPageNumbers(page, totalPages).map((p, i) =>
        p === "…" ? (
          <span key={`gap-${i}`} aria-hidden="true">…</span>
        ) : (
          <button key={p} onClick={() => onPageChange(p)} aria-current={p === page ? "page" : undefined}>
            {p}
          </button>
        ),
      )}
      <button onClick={() => onPageChange(page + 1)} disabled={page === totalPages}>Next</button>
    </nav>
  );
}
```

Keep `page` in the URL (`useSearchParams`); client-side: `items.slice((page - 1) * size, page * size)`.

---

# 5. Modal ⭐

See full accessible version in `26-accessibility.md` §6. Core checklist: portal, backdrop click closes (but not content click), Escape closes, focus moves in, focus trap, restore focus, scroll lock, `role="dialog"` + `aria-modal` + `aria-labelledby`. Mention native `<dialog>` alternative.

---

# 6. Tabs & Accordion ⭐

Compound component Tabs with ARIA roles: `16-component-patterns.md` §6.

```jsx
function Accordion({ items, allowMultiple = false }) {
  const [openIds, setOpenIds] = useState(() => new Set());

  function toggle(id) {
    setOpenIds((prev) => {
      const next = new Set(allowMultiple ? prev : []);
      if (prev.has(id)) next.delete(id);
      else next.add(id);
      return next; // new Set → new reference
    });
  }

  return (
    <div>
      {items.map(({ id, title, content }) => {
        const isOpen = openIds.has(id);
        return (
          <div key={id}>
            <h3>
              <button aria-expanded={isOpen} aria-controls={`panel-${id}`} id={`header-${id}`} onClick={() => toggle(id)}>
                {title} <span aria-hidden="true">{isOpen ? "−" : "+"}</span>
              </button>
            </h3>
            <div id={`panel-${id}`} role="region" aria-labelledby={`header-${id}`} hidden={!isOpen}>
              {content}
            </div>
          </div>
        );
      })}
    </div>
  );
}
```

---

# 7. Star Rating ⭐

```jsx
function StarRating({ max = 5, value, onChange, readOnly = false }) {
  const [hover, setHover] = useState(0);
  const display = hover || value;

  return (
    <div role="radiogroup" aria-label="Rating" onMouseLeave={() => setHover(0)}>
      {Array.from({ length: max }, (_, i) => {
        const star = i + 1;
        return (
          <button
            key={star}
            type="button"
            role="radio"
            aria-checked={value === star}
            aria-label={`${star} star${star > 1 ? "s" : ""}`}
            disabled={readOnly}
            onMouseEnter={() => !readOnly && setHover(star)}
            onClick={() => onChange(star === value ? 0 : star)} // click again to clear
            style={{ color: star <= display ? "gold" : "#ccc", fontSize: 24, background: "none", border: 0 }}
          >
            ★
          </button>
        );
      })}
    </div>
  );
}
```

Extensions: half stars (pointer x-position), keyboard arrows (roving tabindex).

---

# 8. OTP Input ⭐

```jsx
function OtpInput({ length = 6, onComplete }) {
  const [values, setValues] = useState(() => Array(length).fill(""));
  const inputsRef = useRef([]);

  function focus(i) {
    inputsRef.current[Math.max(0, Math.min(i, length - 1))]?.focus();
  }

  function update(next) {
    setValues(next);
    if (next.every((v) => v !== "")) onComplete?.(next.join(""));
  }

  function handleChange(i, e) {
    const digit = e.target.value.replace(/\D/g, "").slice(-1); // last typed digit
    if (!digit) return;
    const next = [...values];
    next[i] = digit;
    update(next);
    focus(i + 1);
  }

  function handleKeyDown(i, e) {
    if (e.key === "Backspace") {
      e.preventDefault();
      const next = [...values];
      if (next[i]) {
        next[i] = "";
      } else if (i > 0) {
        next[i - 1] = "";
        focus(i - 1);
      }
      setValues(next);
    }
    if (e.key === "ArrowLeft") focus(i - 1);
    if (e.key === "ArrowRight") focus(i + 1);
  }

  function handlePaste(e) {
    e.preventDefault();
    const digits = e.clipboardData.getData("text").replace(/\D/g, "").slice(0, length).split("");
    if (!digits.length) return;
    const next = [...values];
    digits.forEach((d, idx) => (next[idx] = d));
    update(next);
    focus(digits.length);
  }

  return (
    <div role="group" aria-label="One-time password">
      {values.map((v, i) => (
        <input
          key={i}
          ref={(el) => (inputsRef.current[i] = el)}
          value={v}
          inputMode="numeric"
          autoComplete={i === 0 ? "one-time-code" : "off"}
          maxLength={1}
          aria-label={`Digit ${i + 1}`}
          onChange={(e) => handleChange(i, e)}
          onKeyDown={(e) => handleKeyDown(i, e)}
          onPaste={handlePaste}
          onFocus={(e) => e.target.select()}
          style={{ width: 40, textAlign: "center" }}
        />
      ))}
    </div>
  );
}
```

---

# 9. Countdown Timer / Stopwatch ⭐

```jsx
function Countdown({ seconds: initial = 60, onEnd }) {
  const [remaining, setRemaining] = useState(initial * 1000);
  const [running, setRunning] = useState(false);
  const endTimeRef = useRef(null);

  useEffect(() => {
    if (!running) return;
    endTimeRef.current = Date.now() + remaining; // based on wall clock → no drift
    const id = setInterval(() => {
      const left = Math.max(0, endTimeRef.current - Date.now());
      setRemaining(left);
      if (left === 0) {
        setRunning(false);
        onEnd?.();
      }
    }, 250);
    return () => clearInterval(id);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [running]);

  const totalSec = Math.ceil(remaining / 1000);
  const mm = String(Math.floor(totalSec / 60)).padStart(2, "0");
  const ss = String(totalSec % 60).padStart(2, "0");

  return (
    <div>
      <p aria-live="off" style={{ fontSize: 32 }}>{mm}:{ss}</p>
      <button onClick={() => setRunning((r) => !r)} disabled={remaining === 0}>{running ? "Pause" : "Start"}</button>
      <button onClick={() => { setRunning(false); setRemaining(initial * 1000); }}>Reset</button>
    </div>
  );
}
```

Talking points: compute from timestamps (intervals drift and are throttled in background tabs); cleanup on pause/unmount; pause preserves `remaining`.

---

# 10. Progress Bar (with queue) ⭐

"Clicking Add starts a progress bar that fills in 2s; multiple bars run at most 3 concurrently."

```jsx
function ProgressBar({ onDone }) {
  const [width, setWidth] = useState(0);
  useEffect(() => {
    const raf = requestAnimationFrame(() => setWidth(100)); // trigger CSS transition after mount
    return () => cancelAnimationFrame(raf);
  }, []);
  return (
    <div className="track" role="progressbar" aria-valuenow={width} aria-valuemin={0} aria-valuemax={100}>
      <div
        className="fill"
        style={{ width: `${width}%`, transition: "width 2s linear" }}
        onTransitionEnd={onDone}
      />
    </div>
  );
}

function ProgressQueue({ concurrency = 3 }) {
  const [bars, setBars] = useState([]); // { id, status: "queued" | "running" | "done" }

  useEffect(() => {
    const running = bars.filter((b) => b.status === "running").length;
    const slots = concurrency - running;
    if (slots <= 0) return;
    const toStart = bars.filter((b) => b.status === "queued").slice(0, slots).map((b) => b.id);
    if (toStart.length) {
      setBars((prev) => prev.map((b) => (toStart.includes(b.id) ? { ...b, status: "running" } : b)));
    }
  }, [bars, concurrency]);

  const add = () => setBars((prev) => [...prev, { id: crypto.randomUUID(), status: "queued" }]);
  const done = (id) => setBars((prev) => prev.map((b) => (b.id === id ? { ...b, status: "done" } : b)));

  return (
    <>
      <button onClick={add}>Add</button>
      {bars.map((b) =>
        b.status === "queued" ? (
          <div key={b.id} className="track" />
        ) : (
          <ProgressBar key={b.id} onDone={() => done(b.id)} />
        ),
      )}
    </>
  );
}
```

(Here the effect sets state derived from state — acceptable because it models a scheduler; alternatively compute "running" as the first N not-done bars during render.)

---

# 11. File Explorer (recursive tree) ⭐

```jsx
const data = {
  id: "root", name: "root", type: "folder",
  children: [
    { id: "1", name: "src", type: "folder", children: [{ id: "2", name: "App.jsx", type: "file" }] },
    { id: "3", name: "package.json", type: "file" },
  ],
};

function addNode(tree, parentId, node) {
  if (tree.id === parentId) return { ...tree, children: [...(tree.children ?? []), node] };
  if (!tree.children) return tree;
  return { ...tree, children: tree.children.map((c) => addNode(c, parentId, node)) };
}

function deleteNode(tree, id) {
  if (!tree.children) return tree;
  return { ...tree, children: tree.children.filter((c) => c.id !== id).map((c) => deleteNode(c, id)) };
}

function FileExplorer() {
  const [tree, setTree] = useState(data);
  const onAdd = (parentId, type) => {
    const name = prompt(`New ${type} name`); // use an inline input in a real implementation
    if (name) setTree((t) => addNode(t, parentId, { id: crypto.randomUUID(), name, type, children: type === "folder" ? [] : undefined }));
  };
  const onDelete = (id) => setTree((t) => deleteNode(t, id));
  return <ul role="tree"><TreeNode node={tree} onAdd={onAdd} onDelete={onDelete} /></ul>;
}

function TreeNode({ node, onAdd, onDelete, level = 1 }) {
  const [open, setOpen] = useState(level === 1);
  const isFolder = node.type === "folder";
  return (
    <li role="treeitem" aria-expanded={isFolder ? open : undefined} aria-level={level}>
      <span onClick={() => isFolder && setOpen((o) => !o)} style={{ cursor: isFolder ? "pointer" : "default" }}>
        {isFolder ? (open ? "📂" : "📁") : "📄"} {node.name}
      </span>
      {isFolder && (
        <>
          <button onClick={() => onAdd(node.id, "file")} aria-label={`Add file to ${node.name}`}>+📄</button>
          <button onClick={() => onAdd(node.id, "folder")} aria-label={`Add folder to ${node.name}`}>+📁</button>
        </>
      )}
      {node.id !== "root" && <button onClick={() => onDelete(node.id)} aria-label={`Delete ${node.name}`}>🗑</button>}
      {isFolder && open && node.children?.length > 0 && (
        <ul role="group" style={{ paddingLeft: 16 }}>
          {node.children.map((child) => (
            <TreeNode key={child.id} node={child} onAdd={onAdd} onDelete={onDelete} level={level + 1} />
          ))}
        </ul>
      )}
    </li>
  );
}
```

Talking points: recursive components, immutable recursive updates, normalized alternative (`{ byId, childrenIds }`) for big trees, sorting folders first, rename, keyboard tree navigation. Same pattern = **nested comments**.

---

# 12. Data Table (sort + filter + paginate) ⭐

```jsx
function DataTable({ rows, columns, pageSize = 10 }) {
  const [query, setQuery] = useState("");
  const [sort, setSort] = useState({ key: null, dir: "asc" });
  const [page, setPage] = useState(1);
  const deferredQuery = useDeferredValue(query);

  const filtered = useMemo(() => {
    const q = deferredQuery.toLowerCase();
    return q ? rows.filter((r) => columns.some((c) => String(r[c.key]).toLowerCase().includes(q))) : rows;
  }, [rows, columns, deferredQuery]);

  const sorted = useMemo(() => {
    if (!sort.key) return filtered;
    return [...filtered].sort((a, b) => {
      const x = a[sort.key], y = b[sort.key];
      const cmp = typeof x === "number" && typeof y === "number" ? x - y : String(x).localeCompare(String(y));
      return sort.dir === "asc" ? cmp : -cmp;
    });
  }, [filtered, sort]);

  const totalPages = Math.max(1, Math.ceil(sorted.length / pageSize));
  const current = Math.min(page, totalPages); // derived clamp instead of effect
  const pageRows = sorted.slice((current - 1) * pageSize, current * pageSize);

  const toggleSort = (key) =>
    setSort((s) => (s.key === key ? { key, dir: s.dir === "asc" ? "desc" : "asc" } : { key, dir: "asc" }));

  return (
    <div>
      <input type="search" placeholder="Filter…" value={query} onChange={(e) => { setQuery(e.target.value); setPage(1); }} />
      <table>
        <thead>
          <tr>
            {columns.map((c) => (
              <th key={c.key} aria-sort={sort.key === c.key ? (sort.dir === "asc" ? "ascending" : "descending") : "none"}>
                <button onClick={() => toggleSort(c.key)}>
                  {c.label} {sort.key === c.key ? (sort.dir === "asc" ? "▲" : "▼") : ""}
                </button>
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {pageRows.length ? (
            pageRows.map((r) => (
              <tr key={r.id}>{columns.map((c) => <td key={c.key}>{c.render ? c.render(r) : r[c.key]}</td>)}</tr>
            ))
          ) : (
            <tr><td colSpan={columns.length}>No matching rows</td></tr>
          )}
        </tbody>
      </table>
      <Pagination page={current} totalPages={totalPages} onPageChange={setPage} />
    </div>
  );
}
```

At scale: server-side sort/filter/paginate (state in URL, TanStack Query), TanStack Table, virtualization.

---

# 13. Virtualized List from Scratch ⭐

```jsx
function VirtualList({ items, itemHeight = 40, height = 400, overscan = 5, renderItem }) {
  const [scrollTop, setScrollTop] = useState(0);

  const totalHeight = items.length * itemHeight;
  const start = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan);
  const end = Math.min(items.length, Math.ceil((scrollTop + height) / itemHeight) + overscan);
  const visible = items.slice(start, end);

  return (
    <div style={{ height, overflowY: "auto" }} onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}>
      <div style={{ height: totalHeight, position: "relative" }}>
        <div style={{ transform: `translateY(${start * itemHeight}px)` }}>
          {visible.map((item, i) => (
            <div key={item.id} style={{ height: itemHeight }}>
              {renderItem(item, start + i)}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

Explain: spacer height = total; only `end - start` rows rendered; overscan for smooth scrolling; dynamic heights need measurement (ResizeObserver) and offset caches.

---

# 14. Tic-Tac-Toe ⭐

```jsx
const LINES = [[0,1,2],[3,4,5],[6,7,8],[0,3,6],[1,4,7],[2,5,8],[0,4,8],[2,4,6]];

function getWinner(squares) {
  for (const [a, b, c] of LINES) {
    if (squares[a] && squares[a] === squares[b] && squares[a] === squares[c]) return { player: squares[a], line: [a, b, c] };
  }
  return null;
}

function TicTacToe() {
  const [history, setHistory] = useState([Array(9).fill(null)]);
  const [step, setStep] = useState(0);
  const squares = history[step];
  const xIsNext = step % 2 === 0;           // derived — no extra state
  const winner = getWinner(squares);
  const isDraw = !winner && squares.every(Boolean);

  function play(i) {
    if (squares[i] || winner) return;
    const next = squares.slice();
    next[i] = xIsNext ? "X" : "O";
    setHistory([...history.slice(0, step + 1), next]); // drop "future" when going back in time
    setStep(step + 1);
  }

  const status = winner ? `Winner: ${winner.player}` : isDraw ? "Draw!" : `Next: ${xIsNext ? "X" : "O"}`;

  return (
    <div>
      <p role="status">{status}</p>
      <div style={{ display: "grid", gridTemplateColumns: "repeat(3, 60px)", gap: 4 }}>
        {squares.map((value, i) => (
          <button
            key={i}
            onClick={() => play(i)}
            aria-label={`Square ${i + 1}${value ? `, ${value}` : ""}`}
            style={{ height: 60, fontSize: 24, background: winner?.line.includes(i) ? "#bbf7d0" : undefined }}
          >
            {value}
          </button>
        ))}
      </div>
      <ol>
        {history.map((_, move) => (
          <li key={move}><button onClick={() => setStep(move)}>{move ? `Go to move #${move}` : "Restart"}</button></li>
        ))}
      </ol>
    </div>
  );
}
```

Extension: N×N board with configurable win length.

---

# 15. More Problems (specs + hints)

| Problem | Key points |
| --- | --- |
| **Multi-step form wizard** | Steps array; per-step validation (Zod); state in one reducer/RHF; back/next; progress indicator; persist draft; summary step |
| **Image carousel** | Index state with wrap-around; autoplay with pause on hover/focus; cleanup interval; keyboard arrows; dots; lazy-load images; `aria-roledescription="carousel"`, live region |
| **Kanban board (drag & drop)** | Columns `{ id, taskIds }` + tasks map; HTML5 DnD (`draggable`, `onDragStart` dataTransfer, `onDragOver` preventDefault, `onDrop`) or dnd-kit; keyboard alternative; persist |
| **Nested comments** | Recursive component (like File Explorer); reply/edit/delete; collapse threads; optimistic add |
| **Traffic light** | Config `{ red: 4000, green: 3000, yellow: 1000 }`; `setTimeout` chain driven by state; cleanup |
| **Chips / tag input** | Enter/comma adds, Backspace removes last, dedupe, max count, paste multiple |
| **Toast notifications** | Context + portal; queue; auto-dismiss timers (pause on hover); `role="status"`/`alert` |
| **Dropdown / select** | Click outside, keyboard nav, portal positioning (Floating UI), listbox roles |
| **Nested checkboxes** | Tree state; parent checked/indeterminate derived from children (`ref.current.indeterminate`) |
| **Grid light-cells / memory game** | Order tracking; deactivate in reverse with timeouts; cleanup |
| **Poll widget / Like button** | Optimistic update + rollback; disable during request; error state |
| **Typing speed test** | Timer from timestamps; derived WPM/accuracy; per-character diff rendering |
| **Calendar / date picker** | Month grid calc (`new Date(y, m, 1).getDay()`), keyboard nav, disabled dates, i18n with `Intl` |
| **Shopping cart** | Reducer; derived totals; coupon; persist; quantity bounds |
| **Snake game** | Game loop with `setInterval`/rAF; direction ref; collision; cleanup |

---

# 16. Final Checklist for Every Solution

- [ ] Works for the happy path
- [ ] Loading, error, empty states
- [ ] No derived data stored in state
- [ ] Stable keys; immutable updates
- [ ] Effects cleaned up (timers, listeners, observers, requests)
- [ ] Race conditions handled for async inputs
- [ ] Keyboard accessible; semantic HTML; labels
- [ ] Reasonable component split
- [ ] You can explain trade-offs and next steps (tests, perf, a11y)
