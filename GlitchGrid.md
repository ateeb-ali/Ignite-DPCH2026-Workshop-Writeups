# GlitchGrid

## Challenge Overview

A stripped 64-bit ELF binary named `glitchgrid` was provided. Running it prints a short banner together with an encrypted flag and a hint about a scrambled 5×4 memory grid that had suffered a single-cell corruption.

---

## Evidence

A single binary was supplied:

- `glitchgrid`

---

## Analysis

### Making it executable and first run

```bash
$ chmod +x glitchgrid
$ ./glitchgrid
[GLITCHGRID] v1.0
a 5x4 memory grid was found after the glitch.
encrypted flag: 06336e17050f565d4a060f150b5d5500040f1304
hint: the columns are scrambled, one cell is corrupted.
```

The program then waits for a 20-byte input.

### XOR with the obvious keystream

`strings` immediately showed the constant `gl1tch` sitting in `.rodata`. Treating the printed hex as a 20-byte ciphertext and XORing it against the repeating keystream `gl1tch` produced:

```
a__cfg11{rl}l1dtggth
```

### Splitting into columns

The resulting 20-byte string was split into five consecutive 4-byte groups:

```
a__c
fg11
{rl}
l1dt
ggth
```

These five groups are the scrambled columns. Re-ordering them back into their natural left-to-right positions and stacking them produced the following grid:

```
fg11
l1dt
a__c
ggth
{rl}
```

Reading the grid top-to-bottom, left-to-right already looked like a flag, except the third row contained `tl1tch` instead of the expected `gl1tch`.

```
flag{g1_gr1d_tl1tch}
```

### Manual correction

Given that the challenge itself is called **GlitchGrid** and the keystream string is literally `gl1tch`, the single corrupted character was changed from `t` to `g`. The corrected grid becomes:



which reads as the final flag.

---

## Flag

```
flag{g1_gr1d_gl1tch}
```

---

## Tools Used

- `chmod` / direct execution
- `strings`
- Python 3 (simple XOR)
- Manual rearrangement of the 4-byte columns
```
