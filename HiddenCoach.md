# HiddenCoach

## Challenge Overview

On 22 January 2024 a senior finance employee at Arup Hong Kong was targeted in a sophisticated deepfake-enabled Business Email Compromise (BEC) attack. Attackers registered phishing infrastructure, harvested credentials via a fake login portal, then convinced the victim to join a video call in which all seven participants were AI-generated deepfakes trained on publicly available Arup meeting recordings. The deepfake CFO ordered 15 wire transfers totalling HKD 200 000 000 (~USD 25.6 M).

CERT-HK recovered three evidence artefacts from the incident. The objective was to analyse them and recover:

1. The attacker’s real name  
2. The exact timestamp of a critical forensic anomaly in the financial records  
3. The attacker’s Telegram account  

**Flag format:** `flag{name:timestamp:telegram}`

---

## Evidence Artefacts

The provided archive (`files.tgz`) contained:

```
evidence_1_phishing_email.eml
evidence_2_transactions.db
evidence_3_osint_archive/
├── ct_logs.txt
├── github_metadata.json
├── telegram_export.json
└── whois_records.txt
```

---

## Analysis

### 1. Phishing Email (`evidence_1_phishing_email.eml`)

The email was sent on 18 January 2024 to `david.chen@arup.com.hk` and impersonated CFO “David Wei”.

Key artefacts extracted from headers and HTML:

- `X-Author-Origin: phishing-kit-v3; author=PENDLE; ref=GF-0042`
- `X-Campaign-ID` (Base64) decoded to `GHOST_FLORD_PENDLE-2024`
- Hidden callback token (Base64) decoded to `linkedin://in/profile/phantom-dev-hk-2024`
- Tracking pixel and reply-to domain pointed to attacker-controlled infrastructure (`*.slmalta.com`, `arup-secure-portal.com`, etc.)

These identifiers later proved useful for pivoting into the OSINT data.

### 2. Transaction Database (`evidence_2_transactions.db`)

SQLite database containing all 15 fraudulent transfers. Relevant schema:

```sql
CREATE TABLE transfers (
    txn_id TEXT PRIMARY KEY,
    timestamp TEXT NOT NULL,
    amount_hkd INTEGER NOT NULL,
    dest_account TEXT NOT NULL,
    dest_bank_swift TEXT NOT NULL,
    country_code TEXT NOT NULL,
    memo_reference TEXT NOT NULL,
    approved_by TEXT NOT NULL,
    approval_method TEXT NOT NULL
);
```

All transfers were approved via `video_call_auth` by `david.chen`. Fourteen records used the timezone offset `+0800` (Hong Kong Time). One record stood out:

```
TXN-ARUP-007 | 2024-01-22 10:01:33 +0700 | 12 500 000 HKD | BREDVUVU | VU
```

This is the **only** entry using Vietnam time (`+0700`). The Telegram export later confirmed the attackers noticed the victim had used his local machine timezone on this transfer.

**Critical forensic anomaly timestamp:** `2024-01-22 10:01:33`

### 3. OSINT Archive

#### WHOIS & Certificate Transparency

Multiple attacker domains (`slmalta.com`, `arup-internal-login.com`, `tracking-domain.xyz`, `update-service-cdn.xyz`, `arup-secure-portal.com`, `cloudscs.com`) shared:

- Registration email: `ghost.floor.ops@protonmail.com`
- Nameserver cluster: `ns1.cloudscs.com` / `ns2.cloudscs.com`
- One registrant country recorded as `VN` (Vietnam)

#### GitHub Metadata (`github_metadata.json`)

Public repository `phantom-dev-hk/deepfakeswap-gf` (fork of a face-swap tool) contained:

- Owner login: `phantom-dev-hk`
- Blog field: `https://nguyenvanhieu.dev`
- Location: `Ho Chi Minh City, Vietnam`
- Bio: “Security researcher | interested in AI and computer vision | HCMC, Vietnam”
- Commit author email matched the ProtonMail address used for domain registration
- Commit messages referenced the campaign ID `GF-0042` and “meeting-quality source material”

The personal blog domain is the strongest real-name indicator present in the evidence.

**Attacker’s real name:** `Nguyen Van Hieu`

#### Telegram Export (`telegram_export.json`)

Intercepted private group “Ghost Floor Ops”. Primary operator:

- Handle: `@GF_Phantom_Ops`
- User ID: `user128472931`

Messages document the entire operation timeline, including:

- Infrastructure setup
- Training of deepfake models on Arup meeting footage
- Credential harvest
- Live video-call execution
- Explicit reference to the timezone mistake on TXN-007
- Operational cleanup

**Attacker’s Telegram account:** `@GF_Phantom_Ops`

---

## Flag Construction

Combining the three recovered values in the required format:

```
flag{Nguyen Van Hieu:2024-01-22 10:01:33:@GF_Phantom_Ops}
```

---

## Summary of Pivots

| Artefact              | Key Finding                          | Pivot                          |
|-----------------------|--------------------------------------|--------------------------------|
| Phishing email        | Campaign ID `GF-0042`, author `PENDLE`, LinkedIn-style token `phantom-dev-hk-2024` | GitHub username & campaign correlation |
| Transactions DB       | Single `+0700` timestamp on TXN-007  | Confirmed by Telegram chat     |
| WHOIS / CT logs       | Shared email `ghost.floor.ops@protonmail.com`, VN registrant | Infrastructure attribution    |
| GitHub                | Blog `nguyenvanhieu.dev` + HCMC      | Real-name attribution          |
| Telegram              | `@GF_Phantom_Ops` running the op     | Operator handle                |

All three required pieces of information were recovered solely from the supplied evidence artefacts.