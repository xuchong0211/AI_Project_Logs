# Bidflow: Final Bid Pack Implementation

**Date:** 2025-02-11  
**Summary:** Implementation of the “Generate Final Bid Pack” flow: confirm dialog, download recording + submission lock, and post-download timeline UI.

---

## Plan (overview)

1. **Confirm UI for download** – Dialog when user clicks “Generate submission pack”: warning (submission locked after proceed), contents list, optional “include logs” checkbox, mandatory disclaimer, Generate button.
2. **Record download** – On Generate: record `generatedAt`, `generatedById`, optional `includeLogs`, pack file name; set “submission locked” for the project.
3. **Submission page states** – **State A:** Pack not generated → show checklist + button that opens dialog. **State B:** Pack generated → show timeline (Submission summary, Acknowledgement of Receipt, Evaluation Period, Award Letter, Contract Start Date) and “Download submission”.
4. **Lock enforcement** – When locked, reject edits to questions, assignees, tags, mandatory attachments (server-side).

---

## 1. Schema & migration

**File:** `db/schema.ts`

- On `projects` table:
  - `submissionPackGeneratedAt` (timestamp, nullable)
  - `submissionPackGeneratedById` (integer, FK to `users`, nullable)
  - `submissionPackIncludeLogs` (boolean, default false)
  - `submissionPackFileName` (text, nullable)
- Lock is implied: `submissionPackGeneratedAt != null` ⇒ submission locked.

**Migration:** `drizzle/0055_submission_pack_generated.sql`

- Adds the four columns and FK `projects_submission_pack_generated_by_id_users_id_fk`.
- Journal: `drizzle/meta/_journal.json` updated with entry for `0055_submission_pack_generated`.

**Apply:** `pnpm run db:push` or `pnpm run db:migrate`.

---

## 2. Backend

### 2.1 Shared zip builder

**File:** `lib/submissionPack.ts`

- `buildSubmissionPackZip(params: { project, questions, clerkOrgId, includeLogs? })`:
  - Builds the same zip as before (final approved answers, referenced docs, appendix).
  - If `includeLogs === true`, adds `project_logs.txt` (placeholder content).
  - Returns `{ buffer, zipFilename }`.
  - Used by both GET download-all and POST generate (no auth inside; caller enforces access).

### 2.2 POST generate (first-time pack + lock)

**File:** `app/api/projects/[projectId]/submission-pack/generate/route.ts`

- **POST** only.
- Auth: Clerk user + project team membership (same as download-all).
- If `project.submissionPackGeneratedAt` already set → **409** “Submission pack has already been generated. Submission is locked.”
- Body: `{ includeLogs?: boolean }` (optional).
- Calls `buildSubmissionPackZip({ project, questions, clerkOrgId, includeLogs })`.
- Updates `projects`: set `submissionPackGeneratedAt`, `submissionPackGeneratedById`, `submissionPackIncludeLogs`, `submissionPackFileName`, `updatedAt`.
- Returns zip with `Content-Disposition: attachment`.

### 2.3 GET download-all (unchanged behaviour, refactored)

**File:** `app/api/projects/[projectId]/questions/download-all/route.ts`

- Refactored to use `buildSubmissionPackZip` with `includeLogs: false`.
- Still used for “Download submission” when submission is locked (re-download same pack).
- Does not write any “pack generated” state; only POST generate does that.

### 2.4 Project API (generator name for timeline)

**File:** `app/api/projects/[projectId]/route.ts`

- GET project: left join `users` on `projects.submissionPackGeneratedById = users.id`.
- Response includes `submissionPackGeneratedByName` (for “Submitted by: …” in timeline).

---

## 3. Submission lock enforcement

**File:** `lib/submissionLock.ts`

- `isSubmissionLocked(projectId): Promise<boolean>`
- `requireSubmissionNotLocked(projectId): Promise<void>` – throws if locked (message: “Submission is locked. The final bid pack has been generated and no further edits are allowed.”)

**Where used:**

- **`app/actions/questions.ts`:**  
  `requireSubmissionNotLocked(projectId)` at start of:  
  `updateQuestionStatus`, `sendBackToSmeSignOff`, `updateQuestion`, `assignQuestion`, `updateQuestionAssignees`, `updateQuestionTags`.
- **`app/actions/questionMandatoryAttachments.ts`:**  
  Same check in:  
  `addQuestionMandatoryAttachment`, `updateQuestionMandatoryAttachmentLabel`, `archiveQuestionMandatoryAttachment`, `setQuestionMandatoryAttachmentDocument`.

Any attempt to change questions, assignees, tags, or mandatory attachments after the pack is generated returns the lock error to the client.

---

## 4. Confirm dialog (image 4)

**File:** `app/(dashboard)/projects/[projectId]/submission/GenerateFinalBidPackDialog.tsx`

- **Props:** `open`, `onOpenChange`, `projectId`, `referencedDocumentNames` (for “Referenced documents” list), `onSuccess`.
- **Content:**
  - Title: “Generate Final Bid Pack”; subtitle: current date/time (e.g. `dd/MM/yyyy 'at' HH:mm`).
  - Warning (amber): “Once you proceed the submission will be locked and no further edits will be allowed.”
  - “The following will be provided in your final pack:” – bullets: All final approved answers; Referenced documents (from `referencedDocumentNames`); Appendix of documents attached to this project.
  - Optional checkbox: “Optional – All logs associated with this project for record keeping purposes” → `includeLogs`.
  - Mandatory checkbox: full disclaimer (accuracy, deadlines, third-party sharing, no guarantee).
  - “Generate Final Bid Pack” button disabled until disclaimer checked.
