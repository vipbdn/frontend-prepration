# 04 — Events & Forms

> **Interview + Understanding Guide** — Roadmap Section 4
>
> Goal: Understand React's event system (synthetic events, delegation at the root), handle forms controlled and uncontrolled, validate properly, and build production forms with React Hook Form + Zod and React 19 form actions.

---

# 1. Handling Events

```jsx
function Toolbar() {
  function handleClick(e) {
    console.log("clicked", e.currentTarget);
  }
  return (
    <>
      <button onClick={handleClick}>Save</button>
      <button onClick={() => remove(id)}>Delete</button> {/* pass arguments */}
    </>
  );
}
```

- camelCase names: `onClick`, `onChange`, `onSubmit`, `onKeyDown`, `onMouseEnter`.
- Pass a **function**, don't call it: `onClick={handleClick}`, not `onClick={handleClick()}`.
- Handlers are the right place for **side effects** (API calls, navigation, state updates).

### Naming convention

- Handlers inside a component: `handleX` (`handleSubmit`)
- Props that accept handlers: `onX` (`onSubmit`, `onSelect`)

---

# 2. Synthetic Events ⭐

React wraps native events in a **SyntheticEvent** — a cross-browser wrapper with the same interface (`target`, `currentTarget`, `preventDefault()`, `stopPropagation()`, `key`, …). The native event is at `e.nativeEvent`.

### Event delegation at the root ⭐

React doesn't attach listeners to each element. Since **React 17**, it attaches **one listener per event type to the root container** (`#root`) and dispatches to the right handlers by walking the fiber tree.

```text
React 16: listeners on `document`
React 17+: listeners on the root container → multiple React apps / versions can coexist on one page
```

Benefits: less memory, handlers work for dynamically rendered elements, consistent behaviour.

### Event pooling (legacy)

React ≤16 reused event objects (you had to call `e.persist()` to use them asynchronously). **Removed in React 17** — you can use `e` in async code now.

### Propagation

```jsx
<div onClick={() => console.log("div")}>
  <button
    onClick={(e) => {
      e.stopPropagation(); // stops bubbling to the div
      console.log("button");
    }}
  >
    Click
  </button>
</div>
```

- All React events **bubble** except `onScroll`.
- Capture phase: append `Capture` → `onClickCapture`.
- React's `onChange` fires on **every keystroke** (like native `input` event), unlike native `change`.
- `onMouseEnter`/`onMouseLeave` don't bubble (same as DOM).
- `onFocus`/`onBlur` in React **do** bubble (React uses `focusin`/`focusout`).

### Mixing native listeners

Native listeners added with `addEventListener` on `document` run **after** React's root listener for bubbling events. Calling `e.stopPropagation()` in React stops React handlers but native `document` listeners may still fire — a common source of "click outside" bugs.

---

# 3. Controlled Forms ⭐

```jsx
function SignupForm() {
  const [form, setForm] = useState({ email: "", password: "", role: "dev", agree: false });
  const [errors, setErrors] = useState({});

  function handleChange(e) {
    const { name, value, type, checked } = e.target;
    setForm((f) => ({ ...f, [name]: type === "checkbox" ? checked : value }));
  }

  function validate(values) {
    const errs = {};
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(values.email)) errs.email = "Invalid email";
    if (values.password.length < 8) errs.password = "Min 8 characters";
    if (!values.agree) errs.agree = "Required";
    return errs;
  }

  function handleSubmit(e) {
    e.preventDefault(); // stop full page reload
    const errs = validate(form);
    setErrors(errs);
    if (Object.keys(errs).length === 0) api.signup(form);
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <label>
        Email
        <input name="email" type="email" value={form.email} onChange={handleChange}
          aria-invalid={!!errors.email} aria-describedby="email-error" />
      </label>
      {errors.email && <p id="email-error" role="alert">{errors.email}</p>}

      <input name="password" type="password" value={form.password} onChange={handleChange} />

      <select name="role" value={form.role} onChange={handleChange}>
        <option value="dev">Developer</option>
        <option value="design">Designer</option>
      </select>

      <label>
        <input name="agree" type="checkbox" checked={form.agree} onChange={handleChange} /> I agree
      </label>

      <button type="submit">Sign up</button>
    </form>
  );
}
```

| Element | Controlled prop |
| --- | --- |
| `input` text, `textarea`, `select` | `value` |
| checkbox / radio | `checked` |
| file | ❌ always uncontrolled (read `e.target.files`) |
| `select multiple` | `value={["a", "b"]}` |

`<textarea value={...}>` and `<select value={...}>` — React normalizes these (no `selected` on options, no text children in textarea).

---

# 4. Uncontrolled Forms with `FormData` ⭐

Often simpler and faster for submit-only forms:

```jsx
function ContactForm() {
  function handleSubmit(e) {
    e.preventDefault();
    const data = Object.fromEntries(new FormData(e.currentTarget));
    // { name: "...", message: "..." }
    api.send(data);
    e.currentTarget.reset();
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" required defaultValue="" />
      <textarea name="message" required minLength={10} />
      <button>Send</button>
    </form>
  );
}
```

Native validation attributes (`required`, `minLength`, `pattern`, `type="email"`) work for free.

---

# 5. React 19 Form Actions

Pass a **function** to `<form action>`. React calls it with `FormData`, manages pending state, and resets uncontrolled fields after success.

