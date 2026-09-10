# Vault Balance

## Challenge Overview

VaultBalance is a small PHP web application that lets the employees of a bank manage their personal digital vaults. Users can create an account, sign in, and view their own balance as well as a short directory of other vaults. The objective is to recover the flag stored somewhere inside the application.

---

## Evidence

The challenge exposes a single web application. The landing page presents a sign-in form and a link to create a new account. After registration the application redirects the user to a dashboard that displays the authenticated user’s vault together with a public directory of other vaults.

---

## Analysis

### Application Behaviour

Registration is unrestricted. Supplying any username and password creates a new account and immediately authenticates the user. The resulting dashboard contains two sections:

- The current user’s own vault (linked as `account.php?id=<own_id>`)
- A table titled “Vault directory” listing several other vaults, each with an “Open” link of the form `account.php?id=<n>`

One entry in the directory is labelled **Executive Vault** and points to `account.php?id=10`.

### Access-Control Check

Requesting the page for the current user’s own vault ID returns the expected details and a “Your vault” tag. Requesting any other ID, including the Executive Vault, is also permitted. The application performs no ownership or authorization check on the `id` parameter of `account.php`.

This is a classic **Insecure Direct Object Reference (IDOR)**.

### Recovering the Flag

Visiting

```
/account.php?id=10
```

while authenticated returns the full details of the Executive Vault. The Description field of that vault contains the flag.

---

## Flag

```
flag{[YOUR FLAG HERE]}
```

---

## Tools Used

- `curl` (account registration, session handling and IDOR testing)
- Browser developer tools (initial request and response inspection)
