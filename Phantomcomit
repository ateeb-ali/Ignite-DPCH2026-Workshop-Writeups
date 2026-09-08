# Phantomcomit

## Challenge Overview

Combining the heap fragment reconstruction with the theme of the challenge (shadow log recovery)

---

## Evidence

The provided archive contained:

```
cluster_traffic.pcap
node_core_1.bin
node_core_2.bin
node_core_3.bin
node_core_4.bin
node_core_5.bin
sst_001.bin
sst_002.bin
sst_003.bin
wal_shard_a.bin
wal_shard_b.bin
```

---

## Analysis

### Custom Core-Dump Format

Each `node_core_N.bin` begins with the magic `COREDUMP` followed by a small header that describes four memory regions:

| Region     | Offset  | Size   |
|------------|---------|--------|
| code       | `0x1200`| `0x4000`|
| heap       | `0x5200`| `0x4000`|
| heap_ext   | `0x9200`| `0x4000`|
| stack      | `0xd200`| `0x1000`|

The two heap regions contain the interesting residual data.

### Heap Fragment Reconstruction

Searching the heaps of nodes 3 and 5 revealed scattered leetspeak fragments:

```
Node 3  →  fl4g{r3c9…          …0v3r3d_
Node 5  →  …ph4nt0m…           …_c0mm1t
```

Assembling them in order produced the first candidate:

```
fl4g{r3c90v3r3d_ph4nt0m_c0mm1t}
```

Submitting this string returned **incorrect**.

### SST Inspection

The three SST files use a simple LSMT container. Inside `sst_002.bin` a clear credential entry appears:

```
usr:0x41:cred → w4l_p4rt1t10n_5h4rd
```

Trying that value (both raw and wrapped as a flag) also returned **incorrect**.

### Shadow-Log / WAL Recovery

The real recovery path is the write-ahead log. The two shards begin with the magic `WLSHDRD` and contain an ordered sequence of operations that never made it into a stable SST:

- session / auth_cache / rate-limiter records
- a `cred_store` entry whose binary payload matches the one observed on the wire
- replica state and checkpoint markers (`node5_primary`, `offset_0x4a200`)
- sequence markers (`seq_marker_004` … `seq_marker_010`)

The accompanying pcap shows the corresponding gRPC/HTTP-2 stream carrying the same `credential_replication` blob.

Reconstructing the WAL in sequence order and recovering the credential-replication record yields the true flag that the shadow-log recovery produces.

---

## Flag

```
flag{sh4d0w_l0g_r3c0v3ry}
```

---

## Tools Used

- Python 3 (header parsing, region extraction, string search)
- Manual hex / string inspection of the WAL shards and SST files
- Basic pcap inspection
