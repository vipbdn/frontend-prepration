# 20 — Regular Expressions

> **Interview + Understanding Guide** — `00-starter-roadmap.md` Section 20
>
> Goal: Learn enough regex for practical frontend validation, parsing and interview questions — and know the JavaScript-specific methods and gotchas (`g` flag + `lastIndex`, `matchAll`, named groups).

---

# 1. Creating Regex

```js
const literal = /hello/i;                   // literal — compiled once, preferred when fixed
const dynamic = new RegExp("hello", "i");   // constructor — for dynamic patterns
const fromUser = new RegExp(escapeRegex(userInput), "g");
```

With the constructor, backslashes must be **doubled** in strings:

```js
/\d+/            ≡  new RegExp("\\d+")
```

### Escaping user input (always do this!)

```js
const escapeRegex = (s) => s.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");

new RegExp(escapeRegex("1+1=2")).test("1+1=2"); // true
```

---

# 2. Syntax Cheat Sheet ⭐

### Character classes

| Pattern | Matches |
| --- | --- |
| `.` | Any char except newline (unless `s` flag) |
| `\d` / `\D` | Digit `[0-9]` / non-digit |
| `\w` / `\W` | Word char `[A-Za-z0-9_]` / non-word |
| `\s` / `\S` | Whitespace (space, tab, newline…) / non-whitespace |
| `[abc]` | One of a, b, c |
| `[^abc]` | Anything except a, b, c |
| `[a-z0-9]` | Ranges |
| `\b` / `\B` | Word boundary / non-boundary |
| `\.` `\*` `\?` | Escaped literal special chars |

### Anchors

| Pattern | Meaning |
| --- | --- |
| `^` | Start of string (or line with `m` flag) |
| `$` | End of string (or line with `m` flag) |

```js
/cat/.test("concatenate");   // true — anywhere
/^cat$/.test("concatenate"); // false — must be the whole string
```

> For **validation**, always anchor with `^...$`, otherwise `"abc123xyz"` passes a "digits only" check.

### Quantifiers

| Pattern | Meaning |
| --- | --- |
| `*` | 0 or more |
| `+` | 1 or more |
| `?` | 0 or 1 (optional) |
| `{n}` | Exactly n |
| `{n,}` | n or more |
| `{n,m}` | Between n and m |
| `*?` `+?` `??` `{n,m}?` | **Lazy** (as few as possible) |

### Greedy vs lazy ⭐

```js
const html = "<b>bold</b><i>italic</i>";

html.match(/<.+>/)[0];  // "<b>bold</b><i>italic</i>" — greedy: as much as possible
html.match(/<.+?>/)[0]; // "<b>" — lazy: as little as possible
```

### Groups & alternation

| Pattern | Meaning |
| --- | --- |
| `(abc)` | **Capturing** group (saved as `$1`, `match[1]`) |
| `(?:abc)` | **Non-capturing** group (grouping only, not saved) |
| `(?<name>abc)` | **Named** capturing group |
| `a\|b` | Alternation: a OR b |
| `\1`, `\k<name>` | Backreference to a captured group |

### Lookarounds (zero-width — check without consuming)

| Pattern | Meaning |
| --- | --- |
| `X(?=Y)` | **Lookahead**: X followed by Y |
| `X(?!Y)` | **Negative lookahead**: X not followed by Y |
| `(?<=Y)X` | **Lookbehind**: X preceded by Y |
| `(?<!Y)X` | **Negative lookbehind**: X not preceded by Y |

```js
"100px 200em".match(/\d+(?=px)/g);  // ["100"]
"$40 €50".match(/(?<=\$)\d+/g);     // ["40"]
```

---

# 3. Flags ⭐

| Flag | Name | Effect |
| --- | --- | --- |
| `g` | global | Find **all** matches (not just first) |
| `i` | ignoreCase | Case-insensitive |
| `m` | multiline | `^`/`$` match at line breaks |
| `s` | dotAll | `.` matches newlines too |
| `u` | unicode | Proper Unicode (emoji, `\p{...}` property escapes) |
| `v` | unicodeSets | Upgraded `u` with set operations (ES2024) |
| `y` | sticky | Match only at `lastIndex` exactly |
| `d` | hasIndices | Adds start/end indices for groups |

```js
/\p{L}+/u.test("नमस्ते"); // true — any Unicode letter (needs u flag)
```

---

# 4. JavaScript Methods ⭐

