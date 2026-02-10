# Question Mandatory Attachments – Implementation Plan

**Date:** 2026-02-10  
**Time:** 11:57

---

## Overview

Replace the `project_questions.mandatory_attachments` text field with a structured model:
- **Labels** come from AI analysis; user can correct, delete, or add extra
- **Uploads** indicate completion for each label
- **Single table** for simplicity
- **Soft delete** (archive) when user removes a mandatory attachment
- **Migration** from existing `mandatory_attachments` text before dropping the column

---

## Data Model

### Single Table: `question_mandatory_attachments`

| Column        | Type      | Notes                                                |
|---------------|-----------|------------------------------------------------------|
| `id`          | serial    | Primary key                                          |
| `question_id` | integer   | FK → project_questions, ON DELETE CASCADE            |
| `label`       | text      | e.g. "Case Study", "Proposal" (editable by user)     |
| `document_id` | integer   | FK → documents, nullable – filled when user uploads  |
| `archived`    | boolean   | `false` = active, `true` = soft-deleted              |
| `sort_order`  | integer   | Optional display order                               |
| `created_at`  | timestamp |                                                     |
| `updated_at`  | timestamp |                                                     |

Each row = one mandatory attachment slot.

---

## Flow

1. **AI analyzes question** → suggests labels (rows inserted, `document_id = null`)
2. **User reviews labels** → can correct label, archive (soft delete), add new
3. **Labels saved** in `question_mandatory_attachments`
4. **User uploads files** → `document_id` updated; upload = "done"
5. **Re-upload** → update `document_id` to new document
6. **Delete from frontend** → set `archived = true` (no physical delete)

---

## Documents Source

- Add `"question_answer"` to `documents.source` (alongside `"project"` | `"clarification"`)
- Question answer documents are stored in the same `documents` table but with distinct source
- `document_id` in `question_mandatory_attachments` references these documents

---

## Migration Order

1. Create `question_mandatory_attachments` table
2. Add `question_answer` support to documents source (if needed)
3. **Data migration:** Parse `mandatory_attachments` text; insert one row per label with `archived = false`, `document_id = null`
4. Deploy application changes
5. **Later:** Drop `mandatory_attachments` column (separate migration after verification)

---

## Implementation Tasks

1. Schema: Add `question_mandatory_attachments` table
2. Drizzle migration: Create table
3. Documents: Add `question_answer` source option
4. Actions: CRUD for question mandatory attachments (list, create, update label, archive, upload)
5. API: Endpoints for list, upsert labels, upload file
6. EditQuestionDialog: Replace textarea with label list (add, edit, archive)
7. Question detail page: Upload UI per attachment slot
8. SubmissionChecklist: Use new table for validation
9. Data migration script: Migrate existing `mandatory_attachments` to new table

---

## Files to Modify

| Area        | Files |
|-------------|-------|
| Schema      | `db/schema.ts` |
| Migration   | `drizzle/0052_*.sql` |
| Actions     | `app/actions/questionMandatoryAttachments.ts` (new) |
| API         | `app/api/questions/[questionId]/mandatory-attachments/` (new) |
| API         | `app/api/questions/[questionId]/mandatory-attachments/[attachmentId]/upload/` (new) |
| UI – Edit   | `components/EditQuestionDialog.tsx` |
| UI – Detail | `app/(dashboard)/projects/[projectId]/questions/[questionId]/page.tsx` |
| Validation  | `app/(dashboard)/projects/[projectId]/submission/SubmissionChecklist.tsx` |
| Question API| `app/api/projects/[projectId]/questions/`, `app/api/questions/[questionId]/` |
| Docx/Export | `lib/docxGenerator.ts`, download routes |

---

## Notes

- Labels are saved first; uploads come later
- Archive = soft delete; filter `archived = false` in queries
- Documents for question answers use `source = "question_answer"` and should have `projectId` for project scope

---

## Implementation Summary (2026-02-10)

Completed implementation:

1. **Schema:** Added `question_mandatory_attachments` table with id, question_id, label, document_id, archived, sort_order, timestamps
2. **Migration:** `drizzle/0052_question_mandatory_attachments.sql`
3. **Actions:** `app/actions/questionMandatoryAttachments.ts` - list, add, update label, archive, set document
4. **Lib helper:** `lib/questionMandatoryAttachments.ts` - getMandatoryAttachmentLabels, getMandatoryAttachmentLabelsBatch
5. **EditQuestionDialog:** Replaced textarea with label list (add, edit, archive) using new actions
6. **QuestionMandatoryAttachmentsSection:** New component for question detail page - upload/replace/remove per attachment
7. **SubmissionChecklist:** Uses `mandatoryAttachments` array (documentId = null means missing)
8. **getQuestions/getQuestion:** Include mandatoryAttachments from new table
9. **Upload register:** Supports `source: "question_answer"`
10. **Docx/finalize/download:** Use getMandatoryAttachmentLabels (new table preferred, legacy fallback)
11. **extractQuestions:** Inserts into question_mandatory_attachments when AI extracts labels
12. **Data migration script:** `scripts/migrate-mandatory-attachments.ts` - run `pnpm run db:migrate-mandatory-attachments` after applying migration

To run migration: `pnpm run db:push` (or `pnpm run db:migrate`), then `pnpm run db:migrate-mandatory-attachments`
