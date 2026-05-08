# Jira Change Request — BSO-742

---

## Summary

**Bulk Discount Code Import Automation (UII0020)**

New ILE program suite to automate the bulk upload of SPAR pricing records from a CSV file on the IFS into production file `UIPPREB0`, replacing the existing manual macro process via MNTSPPRC that operates on ~30,000 lines and historically requires multiple days to complete.

---

## Change Request Details

| Field | Value |
|---|---|
| **Jira Ticket** | BSO-742 |
| **Type** | New Development |
| **Priority** | High |
| **Assignee** | Daniel Stonor |
| **BRD Reference** | BSO-742 — Business Requirements Document |
| **Date Raised** | 16 April 2026 |
| **Target Completion** | 08 May 2026 |
| **Environment** | IBM i (AS/400) — Library derived from `SMDADTAE` data area (e.g. `LIBU08`) |
| **Application Area** | User Interface / Inventory / SPAR Pricing |
| **Menu Path** | `UPMENU -> MNTSPPRC -> LODSPPRC` |

---

## Business Justification

The current process for loading approved discount codes into `UIPPREB0` is performed manually through the MNTSPPRC (Maintain SPAR Prices) screen — one record at a time. With approximately **30,000 records** per upload cycle, this process takes **multiple days** of operator effort and is prone to data entry errors.

This change introduces an automated bulk import that:
- Reduces processing time from **days** to **minutes**
- Eliminates manual data entry errors
- Provides full validation against reference files before insert
- Automatically archives production data before each import
- Logs all failed records with detailed error reasons to a CSV
- Sends styled HTML email notifications with results
- Supports both ad-hoc (interactive) and scheduled execution

---

## Scope of Change

### New Objects Created

| Object | Type | Library | Source Lib/File | Description |
|---|---|---|---|---|
| `UIS0020` | DSPF (Display File) | `UPLNG500` | `UPSRC500/QDDSSRC` | 5-format interactive display file |
| `UIR0020` | *MODULE (SQLRPGLE) | `UPPGM500` | `UPSRC500/QRPGLESRC` | Bulk record validator/processor |
| `UIM0020` | *MODULE (CLLE) | `UPPGM500` | `UPSRC500/QCLLESRC` | CL orchestration module |
| `UII0020` | *PGM (ILE Bound) | `UPPGM500` | — (bound from modules) | Main ILE program (entry: `UIM0020`) |
| `UIC0020` | *PGM (COBOL) | `UPPGM500` | `UPSRC500/QLBLSRC` | Thin COBOL wrapper for DBS menu integration |
| `UIM0020C` | *PGM (CLP) | — | — | Compilation driver (build automation) |

### Existing Objects Modified

| Object | Type | Nature of Change |
|---|---|---|
| `UIPPREB0` | *FILE (Physical) | **No structural change** — records inserted/deleted during processing |
| `UIPPREH0` | *FILE (Physical) | **No structural change** — receives history copies of replaced records |
| `LODSPPRC` | Menu program | Must be updated to call `UIC0020` (COBOL wrapper) |

### No Database Schema Changes

This solution inserts into the existing `UIPPREB0` layout. No DDL changes, no new physical files created in production. Staging and validation work files are created in `QTEMP` (job-scoped, automatically cleaned up).

---

## Technical Design

### Architecture Overview

```
LODSPPRC (DBS Menu)
  └── UIC0020 (COBOL wrapper — receives DS5LINK, calls UII0020)
        └── UII0020 (Bound ILE Program)
              ├── UIM0020 (CLLE — orchestration, screens, email)
              │     ├── UIS0020 (DSPF — 5 record formats)
              │     └── UIR0020 (SQLRPGLE — validation & bulk SQL)
              └── (ACTGRP *CALLER)
```

### Program Flow

#### Interactive Mode (MODE = 'I')

1. **UIS0020M** — Selection screen: Option 1 (Ad-hoc) or Option 2 (Schedule)
2. **UIS0020A** — Ad-hoc confirmation (F6 required to proceed; Enter does NOT confirm)
3. **UIS0020S** — Schedule entry (job name, date DD/MM/YY, time HH:MM:SS with full validation)
4. **UIS0020F** — CSV format reference screen (accessible via F8)
5. **Processing** — Steps 1–6 below
6. **UIS0020C** — Results screen (counts, elapsed time, status badge)

#### Batch Mode (MODE = 'B')

- Skips all interactive screens
- Executes Steps 1–6 directly
- Sends completion/escape messages to job log
- Sends HTML email notification

#### Processing Steps

