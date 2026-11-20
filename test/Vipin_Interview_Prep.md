# Interview Prep – Vipin Kumar (Front-End Developer, 3 Yrs)

> **How to use:** Read Part 0 for both interviews. Then read Part A before the **US Front-End role** and Part B before **Regul Softech (Next.js + SEO)**.
> Answer in your own words. Keep each answer to 60–90 seconds, and give an example from one of your projects whenever you can.

---

## Part 0 – Common Questions (Both Interviews)

### 1. Tell me about yourself.
"I'm Vipin, a Senior Software Engineer (Front-End) at Chetu India with 3 years of experience in React.js, Next.js and TypeScript. I've worked on four projects:
- An online gaming platform, where I built the complete UI with React and Redux Toolkit.
- A career and jobs platform, where I built pages in Next.js and Gatsby with GraphQL while moving the site off a legacy PHP site.
- An AI voice-automation CRM for a bail agency. I led the front-end there and built VAPI voice assistants that push leads into GoHighLevel CRM.
- Currently, a Next.js dealership management platform, where I fix bugs and deliver enhancements from Azure DevOps stories.

I also handle client calls on my own for every project. I was promoted to Senior Software Engineer in June 2026, and now I'm looking for a role where I can work more with Next.js, performance and modern front-end practices."

### 2. Walk me through your best project.
Use the **AI Voice CRM** project, because it shows ownership:
- **Problem:** The bail agency was missing calls, and each missed call was a lost lead.
- **Solution:** We built a VAPI voice assistant that collects caller details. Through a tool call it hits our Node.js API, and the data is saved into GoHighLevel CRM. I built the React + MUI dashboard from scratch, with Redux Toolkit, Formik/Yup forms and an Axios interceptor layer.
- **My role:** I led the front-end, wrote the VAPI prompts and tool integration, and handled the client calls.
- **Result:** No inbound lead gets missed, and the client's manual data entry dropped.

### 3. Why are you looking for a change?
"I've learned a lot at Chetu and was promoted this year. Now I want to work on a product, or with a team where I can go deeper into Next.js, performance and architecture, and take on more ownership. It's about growth, not about any problem with my current company."
*(Never speak negatively about your current employer.)*

### 4. What's your notice period?
"It's 30 days and negotiable. I'll work with my current team for a smooth handover."
⚠️ Your official notice is 90 days. Before the interview, check with HR whether early release or a buyout is possible, so you can keep this promise if you get an offer.

### 5. What are your salary expectations?
"Based on my 3 years of experience, my Next.js and TypeScript work and my promotion, I'm looking at **₹15–18 LPA**. I'm open to discussing it based on the role and the full package."
- Don't give a number first if they haven't asked.
- If they push back, ask: "What is the budget for this role?"
- For a US contract role, an equivalent is **$15–20/hour**.

### 6. What are your strengths?
"Strong React/Next.js fundamentals, clean reusable components, and communication: I handle client calls on my own. I also own tasks end to end, from requirement to deployment."

### 7. What's a weakness?
"Earlier I used to take on too many tasks myself instead of delegating. When I led the CRM front-end, I learned to split the work, set priorities in Jira, and ask for help early."

### 8. Tell me about a challenging bug you fixed.
Pick a real one from the dealership project. Explain it this way:
- **Bug:** Describe the bug, for example a form submitting twice, or stale data showing after an update.
- **Debugging:** Say how you found it: React DevTools, the Network tab, console logs, reproducing it locally in Docker.
- **Root cause:** For example, a missing dependency in `useEffect`, or the cache not being invalidated.
- **Fix:** What you changed.
- **Prevention:** For example, you added a check or a reusable hook so it doesn't happen again.

### 9. How do you handle a disagreement with a client or teammate?
"I listen first and ask why they want it that way. Then I explain the trade-off with facts: time, performance, user experience. If we still disagree, I suggest a small demo or follow the PM's decision, and I note the risk in the ticket."

### 10. Do you have questions for us?
Always ask 2–3:
- "What does the front-end tech stack and deployment process look like?"
- "How is the team structured? Who would I work with daily?"
- "What would success look like for this role in the first 3 months?"

---

## Part A – US Front-End Developer Role

*Focus areas in the JD: HTML/CSS/JS, React, TypeScript, responsive design, cross-browser support, performance, accessibility, REST APIs, debugging, Git, Agile, working with UI/UX designers.*

