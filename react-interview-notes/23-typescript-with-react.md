# 23 — TypeScript with React

> **Interview + Understanding Guide** — Roadmap Section 23
>
> Goal: Type components, props, children, events, hooks, refs, context, reducers and generic components confidently, and use TypeScript patterns (discriminated unions, utility types, polymorphism) that make React code safer.

---

# 1. Typing Props

```tsx
type ButtonProps = {
  label: string;
  variant?: "primary" | "secondary" | "danger"; // union of literals
  disabled?: boolean;
  onClick: (event: React.MouseEvent<HTMLButtonElement>) => void;
};

function Button({ label, variant = "primary", disabled = false, onClick }: ButtonProps) {
  return <button className={variant} disabled={disabled} onClick={onClick}>{label}</button>;
}
```

### `type` vs `interface`

Both work. `interface` supports declaration merging and `extends`; `type` supports unions/intersections and mapped types. Pick one convention per codebase (many use `type` for props).

### Avoid `React.FC`?

```tsx
const Button: React.FC<ButtonProps> = (props) => …; // fine since React 18 (no implicit children)
function Button(props: ButtonProps) { … }          // most common today — simpler, supports generics
```

### Extending native element props ⭐

```tsx
type InputProps = React.ComponentPropsWithoutRef<"input"> & {
  label: string;
  error?: string;
};

function Input({ label, error, id, ...rest }: InputProps) {
  const autoId = useId();
  const inputId = id ?? autoId;
  return (
    <>
      <label htmlFor={inputId}>{label}</label>
      <input id={inputId} aria-invalid={!!error} {...rest} />
      {error && <span role="alert">{error}</span>}
    </>
  );
}
// <Input label="Email" type="email" placeholder="…" onBlur={…} /> — all native props typed
```

| Helper | Use |
| --- | --- |
| `React.ComponentProps<"button">` | Props of a DOM element (includes `ref` in React 19) |
| `React.ComponentPropsWithoutRef<"button">` | Without `ref` |
| `React.ComponentPropsWithRef<"button">` | With `ref` |
| `React.ComponentProps<typeof MyComp>` | Props of an existing component |
| `React.HTMLAttributes<HTMLDivElement>` | Generic HTML attributes |

---

# 2. Typing `children`

```tsx
type CardProps = { title: string; children: React.ReactNode };
```

| Type | Accepts |
| --- | --- |
| `React.ReactNode` | Anything renderable: elements, strings, numbers, arrays, `null`, `undefined`, booleans ✅ (use this) |
| `React.ReactElement` | A single JSX element only |
| `JSX.Element` | Return type of JSX expressions (similar to ReactElement) |
| `(data: T) => React.ReactNode` | Render prop |
| `React.PropsWithChildren<P>` | Adds `children?: ReactNode` to P |

---

# 3. Typing Events ⭐

```tsx
function Form() {
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => setValue(e.target.value);
  const handleSelect = (e: React.ChangeEvent<HTMLSelectElement>) => setOption(e.target.value);
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const data = new FormData(e.currentTarget);
  };
  const handleKey = (e: React.KeyboardEvent<HTMLInputElement>) => { if (e.key === "Enter") submit(); };
  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => console.log(e.clientX);
  const handleFocus = (e: React.FocusEvent<HTMLInputElement>) => {};
}
```

Shortcut types for props: `React.ChangeEventHandler<HTMLInputElement>`, `React.MouseEventHandler<HTMLButtonElement>`.

Tip: write the handler inline once, hover to see the inferred type.

`e.target` vs `e.currentTarget`: `currentTarget` is typed as the element the handler is attached to; `target` is `EventTarget` (could be a child).

---

# 4. Typing Hooks ⭐

### useState

```tsx
const [count, setCount] = useState(0);                     // inferred number
const [user, setUser] = useState<User | null>(null);       // explicit union when starting empty
const [items, setItems] = useState<Item[]>([]);            // empty array needs a type
const [status, setStatus] = useState<"idle" | "loading" | "error">("idle");
```

### useRef

```tsx
const inputRef = useRef<HTMLInputElement>(null);   // DOM ref → RefObject<HTMLInputElement | null>
inputRef.current?.focus();

const timerRef = useRef<number | null>(null);       // mutable value
timerRef.current = window.setTimeout(…);
```

(React 19 types: `useRef` requires an argument; refs are always mutable `RefObject<T>`.)

### useReducer with discriminated unions ⭐

```tsx
type State = { status: "idle" | "loading" | "success" | "error"; data: Todo[]; error?: string };

type Action =
  | { type: "fetch_start" }
  | { type: "fetch_success"; payload: Todo[] }
  | { type: "fetch_error"; error: string }
  | { type: "todo_toggled"; id: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "fetch_start":
      return { ...state, status: "loading" };
    case "fetch_success":
      return { status: "success", data: action.payload }; // payload typed as Todo[] here
    case "fetch_error":
      return { ...state, status: "error", error: action.error };
    case "todo_toggled":
      return { ...state, data: state.data.map((t) => (t.id === action.id ? { ...t, done: !t.done } : t)) };
    default: {
      const _exhaustive: never = action; // compile error if a case is missing
      return state;
    }
  }
}
```

### useContext with a safe hook

```tsx
type AuthContextValue = { user: User | null; login: (c: Credentials) => Promise<void>; logout: () => void };

const AuthContext = createContext<AuthContextValue | null>(null);

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used within AuthProvider");
  return ctx; // narrowed to AuthContextValue
}
```

### useMemo / useCallback — usually inferred

```tsx
const total = useMemo(() => items.reduce((s, i) => s + i.price, 0), [items]); // number
const onSelect = useCallback((id: string) => setSelected(id), []);
```

