# GuessingGame

## Challenge Overview

GuessGame is a small online web application in which every registered player is assigned a secret number. Players can log in or register, view their own score card, and attempt to guess other players’ numbers. Successfully guessing another player’s secret awards points. The application claims that secret numbers are never disclosed — yet a privileged account holds the flag.

---

## Evidence

A single web application was provided at the challenge URL. The client-side JavaScript (`/app.js`) was fully accessible and contained the complete authentication and scoring logic.

---

## Analysis

### Client-Side Cryptography

Inspection of `/app.js` revealed two hard-coded constants used for AES encryption:

```js
const AES_KEY_HEX = '9d2c4f6a1b3e5c7f8a9b0c1d2e3f4a5b';
const AES_IV_HEX  = '1a2b3c4d5e6f708192a3b4c5d6e7f809';
```

The function `encryptId(id)` encrypts a numeric user ID with AES-CBC (PKCS7 padding) and returns the ciphertext as a Base64 string. This value is subsequently submitted as the `token` parameter to the `/api/score` endpoint.

### Score Endpoint Behaviour

The endpoint accepts a POST request of the form:

```json
{ "token": "<encrypted-id>" }
```

It decrypts the token server-side, looks up the corresponding user, and returns a JSON object containing `id`, `username`, `score`, and `secret`. Critically, the endpoint performs **no authentication or ownership check** — any valid token is accepted regardless of the caller’s session.

### Token Forgery

Because the AES key and IV are present in the client-side source, an identical `encryptId` function can be reconstructed in the browser console (after loading CryptoJS). Tokens for arbitrary user IDs can therefore be generated and submitted to `/api/score`.

Querying low sequential IDs produced the following relevant results:

```
ID 1 → {id: 1, username: 'admin', score: 1337, secret: 'flag{\ldots}'}
ID 2 → {id: 2, username: 'player', score: 42, secret: 'my secret number is 7'}
ID 3 → {id: 3, username: 'sally', score: 99, secret: 'my secret number is 42'}
ID 6 → {id: 6, username: 'admin', score: 58, secret: 'my secret number is 59'}
```

User ID 1 (the primary admin account) contained the flag inside the `secret` field.

### Root Cause

The application encrypts a direct object reference (the numeric user ID) with a client-visible key and treats the resulting ciphertext as an opaque, unforgeable token. This provides no real protection against IDOR; the encryption is effectively security theatre.

---

## Flag

```
flag{[YOUR FLAG HERE]}
```

---
## Tools Used

- Browser developer tools (Network tab, Console)
- CryptoJS (loaded from CDN for AES operations)
- Manual sequential ID enumeration against `/api/score`
