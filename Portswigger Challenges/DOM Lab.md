# DOM XSS in `document.write()` using `location.search`

**Platform:** PortSwigger Web Security Academy
**Vulnerability:** DOM-based Cross-Site Scripting (XSS)
**Difficulty:** Practitioner

## Overview

The application contains a stock checker that uses the `storeId` parameter from the URL and writes it directly into the page using `document.write()`.

The goal was to escape the `<select>` element and execute JavaScript.

## Vulnerable Code

```javascript
var stores = ["London","Paris","Milan"];

var store = (new URLSearchParams(window.location.search)).get('storeId');

document.write('<select name="storeId">');

if(store) {
    document.write('<option selected>'+store+'</option>');
}

for(var i=0;i<stores.length;i++) {
    if(stores[i] === store) {
        continue;
    }

    document.write('<option>'+stores[i]+'</option>');
}

document.write('</select>');
```

## Source

The user-controlled value comes from:

```javascript
window.location.search
```

Specifically:

```javascript
.get('storeId')
```

For example:

```text
/product?productId=3&storeId=London
```

produces:

```javascript
store = "London";
```

## Sink

The value is inserted into HTML through:

```javascript
document.write('<option selected>'+store+'</option>');
```

This creates the following structure:

```html
<select name="storeId">
    <option selected>London</option>
</select>
```

The important observation was that the input is placed **inside the `<select>` element**, so the payload needs to escape that HTML context.

## Testing Process

I initially investigated the stock-check request:

```http
POST /product/stock
productId=3&storeId=London
```

Changing the `storeId` changed the returned stock quantity, confirming that it was a valid parameter. However, the response was plain text rather than HTML, so this was not the DOM XSS sink.

I then traced the JavaScript and found that another `storeId` value was taken directly from `window.location.search`.

This gave the following data flow:

```text
URL
 ↓
storeId
 ↓
URLSearchParams.get()
 ↓
store
 ↓
document.write()
 ↓
<option>...</option>
 ↓
<select>
```

## Exploitation

Because the input was inside a `<select>` element, I needed to first escape the existing HTML context and then introduce an executable HTML element.

The successful payload was:

```html
</select><img src=x onerror=alert(1)>
```

Placed into the `storeId` parameter, this caused the browser to leave the original `<select>` context and process the injected element.

## Result

JavaScript execution was successfully triggered with `alert(1)`.

## Key Learning

The important lesson was not the payload itself, but identifying the **context** in which the input was reflected.

For DOM XSS, the reasoning was:

```text
Find source
   ↓
Trace the data
   ↓
Find sink
   ↓
Identify HTML/JS context
   ↓
Determine how to escape that context
   ↓
Construct payload
```

In this lab:

```text
location.search
      ↓
   storeId
      ↓
document.write()
      ↓
<select>
      ↓
escape with </select>
      ↓
execute JavaScript
```
