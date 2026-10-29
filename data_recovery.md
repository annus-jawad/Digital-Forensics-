# Digital Forensics Case Study: Storage Media Data Recovery & File System Mechanics Analysis

**Author:** Annus Jawad

**Role:** Digital Forensics & Incident Response (DFIR) Trainee

**Date:** September 28, 2026

**Environment:** Windows 11 Pro (v10.0.26200), PowerShell 7+

**Tools & Utilities:** Recuva v1.55.0, PhotoRec v7.3-WIP (TestDisk Suite), `Get-FileHash` (SHA-256)

---

## Executive Summary

When files are deleted from traditional file systems, underlying binary data often remains intact in unallocated clusters until overwritten. 

This project presents a controlled forensic investigation evaluating data remanence and recovery methodologies across a **FAT32-formatted USB storage volume**. By executing three deletion scenarios (**Standard Delete**, **Shift+Delete**, and **Quick Format**) across multiple file types, this study contrasts **Metadata-based File System Recovery** against **Signature-based File Carving**. 

### Key Technical Achievements

1. **Forensic Integrity Protocols:** Applied SHA-256 cryptographic hashing prior to deletion and post-recovery to objectively verify byte-level integrity.
2. **Metadata Analysis:** Dissected directory table entry alterations during standard deletion events on FAT32 volumes.
3. **Carving Anomaly Diagnosis:** Identified and verified a 104-byte sector-padding anomaly in signature-based carving outputs using offset-trimmed hash calculations.
4. **Tool Benchmarking:** Comparative assessment of Recuva (Metadata/Deep Scan) vs. PhotoRec (File Carving).

---

## Target Storage & Forensic Baseline

### Storage Media Specifications

