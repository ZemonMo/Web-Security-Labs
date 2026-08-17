# Signed Cookie Authentication — Flask Lab Research

## Overview

This experiment was created to understand how a server can trust a cookie sent back by a browser without storing every individual cookie in a database.

The lab used Flask's signed session cookies and `itsdangerous` to investigate how the cookie payload, timestamp, secret key, and signature work together.

## 1. Cookie Structure

A Flask signed session cookie can contain multiple components separated by dots:

```text
payload.timestamp.signature
```

For example:

```text
eyJ1c2VybmFtZSI6InVzZXIxIn0.xxxxx.xxxxxxxxxxxxx
```

The first component contains serialized session data.

After decoding the payload, we could see information such as:

```json
{"username":"user1"}
```

This demonstrated that the payload itself is not encrypted. The signing mechanism provides integrity/authenticity rather than confidentiality.

## 2. Timestamp

The timestamp changes as new cookies are generated.

We generated several cookies with the same user and observed that the timestamp changed over time.

The timestamp contributes to the signed data and can also be used by the server to enforce cookie/session expiration.

## 3. Secret Key

The most important part of the experiment was understanding the secret key.

The secret key is a server-side value that is used as the cryptographic key when generating the signature.

Conceptually:

```text
session data + timestamp
          +
      secret key
          ↓
        HMAC
          ↓
      signature
```

The secret key is **not stored inside the cookie**.

It is kept by the server.

## 4. How Verification Works

When the browser sends the cookie back, the server does not need to remember the exact cookie it previously issued.

Instead, it takes the received session data and timestamp and calculates the expected signature again using its secret key.

```text
Received:

payload + timestamp + received signature

             ↓

Server:

payload + timestamp + secret key
             ↓
           HMAC
             ↓
       calculated signature
```

The server compares:

```text
received signature == calculated signature
```

If they match, the signature is valid.

If they do not match, the cookie is rejected.

## 5. Cookie Modification Experiment

We modified the session data while keeping the original signature.

For example:

```text
Original:

username=user1
signature=ABC...

Modified:

username=user5
signature=ABC...
```

The server would calculate a new signature for `user5`.

Because the data changed, the calculated signature no longer matched the original signature.

Therefore the cookie became invalid.

This demonstrated why an attacker cannot normally modify a signed session cookie and simply keep the old signature.

## 6. Secret Key Experiment

The lab script used a known secret:

```python
SECRET_KEY = "cookie-lab-secret-key"
```

Changing the secret produced different signatures even when the same session data was used.

This demonstrated that the secret key is an essential input to the signing process.

The important distinction is:

```text
Secret key → input/key
Signature   → cryptographic output
```

The signature does not represent or reveal the secret key.

## 7. Important Security Concept

The HMAC algorithm itself does not have to be secret.

An attacker can know:

* the serialization format
* the signing algorithm
* the hash function
* the cookie structure
* the payload
* the timestamp

The security depends on protecting the secret key.

This follows a fundamental cryptographic principle:

> The algorithm can be public; the key must remain secret.

## 8. Main Finding

The experiment demonstrated that a signed cookie works like a cryptographic proof of integrity.

The server does not ask:

```text
"Have I seen this exact cookie before?"
```

Instead, it asks:

```text
"Does this cookie contain a signature that could have been generated
using my secret key for this exact data?"
```

This explains how signed-cookie sessions can be verified without storing every issued cookie server-side.

## 9. What I Learned

Through this experiment I learned:

1. Cookie payloads can be readable without being modifiable safely.
2. Timestamps can form part of signed session data.
3. HMAC uses a secret key to generate a signature.
4. The secret key is not contained in the cookie.
5. The server can verify a cookie by recalculating its signature.
6. Modifying signed data causes signature verification to fail.
7. Knowing the cryptographic algorithm does not provide the secret key.
8. Signed cookies provide integrity/authenticity, not encryption.

## Ethical Scope

All experiments were performed against a locally controlled Flask laboratory application.

The purpose was to understand signed-cookie authentication, cryptographic integrity, and session verification rather than to access accounts or bypass authentication on systems without authorization.
