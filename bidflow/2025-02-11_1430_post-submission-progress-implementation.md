# Post-submission progress – implementation plan

**Date:** 2025-02-11  
**Scope:** Acknowledgement of receipt (upload receipt + evaluation deadline), submission date, evaluation period (calculated), award outcome (awarded / not awarded), award letter, contract start date.

---

## 1. Process summary

1. **Acknowledgement of receipt** – User uploads receipt file and sets evaluation deadline. Submission date = receipt upload date. "Due within 3 days" is calculated from evaluation deadline.
2. **Evaluation period** – Calculated: start = submission date, end = evaluation deadline (display only).
3. **Award outcome** – User sets Awarded or Not awarded. If not awarded: show "Not awarded" and optional note; no award letter or contract. If awarded: continue to 4 and 5.
4. **Award letter** – When awarded: record award letter date (optional file). 
5. **Contract start date** – When awarded only: user can edit contract start date.

---

## 2. Data model

**New table: `project_submission`** (1:1 with project after pack generated)

| Column | Type | Description |
|--------|------|-------------|
| project_id | text PK, FK projects | Project (submission pack already generated) |
| receipt_document_id | integer FK documents, nullable | Receipt file (document with source=submission_receipt) |
| submission_date | timestamp nullable | Set when receipt is uploaded |
| evaluation_deadline | date nullable | End of evaluation; set in Acknowledgement section |
| award_outcome | text default 'pending' | 'pending' \| 'awarded' \| 'not_awarded' |
| award_outcome_date | timestamp nullable | When outcome was set |
| award_letter_document_id | integer FK documents, nullable | Optional; when awarded |
| contract_start_date | date nullable | When awarded; user-editable |
| award_note | text nullable | Optional note when not_awarded |
| created_at, updated_at | timestamp | |

- **Submission date** = receipt upload date (stored in submission_date).
- **Evaluation period** = computed for display: submission_date → evaluation_deadline.
- **"Due within 3 days"** = UI text derived from evaluation_deadline (e.g. due by deadline − 3 days or within 3 days of deadline).

---

## 3. APIs

- **GET /api/projects/[projectId]/submission-pack**  
  Returns submission state (submission_date, evaluation_deadline, award_outcome, award_outcome_date, contract_start_date, award_note, receipt_document_id, receipt_file_name, award_letter_document_id, award_letter_file_name, etc.). Only when project has submission_pack_generated_at. Used by SubmissionTimeline and dialogs.

- **POST /api/projects/[projectId]/submission-pack/receipt**  
  Multipart: receipt file (any type). Optional field: evaluation_deadline. Creates document (source=submission_receipt), creates/updates project_submission (submission_date, receipt_document_id, optional evaluation_deadline). Returns updated submission state.

- **POST /api/projects/[projectId]/submission-pack/award-letter**  
  Multipart: file + optional awardOutcomeDate. Creates document (source=award_letter), updates project_submission (award_letter_document_id, award_outcome_date). When updating an existing submission, also sets award_outcome = 'awarded'. Returns updated submission state.

- **PATCH /api/projects/[projectId]/submission-pack**  
  Body: { submissionDate?, evaluationDeadline?, awardOutcome?, awardOutcomeDate?, awardLetterDocumentId?, contractStartDate?, awardNote? }. Updates project_submission. When awardOutcome = 'not_awarded': also sets award_outcome_date, award_letter_document_id, contract_start_date to null. Validates: contractStartDate only when awardOutcome=awarded; awardNote when not_awarded.

- **Document upload** for receipt and award letter reuses existing S3/document flow; document sources = 'submission_receipt', 'award_letter'.

---

## 4. UI (SubmissionTimeline)

