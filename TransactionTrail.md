# TransactionTrail

## Challenge Overview

TrustPay is a corporate transfer portal where authenticated users can review their recent transfers and initiate new ones. A valid demo account is available on the sign-in page. The objective is to investigate the transfer functionality and perform an unauthorized transaction to recover the flag.

---

## Evidence

A single web application was provided at the challenge URL. The landing page presents a sign-in form together with the demo credentials `alice` / `password123`. After authentication the application displays the user’s balance and a list of recent transfers, with a link to initiate a new transfer.

---

## Analysis

### Authentication

The demo credentials are accepted without modification:

```
username: alice
password: password123
```

Successful login sets a standard `PHPSESSID` cookie and redirects to the dashboard. The dashboard shows an available balance of `$500.00` and a short history of completed transfers.

### Transfer Form

The “New Transfer” page (`transfer.php`) presents a simple form containing:

- A beneficiary dropdown
- An amount field

The dropdown lists three ordinary beneficiaries (IDs 101–103) and one additional option:

```html
<option value="999" disabled>Global Compliance Hold (restricted)</option>
```

A note below the form states that transfers to restricted accounts are not permitted. The `disabled` attribute prevents selection through the normal browser UI.

### Bypassing the Client-Side Restriction

Because the restriction is enforced only by the HTML `disabled` attribute, it can be ignored by submitting the form parameters directly. A simple POST request that includes the restricted beneficiary ID is sufficient:

```bash
curl -b cookies.txt -X POST \
  -d "beneficiary_id=999&amount=1" \
  http://<challenge-host>/transfer.php
```

(The session cookie obtained after logging in as Alice must be supplied.)

### Server Response

The server processes the request and returns an “Unauthorized transaction escalated” incident page. The page records that a transfer which is not permitted for the account was processed, blocked, and escalated. The incident table contains the beneficiary name, the amount, and an “Incident reference” field that holds the flag.

### Root Cause

The application trusts the client-side form controls for authorization decisions. No server-side check verifies whether the submitted `beneficiary_id` is allowed for the authenticated user. Consequently any authenticated session can transfer funds to the restricted “Global Compliance Hold” account simply by supplying the corresponding identifier.

---

## Flag

```
flag{[YOUR FLAG HERE]}
```

---

## Tools Used

- Browser developer tools (form inspection, cookie extraction)
- `curl` (session handling and direct POST to the restricted beneficiary)
