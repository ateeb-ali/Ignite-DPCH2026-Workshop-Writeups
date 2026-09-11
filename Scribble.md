# Scribble

## Challenge Overview

Scribble Pad is a minimal PHP web application that accepts free-form text input and immediately reflects it back to the user. The challenge description states that a reflected input field leaks the flag when crafted script payloads are submitted, and that the flag itself is injected into the container from the Dockerfile as an environment variable and is only ever handed to a real browser session.

---

## Evidence

A single web application was provided at the challenge URL. The landing page presents a simple form with a text input named `scribble` and a submit button. Submitting any value causes the input to be echoed verbatim inside a `<div class="scribble">` element.

---

## Analysis

### Page Source

Viewing the source of the landing page reveals two important details:

```html
<script src="flag.php"></script>
```

and the complete absence of any output encoding on the reflected value:

```html
<div class="scribble">
    [user-supplied input]
</div>
```

### Behaviour of flag.php

A direct request to `/flag.php` (for example via `curl`) returns only a comment:

```
/* forbidden: open the app in your browser first */
```

The endpoint therefore refuses to serve the flag unless the request arrives in the context of a genuine browser session (proper cookies, User-Agent, and the normal page-load chain).

### Recovering the Flag

Opening the application in a real browser causes the page to load `/flag.php` as a JavaScript resource. The response body is a single assignment:

```js
window.FLAG = "flag{[YOUR FLAG HERE]}";
```

The value is therefore available both in the Network tab (response body of `flag.php`) and in the browser console via `window.FLAG`.

Although the reflected `scribble` parameter is completely unsanitized and would support classic XSS, that path is unnecessary; the flag is already delivered to any legitimate browser session.

### Root Cause

The application intentionally gates the environment-variable flag behind a browser-session check. Once a real browser has loaded the page, the flag is simply written into a global JavaScript variable and can be read without further exploitation.

---

## Flag

```
flag{[YOUR FLAG HERE]}
```

---

## Tools Used

- Browser developer tools (Network tab, Console)
- `curl` (for confirming the forbidden response)