### JavaScript

**A1. `var`, `let` and `const`: what's the difference?**
- `var` is function-scoped and hoisted, with the initial value `undefined`.
- `let` and `const` are block-scoped and sit in the "temporal dead zone" until they are declared.
- `const` can't be reassigned. Objects declared with `const` can still be mutated.

**A2. What is a closure? Give a real use.**
A closure is a function that remembers variables from the scope where it was created, even after that outer function has finished.
- Real uses: debounce functions, data privacy, React hooks.

```js
function counter() { let c = 0; return () => ++c; }
const inc = counter(); inc(); // 1
inc(); // 2
```

**A3. Explain the event loop.**
JavaScript is single-threaded. Here is the order things run in:
1. Synchronous code runs on the call stack.
2. When the stack is empty, all **microtasks** run (Promise `.then`, `queueMicrotask`).
3. Then one **macrotask** runs (`setTimeout`, DOM events), and the cycle repeats.

That's why `Promise.then` runs before `setTimeout(fn, 0)`.

**A4. `==` vs `===`?**
`===` checks both value and type. `==` converts types before comparing. Always use `===`.

**A5. Debounce vs throttle?**
- **Debounce:** run the function only after the user stops triggering it for X ms. Example: a search input.
- **Throttle:** run it at most once every X ms. Example: scroll or resize handlers.

**A6. How do you handle async errors?**
- Wrap `await` in `try/catch`, or use `.catch()` on promises.
- In my CRM project I handled API errors globally in an **Axios response interceptor**: show a toast, and on a 401, log the user out or refresh the token.

### React

**A7. What's the virtual DOM, and how does reconciliation work?**
- React keeps a lightweight copy of the UI in memory.
- When state changes, it builds a new tree, compares it with the old one ("diffing"), and updates only the changed nodes in the real DOM.
- `key` props help React match list items correctly.

**A8. Why shouldn't you use the array index as a key?**
If items are reordered, added or removed, the indexes shift. React then reuses the wrong component state, which causes bugs, for example input values showing up on the wrong row. Use a stable unique ID instead.

**A9. Explain `useEffect` and its cleanup.**
- `useEffect` runs side effects (API calls, subscriptions) after render.
- The dependency array controls when it re-runs.
- The function you return cleans up timers and listeners, so you don't get memory leaks.

**A10. `useMemo` vs `useCallback` vs `React.memo`?**
- `useMemo` caches a computed **value**.
- `useCallback` caches a **function** reference.
- `React.memo` skips re-rendering a component when its props haven't changed.

Use them only where profiling shows unnecessary re-renders.

**A11. How do you decide between local state, Redux Toolkit and TanStack Query?**
- **Local state (`useState`):** UI-only state, like a modal being open.
- **Redux Toolkit:** global client state, like user session, roles and theme. I used it in the gaming and CRM projects.
- **TanStack Query:** **server state**. It handles caching, refetching, loading and error states, and retries. It removes a lot of Redux boilerplate for API data.

**A12. How did you structure reusable components?**
- Small, single-purpose components: Button, Input, Modal, Table.
- Props for variations, plus composition with `children`.
- Shared hooks for logic, like `useFetch` or `useDebounce`.
- A folder structure organised by feature.

In the gaming project, all game cards, modals and forms were built from shared components that matched the Figma designs.

**A13. Controlled vs uncontrolled forms? Why React Hook Form?**
- **Controlled:** the value lives in React state and re-renders on every keystroke.
- **Uncontrolled:** the DOM holds the value, and you read it through refs.
- React Hook Form uses uncontrolled inputs, which means fewer re-renders and better performance. It works with **Yup/Zod** for schema validation.
- I've used Formik + Yup for dynamic, conditional forms in the CRM.

**A14. What is an Axios interceptor? How did you use it?**
Functions that run before every request or after every response.
- **Request interceptor:** attaches the auth token to each request.
- **Response interceptor:** handles 401 (logout or refresh) and 500 (global error toast) in one place.

### TypeScript

**A15. `interface` vs `type`?**
- Both describe the shape of data.
- An `interface` can be extended and merged, which is good for objects and props.
- A `type` can also do unions, intersections and primitives: `type Status = 'idle' | 'loading'`.

**A16. How do you type React props and API responses?**