- **Submission summary** – Submission date: show submission_date or "(Updated when receipt is uploaded)".
- **Acknowledgement of receipt** – Button opens **AcknowledgementOfReceiptDialog** (file upload drop zone + submission date + evaluation deadline). After upload: panel shows file name, Download link, Review (opens **DocumentReviewDialog** in-app), optional Replace.
- **Evaluation period** – Display only: "{submission_date} – {evaluation_deadline}" (or "—" if missing). Status icon complete when deadline passed. Centred heading and text.
- **Award outcome** – "Update status" → confirm dialog → Awarded / Not awarded. If not awarded: "Not awarded" + date + optional note. If awarded: continue to award letter and contract start.
- **Award letter** – Button opens **AwardLetterDialog**: Outcome switch (Awarded / Not awarded), then when Awarded: file upload + award date. When file uploaded: panel shows file name, Download, Review (dialog). Save as Not awarded clears letter, date, contract start.
- **Contract start date** – Shown and editable only when award_outcome = awarded; min date = evaluation_deadline + 1 day; status icon complete when date set.
- **Review** – Eye icon for receipt or award letter opens DocumentReviewDialog (in-app viewer), not a link to the document page.

---

## 5. File list (to create/update)

| Path | Action |
|------|--------|
| db/schema.ts | Add project_submission table |
| drizzle/XXXX_post_submission.sql | Migration |
| app/api/projects/[projectId]/submission-pack/route.ts | GET (state), PATCH (update) |
| app/api/projects/[projectId]/submission-pack/receipt/route.ts | POST (upload receipt) |
| app/api/projects/[projectId]/submission-pack/award-letter/route.ts | POST (upload award letter) |
| app/(dashboard)/projects/[projectId]/submission/SubmissionTimeline.tsx | Receipt, deadline, outcome, contract; dialogs; file name + Download + Review |
| app/(dashboard)/projects/[projectId]/submission/page.tsx | Fetch submission state when locked |
| app/(dashboard)/projects/[projectId]/submission/AcknowledgementOfReceiptDialog.tsx | Dialog: receipt upload + submission date + evaluation deadline |
| app/(dashboard)/projects/[projectId]/submission/AwardLetterDialog.tsx | Dialog: Awarded / Not awarded switch, file upload + award date |
| app/(dashboard)/projects/[projectId]/submission/DocumentReviewDialog.tsx | Dialog: in-app document review (receipt or award letter) |
| app/(dashboard)/projects/[projectId]/submission/submissionTypes.ts | SubmissionStatePayload type |

---

## 6. Un-award process

- User sets outcome to "Not awarded" → store award_outcome_date (and optional award_note).
- Award letter card shows "Not awarded" and date; Contract start date card hidden or "Not applicable".
- No award letter upload or contract start date when not awarded.

---

## 7. Implementation summary (done)

| Path | Action |
|------|--------|
| db/schema.ts | Added projectSubmission table; documents.source comment extended for submission_receipt, award_letter |
| drizzle/0056_project_submission.sql | Migration for project_submission |
| drizzle/meta/_journal.json | Entry for 0056_project_submission |
| app/api/projects/[projectId]/submission-pack/route.ts | **New** GET (submission state), PATCH (evaluationDeadline, awardOutcome, awardOutcomeDate, awardLetterDocumentId, contractStartDate, awardNote) |
| app/api/projects/[projectId]/submission-pack/receipt/route.ts | **New** POST multipart: file + optional evaluationDeadline; creates document (source=submission_receipt), upserts project_submission |
| app/api/projects/[projectId]/submission-pack/award-letter/route.ts | **New** POST multipart: file + optional awardOutcomeDate; creates document (source=award_letter), updates project_submission; sets awardOutcome to "awarded" on update |
| app/(dashboard)/projects/[projectId]/submission/page.tsx | Fetch submission state when locked; pass submissionState + onSubmissionStateChange to SubmissionTimeline |
| app/(dashboard)/projects/[projectId]/submission/SubmissionTimeline.tsx | Uses dialogs for receipt, award letter, document review; status icons; receipt/letter panels show file name + Download + Review (dialog) |
| app/(dashboard)/projects/[projectId]/submission/AcknowledgementOfReceiptDialog.tsx | **New** Dialog: file upload (drop + select), submission date (date only), evaluation deadline (date only); receipt + deadline set together |
| app/(dashboard)/projects/[projectId]/submission/AwardLetterDialog.tsx | **New** Dialog: Outcome switch (Awarded / Not awarded), file upload + award date when Awarded; Save as Not awarded clears letter, date, contract start |
| app/(dashboard)/projects/[projectId]/submission/DocumentReviewDialog.tsx | **New** Dialog: Wraps DocumentReviewContent for in-app review of receipt or award letter (no navigation to document page) |
| app/(dashboard)/projects/[projectId]/submission/submissionTypes.ts | **New** SubmissionStatePayload type shared by timeline and dialogs |

