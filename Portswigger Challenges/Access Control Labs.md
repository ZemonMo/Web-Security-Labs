# PortSwigger Access Control Labs

## Overview

These three PortSwigger Web Security Academy labs demonstrate different access control weaknesses.

The main concept is the difference between:

* **Authentication** — determining who the user is.
* **Authorization** — determining what the authenticated user is allowed to do.

A user can be successfully authenticated while still being incorrectly authorized to perform privileged actions.

---

## Lab 1 — Unprotected Admin Functionality

### Vulnerability

The application exposed an administrator panel without properly restricting access to administrators.

### Discovery

I started by observing the application's requests and looking for hidden functionality.

I used directory/content discovery and found `robots.txt`. The file revealed the administrative path:

```text
/administrator-panel
```

I accessed the endpoint while logged in as the normal lab user.

### Result

The administrator panel was accessible without administrator privileges.

The lab was solved by using the exposed administrative functionality.

### What I learned

An administrative URL being hidden from the normal interface does not provide access control.

The server must explicitly verify the user's authorization before allowing access to administrative functionality.

---

## Lab 2 — Unprotected Admin Functionality With Unpredictable URL

### Vulnerability

The application used an unpredictable-looking URL for the administrator panel, but failed to properly protect the functionality.

### Discovery Method 1 — Burp History

While inspecting Burp HTTP history, I found:

```text
/admin-o80fcu
```

I accessed the endpoint and found that the administrator functionality was accessible.

### Discovery Method 2 — View Source

I then independently checked the page source and found the same administrator URL referenced in the application's code.

This confirmed that the supposedly unpredictable URL could be leaked by the application itself.

### Result

The administrator panel was accessible to the normal lab user.

### What I learned

An unpredictable URL is not an authorization mechanism.

Security through obscurity does not replace server-side authorization.

Even if an administrator URL is difficult to guess, it can still be discovered through:

* HTTP responses
* HTML source
* JavaScript
* links
* application behavior
* other information leaks

The server should still check whether the current user is authorized.

---

## Lab 3 — User Role Controlled by Request Parameter

### Vulnerability

The application trusted a client-controlled cookie to determine whether the current user was an administrator.

### Initial Request

The normal request contained:

```http
GET /my-account?id=wiener HTTP/2
Cookie: Admin=false; session=[REDACTED]
```

Changing the username from `wiener` to `administrator` did not change the authenticated user's privileges because the session still belonged to the normal user.

### Discovery

I captured the request generated when accessing the administrator panel.

The request still contained:

```text
Admin=false
```

This was the important clue.

I changed it to:

```text
Admin=true
```

The administrator functionality became available.

### Privileged Action

I then captured the request responsible for deleting another lab user.

The request also contained:

```text
Admin=false
```

I changed the value to:

```text
Admin=true
```

The privileged action succeeded and the lab was solved.

### Why It Worked

The application was making an authorization decision based on information supplied directly by the client.

Conceptually:

```text
Browser
   |
   | Admin=true
   v
Server
   |
   | "User is administrator"
   v
Privileged functionality
```

A secure application should instead determine the user's privileges from trusted server-side information associated with the authenticated session.

### What I Learned

Authorization checks must not rely on client-controlled values such as:

```text
cookies
query parameters
form parameters
headers
```

unless those values are treated only as untrusted input and independently validated.

The key lesson from this lab was to inspect the **request that performs the privileged action**, not just the page that displays the privileged interface.

---

## Overall Lessons

These three labs demonstrated three different failures:

| Lab                             | Main lesson                                             |
| ------------------------------- | ------------------------------------------------------- |
| Unprotected admin functionality | Privileged functionality must have authorization checks |
| Unpredictable admin URL         | Hiding or randomizing a URL is not security             |
| Client-controlled role          | The client must not control its own authorization level |


Access control testing is therefore not only about finding `/admin`.

It is about understanding **how the application decides who is allowed to perform an action**.
