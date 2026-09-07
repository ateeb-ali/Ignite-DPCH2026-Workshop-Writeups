# Binarybreadcrumbs

## Challenge Overview

A backup job artifact was recovered from cold storage belonging to MeridianCorp. The package contained a Veeam Backup & Replication metadata file together with a SQLite database (and its associated WAL/SHM files) that had been pulled from the guest filesystem of `meridianhr-db01.meridian.corp`.

The database appeared largely empty, yet residual data and metadata breadcrumbs inside both the Veeam job description and the SQLite file itself still held the pieces of a flag that had been deliberately fragmented and partially deleted.

---

## Evidence

The supplied archive (`artifacts.tgz`) contained:

```
BkJob_2026-08-01_030015.vbm
MeridianHR.db
MeridianHR.db-shm
MeridianHR.db-wal
hint.txt
```

The accompanying hint simply identified the VBM as a Veeam Backup & Replication v6.5+ job description and named the source host.

---

## Analysis

### Veeam Metadata (`.vbm`)

The `.vbm` file is an XML document describing a successful scheduled backup of the MeridianHR database. Near the end of the file, inside an HTML-style comment that appears to be an internal verification note, a single Base64 token is present:

```
verification_token: ZmxhZ3t2M2FtX20zdGFf
```

Decoding yields the first half of the flag:

```
flag{v3am_m3ta_
```

### SQLite Database Contents

The database contains three tables (`employees`, `access_log`, `credentials`). Only `access_log` holds live rows:

| id | employee_id | timestamp            | action      | details                  |
|----|-------------|----------------------|-------------|--------------------------|
| 1  | 6           | 2026-07-30T03:14:22Z | data_export | ZmxhZ3t2M2FtX20zdGFf     |
| 2  | 9           | 2026-07-30T03:17:48Z | db_query    | ZDNsM3QzZF9yMHc1X2Z0d30= |

The first `details` value is identical to the token recovered from the VBM.  
The second value Base64-decodes to:

```
d3l3t3d_r0w5_ftw}
```

Together the two strings already form a complete flag.  

### Deleted-Row Residuals

Even without relying on the live `access_log` rows, the same second half can be recovered from free-page / purged-payload remnants still present in the main database file:

```
DELETED_ROW_AUDIT: d3l3t3d_r0w5_ftw}
page_freereclaim tbl=access_log \ldots purged_payload: d3l3t3d_r0w5_ftw}
```

These leftover strings confirm that the second fragment was once stored as ordinary row data and was later deleted, leaving the classic “binary breadcrumbs” that give the challenge its name.

---

## Flag

```
flag{v3am_m3ta_d3l3t3d_r0w5_ftw}
```

---

## Tools Used

- Standard Linux utilities (`strings`, `base64`, `tar`)
- Python 3 + `sqlite3` (schema inspection and live row extraction)
```