| Method | On | Returns |
| --- | --- | --- |
| `regex.test(str)` | RegExp | `true` / `false` |
| `regex.exec(str)` | RegExp | Match array (with groups, `index`) or `null`; with `g`, advances `lastIndex` |
| `str.match(regex)` | String | Without `g`: like `exec`. With `g`: array of **all full matches** (no groups) or `null` |
| `str.matchAll(regex)` | String | Iterator of **all matches with groups** (requires `g`) |
| `str.replace(regex, rep)` | String | New string (first match, or all with `g`) |
| `str.replaceAll(str\|regex, rep)` | String | All matches (regex must have `g`) |
| `str.search(regex)` | String | Index of first match or `-1` |
| `str.split(regex)` | String | Array |

### `test`

```js
/^\d{6}$/.test("560001"); // true — Indian PIN code format
```

### `match` without vs with `g`

```js
const str = "2024-01-15 and 2025-12-31";

str.match(/(\d{4})-(\d{2})/);
// ["2024-01", "2024", "01", index: 0, input: "...", groups: undefined]

str.match(/(\d{4})-(\d{2})/g);
// ["2024-01", "2025-12"] — groups lost with g
```

### `matchAll` — all matches **with** groups ⭐

```js
const re = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/g;

for (const m of str.matchAll(re)) {
  console.log(m.groups.year, m.groups.month, m.index);
}
// 2024 01 0
// 2025 12 15

const dates = [...str.matchAll(re)].map((m) => m.groups);
```

### `replace` — strings, groups, functions

```js
"John Smith".replace(/(\w+)\s(\w+)/, "$2, $1");  // "Smith, John"

"2024-01-15".replace(
  /(?<y>\d+)-(?<m>\d+)-(?<d>\d+)/,
  "$<d>/$<m>/$<y>",
); // "15/01/2024"

"hello world".replace(/\b\w/g, (ch) => ch.toUpperCase()); // "Hello World"

// Replacement function args: (match, group1, group2, ..., offset, string, groups)
"price: 100".replace(/\d+/, (n) => n * 2); // "price: 200"
```

Special patterns: `$&` (whole match), `$1` (group), `$<name>`, `$$` (literal `$`).

### `split`

```js
"a, b;c  d".split(/[,;\s]+/); // ["a", "b", "c", "d"]
```

---

# 5. The `g` Flag + `lastIndex` Trap ⭐⭐⭐

A regex with `g` (or `y`) is **stateful** — `test()` and `exec()` remember `lastIndex`.

```js
const re = /a/g;

re.test("a"); // true  (lastIndex → 1)
re.test("a"); // false ❗ (starts searching at index 1 → no match → lastIndex reset to 0)
re.test("a"); // true
```

Real bug:

```js
const isValid = /^\d+$/g; // ❌ g flag on a reused validator
["1", "2", "3"].filter((s) => isValid.test(s)); // ["1", "3"] ❗
```

**Fix:** don't use `g` with `test()` for validation, or reset `re.lastIndex = 0`, or create the regex inside the function.

### Iterating with `exec`

```js
const re = /\d+/g;
let m;
while ((m = re.exec("a1b22c333")) !== null) {
  console.log(m[0], m.index); // 1 1, 22 3, 333 6
}
```

(Or simply use `matchAll`.)

---

# 6. Practical Validation Patterns

> Keep validation patterns simple and pair them with server-side validation. For emails especially, regex can only check the **format** — actual validity needs a confirmation email.

```js
const patterns = {
  // Simple, practical email check (not RFC-perfect — intentionally)
  email: /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/,

  // 10-digit Indian mobile starting 6–9, optional +91
  indianMobile: /^(?:\+91[-\s]?)?[6-9]\d{9}$/,

  // 6-digit PIN code, first digit 1–9
  pinCode: /^[1-9]\d{5}$/,

  // Strong password: 8+ chars, upper, lower, digit, special — using lookaheads
  password: /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^\w\s]).{8,}$/,

  // Username: 3–16 chars, letters/digits/underscore, starts with a letter
  username: /^[a-zA-Z]\w{2,15}$/,

  // Hex color
  hexColor: /^#(?:[0-9a-fA-F]{3}){1,2}$/,

  // URL slug
  slug: /^[a-z0-9]+(?:-[a-z0-9]+)*$/,

  // Date YYYY-MM-DD (format only)
  isoDate: /^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$/,
};

patterns.password.test("Passw0rd!"); // true
patterns.password.test("password");  // false
```

### How the password lookaheads work

