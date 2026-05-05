# ASN (Advanced Shipping Notice) Processing — End-to-End Flow

## 1. Overview

The ASN processing pipeline notifies customers (primarily internet/PartStore ordering customers) that their parts are being shipped or backordered. It is triggered as part of the **document acknowledgment** process and spans six COBOL programs:

| Program | Role |
|---------|------|
| **PCC0055** | Interactive Document Acknowledgment UI (ACKPCDOC) |
| **PCC0056** | Batch/Auto Document Acknowledgment Engine (PCJN0056A NEPS job) |
| **PCC0191** | ASN Notification Dispatcher — sends data queue or web-service message |
| **FDC0191** | FDC (Field Dealer Customization) hook — populates custom header/ship/backorder fields |
| **PCC8014** | I-O service program for the Customer Order database (PCLCODB0) |
| **PCC8039** | I-O service program for the Acknowledgment Request file (PCLCOAR0) |

---

## 2. Process Trigger — How Acknowledgment Starts

### 2a. Interactive Path — PCC0055

1. A user launches the **ACKPCDOC** function from a menu.
2. PCC0055 displays screen **PCS0055** (format PC300551), where up to **15 document numbers** can be entered with an action code (`*` = acknowledge, `R` = review, `C` = change auto-ack date/time, `B` = pre-bin).
3. Documents are validated:
   - Must exist in PCLCODB0 (customer order header, format PC1COHD0).
   - Doc status must be `D` (shipped), `T` (transferred), or `E` (eligible).
   - Not already in process (`A`), cancelled (`C`), invoiced (`F`), held (`B`), repricing (`G`/`R`), or deleted (`X`).
   - B2C orders (myParts.cat.com) go through additional OMS transmit and tracking-info checks.
   - DPIS Country-of-Origin documents require completed country-code maintenance.
   - Work order segments are checked for record locks.
4. Valid documents are submitted to PCC0056 for processing via the call `CALL "PCC0056"`.

### 2b. Batch / Auto-Acknowledgment Path — PCC0056 (Entry Point)

1. The NEPS job **PCJN0056A** calls PCC0056 directly.
2. PCC0056 reads the **Acknowledgment Request file** (`PCLCOAR0`, format PC1COAK0) sequentially.
3. Each trigger record contains a `REF-DOCUMENT-NO-PC1COAK0` identifying the document to acknowledge.
4. Optionally, a preprocess program **PCC0056A** inspects trigger records for data integrity before processing. If errors are found, the trigger is deleted and the NEPS job moves to the next record.
5. The program reads the corresponding **Customer Order Header** (`PCLCODB0`, format PC1COHD0) and performs the same eligibility checks as the interactive path (doc status, B2C tracking, DPIS, work-order locks, etc.).

---

## 3. Core Acknowledgment Processing — PCC0056

After validation, PCC0056 routes processing based on **transaction code** and **sales type**:

| Tran Code | Sales Type | Route | Description |
|-----------|------------|-------|-------------|
| CS, CT | T | `D100-TRANSFER` | Store-to-store transfer |
| CS, CT | B | `E100-BACKORDER-TRANSFER` | Backorder fill from stock |
| CR, CU, SR, SU | — | `C100-RETURNS` | Return / credit documents |
| CS, CT, SS, ST, CQ, SQ | — | `B100-SALES` | Standard counter/shop sales |
| CD, SD | — | `F100-WORN-CORES` | Worn core distribution |

### Key Processing Steps (B100-SALES path as representative example)

1. **Read parts detail** — iterates through `PCLCODB0` (PC1COPD0) records for the document.
2. **Inventory updates** — updates `PCPPIPT0` (parts master) and `PCPPIST0` (store record) for on-hand quantities, call & demand, last receipt date.
3. **Stock replenishment** — reads/creates `PCLSRDB0` records (PCPSRON0, PCPSRCN0, PCPSRPD0).
4. **Suffix document creation** (`B200-CREATE-NEW-DOCUMENT`) — when backorders exist, creates a new suffix document (A→B→C…→Z) with the backordered parts. Copies header, detail, notes, ship-to, and language records.
5. **Invoice trigger** — determines whether to create an invoice trigger record:
   - Immediate invoice (`W-INVIMD = "Y"`) creates a `WOLINVS0` record or, with Multiple Consolidated Invoicing enabled, a `PCTINVG0` record via `ACPCR0056A`.
   - Consolidated billing (`BIL-IND = "D"`) waits until all related documents are acknowledged.
