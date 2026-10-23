# Pension Scheme Return – File Upload Summary

How each `pension-scheme-return*` service uses file upload, and what data is uploaded. Based on each service's `service.md`:

- [pension-scheme-return/service.md](https://github.com/arturopala/pension-scheme-return/blob/add-service-docs/service.md)
- [pension-scheme-return-frontend/service.md](https://github.com/arturopala/pension-scheme-return-frontend/blob/add-service-docs/service.md)
- [pension-scheme-return-sipp/service.md](https://github.com/arturopala/pension-scheme-return-sipp/blob/add-service-docs/service.md)
- [pension-scheme-return-sipp-frontend/service.md](https://github.com/arturopala/pension-scheme-return-sipp-frontend/blob/add-service-docs/service.md)

## Overview

| Service | Type | Uses file upload? | What is uploaded |
|---|---|---|---|
| [`pension-scheme-return-frontend`](https://github.com/arturopala/pension-scheme-return-frontend/blob/add-service-docs/service.md) | Frontend (UI) | Yes, directly | One CSV of scheme members (bulk member upload) |
| [`pension-scheme-return`](https://github.com/arturopala/pension-scheme-return/blob/add-service-docs/service.md) | Backend | No, not directly | Gets the members from the upload inside the compiled return, as structured data |
| [`pension-scheme-return-sipp-frontend`](https://github.com/arturopala/pension-scheme-return-sipp-frontend/blob/add-service-docs/service.md) | Frontend (UI) | Yes, directly (main data-entry route) | One CSV per asset type: member-level asset transactions |
| [`pension-scheme-return-sipp`](https://github.com/arturopala/pension-scheme-return-sipp/blob/add-service-docs/service.md) | Backend | No, not directly | Gets the validated upload results as JSON transaction lists, one asset class at a time |

Both frontends use the platform's **file upload (virus-scanning) service**. It hosts the upload form, scans the file, calls back, and gives a temporary download URL. Each frontend then downloads the scanned file and validates it itself. Neither backend receives raw files.

---

## pension-scheme-return-frontend

Source: [pension-scheme-return-frontend/service.md](https://github.com/arturopala/pension-scheme-return-frontend/blob/add-service-docs/service.md)

**Purpose of the upload:** a quicker way to fill in the **Members** section of the standard PSR, as an alternative to adding members one by one.

**What is uploaded**
- A single CSV file, filled in from a template the service provides for download.
- One row per scheme member, up to **300 rows**:
  - first name and last name (required)
  - date of birth (required, `dd/mm/yyyy`, not in the future, not before 1900)
  - NINO, **or** a reason for not having one (exactly one of the two)

**Flow**
1. The user downloads the CSV template.
2. The user uploads the completed file through the virus-scanning upload service.
3. The frontend downloads the scanned file and checks it line by line.
4. The result is one of:
   - a success summary
   - an error summary with cell-level references (e.g. `C3`)
   - a "too many errors / too many rows / missing information" page
5. The members that pass validation become part of the session answers. From there they are saved to the backend as a draft ("compiled") return.

**Validation:** required names and date of birth, date format and range, NINO format, NINOs repeated within the file, NINO-or-reason (exactly one), and the 300-row limit.

**Cross-cutting**
- Audit events are raised when a member file is uploaded, downloaded and validated.
- Upload records are kept in the cache (Mongo) and expire after **10 minutes**.
- Upload results can be encrypted at rest (`mongodb.encryption.enabled`, off by default).

---

## pension-scheme-return (backend)

Source: [pension-scheme-return/service.md](https://github.com/arturopala/pension-scheme-return/blob/add-service-docs/service.md)

**No direct file upload.** The backend never sees a file. Members from the CSV upload reach it inside the standard return JSON that the frontend compiles and submits. The backend checks that payload against the ETMP JSON schema (v1.2.0 or v1.1.5), or against the relaxed "optional" schema for pre-populated returns. It is processed the same way whether the members were uploaded or typed in by hand.

---

## pension-scheme-return-sipp-frontend

Source: [pension-scheme-return-sipp-frontend/service.md](https://github.com/arturopala/pension-scheme-return-sipp-frontend/blob/add-service-docs/service.md)

**Purpose of the upload:** the main way to enter SIPP return data. The member-level asset transactions are supplied as **one CSV file per asset type**.

**What is uploaded**

CSV files of member asset transactions, one for each of the six asset types:
1. Interest in land or property (connected party)
2. Arm's-length land or property
3. Tangible moveable property
4. Outstanding loans
5. Unquoted shares
6. Assets other than land or property acquired from a connected party

Each row is one transaction for one member and holds:
- member identity: name, date of birth (also checked against the scheme wind-up date), NINO or a reason for no NINO
- transaction details for that asset type: dates, money amounts, percentages, yes/no answers, addresses and postcodes, countries, company numbers (CRN or a reason for none), and free text within length limits

**File rules**
- Must be `.csv`, at least 1 byte, and no bigger than the configured maximum (default **10 MB**).
- Row 1 is the header and row 2 is a helper row. Data starts at row 3.
- A template for each asset type can be downloaded (Excel).

**Flow**
1. For each asset type the user says whether any were held. A "No" is submitted upstream straight away as an empty declaration.
2. On "Yes", the user uploads the file through the virus-scanning upload service and then confirms the file name.
3. The file is **streamed and validated row by row** in the background, while a loading page polls for progress. The validation reuses the same rules as the forms on the screens. NINOs are upper-cased with whitespace removed, and country names are mapped to country codes.
4. If rows fail:
   - 25 errors or fewer (configurable): an error summary is shown
   - more than 25: a "too many errors" page is shown
   - in both cases the user can **download the original file with an extra error column**
5. If every row passes, the transactions are submitted to the SIPP backend for that asset type, with file-upload audit context. A request over **500 KB** is refused before it is sent, and the user sees a "files exceed limit" page.
6. An uploaded file can later be **removed**. This deletes that asset type's data upstream.
7. Submitted data for each asset type can be **downloaded as CSV**, for any form bundle number or tax-year-and-version pair.

**Storage and data protection**
- Upload metadata and validation state are kept per user, scheme and asset type in Mongo, and expire after 1 hour.
- Validated rows are kept for a short time in the **object store** for **1 day**. Access to the object store uses an internal-auth token.
- The journey store, upload metadata and stored rows are AES-GCM encrypted when `mongodb.encryption.enabled` is on.

**Auditing:** a file-upload audit event is raised for success, for validation errors, and for a size-limit breach.

---

## pension-scheme-return-sipp (backend)

Source: [pension-scheme-return-sipp/service.md](https://github.com/arturopala/pension-scheme-return-sipp/blob/add-service-docs/service.md)

**No direct file upload.** The backend receives the *results* of the frontend's CSV uploads, as JSON lists of transactions for one asset class at a time.

**How uploaded data is applied**
- Each call carries the **complete** list of transactions for one asset class, across all members. The backend fetches the current return from ETMP, merges the list in, and writes the whole return back as *compiled*.
- **The merge replaces; it does not append.** Members are matched exactly on first name + last name + date of birth + NINO:
  - A member who already has transactions for this asset class has them replaced.
  - A member who is in the return but missing from the upload loses their transactions for this asset class.
  - New members are added with status *New*.
  - A member left with no transactions in any asset class is soft-deleted (*Deleted*).
- An empty upload clears the asset class. The declaration "did any member hold X?" is set to Yes or No to match.
- "Delete all assets of one class" is how a removed file is handled on the backend.

**Limits and checks**
- Incoming JSON is checked for structure only.
- The outbound payload is checked against the API 1997 JSON schema and a **500 KB** size limit before it is sent to ETMP.

**Auditing:** uploads that exceed the size limit are audited as `PensionSchemeReturnFileUpload`. Every return write is audited as `PensionSchemeReturnPost`, with the full payload.

⚠️ Full return payloads, including member NINOs, names and dates of birth that came from uploaded files, are written to INFO logs.

---

## Comparison

| Aspect | Standard PSR (frontend) | SIPP PSR (frontend) |
|---|---|---|
| Role of upload | Optional alternative to entering members by hand | Main way to enter asset data |
| Number of files | 1 | Up to 6 (one per asset type) |
| Data content | Member identity only | Member identity + asset transaction details |
| Row limit | 300 members | Bounded by file size (10 MB) and payload size (500 KB) |
| Error feedback | Cell-level error summary | Error summary (≤25) or "too many errors", plus downloadable file with an error column |
| Validated data stored in | Session cache (Mongo, 10-minute upload records) | Object store (1 day) + Mongo metadata (1 hour) |
| Backend receives | Members inside the full compiled return | Full transaction list for one asset class, merged by replacing |
| Download back | Template only | Template + submitted data as CSV + original file with errors |
