# CookieCrumb

## Challenge Overview

CookieCrumb is a bakery-themed web application that presents a simple login form. Users can sign in with a username and password; successful authentication grants access to a personal dashboard. Somewhere in the application a privileged admin session holds the flag. The objective is to escalate from a normal user session to the admin context and recover the flag.

---

## Evidence

The challenge exposes a single web application. The landing page contains a plain sign-in form (username + password). No account credentials were supplied with the challenge.

---

## Analysis

### Application Behaviour

Attempting the most obvious default credentials immediately succeeds:

```
username: guest
password: guest
```

After authentication the application sets a `PHPSESSID` cookie. Inspection of the cookie value in the browser developer tools reveals a clean 32-character hexadecimal string — the exact shape of an MD5 digest rather than a cryptographically random session identifier.

### Session Identifier Construction

The observed session cookie for the `guest` account matches the MD5 hash of the string `"guest"`. This indicates that the application derives the session identifier directly from the username:

```
PHPSESSID = md5(username)
```

Consequently any username whose MD5 is known can be impersonated by simply overwriting the cookie.

### Session Forgery

Using the browser developer tools (Application / Storage → Cookies) the existing `PHPSESSID` value is replaced with the MD5 hash of `"admin"`:

```
21232f297a57a5a743894a0e4a801fc3
```

Refreshing the page (or navigating to any authenticated endpoint) causes the application to treat the request as an authenticated admin session. The flag is returned in the response body of the resulting dashboard / privileged page.

---

## Flag

```
flag{[YOUR FLAG HERE]}
```

---

## Tools Used

- Browser developer tools (cookie inspection and modification)
- Online / local MD5 calculation (hash of `admin`)