6. **Consolidated billing check** (`B130`) — for bill indicator `D`, scans all related documents to confirm they are all acknowledged before releasing the batch to invoicing.
7. **EDI processing** (`B150-DETERMINE-IF-EDI`) — checks `CSLEDII0` to determine if the customer receives EDI acknowledgments; if so, calls **PCC0140**.
8. **Core distribution** — if applicable, calls **PCC0257** / **PCC0285** for core cross-reference processing.
9. **Audit trail** — calls **PCC2201** / **PCC2203** to write parts accounting and audit trail records.
10. **Direct Ship** — for direct-ship documents (Direct Ship Indicator = `A`), creates confirmation copies via `PCLCOPR0` and handles DST backorder transfer/fill pairing.
11. **Header update** — rewrites the PC1COHD0 with updated doc status (`E` or `F`), acknowledgment date/time, and clears the reprice indicator.
12. **Acknowledgment message** — writes a result record to `PCLCOAM0` with success (`Y`) or warning (`W`) status.

---

## 4. ASN Email / Notification — The Internet Order Path

This is the core ASN flow and is triggered **only for internet orders**.

### 4.1 Trigger Condition (PCC0056, section A100-2)

```
IF DOC-STATUS-PC1COHD0 = "D" AND
   TERMINAL-NAME-PC1COHD0 = "INTERNET"
   → Check data queue PSARCHEQ exists
   → If line item count ≤ 250 (ship + backorder)
   → PERFORM 6000-SEND-EMAIL
```

This check occurs **before** the main acknowledgment processing begins, ensuring the ASN notification captures the document's parts before any suffix document creation modifies the data.

### 4.2 Section 6000-SEND-EMAIL (PCC0056)

This section populates the **PC50191** copybook (the ASN data structure) and calls PCC0191:

1. **Initialize** `SHIP-BO-EMAIL-PC50191`
2. **Populate header fields:**
   - `CUST-NO-PC50191` ← Customer number from order header
   - `REF-DOCUMENT-NO-PC50191` ← Document number
   - `STORE-NO-PC50191` ← Store number
   - `DATE-PC50191` ← Current system date (YYYYMMDD)
   - `TIME-PC50191` ← Current system time (HHMMSS)
   - `SHIP-VIA-CODE-PC50191` ← Ship-via code from order header
   - `SHIP-VIA-DESC-PC50191` ← Ship-via description
   - `MSG-TYPE-PC50191` ← `"S"` (shipment) if no suffix, `"B"` (backorder) if suffix exists
3. **Populate dealer code** — reads `PCLSRPA0` (stock replenishment parameters, format PC1SRST0) to get the 4-character dealer code.
4. **Call FDC0191** with detail type `"H"` (header) — allows dealer-custom header fields to be populated.
5. **Read customer email** — reads `PCLCODB0` (format PC1CONT0, note line 50) to retrieve the customer's email address from the order notes.
6. **Load parts arrays** — starts reading PC1COPD0 records for the document:

#### 6400-LOAD-PARTS-ARRAY

For each parts detail record:

| Condition | Action |
|-----------|--------|
| Has backorder qty AND ship qty > 0 | Add to **both** ship array AND backorder array |
| Has backorder qty, no ship qty | Add to **backorder array only** |
| No backorder qty, ship qty > 0 | Add to **ship array only** |

For each entry:
- **Ship array:** `SHIP-QTY-PC50191`, `SHIP-PART-NO-PC50191`, `SHIP-DESC-PC50191`
- **Backorder array:** `BO-QTY-PC50191`, `BO-PART-NO-PC50191`, `BO-DESC-PC50191`
- Calls **FDC0191** with detail type `"S"` (ship) or `"B"` (backorder) after each line item to allow custom fields per line.

7. **Call PCC0191** — `CALL "PCC0191" USING SHIP-BO-EMAIL-PC50191`

---

## 5. PCC0191 — ASN Notification Dispatcher

PCC0191 receives the populated PC50191 copybook and determines the delivery method:

### 5.1 Customer Internet Parameters Check

```
READ PCLINCU0 (Customer Internet Profile) for CUST-NO
IF ASN-TYPE-PC1INCU0 ≠ "D" → EXIT (no ASN for this customer)
```

Only customers with ASN type = `"D"` (DBS ASN) are eligible.

### 5.2 Data Preparation

**Section 1004-MOVE-TO-DATAQ:** Copies the extended data record (with custom fields) into the standard data queue format:
- Header fields (dealer code, customer, document, store, date, time, ship-via, email)
- Ship parts array (up to 250 entries: qty, part number, description)
- Backorder parts array (up to 250 entries: qty, part number, description)
- **XML special character encoding** — email ID, part descriptions, and custom values are scanned for `&`, `<`, `>`, `"`, `'` and replaced with XML entity references (`&amp;`, `&lt;`, `&gt;`, `&quot;`, `&apos;`).

### 5.3 Delivery Method Decision

```
READ CSPWEBF0 (Web Service Parameters) for key "ACPCR00006"
IF SWITCH-CS1WEBF0 = "Y" → Web Service path
ELSE → EXIT (no ASN delivery if web service is off and not DTAQ)
```

### 5.4 Path A — Data Queue (DTAQ) Delivery

If `ASN-TYPE = "D"` and web service is not enabled:
```
CALL "QSNDDTAQ" USING DATA-QUEUE-NAME ("PSARCHEQ")
                       DATA-QUEUE-LIB ("*LIBL")
                       DATA-QUEUE-LENGTH (21700)
                       DATA-QUEUE-RECORD
```

The data queue record (up to 21,700 bytes) is placed on `PSARCHEQ` for consumption by a downstream email-sending process.

### 5.5 Path B — Web Service Delivery

If web service switch = `"Y"`:

1. **Section 1001-MOVE-TO-ASN-VARIABLE:** Copies data into `ASN-DATA-ACPC1ASND0` structure:
   - Header: valid key (`PSF21`), dealer code, customer, document, store, date, time, ship-via, email, message type
   - Ship parts array (up to 250): qty, part, description + custom fields per line (up to 20 custom fields per line item)
   - Backorder parts array (up to 250): qty, part, description + custom fields per line
   - Header custom fields (up to 20)
   - All descriptions are XML-entity-encoded

2. **Add CITI libraries:** `CALL "ACPCM00002" USING ADD-LIBL`

3. **Send via web service:** `CALL "ACPCR00003" USING ASN-DATA-ACPC1ASND0`
   - `ACPCR00003` is the web service client program that transmits the ASN XML to the configured endpoint.

4. **Remove CITI libraries:** `CALL "ACPCM00002" USING REMOVE-LIBL`

---

## 6. FDC0191 — Dealer Custom Fields Hook

FDC0191 is a **dealer-customizable exit point** called by PCC0056 during ASN data population. It is called up to **three times per document**:

| Call | Detail Type | Purpose |
|------|-------------|---------|
| 1 | `"H"` (Header) | Populate header-level custom fields |
| 2 | `"S"` (Ship) | Populate per-ship-line custom fields |
| 3 | `"B"` (Backorder) | Populate per-backorder-line custom fields |

### Parameters Received

| Parameter | Copybook | Description |
|-----------|----------|-------------|
| `SHIP-BO-EMAIL-PC50191` | PC50191 | The ASN data structure being built |
| `CUST-ORDR-HEADER-PC1COHD0` | PC1COHD0 | Current document header |
| `CUST-ORDR-PARTS-DTL-PC1COPD0` | PC1COPD0 | Current parts detail line |
| `FDC-ERRMSG` | — | Error message (output) |
| `FDC-DTL-TYP` | — | `"H"`, `"S"`, or `"B"` |

### Default Implementation (WesTrac)

The current FDC0191 implementation populates:

| Type | Custom Field | Value |
|------|-------------|-------|
| Header | `CUST-NAME-PC50191(1)` = "Customer PO No" | `CUST-PO-NO-PC1COHD0` |
| Ship | `SHIP-CUST-NAME(n,1)` = "Ship Line No" | `CUST-ITEM-NUMBER-PC1COPD0` |
| Ship | `SHIP-CUST-NAME(n,2)` = "Customer Part No" | `CUST-PART-NUMBER-PC1COPD0` |
| Backorder | `BO-CUST-NAME(n,1)` = "BO Line No" | `CUST-ITEM-NUMBER-PC1COPD0` |
| Backorder | `BO-CUST-NAME(n,2)` = "Customer Part No" | `CUST-PART-NUMBER-PC1COPD0` |

