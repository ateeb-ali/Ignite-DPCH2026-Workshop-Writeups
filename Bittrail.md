# BitTrail

## Challenge Description

The flag has been broken apart and every byte has been pushed around with bit-shift operations before being stored in a text file.

---

We are given a file called `encrypted.txt` containing:

```text
1A A2 32 DB 13 89 A3 FA A9 43 89 33 A3 FA 89 A9 FA A1 BB 99 9B 81 6B 99 EB
```

At first glance, it's just a sequence of hexadecimal bytes. Since the challenge is called **BitTrail**, I suspected that the data might have been modified at the bit level.

---

## Step 1 — Looking at the Data

Each value is a byte represented in hexadecimal.

For example:

```text
1A = 00011010
A2 = 10100010
32 = 00110010
DB = 11011011
```

The fact that the data is already neatly separated into bytes made me think this probably wasn't something like Base64 or a normal text encoding.

The challenge name was also a pretty big hint toward **bit manipulation**.

So instead of immediately writing a script, I decided to try some common bitwise operations using an online tool.

---

## Step 2 — Trying Bit Rotation

One operation that stood out was **bit rotation**.

A bit rotation is similar to a bit shift, except that the bits which fall off one side are wrapped around to the other side instead of being discarded.

For example, rotating an 8-bit value to the right by one position:

```text
10110100
     ↓
01011010
```

The `0` that was at the right side wraps around to the left.

I used **CyberChef**, which is a free browser-based tool commonly useful for CTF challenges.

[CyberChef](https://cyberchef.io)

CyberChef has both **Rotate left** and **Rotate right** operations, and its Rotate operation can work on individual bytes.

I entered the hex data and tried a few rotation values.

The important settings were:

```text
Operation: Rotate right
Amount: 3
Carry through: disabled
```

After applying the rotation, the output became readable ASCII:

```text
CTF{b1t_5h1ft_15_4w3s0m3}
```

At this point, the plaintext was clearly a flag, so the rotation was confirmed.

---

## Flag

```text
CTF{b1t_5h1ft_15_4w3s0m3}
```

## Tools Used

* **CyberChef** — used to perform the byte-wise right rotation and reveal the plaintext.
  [CyberChef](https://cyberchef.io)
* **Online Binary Rotate Right Calculator** — another free alternative for experimenting with bit rotations.
  [Binary Rotate Right Calculator](https://binarycon.com/tools/binary-rotate-right-calculator)