```text
^                  start
(?=.*[a-z])        somewhere ahead there's a lowercase letter  (doesn't consume)
(?=.*[A-Z])        ...an uppercase letter
(?=.*\d)           ...a digit
(?=.*[^\w\s])      ...a special character
.{8,}              then: at least 8 characters total
$                  end
```

For URLs, prefer the built-in parser over regex:

```js
const isValidUrl = (s) => {
  try { new URL(s); return true; } catch { return false; }
};
```

---

# 7. Common Interview Tasks

```js
// Count vowels
const countVowels = (s) => (s.match(/[aeiou]/gi) || []).length;
countVowels("Hello World"); // 3

// Remove extra spaces
"  a   b  ".trim().replace(/\s+/g, " "); // "a b"

// camelCase → kebab-case
"backgroundColor".replace(/[A-Z]/g, (c) => `-${c.toLowerCase()}`); // "background-color"

// kebab-case → camelCase
"background-color".replace(/-([a-z])/g, (_, c) => c.toUpperCase()); // "backgroundColor"

// Format number with commas (Western grouping)
"1234567".replace(/\B(?=(\d{3})+(?!\d))/g, ","); // "1,234,567"
// (In real code use Intl.NumberFormat — handles locales like en-IN: 12,34,567)

// Extract hashtags
"Learning #JavaScript and #React".match(/#\w+/g); // ["#JavaScript", "#React"]

// Palindrome check ignoring non-alphanumerics
const isPalindrome = (s) => {
  const clean = s.toLowerCase().replace(/[^a-z0-9]/g, "");
  return clean === [...clean].reverse().join("");
};
isPalindrome("A man, a plan, a canal: Panama"); // true

// Strip HTML tags (display only — NOT a sanitizer!)
"<p>Hi <b>there</b></p>".replace(/<[^>]*>/g, ""); // "Hi there"

// Simple template interpolation
const render = (tpl, data) => tpl.replace(/\{\{\s*(\w+)\s*\}\}/g, (_, key) => data[key] ?? "");
render("Hello {{ name }}!", { name: "Asha" }); // "Hello Asha!"
```

---

# 8. Performance & Security

- **Catastrophic backtracking (ReDoS):** nested quantifiers like `/(a+)+$/` can take exponential time on inputs like `"aaaaaaaaaaaaaaaaaaaaaaaa!"`, freezing the main thread. Avoid nested quantifiers on overlapping patterns; limit input length.
- Don't build regex from unescaped user input.
- **Don't use regex to sanitize HTML** against XSS — use `textContent`, DOMPurify, or framework escaping.
- Hoist constant regexes outside loops/functions (avoid recompiling — mostly an issue with `new RegExp` in hot paths).

---

# 9. Output Questions 🧪

```js
console.log(/^\d+$/.test("123abc"));                    // Q1
console.log("aaa".replace("a", "b"));                    // Q2
console.log("aaa".replace(/a/g, "b"));                   // Q3
console.log("a-b-c".split("-", 2));                      // Q4
const re = /x/g;
console.log(re.test("x"), re.test("x"));                 // Q5
console.log("2024-05".match(/(\d+)-(\d+)/).slice(1));    // Q6
console.log("Hello".search(/l/));                        // Q7
console.log("a.b.c".split("."), "a.b.c".split(/./).length); // Q8
```

<details><summary>Answers</summary>

1. `false` — anchored.
2. `"baa"` — string pattern replaces only the first occurrence.
3. `"bbb"`
4. `["a", "b"]` — limit argument.
5. `true false` — `lastIndex` trap.
6. `["2024", "05"]`
7. `2`
8. `["a", "b", "c"]` and `6` — unescaped `.` matches **every** character, giving 6 empty strings.

</details>

---

# 10. Key Takeaways

- Anchor validation patterns with `^...$`.
- Know classes (`\d \w \s`), quantifiers (greedy vs lazy), groups (capturing, non-capturing, named), lookarounds.
- `match` with `g` drops groups → use `matchAll`.
- `replace` supports `$1`, `$<name>`, and replacer functions.
- `g` + `test/exec` is stateful (`lastIndex`) → bugs in reused validators.
- Escape user input; beware ReDoS; never sanitize HTML with regex.

## Readiness Checklist

- [ ] Write email, phone, password (lookaheads) validators
- [ ] Explain greedy vs lazy with an example
- [ ] Use named groups with `matchAll` and `replace`
- [ ] Explain the `lastIndex` bug
- [ ] Solve camelCase/kebab-case, number formatting, template interpolation
