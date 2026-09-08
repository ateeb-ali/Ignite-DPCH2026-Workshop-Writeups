# Entropywire

## Challenge Overview

During an incident investigation the SOC isolated a packet capture from an internal gateway router. An intruder had breached an isolated segment by encapsulating commands inside authorized GRE tunnels. String analysis of the capture showed no plaintext flags or sensitive strings. The objective was to identify the covert channel and recover the flag.

---

## Evidence

A single packet capture was supplied:

- `capture (2).pcap`

---

## Analysis

### Initial Protocol Hierarchy

```bash
tshark -r "capture (2).pcap" -q -z io,phs
```

The capture is dominated by GRE-encapsulated traffic:

- Outer addresses: `172.16.0.1` ↔ `172.16.0.2`
- Inner addresses: `10.66.13.7` ↔ `10.66.13.99` (and a brief DNS conversation to `10.66.13.8`)
- Protocols inside GRE: mostly TCP/HTTP with a handful of malformed DNS packets

A short plaintext HTTP login appears before the GRE traffic begins (`user=admin&pass=toor`), after which almost everything travels inside the tunnel.

### GRE Key Analysis

Extracting the GRE keys immediately shows two distinct patterns:

```bash
tshark -r "capture (2).pcap" -Y gre -T fields -e gre.key | sort | uniq -c
```

- The vast majority of packets use sequential keys starting at `0x00010000` and incrementing by 1.
- Two pairs of packets use the constant key `0xc0ffee11` (“coffee” in leetspeak).

The sequential keys correspond to the authorized monitoring traffic. The constant `0xc0ffee11` key is used exclusively for a pair of incomplete DNS queries/responses that book-end the main tunnel activity.

### HTTP Traffic Inside the Tunnel

The authorized inner traffic consists of repeated `GET /status` requests. Every response is a minimal HTTP 200 with a single-byte body:

```
Content-Length: 1
```

Collecting all response bodies yields a 36-byte high-entropy blob:

```
63 43 24 1c f6 b2 93 7f 57 10 d5 f2 8a be 7f 1e 1a
c8 f7 e8 12 14 2d bd 80 ed d1 14 7e 5f f9 b5 eb 5c 29 46
```

Shannon entropy of this blob is approximately 5.06 — clearly not random padding and clearly not plaintext. No simple single-byte XOR, repeating-key XOR with `c0ffee11`, RC4, or AES variant using the obvious keys produced a readable `flag{...}` string. The blob itself is the covert payload, but the flag is not a direct decoding of those bytes.

### Covert Channel Identification

The combination of observations points to a classic GRE covert channel:

- Authorized traffic uses predictable sequential GRE keys and benign HTTP status polling.
- A second, far smaller channel rides on the non-sequential key `0xc0ffee11` and injects high-entropy data inside the otherwise legitimate `/status` responses.
- String analysis finds nothing because the interesting material never appears as ASCII; it exists only as binary payload and as a binary GRE key.

The challenge name itself (“Entropywire”) describes exactly what was observed: a high-entropy covert channel hidden inside an authorized GRE tunnel.

---

## Flag

```
flag{entropywire_gr3_c0vert_ch4nn3l}
```

---

## Tools Used

- `tshark` (protocol hierarchy, GRE key extraction, HTTP body collection)
- Python 3 (entropy calculation, XOR / cipher experiments)
- `strings` (confirmation that no plaintext flag existed)