```jsx
import { useActionState } from "react";
import { useFormStatus } from "react-dom";

async function subscribe(prevState, formData) {
  const email = formData.get("email");
  const res = await api.subscribe(email);
  return res.ok ? { ok: true, message: "Subscribed!" } : { ok: false, message: res.error };
}

function SubmitButton() {
  const { pending } = useFormStatus(); // status of the parent <form>
  return <button disabled={pending}>{pending ? "Saving…" : "Subscribe"}</button>;
}

function Newsletter() {
  const [state, formAction, isPending] = useActionState(subscribe, { ok: null, message: "" });
  return (
    <form action={formAction}>
      <input name="email" type="email" required />
      <SubmitButton />
      {state.message && <p role="status">{state.message}</p>}
    </form>
  );
}
```

Works with Server Actions in Next.js too (progressive enhancement). Details in `20-react-19-features.md`.

---

# 6. Validation Strategies

| When | UX |
| --- | --- |
| On submit | Simple; errors appear late |
| On blur ("touched") | Good balance — validate a field when the user leaves it |
| On change | Instant feedback; can be noisy — combine with "touched" |
| Async (e.g. username available) | Debounce + abort stale requests |

Always validate **on the server too** — client validation is UX, not security.

---

# 7. React Hook Form + Zod ⭐ (industry standard)

**Why:** controlled forms re-render on every keystroke and need boilerplate. React Hook Form uses **uncontrolled inputs + refs**, so typing doesn't re-render the whole form; it handles touched/dirty state, errors, arrays, async validation. **Zod** defines the schema once and gives TypeScript types.

```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z
  .object({
    email: z.string().email("Invalid email"),
    password: z.string().min(8, "Min 8 characters"),
    confirm: z.string(),
    age: z.coerce.number().int().min(18, "Must be 18+"),
  })
  .refine((d) => d.password === d.confirm, { message: "Passwords must match", path: ["confirm"] });

type FormValues = z.infer<typeof schema>;

function RegisterForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    reset,
  } = useForm<FormValues>({ resolver: zodResolver(schema), mode: "onBlur" });

  const onSubmit = async (values: FormValues) => {
    await api.register(values);
    reset();
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <input {...register("email")} placeholder="Email" />
      {errors.email && <p role="alert">{errors.email.message}</p>}

      <input type="password" {...register("password")} />
      {errors.password && <p role="alert">{errors.password.message}</p>}

      <input type="password" {...register("confirm")} />
      {errors.confirm && <p role="alert">{errors.confirm.message}</p>}

      <input type="number" {...register("age")} />
      {errors.age && <p role="alert">{errors.age.message}</p>}

      <button disabled={isSubmitting}>Register</button>
    </form>
  );
}
```

Other APIs worth knowing: `Controller` (wrap controlled UI-library inputs), `useFieldArray` (dynamic lists), `watch`, `setValue`, `setError` (server errors), `defaultValues` for edit forms. Alternative: Formik (older, more re-renders).

---

# 8. Dynamic Form Fields (common machine-coding task)

```jsx
function PhoneList() {
  const [phones, setPhones] = useState([{ id: crypto.randomUUID(), value: "" }]);

  const add = () => setPhones((p) => [...p, { id: crypto.randomUUID(), value: "" }]);
  const remove = (id) => setPhones((p) => p.filter((x) => x.id !== id));
  const update = (id, value) => setPhones((p) => p.map((x) => (x.id === id ? { ...x, value } : x)));

  return (
    <>
      {phones.map((p) => (
        <div key={p.id}>
          <input value={p.value} onChange={(e) => update(p.id, e.target.value)} />
          <button type="button" onClick={() => remove(p.id)}>✕</button>
        </div>
      ))}
      <button type="button" onClick={add}>Add phone</button>
    </>
  );
}
```

Note `type="button"` — buttons inside a form default to `type="submit"`.

---

# 9. Common Event/Form Gotchas

- Forgetting `e.preventDefault()` in `onSubmit` → page reloads.
- `<button>` inside `<form>` defaults to submit.
- Reading `e.target.value` inside a `setState` updater after the event (fine since React 17, but prefer reading it first).
- Number inputs give **strings** → convert (`valueAsNumber`, `Number()`, `z.coerce`).
- Controlled input with `value={undefined}` initially → uncontrolled→controlled warning.
- Typing lag in big controlled forms → React Hook Form, split components, `useDeferredValue`.
- IME composition (Chinese/Japanese input): avoid aggressive formatting on every `onChange`.

---

# 10. Interview Answers

**How does React handle events?**

> "React uses synthetic events — a cross-browser wrapper around native events. Instead of attaching listeners to each element, React 17+ attaches one listener per event type at the root container and dispatches to handlers through the component tree. Event pooling was removed in React 17."

**How do you build large forms?**

> "For anything non-trivial I use React Hook Form with a Zod schema: inputs are uncontrolled so typing doesn't re-render the whole form, the schema drives both validation and TypeScript types, and it handles touched, dirty, submitting and field arrays. I validate on blur for UX and always re-validate on the server."

---

# 11. Key Takeaways

- Pass functions, not calls; `handleX` / `onX` naming.
- Synthetic events, root-level delegation (React 17+), no pooling.
- Controlled: `value`/`checked` + `onChange`; file inputs are uncontrolled.
- Uncontrolled + `FormData` is simple for submit-only forms.
- React 19: `<form action>`, `useActionState`, `useFormStatus`.
- Production: React Hook Form + Zod; server-side validation always.

## Readiness Checklist

- [ ] Explain synthetic events and delegation changes in React 17
- [ ] Build a controlled form with one generic `handleChange`
- [ ] Build the same form uncontrolled with `FormData`
- [ ] Build a RHF + Zod form with cross-field validation
- [ ] Build dynamic add/remove fields
