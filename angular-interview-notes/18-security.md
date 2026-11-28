# 18 — Security 🔥

> **Interview + Understanding Guide** — Roadmap Section 18
>
> Goal: Explain how Angular protects against XSS (sanitization, contexts), when bypassing is (un)safe, CSRF/XSRF support, CSP & Trusted Types, auth best practices, and common vulnerabilities in SPAs.

---

# 1. XSS & Angular's Built-in Protection ⭐⭐⭐

**Cross-Site Scripting (XSS)**: an attacker injects script into your page (via user content, URL params, API data) that runs with your users' privileges (steals tokens, performs actions).

Angular treats **all values as untrusted by default**:

- **Interpolation** `{{ value }}` → always **escaped** as text (`<script>` shows as text).
- **Property/attribute bindings** → **sanitized** by security context.

| Security context | Example | Angular does |
| --- | --- | --- |
| HTML | `[innerHTML]="html"` | Strips dangerous tags/attrs (`<script>`, `onerror=`) and warns in console |
| Style | `[style]` | Sanitizes |
| URL | `[href]="url"` | Blocks `javascript:` URLs (prefixes `unsafe:`) |
| Resource URL | `<iframe [src]>`, `<script [src]>` | **Throws** unless explicitly trusted |

```ts
// user-provided comment
comment = `<img src=x onerror="alert('xss')"><b>Nice!</b>`;
```

```html
<p>{{ comment }}</p>               <!-- shows the raw text, safe -->
<p [innerHTML]="comment"></p>      <!-- renders <b>Nice!</b>, strips onerror -->
```

### Templates are compiled code

Angular templates are compiled (AOT) and **never built from user input** — that's why there's no template injection in normal usage. ⚠️ Never dynamically compile templates with user content.

---

# 2. Bypassing Sanitization (`DomSanitizer`) ⚠️

```ts
private readonly sanitizer = inject(DomSanitizer);

// Embedding a YouTube video by ID — validate the input first!
safeVideoUrl(id: string): SafeResourceUrl {
  if (!/^[\w-]{11}$/.test(id)) throw new Error("Invalid id");
  return this.sanitizer.bypassSecurityTrustResourceUrl(`https://www.youtube.com/embed/${id}`);
}
```

Methods: `bypassSecurityTrustHtml`, `Style`, `Script`, `Url`, `ResourceUrl`.

- Only bypass for **trusted, validated** values (never raw user input)
- Prefer sanitizing yourself (`sanitizer.sanitize(SecurityContext.HTML, value)`) or using a vetted library (DOMPurify) for rich text
- Avoid direct DOM APIs (`ElementRef.nativeElement.innerHTML = …`) — they bypass Angular's sanitization completely

---

# 3. Content Security Policy & Trusted Types

- **CSP** header restricts sources of scripts/styles: e.g. `script-src 'self' 'nonce-RANDOM'`.
- Angular supports **CSP nonces**: `ngCspNonce` attribute on the root element or `CSP_NONCE` token; CLI `autoCsp` option generates hash-based CSP for SSG/SSR.
- **Trusted Types** (`require-trusted-types-for 'script'`) — Angular is Trusted-Types compatible (`trusted-types angular angular#bundler`), preventing DOM XSS sinks from accepting strings.

---

# 4. CSRF / XSRF ⭐

**CSRF**: a malicious site makes the user's browser send an authenticated request (cookies auto-attached) to your API.

Angular's `HttpClient` supports the **cookie-to-header** pattern:
1. Server sets a cookie `XSRF-TOKEN` (readable by JS).
2. `HttpClient` reads it and sends header `X-XSRF-TOKEN` on mutating requests (POST/PUT/DELETE) to **same-origin, relative URLs**.
3. Server verifies header === cookie (attacker's site can't read your cookie).

```ts
provideHttpClient(withXsrfConfiguration({ cookieName: "MY-XSRF", headerName: "X-MY-XSRF" }));
// disable: withNoXsrfProtection()
```

Plus: `SameSite=Lax/Strict` cookies, server-side CSRF validation. Bearer tokens in headers (not cookies) aren't vulnerable to CSRF but are to XSS.

---

# 5. Authentication & Token Storage ⭐

| Storage | XSS risk | CSRF risk | Notes |
| --- | --- | --- | --- |
| `localStorage`/`sessionStorage` | ❌ readable by injected JS | ✅ safe | Simple; requires strong XSS defenses |
| Memory (signal/service) | Lower (lost on refresh) | ✅ | Combine with refresh token cookie |
| **HttpOnly, Secure, SameSite cookie** | ✅ not readable by JS | Needs CSRF protection | Generally recommended (BFF pattern) |

Best practices:
- OAuth2/OIDC with **Authorization Code + PKCE** (libraries: angular-oauth2-oidc, angular-auth-oidc-client, MSAL)
- Short-lived access tokens, refresh token rotation
- Auth interceptor attaches tokens only to **your** API domains
- **Guards are UX only** — authorization must be enforced server-side
- Don't put secrets/API keys in Angular code or `environment.ts` — everything shipped is public

---

# 6. Other Concerns

- **Dependency vulnerabilities**: `npm audit`, Dependabot/Renovate, keep Angular updated (security fixes land in supported versions only — check LTS policy)
- **Open redirects**: validate `returnUrl` query params are internal paths before navigating
- **Clickjacking**: `X-Frame-Options`/`frame-ancestors` headers
- **Sensitive data in URLs/logs**: avoid tokens in query strings
- **SSR**: protect against SSRF (validate hosts used by server-side HTTP; Angular SSR has `allowedHosts` checks), don't leak server-only data into transfer state
- HTTPS everywhere, HSTS

---

# 7. Interview Answers

**How does Angular prevent XSS?**

> "It treats all values as untrusted. Interpolation is always escaped as text, and property bindings are sanitized according to their security context — HTML, style, URL or resource URL — so `[innerHTML]` strips scripts and event handlers, `javascript:` URLs are neutralized, and untrusted resource URLs throw. Templates are compiled ahead of time, never from user input. Risks come from bypassing that: `bypassSecurityTrust*` with unvalidated input or touching the DOM directly through `nativeElement`."

**When would you use `DomSanitizer.bypassSecurityTrust*`?**

> "Only for values I control or have validated — like building an embed URL from a validated video ID or rendering HTML already cleaned by a trusted sanitizer. Never for raw user input, because it disables Angular's protection for that value."

**How does Angular help with CSRF?**

> "HttpClient implements the cookie-to-header pattern: it reads an XSRF-TOKEN cookie set by the server and sends it back as an X-XSRF-TOKEN header on mutating same-origin requests; the server compares them. Combined with SameSite cookies and server validation, that blocks cross-site forged requests."

**Where should you store JWTs?**

> "There's a trade-off. localStorage is easy but any XSS can steal the token. HttpOnly secure SameSite cookies aren't readable by JavaScript, which is generally safer, but you need CSRF protection. A common approach is a backend-for-frontend with HttpOnly session cookies, or short-lived access tokens kept in memory with a refresh token in an HttpOnly cookie."

---

# 8. Key Takeaways

- Escaping + context-aware sanitization by default; don't bypass with untrusted data; avoid raw DOM APIs.
- CSP nonces & Trusted Types supported.
- XSRF cookie-to-header built into HttpClient.
- Guards ≠ security; secrets don't belong in frontend code.

## Readiness Checklist

- [ ] Demonstrate `{{ }}` vs `[innerHTML]` with a malicious string
- [ ] Safely embed a YouTube iframe
- [ ] Explain XSRF flow
- [ ] Compare token storage options
