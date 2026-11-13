# 24 — Styling in React

> **Interview + Understanding Guide** — Roadmap Section 24
>
> Goal: Compare styling approaches (plain CSS, CSS Modules, Sass, utility-first Tailwind, CSS-in-JS runtime and zero-runtime), know their trade-offs with SSR/RSC and performance, and implement theming, dark mode and responsive design.

---

# 1. Approaches Overview

| Approach | Example | Scoping | Runtime cost | RSC/SSR friendly |
| --- | --- | --- | --- | --- |
| Global CSS | `import "./app.css"` | ❌ global | none | ✅ |
| **CSS Modules** | `styles.button` | ✅ local class names | none | ✅ |
| Sass/SCSS (+ modules) | variables, nesting, mixins | via modules | none | ✅ |
| **Tailwind CSS** | `className="px-4 py-2"` | utility classes | none | ✅ |
| Runtime CSS-in-JS | styled-components, Emotion | ✅ | ❌ JS at runtime | ⚠️ needs client components / setup |
| Zero-runtime CSS-in-JS | vanilla-extract, Panda CSS, Linaria, StyleX | ✅ | none (extracted at build) | ✅ |
| Inline styles | `style={{ color }}` | element | small | ✅ (no pseudo/media queries) |
| Component libraries | MUI, Chakra, Mantine, Ant Design, shadcn/ui | — | varies | varies |

---

# 2. Plain / Global CSS

```jsx
import "./Button.css";
<button className="btn btn-primary">Save</button>
```

Problems at scale: global namespace collisions, specificity wars, dead CSS hard to remove. Mitigated with conventions (BEM: `.card__title--active`).

---

# 3. CSS Modules ⭐

```css
/* Button.module.css */
.button { padding: 8px 16px; border-radius: 6px; }
.primary { background: var(--color-primary); color: white; }
.button:hover { opacity: 0.9; }
```

```jsx
import styles from "./Button.module.css";

function Button({ variant = "primary", className, ...props }) {
  return <button className={`${styles.button} ${styles[variant]} ${className ?? ""}`} {...props} />;
}
```

- Class names compiled to unique hashes (`Button_button__x7Fz2`) → **no collisions**.
- Plain CSS: pseudo-classes, media queries, zero runtime, works with SSR/RSC.
- Use `clsx`/`classnames` for conditional classes: `clsx(styles.button, isActive && styles.active)`.

---

# 4. Tailwind CSS ⭐ (very popular)

Utility-first: compose styles from small single-purpose classes.

```jsx
function Card({ title, children, highlighted }) {
  return (
    <div
      className={clsx(
        "rounded-xl border p-6 shadow-sm transition hover:shadow-md",
        "dark:border-gray-700 dark:bg-gray-900",
        highlighted && "border-blue-500 ring-2 ring-blue-200",
      )}
    >
      <h2 className="mb-2 text-lg font-semibold md:text-xl">{title}</h2>
      <div className="text-sm text-gray-600 dark:text-gray-300">{children}</div>
    </div>
  );
}
```

**Pros:** no naming, no dead CSS (only used classes are generated), consistent design tokens, fast to iterate, colocated styles, tiny CSS output, no runtime.
**Cons:** long class strings, learning curve, harder to read for some, needs discipline for consistent components.

Tooling: `tailwind-merge` (resolve conflicting classes when overriding), **`cva`** (class-variance-authority) for variants, Prettier plugin for class sorting.

```tsx
import { cva, type VariantProps } from "class-variance-authority";

const button = cva("inline-flex items-center rounded-md font-medium focus-visible:ring-2", {
  variants: {
    variant: { primary: "bg-blue-600 text-white hover:bg-blue-700", ghost: "hover:bg-gray-100" },
    size: { sm: "h-8 px-3 text-sm", md: "h-10 px-4" },
  },
  defaultVariants: { variant: "primary", size: "md" },
});

type ButtonProps = React.ComponentProps<"button"> & VariantProps<typeof button>;
export function Button({ variant, size, className, ...props }: ButtonProps) {
  return <button className={twMerge(button({ variant, size }), className)} {...props} />;
}
```

(This is the shadcn/ui approach.)

---

# 5. CSS-in-JS (runtime): styled-components / Emotion

```jsx
import styled, { css, ThemeProvider } from "styled-components";

const Button = styled.button`
  padding: 8px 16px;
  border-radius: 6px;
  background: ${({ theme }) => theme.colors.primary};
  ${({ $variant }) => $variant === "ghost" && css`background: transparent;`}
  &:hover { opacity: 0.9; }
  @media (max-width: 600px) { width: 100%; }
`;

<ThemeProvider theme={{ colors: { primary: "#4f46e5" } }}>
  <Button $variant="ghost">Save</Button>
</ThemeProvider>
```

**Pros:** scoped styles, dynamic styles from props, theming, colocated, full CSS features.
**Cons:** **runtime cost** (styles generated/injected during render → slower, especially with frequent re-renders), bigger bundles, SSR setup needed, **not compatible with Server Components** (needs client components; context-based theming). The React team and libraries have moved toward **zero-runtime** solutions; styled-components entered maintenance mode (2025).