### Custom hooks — return tuples `as const`

```tsx
function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn((v) => !v), []);
  return [on, toggle] as const; // readonly [boolean, () => void] — not (boolean | (() => void))[]
}
```

---

# 5. Discriminated Unions for Props ⭐

Model mutually exclusive props so invalid combinations don't compile.

```tsx
type AsyncState<T> =
  | { status: "loading" }
  | { status: "error"; error: Error }
  | { status: "success"; data: T };

function UserView({ state }: { state: AsyncState<User> }) {
  switch (state.status) {
    case "loading": return <Spinner />;
    case "error": return <p>{state.error.message}</p>;  // error available only here
    case "success": return <h1>{state.data.name}</h1>;  // data available only here
  }
}
```

```tsx
// Either a link or a button — never both href and onClick confusion
type ActionProps =
  | { as: "link"; href: string; onClick?: never }
  | { as: "button"; onClick: () => void; href?: never };
```

---

# 6. Generic Components ⭐

```tsx
type ListProps<T> = {
  items: T[];
  getKey: (item: T) => string | number;
  renderItem: (item: T) => React.ReactNode;
  emptyText?: string;
};

function List<T>({ items, getKey, renderItem, emptyText = "No items" }: ListProps<T>) {
  if (!items.length) return <p>{emptyText}</p>;
  return <ul>{items.map((item) => <li key={getKey(item)}>{renderItem(item)}</li>)}</ul>;
}

<List items={users} getKey={(u) => u.id} renderItem={(u) => u.name} /> // T inferred as User
```

Generic `Select`:

```tsx
type SelectProps<T> = {
  options: T[];
  value: T | null;
  onChange: (value: T) => void;
  getLabel: (option: T) => string;
};
```

In `.tsx` arrow functions, write `<T,>` to avoid JSX ambiguity: `const List = <T,>(props: ListProps<T>) => …`.

---

# 7. Polymorphic Components

```tsx
type PolymorphicProps<C extends React.ElementType, P = {}> = P & {
  as?: C;
} & Omit<React.ComponentPropsWithoutRef<C>, keyof P | "as">;

function Text<C extends React.ElementType = "span">({ as, ...rest }: PolymorphicProps<C, { size?: "sm" | "lg" }>) {
  const Component = as ?? "span";
  return <Component {...rest} />;
}

<Text as="label" htmlFor="x">Label</Text>   // htmlFor allowed
<Text as="a" href="/">Home</Text>             // href allowed
```

---

# 8. Useful Utility Types

```tsx
Partial<User>                  // all optional (edit forms)
Required<User>
Pick<User, "id" | "name">      // subset
Omit<User, "password">         // exclude
Record<Status, string>         // { idle: string; loading: string; … }
ReturnType<typeof useAuth>
Parameters<typeof fetchUser>
Awaited<ReturnType<typeof getUser>> // resolved type of a promise
NonNullable<T>
Extract<Action, { type: "fetch_success" }>
```

```tsx
// Derive types from runtime constants
const ROLES = ["admin", "editor", "viewer"] as const;
type Role = (typeof ROLES)[number]; // "admin" | "editor" | "viewer"

// Satisfies: validate without widening
const routes = { home: "/", profile: "/profile" } satisfies Record<string, string>;
```

### Types from schemas (single source of truth)

```tsx
const UserSchema = z.object({ id: z.string(), name: z.string(), email: z.string().email() });
type User = z.infer<typeof UserSchema>;
const user = UserSchema.parse(await res.json()); // runtime validation + static type
```

---

# 9. Typing API Calls

```tsx
async function fetchJSON<T>(url: string, init?: RequestInit): Promise<T> {
  const res = await fetch(url, init);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<T>; // ⚠️ a cast — validate with Zod for untrusted data
}

const { data } = useQuery({ queryKey: ["user", id], queryFn: () => fetchJSON<User>(`/api/users/${id}`) });
// data: User | undefined
```

---

# 10. Common Mistakes

- `any` everywhere (use `unknown` + narrowing)
- `useState([])` without a type → `never[]`
- Non-null assertions (`ref.current!`) instead of checks
- Typing `children` as `JSX.Element` (rejects strings/arrays)
- Casting API responses without validation
- Over-engineering types in app code — prefer readable types

---

# 11. Interview Answers

**How do you type a component that extends a native element?**

> "I intersect `React.ComponentPropsWithoutRef<'button'>` — or `ComponentProps` if I want `ref` — with my custom props, destructure mine, and spread the rest onto the element. That way consumers get every native attribute typed."

**How do you type a reducer?**

> "State as a type, actions as a discriminated union on `type`. In the switch TypeScript narrows each case so payloads are typed, and a `never` check in the default case catches missing cases at compile time."

**What's a generic component?**

> "A component parameterized by a type, like `List<T>` taking `items: T[]` and `renderItem: (item: T) => ReactNode`. TypeScript infers `T` from the props, so render callbacks are fully typed without casting."

---

# 12. Key Takeaways

- Props via `type`/`interface`; extend native props with `ComponentPropsWithoutRef<"el">`.
- `children: React.ReactNode`.
- Event types: `ChangeEvent`, `FormEvent`, `MouseEvent`, `KeyboardEvent` with element generics.
- `useState<T | null>(null)`, `useRef<HTMLInputElement>(null)`, discriminated-union reducers with exhaustive checks.
- Context with `null` default + throwing hook.
- Generic and polymorphic components; `as const` tuples.
- Zod for runtime validation + inferred types.

## Readiness Checklist

- [ ] Type an Input extending native props
- [ ] Type a reducer with exhaustive switch
- [ ] Write a generic List/Select component
- [ ] Model async state with a discriminated union
- [ ] Type a context + custom hook