* **Target Device:** 16 GB USB Flash Memory (`\\.\PhysicalDrive3`, $15 \text{ GB}$ / $14 \text{ GiB}$)
* **File System:** FAT32 (`LABFAT`)
* **Volume Structure:** 8,192-byte Cluster Size | 30,720,000 Total Sectors
* **Forensic Safety Constraints:** The target drive was isolated from system operations. All recovery artifacts were exported strictly to external local storage (`C:\Recovered\`) to prevent write operations on the target media.

### Initial File Baseline & Hash Table

| File Name | Extension | Category | Exact Size (Bytes) | Baseline SHA-256 Fingerprint |
| :--- | :--- | :--- | :--- | :--- |
| `note.txt` | `.txt` | ASCII Plain Text | 54,018 | *Pre-computed Forensic Hash* |
| `doc.pdf` | `.pdf` | Portable Document Format | 71,134 | *Pre-computed Forensic Hash* |
| `photo.jpg` | `.jpg` | JPEG Image Data | 303,512 | `86D29B28FEE86ECA16000E5605D2AA09348B949531216631B7FDAC20FB206EBB` |

---

## Deletion Scenarios & Methodology

```
┌────────────────────────────────────────────────────────────────────────┐
│                        FAT32 Target Media                              │
│                      (Initial Baseline Set)                            │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
       ┌────────────────────────────┼────────────────────────────┐
       ▼                            ▼                            ▼
┌──────────────┐             ┌──────────────┐             ┌──────────────┐
│ Scenario A   │             │ Scenario B   │             │ Scenario C   │
│ Standard     │             │ Shift+Delete │             │ Quick Format │
│ Deletion     │             │ Operation    │             │ Operation    │
│ (`note.txt`) │             │ (`doc.pdf`)  │             │ (`photo.jpg`)│
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
│       Recuva 1.55          │                     │    PhotoRec 7.3   │
│ ── Standard Metadata Scan  │                     │ ── Signature-Based│
│ ── Signature Deep Scan     │                     │    File Carving   │
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

### Scenario Breakdown

1. **Scenario A (Standard Delete):** `note.txt` removed via Windows Explorer context menu. On removable volumes, Windows bypasses the Recycle Bin and updates directory markers immediately.
2. **Scenario B (Shift+Delete):** `doc.pdf` subjected to explicit bypass deletion.
3. **Scenario C (Quick Format):** Drive subjected to a high-level format (`Quick Format`), overwriting file system structures (Boot Sector, FAT, Root Directory) while leaving cluster payload areas intact.

---

## Experimental Results Matrix

| Attempt # | Target File | Scenario | Recovery Tool | Recovery Status | SHA-256 Match | Filename Outcome | Data Integrity Notes |
| :---: | :--- | :--- | :--- | :---: | :---: | :--- | :--- |
| **01** | `note.txt` | A: Normal Delete | Recuva (Standard) | **Success** | **Match** | Partial (`_ote.txt`) | Full recovery via directory entry. |
| **02** | `note.txt` | A: Normal Delete | PhotoRec | **Failed** | N/A | N/A | Missed: Plaintext lacks magic bytes. |
| **03** | `doc.pdf` | B: Shift+Delete | Recuva (Standard) | **Success** | **Match** | Partial (`_oc.pdf`) | Full recovery via directory entry. |
| **04** | `doc.pdf` | B: Shift+Delete | PhotoRec | **Success** | **Match** | Modified (`f0032832...`) | Recovered via PDF header signatures. |
| **05** | `photo.jpg` | C: Quick Format | Recuva (Deep Scan) | **Success** | **Mismatch** | Generic (`[000002].jpg`) | 104 extra padding bytes added. |
| **06** | `photo.jpg` | C: Quick Format | PhotoRec | **Success** | **Match** | Generic (`f0033088.jpg`) | Exact EOF marker match. |
| **07** | `doc.pdf` | C: Quick Format | Recuva (Deep Scan) | **Success** | **Match** | Generic (`[000001].pdf`) | Unallocated space carved successfully. |
| **08** | `doc.pdf` | C: Quick Format | PhotoRec | **Success** | **Match** | Modified (`f0032832...`) | Unallocated space carved successfully. |
| **09** | `note.txt` | C: Quick Format | Recuva (Deep Scan) | **Failed** | N/A | N/A | Missed: Directory deleted & no signature. |
| **10** | `note.txt` | C: Quick Format | PhotoRec | **Failed** | N/A | N/A | Missed: Plaintext lacks magic bytes. |

---

## Deep-Dive Technical Analysis

### 1. FAT32 File System Mechanics During Deletion

When a file is deleted under FAT32:
* **Directory Entry Transformation:** The first character of the filename in the directory record is overwritten with hex byte `0xE5` (represented as `_` by recovery software).
* **Cluster Allocation Table:** The File Allocation Table entries corresponding to the file's cluster chain are updated to `0x00000000` (Marked Unallocated).
* **Data Payload Payload Preservation:** Binary clusters containing the actual file payload are unmodified until target allocation blocks are reassigned to new I/O operations.

```
Initial Directory Entry:  [ n ] [ o ] [ t ] [ e ] [ . ] [ t ] [ x ] [ t ]  ---> Cluster Pointer: 0x0032
Post-Deletion Entry:     [0xE5] [ o ] [ t ] [ e ] [ . ] [ t ] [ x ] [ t ]  ---> Cluster Pointer: 0x0000 (Data intact at 0x0032)
```

### 2. Diagnosis of the Recuva Padding Anomaly (Attempt #05)

During Scenario C, Recuva's Deep Scan successfully carved `photo.jpg`, but returned an invalid full-file SHA-256 hash. Investigation revealed:

* **Original Payload Size:** 303,512 bytes
* **Recovered Payload Size:** 303,616 bytes ($\Delta = +104 \text{ bytes}$)
* **Sector Boundary Multiplier:** $303,616 \div 512 = 593.0 \text{ exact sectors}$

```
Original JPEG Data (303,512 Bytes)                    Sector Boundary (303,616 Bytes)
[...................................................| Padding (104 Bytes) ]
^                                                   ^
Start of File                                       Calculated Sector End
```

**Diagnostic Verification:** Executing a PowerShell partial-stream hash over the first 303,512 bytes yielded the exact SHA-256 baseline hash, confirming that the image data was uncorrupted and that the mismatch was caused solely by trailing 512-byte sector padding.

```powershell
# PowerShell verification of partial byte stream
$bytes = [System.IO.File]::ReadAllBytes("C:\Recovered\Recuva\[000002].jpg")
$stream = New-Object System.IO.MemoryStream($bytes, 0, 303512)
(Get-FileHash -InputStream $stream -Algorithm SHA256).Hash
```

### 3. Tool Comparison: Metadata Scan vs. Signature Carving

| Comparison Vector | File-System Metadata Recovery (Recuva Standard) | Raw File Signature Carving (PhotoRec / Deep Scan) |
| :--- | :--- | :--- |
| **Primary Method** | Parses directory table entries and FAT pointer structures. | Scans raw storage sectors for file headers/footers (magic bytes). |
| **Filename Retention** | Partial (First character replaced by `_`). | Lost (Replaced with arbitrary sequence IDs or embedded metadata). |
| **Plain Text Handling** | **High Success:** Recovers `.txt` files if metadata is present. | **Ineffective:** Plaintext files lack distinct file header magic bytes. |
| **Format Immunity** | Fails if root directory/FAT tables are formatted. | High resistance; ignores file system structures entirely. |
| **File Fragmentation** | Reconstructs non-contiguous files via cluster chains. | Fails or corrupts fragmented files without active chain pointers. |

---

## Defensive & Incident Response Engineering Takeaways

1. **Data Remanence Risks:** Neither standard OS deletion commands nor Quick Format operations destroy binary payload data. Media must undergo cryptographic wipe procedures (e.g., NIST SP 800-88 compliant zero-fills) or full disk encryption (FDE) before disposal.
2. **Volatile Evidence Preservation:** In incident response engagements, target media must be write-blocked immediately. Any write operations by the OS can overwrite unallocated space and destroy recoverable artifacts.
3. **Forensic Integrity Verification:** Cryptographic hashing (SHA-256/MD5) is essential to validate that evidence extraction processes do not modify original source artifacts.