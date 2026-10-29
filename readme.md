# Storage Media Data Recovery & FAT32 File System Analysis

An experimental digital forensics project evaluating data remanence and recovery methodologies across a FAT32 storage volume. This project contrasts **Metadata-based File System Recovery** with **Signature-based File Carving** across three deletion scenarios (**Standard Delete**, **Shift+Delete**, and **Quick Format**) and validates all recovered artifacts using SHA-256 cryptographic hashing.

## Key Highlights & Findings

* **Forensic Integrity:** 100% of recoverable files were verified byte-for-byte against pre-deletion SHA-256 baseline hashes.
* **Sector-Padding Anomaly Analysis:** Diagnosed a 104-byte padding discrepancy in carved JPEGs using PowerShell partial-stream byte hashing.
* **Tool Comparison:** Benchmarked Recuva (Metadata/Deep Scan) against PhotoRec (Signature Carving).
* **FAT32 Analysis:** Detailed breakdown of directory table alterations (`0xE5` marker) and cluster chain behavior during deletion events.

### Experimental Recovery Matrix

| Attempt | Target File | Scenario | Tool | Status | SHA-256 Match | Filename Outcome |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **01** | `note.txt` | Normal Delete | Recuva (Standard) | **Success** | **Match** | Partial (`_ote.txt`) |
| **02** | `note.txt` | Normal Delete | PhotoRec | **Failed** | N/A | Missed (No magic bytes) |
| **03** | `doc.pdf` | Shift+Delete | Recuva (Standard) | **Success** | **Match** | Partial (`_oc.pdf`) |
| **04** | `doc.pdf` | Shift+Delete | PhotoRec | **Success** | **Match** | Renamed (`f0032832...`) |
| **05** | `photo.jpg` | Quick Format | Recuva (Deep Scan) | **Success** | **Mismatch\*** | Generic (`[000002].jpg`) |
| **06** | `photo.jpg` | Quick Format | PhotoRec | **Success** | **Match** | Generic (`f0033088.jpg`) |
| **07** | `doc.pdf` | Quick Format | Recuva (Deep Scan) | **Success** | **Match** | Generic (`[000001].pdf`) |
| **08** | `doc.pdf` | Quick Format | PhotoRec | **Success** | **Match** | Renamed (`f0032832...`) |
| **09** | `note.txt` | Quick Format | Recuva (Deep Scan) | **Failed** | N/A | Missed (Directory wiped) |
| **10** | `note.txt` | Quick Format | PhotoRec | **Failed** | N/A | Missed (No magic bytes) |

*\*Content verified identical after trimming trailing 104-byte sector-boundary padding.*

---

## Technical Architecture & Methodology

```
┌────────────────────────────────────────────────────────────────────────┐
│                        FAT32 Target Media                              │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
       ┌────────────────────────────┼────────────────────────────┐
       ▼                            ▼                            ▼
┌──────────────┐             ┌──────────────┐             ┌──────────────┐
│ Scenario A   │             │ Scenario B   │             │ Scenario C   │
│ Standard     │             │ Shift+Delete │             │ Quick Format │
│ Deletion     │             │ Operation    │             │ Operation    │
└──────┬───────┘             └──────┬───────┘             └──────┬───────┘
       │                            │                            │
       └────────────────────────────┼────────────────────────────┘
                                    │
                                    ▼
                     ┌─────────────────────────────┐
                     │ Parallel Recovery Pipelines │
                     └──────────────┬──────────────┘
                                    │
           ┌────────────────────────┴────────────────────────┐
           ▼                                                 ▼
┌────────────────────────────┐                     ┌───────────────────┐
│ Recuva (Metadata/Deep Scan)│                     │ PhotoRec (Carver) │
└──────────┬─────────────────┘                     └─────────┬─────────┘
           │                                                 │
           └────────────────────────┬────────────────────────┘
                                    │
                                    ▼
                     ┌─────────────────────────────┐
                     │ Hash Integrity Verification │
                     │       (SHA-256 Engine)      │
                     └─────────────────────────────┘
```

---


---

## Core Findings & Defensive Takeaways

1. **Standard Delete & Quick Format Do Not Wipe Data:** On FAT32, deleting a file merely alters the directory entry (replacing the first character with `0xE5`) and marks the clusters as unallocated. Quick format rewrites the file system structures but leaves data areas untouched.
2. **Metadata Recovery vs. Signature Carving:**
   * **Recuva (Metadata):** Preserves file names (partially) and recovers non-signature files like plain text (`.txt`), but fails if directory tables are formatted.
   * **PhotoRec (Carving):** Ignores file system structures entirely and reads raw sectors for file signatures (magic bytes). Highly effective post-format, but loses original file names and cannot recover unstructured plain text.
3. **Forensic Safety Constraints:** To prevent data overwriting during analysis, target drives must remain strictly write-blocked, and all extracted artifacts must be saved to external target drives.

---

## Author & Contact

* **Author:** Annus Jawad
* **Role:** Digital Forensics & Incident Response (DFIR) Trainee
* **LinkedIn:** [Your LinkedIn Profile]
* **GitHub:** [Your GitHub Profile]