---

## 8. Post-implementation refinements (done)

### 8.1 Receipt & evaluation

- Receipt file can be any type (no restriction to PDF).
- Evaluation deadline shown in Evaluation Period; when set, deadline input can be edited in the Acknowledgement dialog only.
- Deadline and receipt must be set/uploaded together in one action (dialog).

### 8.2 Acknowledgement of Receipt dialog

- Opens from a button in the Acknowledgement of Receipt panel.
- Contains: file upload (drop zone + select), evaluation deadline (single date input), submission date (editable, date only) above evaluation period.
- If evaluation period/deadline is already set, shows it and allows editing only the evaluation date.
- Drop zone: icon and text centred.

### 8.3 Status icons

- Shared `StatusIcon` from `@/components/StatusIcon` used across SubmissionTimeline (complete / outstanding / at_risk).
- Acknowledgement: complete (CircleCheck) only when both receipt uploaded and evaluation deadline set.
- Evaluation Period: complete when evaluation deadline has passed; panel heading and text centred.
- Contract Start Date: complete only when `contractStartDate` is set; min date = evaluation deadline + 1 day; Edit pre-fills current value; validation and inline error.

### 8.4 Award outcome

- Confirm dialog before setting Awarded or Not awarded (from timeline buttons).

### 8.5 Award Letter

- Panel: upload button opens Award Letter dialog (same styling as Acknowledgement: drop file + award date).
- Supports: upload, replace, remove, and date-only update.
- When a file is uploaded: panel shows file name with **Download** (link to `/api/documents/{id}/download`) and **Review** (opens review dialog, not document page).

### 8.6 Review = dialog (not document page)

- **Review** (Eye icon) for receipt or award letter opens a **dialog** to review the document in-app.
- `DocumentReviewDialog` wraps `DocumentReviewContent` (from clarifications): fetches `/api/documents/{id}/view`, renders PDF (iframe), image, Word (Office Online), or “download to view” for other types.
- SubmissionTimeline state: `reviewDialogOpen`, `reviewDocumentId`, `reviewDocumentFileName`; Eye icon is a button that sets state and opens the dialog (no `Link` to document page).

### 8.7 Award Letter dialog: Awarded / Not awarded switch

- Dialog has an **Outcome** switch at the top: **Awarded** | **Not awarded** (segmented control); on open, reflects current outcome (or “Awarded” when pending).
- **When “Not awarded” selected:** Upload and award date sections hidden. Message: “Saving as Not awarded will remove the award letter (if any), award date, and contract start date.” **Save as Not awarded** → PATCH `awardOutcome: "not_awarded"`.
- **API (PATCH):** When `awardOutcome` is set to `"not_awarded"`, the handler also sets `awardOutcomeDate = null`, `awardLetterDocumentId = null`, `contractStartDate = null` (clears all three).

### 8.8 Switching from Not awarded to Awarded

- When user switches to **Awarded** in the dialog and saves:
  - **Upload letter:** Award-letter POST now sets `awardOutcome: "awarded"` when updating existing submission (`app/api/projects/[projectId]/submission-pack/award-letter/route.ts`).
  - **Date-only save:** Dialog PATCH sends `awardOutcome: "awarded"` together with `awardOutcomeDate` so the outcome is updated when only the date is changed (`AwardLetterDialog.tsx`).
