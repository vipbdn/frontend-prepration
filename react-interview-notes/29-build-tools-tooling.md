# 29 — Build Tools & Tooling

> **Interview + Understanding Guide** — Roadmap Section 29
>
> Goal: Understand what happens between your JSX/TS source and the browser: bundlers (Vite, webpack, Rspack, Turbopack), transpilers (Babel, SWC, esbuild), dev servers and HMR, environment variables, linting/formatting, package managers, and CI/CD for React apps.

---

# 1. Why Build Tools?

Browsers don't understand **JSX**, **TypeScript**, bare imports (`import React from "react"`), CSS Modules or image imports, and shipping thousands of unminified modules is slow.

Build tools:
1. **Transpile** JSX/TS → JS (and down-level syntax for target browsers)
2. **Resolve & bundle** modules (`node_modules`, aliases)
3. Process **CSS** (PostCSS, Tailwind, Modules, Sass) and **assets**
4. **Optimize**: minify, tree-shake, code-split, hash filenames, compress
5. Provide a **dev server** with **HMR / Fast Refresh**

---

# 2. Transpilers

| Tool | Written in | Notes |
| --- | --- | --- |
| **Babel** | JS | Most configurable, huge plugin ecosystem (React Compiler is a Babel plugin), slower |
| **SWC** | Rust | ~20× faster than Babel; used by Next.js, Rspack, Vite plugin |
| **esbuild** | Go | Extremely fast transpile/bundle; Vite uses it for dependency pre-bundling (and TS/JSX transforms) |
| **TypeScript `tsc`** | TS | Type-checking; transpiling usually delegated to faster tools |

> Fast transpilers **strip types without checking** them → run `tsc --noEmit` separately (CI, editor) for type safety.

---

# 3. Bundlers

| | Vite | webpack | Rspack | Turbopack |
| --- | --- | --- | --- | --- |
| Dev model | **Native ESM** served on demand (no bundling in dev) | Bundles everything | webpack-compatible, Rust | Incremental, Rust (Next.js) |
| Dev startup | Very fast | Slow on big apps | Fast | Fast |
| Prod build | Rollup (Rolldown in newer versions) | webpack | Rspack | Turbopack (Next.js 16 default) |
| Config | Minimal | Powerful, verbose | webpack-like | Next.js-integrated |
| Ecosystem | Large, growing | Largest, mature | webpack plugins mostly work | Next.js only |

**Create React App is deprecated** (2025). New React projects: **Vite** for SPAs, or a framework (Next.js, React Router framework mode, Expo).

### How Vite dev works

```text
Browser requests /src/main.tsx
  → Vite transforms that single file (esbuild/SWC) and serves it as ESM
  → browser follows imports, requesting modules on demand
  → node_modules dependencies are pre-bundled once with esbuild (fewer requests, CJS→ESM)
Edit a file → only that module is re-transformed → HMR update
```

### Key webpack concepts (still asked)

| Concept | Meaning |
| --- | --- |
| Entry / output | Where bundling starts / where files go |
| Loaders | Transform files (`babel-loader`, `css-loader`, `ts-loader`) |
| Plugins | Hook into the build (`HtmlWebpackPlugin`, `DefinePlugin`, `MiniCssExtractPlugin`) |
| Mode | `development` vs `production` optimizations |
| Code splitting | `import()` → separate chunks; `SplitChunksPlugin` for vendor chunks |
| Module Federation | Share modules between separately built apps (micro-frontends) |

---

# 4. HMR & Fast Refresh

- **HMR** (Hot Module Replacement) swaps changed modules in the running app without a full reload.
- **React Fast Refresh** re-renders edited components **preserving their state** (when a file only exports components). Editing non-component exports or hook order causes a remount/full reload.

---

# 5. Production Optimizations

| Optimization | Effect |
| --- | --- |
| Minification (Terser, esbuild, SWC) | Smaller JS/CSS |
| Tree shaking | Remove unused ESM exports |
| Code splitting | Smaller initial bundle (`import()`, lazy routes) |
| Content hashing (`main.4f3a2.js`) | Long-term caching with `Cache-Control: immutable` |
| Compression | Brotli/gzip at server/CDN |
| `process.env.NODE_ENV = "production"` | Strips React dev warnings/checks (much faster React) |
| Source maps | Debuggable prod errors (upload privately to Sentry) |
| Browserslist targets | Avoid unnecessary polyfills/transpilation |
| Asset optimization | Image compression, inlining small assets |

Bundle analysis: `rollup-plugin-visualizer`, `webpack-bundle-analyzer`, `source-map-explorer`. Performance budgets: `size-limit`, Lighthouse CI.

---

