# FaultyOracle

## Evidence

A single binary was supplied:

- `faulty_oracle.bin`

---

## Analysis

### First Interaction

```bash
$ chmod +x faulty_oracle
$ ./faulty_oracle --help
FaultyOracle v1.0  --  AES-128-CBC padding oracle

  ./faulty_oracle flag          print encrypted flag (hex IV||CT)
  ./faulty_oracle <hex IV||CT>  one-shot query -> VALID | INVALID
  ./faulty_oracle               oracle mode, read hex queries from
                                stdin (one per line, one answer per
                                line on stdout)

$ ./faulty_oracle flag
3f1c92a7d84e6b05c2a9f37e1d40b68c
87479fea6803c8d8814943d86d6eba20e39b63727231863befa21f0798fee0a044831f26912fd40923293eade322126d
```

The output is a 64-byte hex string: a 16-byte IV followed by three 16-byte ciphertext blocks.

### Padding Oracle Attack

Because the binary acts as a perfect padding oracle, each ciphertext block can be decrypted independently by recovering its intermediate state (the value just after AES decryption and before the final XOR with the previous ciphertext block).

For a given target block `C` and the preceding block `P` (which may be the IV):

1. Construct a modified preceding block `C'` that forces a chosen padding length.
2. Brute-force the last byte of `C'` until the oracle returns `VALID`. That byte, XORed with the desired padding value, yields the corresponding intermediate byte.
3. Move one byte to the left and repeat, always maintaining valid padding for the already-recovered suffix.

The recovered intermediate block is then XORed with the original preceding block to obtain the plaintext block.

A short Python script automates the process by spawning the binary for every candidate query:

```python
import subprocess

ORACLE = "./faulty_oracle"

def query(hex_data: str) -> bool:
    result = subprocess.run([ORACLE, hex_data], capture_output=True, text=True)
    return result.stdout.strip() == "VALID"

def decrypt_block(prev: bytes, curr: bytes) -> bytes:
    intermediate = bytearray(16)
    plaintext = bytearray(16)
    for pad_len in range(1, 17):
        crafted = bytearray(16)
        for i in range(16 - pad_len + 1, 16):
            crafted[i] = intermediate[i] ^ pad_len
        for guess in range(256):
            crafted[16 - pad_len] = guess
            if query((crafted + curr).hex()):
                intermediate[16 - pad_len] = guess ^ pad_len
                plaintext[16 - pad_len] = intermediate[16 - pad_len] ^ prev[16 - pad_len]
                break
    return bytes(plaintext)
```

Applying the routine to each of the three ciphertext blocks yields:

```
flag{p4dd1ng_0r4cl3_15_n0t_ju5t_f0r_0r4cl3s}
```

after PKCS#7 padding is stripped from the final block.

---

## Flag

```
flag{p4dd1ng_0r4cl3_15_n0t_ju5t_f0r_0r4cl3s}
```

---

## Tools Used

- `chmod` / direct execution of the binary
- Python 3 (padding-oracle automation via subprocess)
