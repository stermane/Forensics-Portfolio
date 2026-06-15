# Case 001: Windows11-SuspiciousUser-001

## Case Summary
This case simulates a digital forensic investigation involving a Windows 11
workstation suspected of unauthorized access to sensitive compay data. The 
"suspect" allegedly accessed confidential files, attempted to delete evidence,
and cleared browser history to cover their tracks. 

**Examiner:** Eric Sterman
**Case Number:** 2026-01
**Organization:** Eric's Home Lab
**Date of Acquisition:** June 12, 2026

## Scenario
A user on a corporate workstation is suspected of:
- Accessing and storing sensitive files (passwords, financial data, employee
  records, personal information) in a folder named 'secret_docs'
- Deleting the folder and emptying the Recycle Bin to destroy evidence
- Browsing multiple external websites and clearing browsing history afterward

## Evidence Acquisition

**Tool Used:** FTK Imager 8.2.0
**Source:** Windows 11 VM (C:\ drive, logical image)
**Image Format:** E01 (compression level 6, 1500MB segments)
**Output:** 20 segments (E01-E20)
**Destination:** External SSD (Eric's Cyber - Network SSD)


**Acquisition Time:** June 12, 2026 16:22:46 - 16:44:33

**Hash Verification:**
| Hash Type | Value |
|- - - - - -|- - - -|
| MD5  | ecfcbba56596c1d2225a2093dacec1D |
| SHA1 | 62173c5309c33ca177eb099155c19242fd9cc253 |

Both source and image hashes matched - verification **PASSED**.

![Verification Report](verification_report.png)

## Analysis Platform
- **Tool:** Autopsy 4.22.1 (Kali Linux)
- **Ingest Module Used:** Recent Activity
- Image loaded as a 20-segment E01 data source; all segments auto-detected,

## Findings

### 1. Recent Documents - Planted Sensitive Files Recovered
The recent Documents artifact (63 entries) revealed the 'secret_docs' folder
and all four files placed inside it:
- 'passwords.txt'
- 'employee_records.txt'
- 'financial_data.txt'
- 'personal_info.txt'

This confirms the files were accessed/opened on the system, even after the
folder was deleted. 

![Recent Documents](recent_documents.png)

### 2. Deleted Files & $Recycle Bin — Evidence Recovery After Deletion
- **Deleted Files:** 1,486 total deleted files were recoverable from the image.
- **$Recycle Bin:** 4 items recovered under user SID
  `S-1-5-21-3367154262-1791293016-1544316633-1001`, corresponding to the
  deleted `secret_docs` files.
- Content of the deleted `$R` files was readable in Autopsy's Text viewer,
  confirming the data was **not overwritten** and is fully recoverable.

**Key takeaway:** Deleting files and emptying the Recycle Bin does not
permanently remove data — the underlying content remains recoverable via
forensic imaging and analysis.

![Deleted Files](deleted_files.png)

### 3. Web History & Web Search — Anti-Forensic Indicator
Standard Recent Activity parsing of **Web History (18)** and **Web Search (6)**
returned only entries related to FTK Imager — activity that occurred *after*
the simulated incident, during the imaging process itself. No browsing
activity from the simulated "suspect" session was present in the live history
database.

This is consistent with the user clearing their browser history as an
anti-forensic technique. While the live history records were cleared,
artifacts in Web Cache (Finding #3) still preserved evidence of the underlying
activity — demonstrating that clearing history alone does not eliminate all
browser-related evidence.

![Web History](web_history.png)
![Web Search](web_search.png)

**Note for future work:** Recovering the actual cleared history entries would
require carving unallocated space or analyzing `WebCacheV01.dat` directly —
a good follow-up exercise for a future case study.

## Tools & Commands Used
- **FTK Imager 8.2.0** — disk imaging (E01 format, logical drive acquisition)
- **Autopsy 4.22.1** — forensic analysis (Recent Activity ingest module)
- **VMware Workstation** — VM management, snapshot ("Pre-Forensic-Investigation"),
  removable device passthrough for evidence transfer

## Conclusion
This case demonstrates that even when a user actively attempts to delete
sensitive files, empty the Recycle Bin, and clear browser history, substantial
evidence remains recoverable through proper forensic imaging and analysis.
Recent Documents, Deleted Files, $Recycle Bin entries, and Web Cache all
preserved usable evidence of the simulated activity, while the cleared Web
History served as its own indicator of anti-forensic behavior.

## Chain of Custody / Evidence Handling Notes
- Evidence image stored on external SSD, not committed to this repository due
  to file size (raw E01 segments excluded via .gitignore)
- Only documentation, screenshots, and verification reports are published here