| Step | Description | Key Commands/Calls |
|---|---|---|
| **1** | Verify IFS source CSV exists | `CHKOBJLNK` |
| **2** | Archive `UIPPREB0` before import (1 month retention) | `CRTARCF` to `ARCDBF/SPRC` |
| **3a** | Create staging file `QTEMP/UISTG020` from production layout | `CRTDUPOBJ`, `RNMM` |
| **3b** | Count total CSV data rows (header excluded) | `CPYFRMSTMF` + `RTVMBRD` |
| **3c** | Import CSV into staging file | `CPYFRMIMPF` (CCSID 1208→37, comma-delimited, skip header) |
| **4** | Validate, deduplicate, and insert via `UIR0020` | `CALLPRC UIR0020` |
| **5** | Calculate elapsed time, determine status (S/P/F) | Internal calculation |
| **6** | Archive processed CSV to `/Inventory/Archive/` with timestamp | `CPY` via `QCMDEXC` |
| **7** | Display results (interactive) or send job log message (batch) | `SNDRCVF` / `SNDPGMMSG` |
| **8** | Send HTML email notification (with error CSV attachment if applicable) | `SNDSMTPEMM` via `SBMJOB` |
| **9** | Clean up error CSV asynchronously | `QSH CMD('sleep 5 && rm -f ...')` via `SBMJOB` |

### UIR0020 — Validation & Bulk Processing (SQLRPGLE)

#### Phase 1: Set-Based Validation
- Bulk inserts all staging records into `QTEMP/UIVAL020`
- Flags blank mandatory fields (CUNO, SOS1, etc.)
- Flags invalid AGRI indicator (must be S/P/R/M)
- Validates date format and range (DDMMYYYY packed 8,0)
- Validates Finish Date > Start Date (compared as YYYYMMDD)
- Validates against reference files via bulk JOINs:
  - **Customer** → `CILNAME0`
  - **SOS** → `PCPSRSS0`
  - **BECTYC** → `PCPPIPT0`
  - **CMCD** → `PCPPIPT0`
  - **Part Number** → `PCPPIPT0`
- Determines record type for valid records
- Writes failed records with reason to IFS error CSV via C-runtime `open()`/`write()`/`close()`

#### Phase 2: Set-Based Bulk Operations
- Bulk copy matching duplicates to history (`UIPPREH0`)
- Bulk delete matched duplicates from `UIPPREB0`
- Bulk insert all validated records into `UIPPREB0`

> **Performance Note:** Phase 1+2 replaced a row-by-row approach (5 SELECT lookups × ~82K rows = ~410K individual I/O cycles) with set-based SQL, reducing to 3 bulk statements plus per-reference-file UPDATE statements.

### CSV Format

| Col | Field | Type | Len | Description |
|---|---|---|---|---|
| 1 | CUNO | Text | 7 | Customer Number |
| 2 | SOS1 | Text | 3 | Source of Supply |
| 3 | PANO20 | Text | 20 | Part Number |
| 4 | CATPANO16 | Text | 20 | C of A Part Number |
| 5 | CMCD | Text | 2 | Commodity Code |
| 6 | BECTYC | Text | 3 | Bus Econ Comm Code |
| 7 | CUSTDISC | Numeric | 5,2 | Cust Discount (0–100) |
| 8 | WESTDISC | Numeric | 5,2 | WesTrac Discount (0–100) |
| 9 | SPSELL | Numeric | 15,2 | Special Sell Price (≥0) |
| 10 | SPCOST | Numeric | 15,2 | Special Cost Price (≥0) |
| 11 | SDATE | Numeric | 8,0 | Start Date (DDMMYYYY) |
| 12 | FDATE | Numeric | 8,0 | Finish Date (DDMMYYYY) |
| 13 | AGRI | Text | 1 | Agreement Indicator (S/P/R/M) |
| 14 | CUSTYP | Text | 1 | Customer Type Indicator |
| 15 | AGRNO | Text | 15 | Agreement Number |
| 16 | REMARK | Text | 50 | Remarks (optional) |

**CSV Requirements:** Comma-delimited, header row present, CRLF line endings, UTF-8 encoding (CCSID 1208).

### IFS Paths

| Path | Purpose |
|---|---|
| `/Inventory/UIPPREB0.csv` | Source CSV (uploaded by Inventory team) |
| `/Inventory/UIPPREB0_Import_Errors.csv` | Error output (per-record failure reasons) |
| `/Inventory/Archive/UIPPREB0_DDMMYY_HHMMSS.csv` | Archived source CSV (timestamped) |

### Email Notification

