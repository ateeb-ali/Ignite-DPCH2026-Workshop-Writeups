# Lateral Echo

## Challenge Overview

A set of forensic artefacts was recovered from a workstation that had executed a short PowerShell script. The script read two values from the current-user registry hive, XOR-decoded a hard-coded hex blob, and transmitted the result over a local TCP socket. The objective was to reconstruct the exact data flow and recover the flag that had been deliberately obfuscated inside the payload.

---

## Evidence

The supplied archive (`Lateral_Echo.zip`) contained three files:

```
PowerShell_Operational.evtx
Network_Traffic.pcap
NTUSER.DAT
```

- `PowerShell_Operational.evtx` – Microsoft-Windows-PowerShell/Operational event log (Event ID 4104)
- `Network_Traffic.pcap` – short localhost TCP capture on port 4433
- `NTUSER.DAT` – partial current-user registry hive containing the script’s configuration values

---

## Analysis

### PowerShell Script Block (Event ID 4104)

The single 4104 record contains the full script that was executed:

```powershell
$settings = Get-ItemProperty -Path 'HKCU:\Software\AppData\Settings'
$op = $settings.SessionID
$hex = '291c04151a0358010a7f07502d07441b5c007a4106473e195d5a1a7c0218'
$bytes = New-Object byte[] ($hex.Length / 2)
for ($i = 0; $i -lt $hex.Length; $i += 2) {
    $bytes[$i/2] = [Convert]::ToByte($hex.Substring($i, 2), 16)
}
$xored = for ($i = 0; $i -lt $bytes.Length; $i++) {
    $bytes[$i] -bxor [byte]$op[$i % $op.Length]
}
$client = New-Object System.Net.Sockets.TcpClient('127.0.0.1', $settings.TargetPort)
$out = ($xored | ForEach-Object { '{0:x2}' -f $_ }) -join ''
$payload = [System.Text.Encoding]::ASCII.GetBytes($out)
$stream = $client.GetStream()
$stream.Write($payload, 0, $payload.Length)
$stream.Flush()
Start-Sleep -Milliseconds 200
$client.Close()
```

Key observations:

- Configuration is read from `HKCU:\Software\AppData\Settings`
- A fixed hex string is converted to bytes
- Those bytes are XOR’d with the repeating `SessionID` value
- The resulting bytes are re-encoded as a continuous hex string and sent to `127.0.0.1` on the port stored in `TargetPort`

### Registry Hive (`NTUSER.DAT`)

The hive is minimal but contains the exact path referenced by the script:

```
Software\AppData\Settings
├── SessionID   = "Operation"   (REG_SZ, UTF-16LE)
└── TargetPort  = 4433          (REG_DWORD, 0x1151)
```

These two values are the only runtime secrets required to reverse the obfuscation.

### Network Capture (`Network_Traffic.pcap`)

A short localhost TCP conversation on port 4433 is present. After the three-way handshake two data segments are sent:

```
291c04151a0358010a7f07502d07441b
5c007a4106473e195d5a1a7c0218\r\n
```

Concatenated, the payload matches the hard-coded hex string from the PowerShell script exactly. No other traffic is present.

### XOR Decryption

With the key recovered from the registry (`Operation`) the original hex blob can be decoded:

```python
hexstr = "291c04151a0358010a7f07502d07441b5c007a4106473e195d5a1a7c0218"
key    = b"Operation"
data   = bytes.fromhex(hexstr)
flag   = bytes(b ^ key[i % len(key)] for i, b in enumerate(data))
print(flag.decode())
```

This yields the clear-text flag.

---

## Flag

```
flag{w1nd0w5_f0r3n51c5_m45t3r}
```

---

## Tools Used

- `evtx_dump.py` / python-evtx (PowerShell event log)
- `tshark` (pcap inspection)
- `python-registry` + `strings` / `xxd` (NTUSER.DAT)
- Python 3 (XOR decryption)