```ts
interface User { id: string; name: string; email: string }
interface Props { user: User; onSelect?: (id: string) => void }
const res = await axios.get<User[]>('/api/users');
```

Also worth knowing:
- **Generics:** like `ApiResponse<T>`.
- **Utility types:** `Partial`, `Pick`, `Omit`, `Record`.

### HTML / CSS / Responsive / Accessibility

**A17. How do you make a site responsive?**
- Build mobile-first.
- Use flexible layouts with **Flexbox and Grid**.
- Use relative units: `rem`, `%`, `vw`.
- Use media queries, or Tailwind/Bootstrap breakpoints.
- Use responsive images: `srcset`, `next/image`.
- Test on real devices and in Chrome DevTools device mode.

**A18. Flexbox vs Grid?**
- **Flexbox:** one dimension, a row or a column. Good for navbars and aligning items.
- **Grid:** two dimensions, rows and columns together. Good for page layouts and card grids.

**A19. How do you ensure cross-browser compatibility?**
- Use standard CSS. Autoprefixer, which Tailwind and Next.js include, adds vendor prefixes.
- Check feature support on caniuse.com.
- Test in Chrome, Firefox, Safari and Edge.
- Watch out for Safari issues with dates, flex gaps and `100vh` on mobile.

**A20. What do you know about web accessibility (a11y)?**
- Use semantic HTML: `button`, `nav`, `main`, `label`.
- Add `alt` text to images.
- Make sure everything works with the keyboard and focus states are visible.
- Keep enough colour contrast.
- Use ARIA attributes only when native HTML isn't enough.
- Check with Lighthouse's accessibility audit.

### Performance

**A21. How have you improved front-end performance?**
On the career platform:
- **Image lazy loading.**
- **Code splitting / chunking.**
- **Lazy-loading components:** `React.lazy` + `Suspense`, `next/dynamic`.

All of these improved our Lighthouse scores. Other techniques:
- Memoization.
- Debouncing API calls.
- Smaller bundles (tree-shaking, removing unused libraries).
- Optimized images (WebP).
- Caching API data with TanStack Query.

### APIs / Debugging / Testing

**A22. How do you integrate a REST API cleanly?**
- One central Axios instance with the base URL and interceptors.
- An API service layer, so components don't call Axios directly.
- Loading, error and empty states for every call.
- TypeScript types for responses.

**A23. How do you debug a UI issue in production?**
1. Reproduce it: which browser, which steps.
2. Check the console and the Network tab for the failed request, its status and payload.
3. Use React DevTools to inspect props and state, and Redux DevTools for actions.
4. Fix it locally (in Docker for the dealership project).
5. Raise a PR with a clear description so it gets reviewed.

**A24. How do you test your code?**
Be honest about where you are:
- "Most of my testing has been manual: functional and cross-browser testing, plus edge cases like empty, error and slow-network states.
- I'm familiar with **Jest and React Testing Library** concepts: render a component, find elements by role, fire events, assert the result."

Prepare one small example. Learn it before the interview if you haven't used it yet.

### Git / Agile / Collaboration

**A25. Explain your Git workflow.**
1. Create a feature branch from `develop`/`main`: `feature/ticket-123`.
2. Make small commits with clear messages.
3. Pull and rebase to stay updated.
4. Raise a PR and address review comments.
5. Merge.

To resolve a merge conflict: open the conflicting files, keep the correct changes, test, then commit.

**A26. `git merge` vs `git rebase`?**
- **Merge** keeps the full history and adds a merge commit.
- **Rebase** rewrites your commits on top of the target branch, so history stays linear.
- Never rebase a shared or public branch.

**A27. What does your Agile process look like?**
- Two-week sprints with sprint planning, daily stand-ups and a retro.
- Tickets in Jira (career platform) and Azure DevOps (dealership).
- I estimate stories, update their status, and demo the work to clients.

**A28. How do you work with UI/UX designers?**
- I review the Figma file early and ask about missing states: hover, error, empty, mobile.
- I build reusable components that match the design system.
- I raise feasibility or performance concerns before I start building, not after.

**A29. You'll work with a US team. How do you handle time-zone differences and remote work?**
"I already talk directly with clients on calls. I keep a few overlap hours, send clear written updates at the end of my day, document decisions in tickets, and say early if I'm blocked."

---

## Part B – Regul Softech (Next.js + SEO + SSR/SSG)

