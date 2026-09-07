# Non Zero Drift

## Challenge Overview

An 8 KB OTA firmware image (`firmware.bin`) was encrypted with a hand-rolled ChaCha20 implementation. The encryption routine resets the lower byte of the nonce to `0x00` at every 4096-byte chunk boundary. Consequently both 4096-byte chunks in the file are encrypted under the identical keystream.

The goal is to recover the plaintext (and the flag contained within it) by exploiting the keystream reuse.

---

## Evidence

A single binary was supplied:

- `firmware.bin` (8192 bytes)

---

## Analysis

### Keystream Reuse

Because the nonce’s least-significant byte is forced to zero at every 4096-byte boundary, the two halves of the file share exactly the same keystream `K`:

```
C1[i] = P1[i] ⊕ K[i]
C2[i] = P2[i] ⊕ K[i]
```

XORing the two ciphertext chunks therefore cancels the keystream completely:

```
C1[i] ⊕ C2[i] = P1[i] ⊕ P2[i]
```

No ChaCha20 key recovery or internal state reconstruction is required; the problem reduces to a classic two-time pad.

### XOR of the Two Chunks

```python
data = open("firmware.bin", "rb").read()
chunk1, chunk2 = data[:4096], data[4096:]
xored = bytes(a ^ b for a, b in zip(chunk1, chunk2))
```

Inspection of the resulting difference immediately reveals two useful properties:

1. A long trailing run of zero bytes — both plaintexts are identical (zero-padded) in that region.
2. Near the beginning of the XOR output, readable ASCII fragments appear, including a build-string and, a little further in, a clear flag string.

### Why the Flag Appears Directly

At the offset where the flag resides, one plaintext contains the literal flag bytes while the other plaintext contains zero bytes (padding). Because

```
flag_byte ⊕ 0 = flag_byte
```

the XOR difference reproduces the flag characters in the clear. No further keystream recovery is necessary.

---

## Flag

```
flag{n0nZ3r0_dr1ft_k3ystr34m_r3us3}
```

---

## Tools Used

- Python 3 (chunk splitting, XOR, ASCII inspection)
```