# 28 — Architecture & Best Practices

> **Interview + Understanding Guide** — Roadmap Section 28
>
> Goal: Structure React codebases that scale with features and teams: folder structures, layering (UI / hooks / services / data), API layer design, design systems, monorepos, micro-frontends, configuration, code quality and conventions. At 4 YOE, interviewers expect you to justify these decisions.

---

# 1. Folder Structure ⭐

### Type-based (small apps)

```text
src/
├── components/
├── hooks/
├── pages/
├── services/
├── utils/
└── context/
```

Simple, but in large apps a single feature is scattered across 6 folders.

### Feature-based (recommended for medium/large apps) ⭐

```text
src/
├── app/                    # app shell: providers, router, global styles, store setup
│   ├── providers.tsx
│   ├── router.tsx
│   └── store.ts
├── features/               # vertical slices — each owns its UI, logic, API, types
│   ├── auth/
│   │   ├── api/            # loginRequest, useCurrentUser (TanStack Query hooks)
│   │   ├── components/     # LoginForm, ProtectedRoute
│   │   ├── hooks/          # useAuth
│   │   ├── types.ts
│   │   └── index.ts        # public API of the feature
│   ├── cart/
│   └── products/
├── shared/ (or components/, lib/)
│   ├── ui/                 # design system: Button, Input, Modal
│   ├── hooks/              # useDebounce, useMediaQuery
│   ├── lib/                # apiClient, formatters, analytics
│   └── types/
├── pages/ or routes/       # route components composing features (thin)
└── main.tsx
```

Rules:
- Features **don't import each other's internals** — only from `features/x/index.ts` (enforce with ESLint `import/no-restricted-paths` or boundaries plugins).
- `shared` never imports from `features`.
- Route/page components are **thin**: compose features, read params.
- Colocate tests, stories and styles with components (`Button.tsx`, `Button.test.tsx`, `Button.stories.tsx`).

Alternative named methodology: **Feature-Sliced Design** (layers: app → pages → widgets → features → entities → shared).

---

# 2. Layering & Separation of Concerns ⭐

```text
┌──────────────────────────────┐
│ UI components (presentational)│  render props, no fetching
├──────────────────────────────┤
│ Hooks / containers            │  useProducts(), useCheckout() — orchestrate state & data
├──────────────────────────────┤
│ Data layer (TanStack Query)   │  query keys, caching, mutations
├──────────────────────────────┤
│ API client / services          │  fetch wrappers, auth headers, error mapping, DTO → domain
├──────────────────────────────┤
│ Domain utils (pure functions)  │  pricing, validation, formatting — easy to unit test
└──────────────────────────────┘
```

### API layer

```ts
// shared/lib/apiClient.ts
export class ApiError extends Error {
  constructor(public status: number, message: string, public body?: unknown) { super(message); }
}

export async function apiClient<T>(path: string, init: RequestInit = {}): Promise<T> {
  const res = await fetch(`${import.meta.env.VITE_API_URL}${path}`, {
    credentials: "include",
    headers: { "Content-Type": "application/json", ...init.headers },
    ...init,
  });
  if (!res.ok) throw new ApiError(res.status, res.statusText, await res.json().catch(() => null));
  return res.status === 204 ? (undefined as T) : res.json();
}

// features/products/api/getProducts.ts
export const productKeys = { all: ["products"] as const, list: (f: Filters) => [...productKeys.all, f] as const };

export function useProducts(filters: Filters) {
  return useQuery({
    queryKey: productKeys.list(filters),
    queryFn: () => apiClient<ProductDTO[]>(`/products?${new URLSearchParams(filters)}`).then(toProducts),
  });
}
```

Benefits: one place for auth headers, errors, base URL, retries, logging; components never call `fetch` directly; easy to mock (MSW).

---

# 3. Component Design Guidelines

- **Single responsibility** — split when a component does several things or exceeds ~200–300 lines.
- Prefer **composition** (children/slots) over boolean prop explosions.
- Keep components **pure**; side effects in hooks/handlers.
- **Colocate state**; lift only when needed.
- Derive data instead of syncing state.
- Name by **domain** (`InvoiceTable`), not implementation (`BlueBox`).
- Extract custom hooks for reusable or complex logic.
- Shared UI components: accessible, ref-forwarding, spread rest props, controlled + uncontrolled.

---

# 4. Design Systems

```text
Design tokens (color, spacing, type, radius, shadows)
      ↓
Primitives (Box, Stack, Text) + headless behaviour (Radix / React Aria)
      ↓
Components (Button, Input, Select, Modal, Toast, Table)
      ↓
Patterns (Forms, Empty states, Page layouts)
```

- Documented in **Storybook**; visual regression (Chromatic); a11y checks.
- Published as a versioned package (monorepo) with changelogs (Changesets).
- Tokens shared with designers (Figma variables → Style Dictionary → CSS variables / Tailwind config).

---

# 5. Monorepos

```text
apps/
  web/            (Next.js)
  admin/          (Vite SPA)
  mobile/         (React Native / Expo)
packages/
  ui/             (design system)
  api-client/     (typed SDK)
  config/         (eslint, tsconfig, tailwind presets)
  utils/
```