*Focus areas in the JD: React, Next.js, SEO optimization, SSR/SSG, website performance, HTML5/CSS3/JS, responsive UI, REST APIs, Git.*

### Next.js Core

**B1. What's the difference between CSR, SSR, SSG and ISR?**

| Type | When HTML is built | Use case |
|---|---|---|
| **CSR** | In the browser, after the JavaScript loads | Dashboards, pages behind a login |
| **SSR** | On the server, for every request | Personalized or frequently changing data |
| **SSG** | At build time | Blogs, marketing pages, docs |
| **ISR** | At build time, then regenerated after X seconds | Job listings, product pages |

On the career platform, we used SSG for blogs and static content (Gatsby and Next.js) and SSR or fresh fetching for dynamic pages.

**B2. How do you do SSR, SSG and ISR in the App Router?**
Server components fetch data directly:

```ts
// SSG (default – cached)
await fetch(url);
// SSR (no cache)
await fetch(url, { cache: 'no-store' });
// ISR (revalidate every 60s)
await fetch(url, { next: { revalidate: 60 } });
```

In the **Pages Router**, the equivalents are:
- `getStaticProps` → SSG
- `getServerSideProps` → SSR
- `getStaticProps` + `revalidate` → ISR
- `getStaticPaths` → dynamic static routes

**B3. Server Components vs Client Components?**
- **Server Components** (the default in the App Router) run on the server. They send no JavaScript to the browser and can fetch data and access secrets.
- **Client Components** (`'use client'`) are needed for state, effects, event handlers and browser APIs.
- Best practice: keep most components on the server and push `'use client'` down to small interactive pieces.

**B4. What is hydration? What causes a hydration error?**
Hydration is when React attaches event listeners to the server-rendered HTML in the browser. A hydration error happens when the server HTML and the client's first render don't match. Common causes:
- Using `window`, `Date.now()` or `Math.random()` during render.
- Invalid HTML nesting.

Fixes:
- Use `useEffect` for browser-only code.
- Use `dynamic(..., { ssr: false })`.

**B5. App Router vs Pages Router?**
- **App Router** (`app/`): server components, nested layouts, `loading.tsx` and `error.tsx`, the Metadata API, streaming.
- **Pages Router** (`pages/`): the older model with `getServerSideProps` and `getStaticProps`.

I've worked with both.

**B6. How does Next.js routing work?**
- Routing is based on files and folders.
- Dynamic routes: `[slug]`.
- Catch-all routes: `[...slug]`.
- Route groups: `(group)`.
- Shared UI goes in `layout.tsx`.
- Navigation uses `<Link>`, which prefetches pages.

**B7. Next.js vs Gatsby? You've used both.**
- **Gatsby:** mainly SSG, with a GraphQL data layer at build time. Great for content sites, but build times grow as the site grows.
- **Next.js:** flexible. You can choose SSR, SSG, ISR or CSR for each page, and it has API routes and better support for dynamic content.

That flexibility is why many teams, including ours for parts of the platform, use Next.js for dynamic sections.

### SEO

**B8. Why is Next.js better for SEO than a plain React app?**
- A plain React app (CSR) sends an empty HTML shell, so search crawlers may not see the content.
- Next.js sends fully rendered HTML through SSR or SSG, so the content and meta tags are visible to crawlers right away.
- Pages also load faster, and speed is a ranking factor too.

**B9. How do you add SEO metadata in Next.js?**
App Router, static metadata:

```ts
export const metadata = {
  title: 'Jobs | Brand',
  description: 'Find the latest jobs...',
  openGraph: { title: '...', images: ['/og.png'] },
  alternates: { canonical: 'https://site.com/jobs' },
};
```

For dynamic pages, use `generateMetadata({ params })` to set the title and description per item. In the Pages Router, use `next/head`.

On the career platform, I added metadata to pages based on the client's requirements.

**B10. Which on-page SEO elements do you take care of?**
- A unique `<title>` and meta description on every page.
- One `<h1>` per page and a proper heading hierarchy.
- Semantic HTML.
- `alt` text on images.
- Clean URLs (slugs).
- Canonical tags to avoid duplicate content.
- Open Graph and Twitter tags for social sharing.
- Internal linking.
- Mobile-friendly pages.
- Fast loading.