- **On Generate:**  
  POST `/api/projects/[projectId]/submission-pack/generate` with `{ includeLogs }` → trigger file download from response blob → `onSuccess()` → `onOpenChange(false)`.

---

## 5. Submission page: State A vs State B

**File:** `app/(dashboard)/projects/[projectId]/submission/page.tsx`

- **Project type** extended with:  
  `submissionPackGeneratedAt`, `submissionPackGeneratedById`, `submissionPackFileName`, `submissionPackGeneratedByName`.
- **State check:** `submissionLocked = project?.submissionPackGeneratedAt != null`.

**State A (pack not generated):**

- Renders existing Submission checklist card (workflow steps, deadlines) and “Generate submission pack” button.
- Button opens `GenerateFinalBidPackDialog` (no direct download).
- Dialog `onSuccess` calls `refreshProject()` (re-fetch project) so UI can switch to State B.
- Rest of page: `SubmissionChecklist`, `SubmissionAnswerList` (unchanged).

**State B (pack generated):**

- Renders only `SubmissionTimeline` (no checklist/answer list).
- “Download submission” in timeline calls existing GET `/api/projects/[projectId]/questions/download-all` (same zip, no lock write).

---

## 6. Post-download timeline (image 5)

**File:** `app/(dashboard)/projects/[projectId]/submission/SubmissionTimeline.tsx`

- **Props:** `projectId`, `submissionPackGeneratedAt`, `submissionPackFileName`, `submissionPackGeneratedByName`, `externalDeadlineFormatted`, `isDownloading`, `onDownloadSubmission`.
- **Sections (vertical cards):**
  1. **Submission summary** – Lock icon; “Submission pack generated on”, “External deadline”, “Submitted by: … (Bid Manager)”, “Submission pack name: … (read-only)”, “Submission date: (Updated when receipt is uploaded)”; **Download submission** button (calls `onDownloadSubmission`).
  2. **Acknowledgement of Receipt** – “Awaiting commissioner (due within 3 days)”; **Upload** button (disabled placeholder).
  3. **Evaluation Period** – Placeholder text.
  4. **Award Letter** – “Update status” button (disabled placeholder).
  5. **Contract Start Date** – “Edit” button (disabled placeholder).

Receipt upload, evaluation dates, award letter, and contract start date are left for a later phase.

---

## 7. File list (created/updated)

| Path | Action |
|------|--------|
| `db/schema.ts` | Updated (4 new columns on `projects`) |
| `drizzle/0055_submission_pack_generated.sql` | Created |
| `drizzle/meta/_journal.json` | Updated |
| `lib/submissionPack.ts` | Created |
| `lib/submissionLock.ts` | Created |
| `app/api/projects/[projectId]/submission-pack/generate/route.ts` | Created |
| `app/api/projects/[projectId]/questions/download-all/route.ts` | Refactored (use `buildSubmissionPackZip`) |
| `app/api/projects/[projectId]/route.ts` | Updated (join user, return `submissionPackGeneratedByName`) |
| `app/actions/questions.ts` | Updated (lock check in 6 functions) |
| `app/actions/questionMandatoryAttachments.ts` | Updated (lock check in 4 functions) |
| `app/(dashboard)/projects/[projectId]/submission/GenerateFinalBidPackDialog.tsx` | Created |
| `app/(dashboard)/projects/[projectId]/submission/SubmissionTimeline.tsx` | Created |
| `app/(dashboard)/projects/[projectId]/submission/page.tsx` | Updated (dialog, State A/B, timeline) |

---

## 8. User flow

1. User opens Submission page → sees checklist and “Generate submission pack”.
2. Clicks “Generate submission pack” → **Generate Final Bid Pack** dialog opens (warning, contents, optional logs, mandatory disclaimer).
3. Checks disclaimer (and optionally “Include logs”) → Clicks “Generate Final Bid Pack” → POST generate runs → zip downloads → project is updated and locked → dialog closes → page refetches project.
4. Page now shows **timeline** (State B): Submission summary with “Download submission”, plus placeholders for Acknowledgement, Evaluation, Award, Contract.
5. “Download submission” re-downloads the same pack via GET download-all.
6. Any edit to questions/assignees/tags/mandatory attachments after lock is rejected by the server with the lock message.

---

## 9. Possible follow-ups (not in this implementation)

- **Acknowledgement of Receipt:** Upload final manual edit / receipt file(s); store and show “Submission date” when receipt is uploaded; “due within 3 days” logic.
- **Evaluation period:** Store and display date range (e.g. 2–30 Oct 2025).
- **Award letter:** “Update status” to record awarded/not awarded and date.
- **Contract start date:** “Edit” to set/update when awarded; show only when award is positive.
- **Answer content saves:** If answer text is persisted via another API (e.g. Git/versions), add a lock check there so content cannot be edited after pack generation.