Tools: **pnpm workspaces**, **Turborepo** (task caching, pipelines), **Nx** (generators, dependency graph, affected builds).
Benefits: shared code without publishing, atomic changes across apps, consistent tooling. Costs: CI complexity, tooling knowledge, boundaries discipline.

---

# 6. Micro-Frontends ⭐ (frequently asked at 4+ YOE)

Split a large frontend into **independently developed and deployed** apps owned by different teams.

| Approach | How | Notes |
| --- | --- | --- |
| **Module Federation** (webpack 5 / Rspack / Vite plugin) | Apps expose/consume modules at runtime, sharing React as a singleton | Most popular for React |
| Build-time packages | Each MFE is an npm package composed at build | Not independently deployable |
| Route-based composition | Different paths served by different apps (reverse proxy) | Simplest; full reloads between apps |
| iframes | Strong isolation | Poor UX/integration, SEO, performance |
| Web Components | Framework-agnostic wrappers | Good for mixed frameworks |
| single-spa | Orchestrates multiple frameworks | Mature, more config |

```js
// host webpack.config.js (Module Federation)
new ModuleFederationPlugin({
  name: "shell",
  remotes: { checkout: "checkout@https://checkout.example.com/remoteEntry.js" },
  shared: { react: { singleton: true, requiredVersion: "^19.0.0" }, "react-dom": { singleton: true } },
});

// host app
const CheckoutApp = lazy(() => import("checkout/App"));
```

**When:** many teams, independent release cycles, very large app, gradual migration from a legacy framework.
**When not:** small/medium teams — the overhead (shared dependency versions, consistent UX, cross-app state, performance, testing, deployment) usually outweighs benefits. A well-structured monorepo often solves the same organizational problem.

Communication between MFEs: URL, custom events / event bus, shared minimal store, props from the shell. Avoid tight coupling.

---

# 7. Configuration & Environments

- `.env`, `.env.development`, `.env.production`; validate env vars at startup (Zod) so missing config fails fast.
- Feature flags (LaunchDarkly, Unleash, GrowthBook, simple JSON) for gradual rollouts & A/B tests.
- Runtime config for multi-environment Docker images (fetch `/config.json` at boot) vs build-time env vars.

---

# 8. Error Handling & Observability

- Error boundaries at app/route/widget level.
- Centralized API error mapping → user-friendly messages + toasts.
- Monitoring: **Sentry** (errors, source maps, session replay), Datadog/New Relic RUM, LogRocket.
- **Web Vitals** reporting (`web-vitals` library) to analytics.
- Structured logging with context (user id, route, release version).

---

# 9. Code Quality & Conventions

| Area | Practice |
| --- | --- |
| Types | TypeScript `strict`; no `any`; Zod at boundaries |
| Linting | ESLint (react-hooks, jsx-a11y, import order, boundaries), Prettier |
| Commits | Conventional Commits; Husky + lint-staged pre-commit |
| PRs | Small, reviewed, with tests; preview deployments |
| Naming | `PascalCase` components, `useX` hooks, `handleX` handlers / `onX` props, `isX/hasX` booleans |
| Files | One component per file; `index.ts` barrels only at feature boundaries |
| Docs | README per feature/package; ADRs (Architecture Decision Records) for big choices |
| Dead code | `knip`, `ts-prune` |

---

# 10. Scalability Checklist (talk track)

- Feature-based structure with enforced boundaries
- Server state in TanStack Query; small client state in Zustand/Context; URL state in params
- Typed API client & generated types (OpenAPI → `openapi-typescript`, GraphQL codegen)
- Shared design system with Storybook
- Route-level code splitting; performance budgets in CI
- Testing pyramid with MSW; E2E for critical flows
- CI/CD with preview environments, lint/typecheck/test gates
- Observability (Sentry + Web Vitals)
- Feature flags for safe releases
- Documentation & ADRs

---

# 11. Interview Answers

**How do you structure a large React application?**

> "Feature-based: an `app` folder for providers, routing and store setup; a `features` folder where each feature owns its components, hooks, API calls and types and exposes a public `index`; and a `shared` layer for the design system, generic hooks and the API client. Pages are thin and compose features. I enforce import boundaries with ESLint, keep server state in TanStack Query behind feature-level hooks, and colocate tests and stories."

**When would you use micro-frontends?**

> "When several autonomous teams need to develop and deploy parts of a large product independently, or when migrating gradually from a legacy app. Module Federation with React as a shared singleton is the common approach. For smaller organizations I'd avoid it — the costs in shared dependency management, UX consistency, performance and testing are high, and a monorepo with clear boundaries usually solves the team-scaling problem."

---

# 12. Key Takeaways

- Feature-based folders with public APIs and enforced boundaries.
- Layers: UI → hooks → data (Query) → API client → pure domain logic.
- Centralized, typed API client; components don't call `fetch` directly.
- Design system with tokens, headless primitives, Storybook.
- Monorepos (pnpm + Turborepo/Nx) for multi-app code sharing.
- Micro-frontends only when team/deployment independence justifies the cost.
- Strict TS, linting, CI gates, observability, feature flags, ADRs.

## Readiness Checklist

- [ ] Draw a feature-based folder structure for an e-commerce app
- [ ] Write a typed API client + feature query hook
- [ ] Explain design system layers
- [ ] Compare monorepo vs micro-frontends
- [ ] Present a "how would you scale this codebase" answer