This allows the ASN email/XML to include the customer's own PO number, item numbers, and part numbers alongside the DBS part information.

---

## 7. I-O Service Programs

### PCC8014 — Customer Order File I-O

- Provides standardized CRUD operations on `PCLCODB0` (multi-format logical file).
- Supports formats: PC1COHD0 (header), PC1COPD0 (parts detail), PC1COBO0 (backorder), PC1COMD0 (misc detail), PC1CONT0 (notes), PC1COCN0 (case number).
- Called by PCC0055 to read document data during interactive acknowledgment.
- Uses linkage section parameters: FUNCTION, FORMAT-NAME, FILE-STATUS, KEY-AREA, RECORD-AREA.

### PCC8039 — Acknowledgment Request File I-O

- Provides standardized CRUD operations on `PCLCOAR0` (format PC1COAK0).
- Supports: read, read-next, read-prior, write, rewrite, delete, start operations.
- Opened I-O (read-write) to allow both reading trigger records and writing/updating result records.
- Handles record lock detection and returns lock messages via MESSAGE-AREA parameter.

---

## 8. Data Flow Diagram

```
┌─────────────┐     ┌─────────────┐
│   PCC0055   │     │  PCJN0056A  │
│ Interactive │     │  NEPS Job   │
│   (User)    │     │  (Batch)    │
└──────┬──────┘     └──────┬──────┘
       │                   │
       │  CALL PCC0056     │  Direct Entry
       ▼                   ▼
┌──────────────────────────────────┐
│           PCC0056                │
│   Document Acknowledgment       │
│                                  │
│  ┌─ PCC8014 ─► PCLCODB0        │ Read order header/detail
│  │                               │
│  ├─ PCC8039 ─► PCLCOAR0        │ Read/write ack requests
│  │                               │
│  ├─ B100/C100/D100/E100/F100    │ Sales/Returns/Transfers/BO/Cores
│  │                               │
│  ├─ 6000-SEND-EMAIL ───────┐    │ (Internet orders only)
│  │                          │    │
│  │  ┌───────────────────┐   │    │
│  │  │ 6400-LOAD-PARTS   │   │    │
│  │  │   ┌───────────┐   │   │    │
│  │  │   │ FDC0191   │   │   │    │ Custom fields per line
│  │  │   └───────────┘   │   │    │
│  │  └───────────────────┘   │    │
│  │                          │    │
│  │  CALL PCC0191 ◄──────────┘    │
│  │                               │
│  └─ A200-WRITE-MESSAGE ─► PCLCOAM0  │ Result/status
└──────────────────────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│           PCC0191                │
│   ASN Notification Dispatcher    │
│                                  │
│  Read PCLINCU0 (Customer IP)     │
│  Check ASN-TYPE = "D"            │
│                                  │
│  XML encode special characters   │
│                                  │
│  ┌────────────┬─────────────┐    │
│  │ Path A     │ Path B      │    │
│  │ QSNDDTAQ  │ ACPCR00003  │    │
│  │ PSARCHEQ  │ Web Service  │    │
│  └────────────┴─────────────┘    │
└──────────────────────────────────┘
                │
                ▼
     ┌─────────────────────┐
     │ Downstream Consumer │
     │ (Email / XML ASN)   │
     └─────────────────────┘
```

---

## 9. Key Database Files