---

# 6. Zero-Runtime CSS-in-JS

Write styles in TS/JS, extracted to static CSS at build time.

```ts
// vanilla-extract: button.css.ts
import { style } from "@vanilla-extract/css";
export const button = style({ padding: "8px 16px", ":hover": { opacity: 0.9 } });
```

Examples: vanilla-extract, Panda CSS, Linaria, Meta's **StyleX**, Pigment CSS (MUI). Type-safe tokens, no runtime, RSC-compatible.

---

# 7. Inline Styles

```jsx
<div style={{ width: `${progress}%`, backgroundColor: color }} />
```

Good for **truly dynamic values** (progress %, positions, user-chosen colors). Can't do pseudo-classes, media queries, keyframes; creates new objects each render. Combine with CSS variables:

```jsx
<div className="progress" style={{ "--progress": `${progress}%` }} />
// .progress::after { width: var(--progress); }
```

---

# 8. Theming & Dark Mode ⭐

### CSS custom properties (framework-agnostic, zero runtime)

```css
:root {
  --bg: #ffffff;
  --text: #111827;
  --primary: #4f46e5;
}
[data-theme="dark"] {
  --bg: #0b1120;
  --text: #e5e7eb;
}
body { background: var(--bg); color: var(--text); }
```

```jsx
function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage("theme", "system");

  useEffect(() => {
    const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;
    const resolved = theme === "system" ? (prefersDark ? "dark" : "light") : theme;
    document.documentElement.dataset.theme = resolved;
  }, [theme]);

  return (
    <select value={theme} onChange={(e) => setTheme(e.target.value)}>
      <option value="system">System</option>
      <option value="light">Light</option>
      <option value="dark">Dark</option>
    </select>
  );
}
```

**Avoid the flash of wrong theme (FOUC) with SSR:** set the theme attribute with a tiny **blocking inline script** in `<head>` before React hydrates (libraries like `next-themes` do this).

Tailwind: `darkMode: "class"` (or selector strategy) + `dark:` variants.

---

# 9. Responsive Design

- CSS media queries / Tailwind breakpoints (`sm: md: lg:`) — preferred (no JS, no hydration issues).
- **Container queries** (`@container`) — components adapt to their container, not the viewport (great for reusable components).
- JS (`useMediaQuery`) only when you must render **different components** (and beware SSR mismatch).
- Fluid typography: `font-size: clamp(1rem, 2vw + 0.5rem, 1.5rem)`.

---

# 10. Animations

| Tool | Use |
| --- | --- |
| CSS transitions/animations | Most hover/enter effects; animate `transform` & `opacity` |
| **Framer Motion (Motion)** | Declarative React animations, layout animations, exit animations (`AnimatePresence`), gestures |
| React Spring | Physics-based |
| View Transitions API | Page/state transitions (React `<ViewTransition>` experimental) |
| `prefers-reduced-motion` | Respect accessibility settings |

---

# 11. Design Systems

- **Design tokens** (colors, spacing, typography, radii) as CSS variables / Tailwind config / TS constants.
- Component library (Button, Input, Modal…) documented in **Storybook**, with visual regression tests (Chromatic).
- Accessible primitives (Radix, React Aria) + your styles.
- Versioned package in a monorepo for multiple apps.

---

# 12. Choosing (interview answer framework)

| Need | Choice |
| --- | --- |
| Next.js App Router / RSC | Tailwind, CSS Modules, or zero-runtime CSS-in-JS |
| Fast product development with design tokens | Tailwind + cva + Radix/shadcn |
| Team prefers plain CSS, simple app | CSS Modules (+ Sass) |
| Existing styled-components codebase | Keep; consider gradual migration for RSC |
| Enterprise admin UI quickly | MUI / Ant Design / Mantine |

---

# 13. Interview Answers

**Which styling approach do you prefer and why?**

> "For modern React — especially Next.js with Server Components — I prefer Tailwind or CSS Modules: zero runtime, scoped or utility-based, and fully compatible with SSR and RSC. I use `cva` for variants and CSS variables for theming. Runtime CSS-in-JS like styled-components gives great DX for dynamic styles but costs runtime performance and needs client components, so the ecosystem is moving to zero-runtime tools."

**How do you implement dark mode without a flash?**

> "Define colors as CSS variables switched by a `data-theme` attribute or class, store the preference, default to `prefers-color-scheme`, and set the attribute in a small inline script in `<head>` before hydration so the first paint already has the right theme."

---

# 14. Key Takeaways

- CSS Modules: scoped, zero runtime. Tailwind: utility-first, tokens, zero runtime.
- Runtime CSS-in-JS: dynamic & themable but runtime cost and RSC friction; zero-runtime alternatives exist.
- Inline styles for dynamic values; prefer CSS variables.
- Theming with CSS variables; prevent theme flash with an inline head script.
- Media/container queries over JS for responsiveness.

## Readiness Checklist

- [ ] Build the same Button with CSS Modules, Tailwind+cva and styled-components
- [ ] Implement dark mode with system preference and no flash
- [ ] Explain runtime vs zero-runtime CSS-in-JS
- [ ] Use container queries in a card component