- Sent via `SNDSMTPEMM` under system user profile (e.g. `XUPT08CDIS`)
- Styled HTML email with WesTrac branding (black/gold header, logo)
- Includes stats table: Job ID, Date/Time, Records Read/Loaded/Failed, Elapsed Time
- Colour-coded status badge: Green (Success), Amber (Partial), Red (Failed)
- Error CSV attached as `.txt` when failures exist

---

## IFS & File Dependencies

### Reference Files Accessed (Read-Only Validation)

| File | Purpose |
|---|---|
| `CILNAME0` | Customer master — validate CUNO exists |
| `PCPSRSS0` | SOS reference — validate SOS1 exists |
| `UCLPIPT0` | Part number master — validate PANO20 exists |
| `UCLPIPT1` | Part number alternate — secondary lookup |
| `PCLPICM0` | Commodity code reference — validate CMCD exists |
| `PCPPIPT0` | BECTYC / CMCD / Part reference |

### Production Files (Read/Write)

| File | Operations |
|---|---|
| `UIPPREB0` | DELETE (matched duplicates), INSERT (validated records) |
| `UIPPREH0` | INSERT (history/audit copy of replaced records) |

### System Objects

| Object | Purpose |
|---|---|
| `SMDADTAE` | Data area — production library name |
| `ARCDBF/SPRC` | Archive library/file for `CRTARCF` backups |

---

## Compilation & Deployment

### Build Procedure

Run `CALL UIM0020C` — the automated compilation driver. Sequence:

1. `CRTDSPF` — Display file `UIS0020` into `UPLNG500`
2. `CRTSQLRPGI` — SQLRPGLE module `UIR0020` into `UPPGM500`
3. `CRTCLMOD` — CLLE module `UIM0020` into `UPPGM500`
4. `CRTPGM` — Bind `UIM0020` + `UIR0020` into `UII0020` (`ACTGRP(*CALLER)`)
5. `DLTMOD` — Clean up intermediate modules
6. `CRTCBLPGM` — COBOL wrapper `UIC0020` into `UPPGM500`

### Source Libraries

| Library | Source File | Members |
|---|---|---|
| `UPSRC500` | `QDDSSRC` | `UIS0020` |
| `UPSRC500` | `QRPGLESRC` | `UIR0020` |
| `UPSRC500` | `QCLLESRC` | `UIM0020` |
| `UPSRC500` | `QLBLSRC` | `UIC0020` |

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Production data corruption in `UIPPREB0` | Low | High | Automatic `CRTARCF` archive taken before every import (Step 2). 1-month retention in `ARCDBF/SPRC`. |
| CSV format mismatch / bad data | Medium | Medium | Multi-layer validation: `CPYFRMIMPF` parse errors caught, UIR0020 validates every field against reference files. Failed records logged with reason. |
| Duplicate records overwriting valid data | Low | Medium | Duplicate detection with history copy to `UIPPREH0` before delete+re-add. Full audit trail preserved. |
| Email notification failure | Low | Low | `MONMSG` on `SNDSMTPEMM`. Processing results still shown on screen (interactive) or job log (batch). |
| IFS file not present at runtime | Medium | Low | `CHKOBJLNK` check at Step 1 with clear error message before any processing begins. |
| Scheduled job runs with stale CSV | Low | Medium | CSV is archived after processing; next run will fail at Step 1 if no new CSV uploaded. |

---

## Rollback Plan

1. **Restore `UIPPREB0`** from the automatic archive in `ARCDBF/SPRC` (created at Step 2 of every run)
2. **Remove new objects:** `DLTPGM UII0020`, `DLTPGM UIC0020`, `DLTF UIS0020`
3. **Revert menu entry** in `LODSPPRC` to remove the `UIC0020` call
4. **No database schema changes** to revert — `UIPPREB0` and `UIPPREH0` structures are unchanged

---

## Testing Evidence

### Test Scenarios

| # | Scenario | Expected Result | Status |
|---|---|---|---|
| 1 | Ad-hoc load with valid CSV (~30K rows) | All records loaded, status = SUCCESS, email sent | |
| 2 | Ad-hoc load with mixed valid/invalid rows | Valid loaded, invalid logged to error CSV, status = PARTIAL | |
| 3 | Ad-hoc load with all invalid rows | Zero loaded, all logged, status = FAILED | |
| 4 | CSV file missing from IFS | Fails at Step 1 with clear message, no data touched | |
| 5 | Duplicate records in CSV (same CUNO/SOS1/PANO20) | Existing deleted, history written to `UIPPREH0`, new inserted | |
| 6 | Schedule for future date/time | `ADDJOBSCDE` created successfully, confirmation message displayed | |
| 7 | Schedule with past date | Rejected with "Date cannot be in the past" | |
| 8 | Schedule with past time (today) | Rejected with "Time cannot be in the past for today" | |
| 9 | Batch mode execution (MODE='B') | No screens displayed, processes directly, sends email | |
| 10 | F6 confirmation required on ad-hoc screen | Enter key redisplays with reminder; only F6 proceeds | |
| 11 | F8 CSV format reference screen | Displays column layout; F12 returns to main | |
| 12 | Email with error attachment | HTML email received with error CSV attached | |
| 13 | Email without errors | HTML email received, no attachment | |
| 14 | Archive created before import | `ARCDBF/SPRC` contains timestamped backup | |

