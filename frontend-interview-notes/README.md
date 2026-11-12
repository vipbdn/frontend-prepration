# 🚀 Frontend Interview Prep — Advanced JavaScript Notes

> Complete notes for every section of the roadmap (`00-starter-roadmap.md`), written for a **3-YOE Frontend / React developer**.
>
> Every file follows the same structure: **concept → internals → code → React connection → output questions → interview answers → key takeaways → readiness checklist**.
>
> All implementations and output-question answers were checked by running them in Node.js. Browser-only behaviour (DOM, storage, rendering) is explained, not executed.

---

## 📚 File Index

| # | File | Roadmap section | Priority |
| --- | --- | --- | --- |
| 00 | [00-starter-roadmap.md](00-starter-roadmap.md) | The roadmap & progress tracker | — |
| 01 | [01-javascript-fundamentals-and-internals.md](01-javascript-fundamentals-and-internals.md) | 1. Fundamentals & Internals | ⭐ |
| 02a | [02a-closures.md](02a-closures.md) | 2. Closures | 🔥 |
| 02b | [02b-functional-programming.md](02b-functional-programming.md) | 2. Functional Programming, compose/pipe/curry/once | 🔥 |
| 03 | [03-objects-and-prototypes.md](03-objects-and-prototypes.md) | 3. Objects & Prototypes, `new`, classes, `instanceof` | 🔥 |
| 04 | [04-arrays-and-data-transformation.md](04-arrays-and-data-transformation.md) | 4. Arrays & Data Transformation | ⭐ |
| 05 | [05-shallow-and-deep-copy.md](05-shallow-and-deep-copy.md) | 5. Shallow & Deep Copy, `deepClone`, `deepEqual` | 🔥 |
| 06 | [06-promises.md](06-promises.md) | 6. Promises + all implementations | 🔥🔥🔥 |
| 07 | [07-async-javascript.md](07-async-javascript.md) | 7. Async JavaScript | 🔥🔥 |
| 08 | [08-event-loop.md](08-event-loop.md) | 8. Event Loop + 10 output questions | 🔥🔥🔥 |
| 09 | [09-debounce-and-throttle.md](09-debounce-and-throttle.md) | 9. Debounce & Throttle (+ React hooks) | 🔥🔥🔥 |
| 10 | [10-memoization-and-caching.md](10-memoization-and-caching.md) | 10. Memoization, LRU, React memo hooks | 🔥🔥 |
| 11 | [11-this-and-function-binding.md](11-this-and-function-binding.md) | 11. `this`, call/apply/bind + polyfills | 🔥🔥🔥 |
| 12 | [12-dom-and-browser-apis.md](12-dom-and-browser-apis.md) | 12. DOM, Events, Delegation, Observers | 🔥🔥 |
| 13 | [13-browser-storage.md](13-browser-storage.md) | 13. Browser Storage (+ where to store tokens) | ⭐ |
| 14 | [14-memory-management.md](14-memory-management.md) | 14. Memory Management, GC, AbortController | 🔥🔥🔥 |
| 14b | [14b-memory-leaks-devtools.md](14b-memory-leaks-devtools.md) | 14. React leaks & DevTools Memory tab (your original notes) | 🔥 |
| 15 | [15-error-handling.md](15-error-handling.md) | 15. Error Handling, custom errors, error boundaries | ⭐ |
| 16 | [16-javascript-modules.md](16-javascript-modules.md) | 16. Modules, code splitting, tree shaking | ⭐ |
| 17 | [17-map-set-weakmap-weakset.md](17-map-set-weakmap-weakset.md) | 17. Map / Set / WeakMap / WeakSet | ⭐ |
| 18 | [18-iterators-and-generators.md](18-iterators-and-generators.md) | 18. Iterators & Generators | — |
| 19 | [19-symbols.md](19-symbols.md) | 19. Symbols | — |
| 20 | [20-regular-expressions.md](20-regular-expressions.md) | 20. Regular Expressions | — |
| 21 | [21-utility-implementations.md](21-utility-implementations.md) | 21. All utility implementations in one place | 🔥🔥🔥 |
| 22 | [22-polyfills.md](22-polyfills.md) | 22. Array / Function / Promise polyfills | 🔥🔥🔥 |
| 23 | [23-javascript-performance.md](23-javascript-performance.md) | 23. Performance, rendering, Web Vitals | 🔥🔥 |
| 24 | [24-web-workers.md](24-web-workers.md) | 24. Web Workers & Concurrency | — |
| 25 | [25-frontend-security.md](25-frontend-security.md) | 25. XSS, CSRF, CORS, CSP, prototype pollution | ⭐ |
| 26 | [26-react-advanced-javascript.md](26-react-advanced-javascript.md) | 26. React + Advanced JS | 🔥🔥🔥 |
| 27 | [27-interview-coding-patterns.md](27-interview-coding-patterns.md) | 27. Coding patterns, Event Emitter, mock rounds | 🔥🔥🔥 |

---

## 🎯 Suggested Study Order (follows the roadmap's 8 phases)

| Phase | Files | Focus |
| --- | --- | --- |
| 1 — Internals | 01, 11 | Execution context, scope, hoisting, `this` |
| 2 — Closures & Functions | 02a, 02b, 09, 10 | Closures, FP, debounce/throttle, memoization |
| 3 — Objects | 03, 05, 11 (call/apply/bind) | Prototypes, `new`, classes, copying |
| 4 — Async | 06, 07, 08, 15 | Promises, async/await, event loop, errors |
| 5 — Browser | 12, 13, 25 | DOM, events, storage, security |
| 6 — Memory & Performance | 14, 14b, 23, 24 | GC, leaks, rendering, workers |
| 7 — Implementations | 21, 22, 27 | Write everything from scratch |
| 8 — React connection | 26 | Stale closures, immutability, async React |
| Extras | 04, 16, 17, 18, 19, 20 | Arrays, modules, collections, iterators, symbols, regex |

---

## 🏆 If You Only Have One Week

| Day | Files |
| --- | --- |
| 1 | 02a Closures · 11 `this` |
| 2 | 06 Promises · 07 Async |
| 3 | 08 Event Loop (do every output question twice) |
| 4 | 09 Debounce/Throttle · 10 Memoization |
| 5 | 03 Prototypes · 05 Deep Copy |
| 6 | 22 Polyfills · 21 Utilities |
| 7 | 26 React + JS · 27 Mock Round 1 & 2 |

---

## 🔁 Revision Method (from the roadmap)

- **Day 1:** read the file, type every example yourself, implement from scratch
- **Day 3:** re-read Key Takeaways, solve the output questions without looking
- **Day 7:** implement the utilities from memory, explain the "Interview Answer" sections out loud
- **Day 30:** quick revision with the cheat sheets + one mock round from file 27

> **Don't mark a topic complete just because you read it.**
> Understand → Explain → Implement → Debug → Apply in React → Answer interview questions.