| Physical File | Logical File | Format | Description |
|---------------|-------------|--------|-------------|
| PCPCOHD0 | PCLCODB0 | PC1COHD0 | Customer order header |
| PCPCOPD0 | PCLCODB0 | PC1COPD0 | Customer order parts detail |
| PCPCOBO0 | PCLCODB0 | PC1COBO0 | Customer order backorder detail |
| PCPCONT0 | PCLCODB0 | PC1CONT0 | Customer order notes (email at line 50) |
| PCPCOMD0 | PCLCODB0 | PC1COMD0 | Customer order misc detail |
| PCPCOAK0 | PCLCOAR0 | PC1COAK0 | Acknowledgment request/trigger records |
| PCPCOAK0 | PCLCOAM0 | — | Acknowledgment message/result records |
| PCPCOPA0 | PCLCOPA0 | PC1COPA0 | Customer order parameters |
| PCPCOSP0 | PCLCOSP0 | PC1COSP0 | Customer order ship-to |
| PCPPIPT0 | PCLPICM0 | PC1PIPT0 | Parts inventory part master |
| PCPPIST0 | PCLPICM0 | PC1PIST0 | Parts inventory store record |
| PCPSRON0 | PCLSRDB0 | PC1SRON0 | Stock replenishment order |
| PCPSRCN0 | PCLSRDB0 | PC1SRCN0 | Stock replenishment case |
| PCPSRPD0 | PCLSRDB0/PCLSRPD0 | PC1SRPD0 | Stock replenishment parts detail |
| PCLINCU0 | — | PC1INCU0 | Customer internet profile (ASN type) |
| CSPWEBF0 | — | CS1WEBF0 | Web service parameters |
| WOPINVS0 | WOLINVS0 | WO1INVS0 | Invoice trigger file |
| PCTINVG0 | — | — | Multiple consolidated invoicing trigger |
| PSARCHEQ | — | — | Data queue for ASN email messages |
| CSPEDIC0 | CSLEDII0 | CS1EDIC0 | EDI customer profile |

---

## 10. Key Copybooks

| Copybook | Purpose |
|----------|---------|
| **PC50191** | ASN email/notification data structure — header + ship array (250) + backorder array (250) + custom fields |
| **ACPC1ASND0** | ASN web service data structure (extended, with custom fields per line) |
| **PC1COHD0** | Customer order header layout |
| **PC1COPD0** | Customer order parts detail layout |
| **PC1COAK0** | Acknowledgment request record layout |
| **PC1INCU0** | Customer internet parameters layout |
| **CS1WEBF0** | Web service switch record layout |
| **DS5FUNC** | Standard I-O function codes (read, write, rewrite, delete, start, etc.) |
| **DS5STAT** | Standard file status codes |

---

## 11. Processing Constraints & Limits

| Constraint | Limit | Behaviour |
|------------|-------|-----------|
| Parts per ASN | 250 ship + 250 backorder | If either count > 250, email is **skipped** |
| Custom fields per line | 20 | Array dimension in PC50191 |
| Custom header fields | 20 | Array dimension in PC50191 |
| Data queue record size | 21,700 bytes | PSARCHEQ record length |
| Suffix documents | A through Z (26 max) | Error written to PCPCOAK0 if exceeded |
| Record lock retries | 5 attempts | Then rollback + skip to next trigger |

---

## 12. Error Handling

- **Record locks:** Up to 5 retry attempts. On the 5th failure, a ROLLBACK is issued and the program moves to the next trigger record. Lock info is journaled with type `PC`.
- **Missing records:** "Document not found" or "not eligible" messages written to PCPCOAK0 with indicator `W` (warning).
- **Preprocess errors (PCC0056A):** If the preprocess program detects data integrity issues, it sets `DELETE-TRIGGER-SW = "Y"`, the trigger is cleaned up, and processing continues with the next record.
- **Disaster routine (Z999):** Fatal errors (permanent I-O errors, undefined functions) are routed to the disaster routine which logs the error context (file, format, paragraph, key, status) and terminates the job.
- **System shutdown:** PCC0056 checks the SYSTEM-SHUTDOWN special name and exits gracefully if a shutdown is pending.

---

## 13. Commitment Control

PCC0056 operates under **commitment control** for the following files:
- PCLCOAR0, PCLCOAM0, PCLCODB0 (multiple opens), PCLCOPA0, PCLCOSP0, PCLSRDB0, PCLPICM0, WOLINVS0, PCLCCPR0, PCLLDOC0, PCLCOBO0, PCLPOTF0, WOLHDRS0, PCLSRPD0/PCLSRPD2, PCLCOPR0, PCLDPME0

A COMMIT is issued after successfully processing each acknowledgment request. A ROLLBACK is issued on record lock timeout or processing errors to ensure data consistency.
