# PortSwigger Web Security Academy — Tuesday Labs

Three Apprentice-level labs completed while practicing web application security:

1. SQL Injection
2. Stored XSS
3. Access Control

---

## 1. SQL Injection — WHERE Clause

**Lab:** SQL injection vulnerability in WHERE clause allowing retrieval of hidden data
**Difficulty:** Apprentice
**Status:** Solved ✅

### Objective

The application used a category parameter to filter products. The goal was to manipulate the query so that unreleased products were displayed.

### Initial Request

```http
GET /filter?category=Corporate+gifts
```

The application appeared to use a query conceptually similar to:

```sql
SELECT * FROM products
WHERE category = 'Corporate gifts'
AND released = 1
```

The `released = 1` condition prevented unreleased products from being returned.

### Testing

A single quote was submitted through the `category` parameter.

The application responded with:

```http
HTTP 500 Internal Server Error
```

This indicated that the input was reaching an SQL query and affecting its syntax.

The next step was understanding SQL comments and Boolean conditions.

The successful test was:

```text
' OR 1=1 --
```

Conceptually, this changed the query logic so that the original restriction could no longer prevent the matching rows from being returned.

The lab then displayed the previously hidden products.

### What I Learned

* `SELECT` retrieves data.
* `FROM` specifies the table.
* `WHERE` filters rows.
* `OR` can change Boolean logic.
* `1=1` is always true.
* `--` starts an SQL comment in the relevant SQL context.
* User-controlled input can become dangerous when directly incorporated into SQL queries.

### Methodology

```text
Find input
    ↓
Observe application behavior
    ↓
Test syntax
    ↓
Identify backend context
    ↓
Understand the query
    ↓
Determine how input changes its logic
```

---

# 2. Stored XSS — Anchor href Attribute

**Lab:** Stored XSS into anchor `href` attribute with double quotes HTML-encoded
**Difficulty:** Apprentice
**Status:** Solved ✅

### Objective

Submit a comment containing an XSS payload that executes when the comment author's name is clicked.

### Initial Investigation

I first tested the comment body:

```text
TESTAUTHOR123
```

The value appeared inside a paragraph:

```html
<p>TESTAUTHOR123</p>
```

This showed that the comment body was not the interesting injection point.

I then tested the Website field:

```text
TESTSITE123
```

The value appeared in the author's link:

```html
<a id="author" href="TESTSITE123">john</a>
```

This identified the injection context as an HTML `href` attribute.

### Context Analysis

The lab stated that double quotes were HTML-encoded.

Therefore, escaping the `href` attribute with a quote was not the appropriate approach.

Instead, I considered what could execute while remaining inside an existing `href`.

The successful payload was:

```text
javascript:alert(1)
```

After submitting the comment and clicking the author's name, the JavaScript executed and the lab was solved.

### What I Learned

The important part of XSS testing is not simply looking for a generic payload.

The first question should be:

> **Where exactly does my input appear?**

Different contexts require different approaches:

```text
HTML body
    ↓
Attribute
    ↓
JavaScript
    ↓
URL
    ↓
CSS
```

In this lab, the input was inside an existing `href`, so the `href` itself became the attack surface.

### Methodology

```text
Find input
    ↓
Find reflection/storage point
    ↓
Identify exact context
    ↓
Check encoding/filtering
    ↓
Choose context-appropriate test
    ↓
Verify execution
```

---

# 3. Access Control — Admin Panel

**Lab:** Accessing the admin panel
**Difficulty:** Apprentice
**Status:** Solved ✅

### Objective

Log in as the normal user `wiener`, access the administrator panel, and delete the user `carlos`.

The lab specified that the administrator role was associated with:

```text
roleid = 2
```

### Initial Investigation

I logged in using the provided credentials and captured the request:

```http
POST /login
username=wiener
password=peter
```

The successful login returned a session cookie and redirected to:

```text
/my-account?id=wiener
```

The account page showed normal user information, but the HTML did not expose the user's role.

I also tested changing the account ID:

```text
/my-account?id=carlos
```

This resulted in a redirect to the login page, so simply changing the ID was not enough.

### Finding the Missing Information

The important clue came from another account function: the **change-email functionality**.

The response contained JSON with user information, including:

```json
"roleid": 1
```

This showed that role information was being exposed through an application response.

The lab's requirement was:

```text
roleid = 2
```

By analyzing the application's account functionality and how it handled the role information, I was able to reach the administrator panel.

From `/admin`, I could perform the required administrative action and delete `carlos`.

### What I Learned

Access-control testing is not limited to trying obvious URLs such as:

```text
/admin
```

A better approach is to map the application's functionality and look for places where the server exposes or processes:

* User IDs
* Roles
* Permissions
* Object IDs
* Account information
* JSON/API responses
* State-changing requests

The important question is:

> **What does the server trust when deciding what I am allowed to do?**

### Methodology

```text
Map account functionality
    ↓
Capture requests
    ↓
Inspect responses
    ↓
Look for user/role/object information
    ↓
Understand the authorization boundary
    ↓
Test whether the server properly enforces it
```

---

# Overall Lessons

These three labs demonstrated three different ways user-controlled data can affect an application:

| Vulnerability  | Trust Boundary              | Main Lesson                                                 |
| -------------- | --------------------------- | ----------------------------------------------------------- |
| SQL Injection  | Application → Database      | Understand how input changes backend query logic            |
| Stored XSS     | Application → Browser       | Identify the exact context where input is rendered          |
| Access Control | User → Authorization System | Determine what the server trusts when enforcing permissions |

The common workflow across all three was:

```text
1. Observe
2. Map the application
3. Identify the input
4. Find where the input goes
5. Understand the backend behavior
6. Form a hypothesis
7. Test the hypothesis
8. Verify the result
```

## Key Takeaway

A useful security mindset is to stop thinking only in terms of payloads.

Instead:

> **Follow the data.**

For SQL injection, follow the input into the database query.

For XSS, follow the input into the browser and identify its rendering context.

For access control, follow the user's identity and authorization data through the application's requests and responses.

Understanding the data flow makes it easier to identify where trust boundaries can fail.

**Labs completed: 3/3** ✅