---

## Screenshots

> **Instructions:** Replace each placeholder below with the relevant screenshot. Capture the green-screen or browser output as appropriate.

---

### Screenshot 1: Selection Screen (UIS0020M)
> Shows the initial selection screen with Option 1 (Ad-hoc) and Option 2 (Schedule), IFS source/target paths, and import rules.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF UIS0020M HERE <<<             ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 2: Ad-hoc Confirmation Screen (UIS0020A)
> Shows the F6 confirmation prompt before starting immediate processing.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF UIS0020A HERE <<<             ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 3: Schedule Entry Screen (UIS0020S)
> Shows the job scheduling fields (job name, date, time, description) with validation example.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF UIS0020S HERE <<<             ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 4: CSV Format Reference Screen (UIS0020F)
> Shows the expected CSV column layout reference accessible via F8.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF UIS0020F HERE <<<             ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 5: Results Screen — Successful Load (UIS0020C)
> Shows the results screen after a fully successful import (all records loaded, status = SUCCESS).

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF UIS0020C (SUCCESS) HERE <<<   ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 6: Results Screen — Partial Success (UIS0020C)
> Shows the results screen after import with some failed records (status = PARTIAL).

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF UIS0020C (PARTIAL) HERE <<<   ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 7: Results Screen — Failed Load (UIS0020C)
> Shows the results screen after a completely failed import (status = FAILED).

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF UIS0020C (FAILED) HERE <<<    ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 8: Email Notification — Success
> Shows the HTML email received after a successful import (WesTrac branded, green status badge).

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF SUCCESS EMAIL HERE <<<        ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 9: Email Notification — Partial (with Error Attachment)
> Shows the HTML email received after a partial import (amber status badge, error CSV attached).

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF PARTIAL EMAIL HERE <<<        ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 10: Email Notification — Failed
> Shows the HTML email received after a failed import (red status badge).

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF FAILED EMAIL HERE <<<         ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 11: Error CSV Sample
> Shows a sample of the `UIPPREB0_Import_Errors.csv` output with ERROR_REASON column populated.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF ERROR CSV HERE <<<            ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 12: Archive File in ARCDBF
> Shows the archive entry created in `ARCDBF/SPRC` before import.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF ARCDBF ARCHIVE HERE <<<       ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 13: IFS Archive Directory
> Shows the `/Inventory/Archive/` directory with timestamped CSV backups.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF IFS ARCHIVE DIR HERE <<<      ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 14: Scheduled Job Entry (WRKJOBSCDE)
> Shows the scheduled job entry created via Option 2.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF WRKJOBSCDE HERE <<<           ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 15: Compilation — UIM0020C Success Messages
> Shows the completion messages from running `CALL UIM0020C` (all components compiled successfully).

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF COMPILE MESSAGES HERE <<<     ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 16: Validation Error — Invalid Option
> Shows UIS0020M with the "INVALID OPTION - ENTER 1/2" error message highlighted.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF INVALID OPTION HERE <<<       ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 17: Schedule Validation — Past Date/Time
> Shows UIS0020S with date or time past-check error highlighted.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF PAST DATE ERROR HERE <<<      ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 18: UIPPREB0 Record Verification (Optional)
> Shows a DSPPFM or SQL query of `UIPPREB0` confirming records were loaded correctly.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF UIPPREB0 DATA HERE <<<        ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

### Screenshot 19: UIPPREH0 History Records (Optional)
> Shows history records written to `UIPPREH0` for duplicate replacements.

```
╔══════════════════════════════════════════════════════════╗
║                                                          ║
║   >>> INSERT SCREENSHOT OF UIPPREH0 HISTORY HERE <<<     ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

## Approvals

| Role | Name | Date | Signature |
|---|---|---|---|
| Developer | Daniel Stonor | | |
| Reviewer | | | |
| Business Owner | | | |
| Change Manager | | | |

---

*Document generated: 08 May 2026*
*BRD Reference: BSO-742 — Business Requirements Document*