**B11. What are sitemap.xml and robots.txt? How do you add them in Next.js?**
- **Sitemap:** lists every URL so search engines can find them.
- **robots.txt:** tells crawlers what they may or may not crawl.

In the App Router, add `app/sitemap.ts` and `app/robots.ts`, which generate both automatically. Older setups use the `next-sitemap` package.

**B12. What is structured data (JSON-LD)?**
Structured data is schema.org information in a `<script type="application/ld+json">` tag. It helps Google show rich results, for example a **JobPosting** schema on a job page, or **Article** on a blog. This is very relevant to a jobs or blog platform like the one I worked on.

### Performance / Core Web Vitals

**B13. What are Core Web Vitals?**
- **LCP (Largest Contentful Paint):** how fast the main content loads. Target: under 2.5 s.
- **INP (Interaction to Next Paint):** how fast the page responds to user input. Target: under 200 ms.
- **CLS (Cumulative Layout Shift):** visual stability. Target: under 0.1.

**B14. How do you improve LCP, CLS and INP?**
- **LCP:**
  - Use `next/image` with `priority` on the hero image.
  - Optimize image size and format.
  - Use SSR/SSG.
  - Preload fonts with `next/font`.
- **CLS:**
  - Always set image width and height.
  - Reserve space for ads and embeds.
  - Use `next/font` to avoid the layout jump when the font loads.
- **INP:**
  - Reduce JavaScript: code splitting, server components.
  - Avoid long tasks.
  - Debounce handlers.

**B15. What does `next/image` do?**
- Lazy loads images automatically.
- Serves responsive sizes and modern formats (WebP/AVIF).
- Prevents layout shift because width and height are required.
- `priority` marks above-the-fold images to load first.

**B16. How did you improve Lighthouse scores on your project?**
"On the career platform, we:
- Lazy-loaded images below the fold.
- Split the bundle into chunks.
- Lazy-loaded heavy components with `next/dynamic` / `React.lazy`, so the initial JavaScript was smaller.

The Performance score improved. I'd also check unused JavaScript and CSS, third-party scripts (load them with `next/script` and `strategy="lazyOnload"`), and font loading."

**B17. What is `next/dynamic`?**
It lazy-loads a component, which splits the code into a separate chunk. Use `{ ssr: false }` for browser-only components, like charts or editors that use `window`.

### Data, APIs and Misc

**B18. How do you fetch data in Next.js: server or client?**
- **On the server** (server components, `getServerSideProps`): for SEO content and secure data.
- **On the client** (TanStack Query, `useEffect`): for user-specific or interactive data after the page loads.

On the dealership platform, we integrate Django REST APIs. On the career platform, we used GraphQL plus REST.

**B19. What are API routes / Route Handlers?**
Backend endpoints inside Next.js (`app/api/.../route.ts`). Use them for small server tasks, like form submissions, proxying third-party APIs or hiding API keys.

**B20. What is middleware in Next.js?**
Code that runs before a request completes (`middleware.ts`). Typical uses: auth redirects, geo or locale redirects, A/B testing.

**B21. How do you handle environment variables?**
- Store them in `.env.local`.
- Only variables prefixed with `NEXT_PUBLIC_` are exposed to the browser. Keep secrets unprefixed and use them only on the server.

**B22. How would you migrate an old site to Next.js without losing SEO?**
"Based on our PHP → Gatsby/Next.js move:
- Keep the same URLs, or add **301 redirects** in `next.config.js`.
- Keep the same titles and meta descriptions.
- Update the sitemap and submit it in Google Search Console.
- Keep canonical tags.
- Compare Lighthouse scores and indexing before and after."

**B23. What is Docker, and how do you use it?**
"Docker packages an app with its environment into containers, so it runs the same everywhere. On the dealership project, the frontend and the Django backend run in Docker containers locally. I use `docker compose up`, check logs with `docker logs`, and rebuild images when dependencies change."

---

## Last-Minute Checklist
- [ ] Re-read your resume. Every line on it can become a question.
- [ ] Prepare one real bug story and one leadership story (the CRM project).
- [ ] Be ready to explain the VAPI → backend → GoHighLevel flow simply.
- [ ] Practise one live-coding task: a todo list, a debounced search box, or fetch-and-display with loading and error states.
- [ ] Check with HR about early release from your notice period.
- [ ] Set up a stable internet connection and headset, and test your camera before a remote interview.
