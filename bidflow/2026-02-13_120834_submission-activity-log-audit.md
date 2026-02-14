# Submission page – activity log audit

Every user action on the submission page that changes data is logged to `user_activities` and shown in the Activity tab (category `submission`, entityType `project`, entityId = projectId).

## User actions and activity types

| # | User action | API | Activity type | Where logged |
|---|-------------|-----|---------------|--------------|
| 1 | Generate submission pack | `POST /api/projects/[id]/submission-pack/generate` | `generate_submission_pack` | `submission-pack/generate/route.ts` |
| 2 | Download submission pack | `GET /api/projects/[id]/questions/download-all` | `download_submission_pack` | `questions/download-all/route.ts` |
| 3 | Upload acknowledgement receipt (first time) | `POST /api/projects/[id]/submission-pack/receipt` | `save_receipt` | `submission-pack/receipt/route.ts` |
| 4 | Replace acknowledgement receipt | `POST /api/projects/[id]/submission-pack/receipt` | `update_receipt` | `submission-pack/receipt/route.ts` |
| 5a | Set receipt dates (first time, no prior dates) | `PATCH` with submissionDate/evaluationDeadline | `set_receipt_dates` | `submission-pack/route.ts` (PATCH) |
| 5b | Update receipt dates (change existing dates) | `PATCH` with submissionDate/evaluationDeadline | `update_receipt` (dateOnly, previous* + new) | `submission-pack/route.ts` (PATCH) |
| 6 | Remove acknowledgement receipt | `PATCH` with `receiptDocumentId: null` | `remove_receipt` | `submission-pack/route.ts` (PATCH) |
| 7 | Set award outcome (Awarded / Not awarded) from confirm dialog | `PATCH` with `awardOutcome` | `set_award_outcome` | `submission-pack/route.ts` (PATCH) |
| 8 | Save as Not awarded (in Award Letter dialog) | `PATCH` with `awardOutcome: "not_awarded"` | `set_award_outcome` | `submission-pack/route.ts` (PATCH) |
| 9 | Upload award letter (first time) | `POST /api/projects/[id]/submission-pack/award-letter` | `save_award_letter` | `submission-pack/award-letter/route.ts` |
| 10 | Replace award letter | `POST /api/projects/[id]/submission-pack/award-letter` | `update_award_letter` | `submission-pack/award-letter/route.ts` |
| 11 | Save award letter date only | `PATCH` with `awardOutcomeDate` | `update_award_letter` (metadata.dateOnly) | `submission-pack/route.ts` (PATCH) |
| 12 | Remove award letter | `PATCH` with `awardLetterDocumentId: null` | `remove_award_letter` | `submission-pack/route.ts` (PATCH) |
| 13 | Set contract start date (first time) | `PATCH` with `contractStartDate` | `save_contract_start_date` | `submission-pack/route.ts` (PATCH) |
| 14 | Update contract start date | `PATCH` with `contractStartDate` | `update_contract_start_date` | `submission-pack/route.ts` (PATCH) |

## UI → API flow

- **State A:** Generate pack → `GenerateFinalBidPackDialog` → POST generate → log `generate_submission_pack`.
- **State B:** Download → page `handleDownloadSubmission` → GET download-all → log `download_submission_pack`.
- **Receipt:** `AcknowledgementOfReceiptDialog` → POST receipt or PATCH (dates-only / remove) → logs in receipt route or PATCH route.
- **Award outcome:** Timeline confirm dialog or `AwardLetterDialog` “Save as Not awarded” → PATCH → log `set_award_outcome` in PATCH route.
- **Award letter:** `AwardLetterDialog` → POST award-letter or PATCH (date-only / remove) → logs in award-letter route or PATCH route.
- **Contract date:** Timeline inline edit → PATCH contractStartDate → log save/update in PATCH route.

## Types and display

- **Types:** `lib/activityLogger.ts` – `ActivityType` and helpers (`logSaveReceipt`, `logUpdateReceipt`, …).
- **Display:** `components/ActivityLogItem.tsx` – one `case` per submission type with detailed labels for receipt: **save_receipt** includes file name and date details when set; **set_receipt_dates** for first-time date set (e.g. "set acknowledgement receipt dates: submission date to X, evaluation deadline to Y"); **update_receipt** for date-only shows "submission date from A to B; evaluation deadline from C to D", for file replace shows file name and optional date details. Other submission types unchanged (e.g. removed_receipt, set_award_outcome, generate/download pack). “saved acknowledgement receipt”, “updated acknowledgement receipt (dates only)”, “removed the acknowledgement receipt”, “set award outcome to Awarded/Not awarded”, “generated the submission pack”, “downloaded the submission pack”).

Read-only actions (e.g. opening receipt/award letter review dialogs) are not logged.
