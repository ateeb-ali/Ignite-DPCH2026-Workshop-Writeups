# GhostProcess

## Challenge Overview

A Debian package named `ghostprocess_1.0_amd64.deb` was recovered as part of an operating-system forensics exercise. The package claims to install a background service that “continuously leaks sensitive information over localhost.” The objective was to determine what the service actually does, locate the sensitive material it was designed to exfiltrate, and recover the flag that had been deliberately obfuscated inside the binary.

---

## Evidence

The supplied artefact was a single Debian package:

```
ghostprocess_1.0_amd64.deb
```

Extraction revealed the following layout:

```
./etc/systemd/system/ghostprocess.service
./usr/local/sbin/ghostprocess
./usr/share/doc/ghostprocess/changelog.Debian.gz
./usr/share/doc/ghostprocess/copyright
DEBIAN/control
DEBIAN/postinst
```

---

## Analysis

### Package Metadata & Service Definition

The control file describes a utility package that depends only on `libc6`. The accompanying `postinst` script enables and restarts a systemd unit on installation. The unit itself is straightforward:

```
[Unit]
Description=GhostProcess suspicious background service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/sbin/ghostprocess
Restart=always
RestartSec=1
User=root

[Install]
WantedBy=multi-user.target
```

The binary therefore runs as root, restarts indefinitely, and is expected to remain resident.

### Binary Behaviour

`ghostprocess` is a small, non-stripped ELF 64-bit PIE. Static inspection of strings and symbols immediately surfaces several interesting artefacts:

- the process name string `kworker/0:1`
- the banner `ghostprocess telemetry dump:`
- a temporary-file template `/tmp/.sysinfo-XXXXXX`
- a symbol named `enc_flag`

Dynamic analysis (and the corresponding disassembly of `main`) shows the following sequence:

1. The process double-forks and calls `setsid` to detach from the controlling terminal.
2. It uses `prctl(PR_SET_NAME, "kworker/0:1")` to masquerade as a kernel worker thread.
3. Command-line arguments are overwritten with zeros (classic argv scrubbing).
4. A 35-byte buffer is populated by XORing the contents of `enc_flag` with the constant `0x5a`.
5. A temporary file is created via `mkstemp`, the telemetry banner plus the decrypted buffer are written to it, the file is `fsync`ed, and then immediately unlinked. The data therefore exists only briefly in the page cache / unlinked inode.
6. A TCP listener is bound to `127.0.0.1:27002`. On every accepted connection the same 35-byte buffer is written to the client and the socket is closed.

In short, the service hides itself, briefly materialises the flag in an unlinked temporary file, and continuously serves the same material over a localhost socket.

### Flag Recovery

The encrypted material lives in the `.rodata` section under the symbol `enc_flag`:

```
00002040  3c 36 3b 3d 21 3d 32 6a  29 2e 05 2a 28 6a 39 69  |<6;=!=2j)..*(j9i|
00002050  6f 6f 05 6b 34 05 2e 32  69 05 37 6e 39 32 6b 34  |oo.k4..2i.7n92k4|
00002060  69 27 50 00                                       |i'P.|
```

A simple single-byte XOR with `0x5a` recovers the plaintext:

```python
python3 -c '
data = bytes.fromhex("3c363b3d213d326a292e052a286a39696f6f056b34052e326905376e39326b3469275000")
print(bytes(b ^ 0x5a for b in data).decode())
'
```

yielding:

```
flag{gh0st_pr0c355_1n_th3_m4ch1n3}
```

The same plaintext is the buffer that the binary later writes both to the transient temporary file and to every client that connects to the localhost listener.

---

## Flag

```
flag{gh0st_pr0c355_1n_th3_m4ch1n3}
```

---

## Tools Used

- `dpkg-deb` (package extraction and inspection)
- `readelf` / `objdump` / `nm` / `strings` (static binary analysis)
- Python 3 (XOR decryption)
```