# 6. Environment Variables

| Tool | Client-exposed prefix | Access |
| --- | --- | --- |
| Vite | `VITE_` | `import.meta.env.VITE_API_URL` |
| Next.js | `NEXT_PUBLIC_` | `process.env.NEXT_PUBLIC_API_URL` |
| CRA (legacy) | `REACT_APP_` | `process.env.REACT_APP_API_URL` |

Values are **inlined at build time** → public, and changing them requires a rebuild. For one Docker image across environments, load runtime config (`/config.json`) at startup.

---

# 7. Linting & Formatting

```js
// eslint.config.js (flat config)
import js from "@eslint/js";
import tseslint from "typescript-eslint";
import reactHooks from "eslint-plugin-react-hooks";
import jsxA11y from "eslint-plugin-jsx-a11y";

export default [
  js.configs.recommended,
  ...tseslint.configs.recommended,
  reactHooks.configs["recommended-latest"], // rules-of-hooks, exhaustive-deps, compiler rules
  jsxA11y.flatConfigs.recommended,
];
```

- **ESLint** catches bugs (hooks rules!), a11y, import boundaries.
- **Prettier** formats (or **Biome** — Rust linter+formatter, very fast; **oxlint**).
- **Husky + lint-staged**: run lint/format on staged files before commit.
- **commitlint**: enforce Conventional Commits.

---

# 8. Package Managers

| | npm | Yarn (Berry) | pnpm |
| --- | --- | --- | --- |
| Disk usage | Copies per project | PnP / node_modules | **Content-addressable store + hard links** (efficient) |
| Speed | OK | Fast | Fast |
| Strictness | Hoisting allows phantom deps | PnP strict | Strict (no phantom dependencies) |
| Monorepos | workspaces | workspaces | workspaces (popular with Turborepo) |

Lockfiles (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`) must be committed. `npm ci` / `pnpm install --frozen-lockfile` in CI. Semver: `^1.2.3` (minor+patch), `~1.2.3` (patch).

---

# 9. TypeScript Config Essentials

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["DOM", "DOM.Iterable", "ES2022"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "skipLibCheck": true,
    "noEmit": true,
    "paths": { "@/*": ["./src/*"] }
  }
}
```

---

# 10. CI/CD Pipeline for a React App

```yaml
# .github/workflows/ci.yml (simplified)
name: CI
on: [pull_request]
jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test -- --coverage
      - run: pnpm build
      - run: pnpm exec playwright install --with-deps && pnpm e2e
```

Deployment:
- **Static SPA**: build → upload `dist/` to S3+CloudFront / Netlify / Vercel / Cloudflare Pages; configure SPA fallback to `index.html`; immutable caching for hashed assets, `no-cache` for `index.html`.
- **Next.js**: Vercel, or Node server / Docker / standalone output on any host.
- **Preview deployments** per PR; feature flags for gradual rollout; quick rollback.

Docker for an SPA (multi-stage):

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf   # try_files $uri /index.html;
```

---

# 11. Interview Answers

**Vite vs webpack?**

> "Vite serves source files as native ES modules during development, transforming each file on demand and pre-bundling dependencies with esbuild, so startup and HMR are very fast regardless of app size. For production it bundles with Rollup. webpack bundles the whole app even in development, which is slower, but it's extremely configurable with a huge plugin ecosystem and features like Module Federation. For new SPAs I'd choose Vite; Rspack is a good webpack-compatible speed upgrade for existing projects."

**What happens when you run `npm run build`?**

> "The bundler starts from the entry, transpiles TS and JSX, resolves imports into a module graph, tree-shakes unused exports, splits code at dynamic imports, processes CSS and assets, minifies everything, adds content hashes for caching, sets `NODE_ENV=production` so React's dev checks are removed, and writes the output with source maps."

---

# 12. Key Takeaways

- Build = transpile + bundle + optimize + dev server/HMR.
- Babel (flexible) vs SWC/esbuild (fast); type-check separately with `tsc`.
- Vite for SPAs (native ESM dev, Rollup/Rolldown prod); webpack/Rspack for advanced configs & Module Federation; CRA deprecated.
- Fast Refresh preserves component state.
- Env vars with client prefixes are public and build-time.
- ESLint (hooks + a11y), Prettier/Biome, Husky; pnpm for monorepos.
- CI: install → lint → typecheck → test → build → e2e → deploy with caching rules.

## Readiness Checklist

- [ ] Explain Vite's dev architecture
- [ ] Explain loaders vs plugins in webpack
- [ ] List 8 production optimizations
- [ ] Configure ESLint with react-hooks rules
- [ ] Write a CI pipeline and SPA deployment caching strategy
