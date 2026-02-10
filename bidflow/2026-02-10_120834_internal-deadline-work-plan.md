# Internal Deadline Work-Plan (Submission Checklist)

## Overview

The **Internal deadline** item on the Submission checklist uses a **work-plan** to decide whether the project is **On track**, **At risk**, or **Overdue**. It combines:

- **Time:** Period from project creation to the internal deadline (and days left / elapsed).
- **Work:** Total "weight" of all questions (by importance) and "done" weight from each question's **current status**.

So the status reflects both "how much time is left?" and "are we on plan given the work done so far?". The UI shows a short **conclusion** (e.g. "On track (12 days left)") and a **reason** (why we're on track, at risk, or overdue).

---

## What the user sees

### Conclusion (subText)

- **On track:** `On track (X days left)` — green check.
- **At risk:** `At risk (X days left)` or `X day(s) overdue` — amber.
- **Overdue:** `X day(s) overdue` — amber (deadline passed).
- **No deadline:** `No deadline set` — neutral.

### Why (explanation)

When "Show details" is on, the checklist shows a short paragraph under **Internal deadline** that explains the conclusion, for example:

- **On track:**  
  *"On plan: 45% of planned work done; expected ~42% by today. 12 days left until the internal deadline. Progress is based on question weighting and status (e.g. drafting, sign-off, completed)."*

- **At risk (time):**  
  *"Only 5 day(s) left until the internal deadline. Work completed: 30% of planned (expected ~60% by now). Focus on completing remaining questions or adjust the deadline."*

- **At risk (progress):**  
  *"Behind plan: 35% of planned work done vs ~55% expected by today (14 days left). Progress is measured by question weighting and current status. Consider reallocating resources or revising the internal deadline."*

- **Overdue:**  
  *"The internal deadline has passed. You are 3 day(s) overdue. Completed work: 70% of planned (20 questions). Consider updating the internal deadline or prioritizing remaining questions."*

So the user always sees both **what** the status is and **why** (time, progress, or both).

---

## How it's calculated

1. **Total work**  
   Each question has a **weight** (from its `weighting` field). We map that to a number (e.g. High=3, Medium=2, Low=1) and sum over all questions → **total weighted work**.

2. **Done work**  
   Each question has a **status** (unassigned, drafting, completed, etc.). Each status has a **completion fraction** (0–1). For each question we add `weight × completion` and sum → **done weighted work**.

3. **Expected work by today**  
   We assume a **linear plan**: by today we "should" have done  
   `(days elapsed / total days to deadline) × total weighted work`  
   → **expected work**.

4. **Comparison**
   - **Overdue:** Deadline date has passed → **At risk** (and explanation says overdue).
   - **At risk (time):** Days left ≤ threshold (e.g. 7) → **At risk**.
   - **At risk (progress):** Done work is more than X% behind expected (e.g. 20%) → **At risk**.
   - **On track:** Not overdue, enough days left, and progress within tolerance of expected → **Complete** (green).

All of this is implemented in `app/(dashboard)/projects/[projectId]/submission/internalDeadlineWorkPlan.ts`, and the result (level, subText, explanation) is passed into the checklist so it can show the conclusion and the "why" in the UI.

---

## Parameters (for alignment)

These constants are defined in `internalDeadlineWorkPlan.ts` so product/ops can align behaviour later without changing the overall logic.

| Parameter | Meaning | Default | Tuning |
|-----------|--------|---------|--------|
| **WEIGHT_MAP** | Question weighting text → numeric work (High/Medium/Low, etc.). | high=3, medium=2, low=1 | Adjust if you use different labels or want different effort ratios. |
| **STATUS_COMPLETION** | Completion fraction (0–1) per question status (unassigned → completed). | e.g. drafting=0.35, completed=1 | Increase/decrease to reflect how much "done" each stage represents. |
| **ON_TRACK_TOLERANCE** | Still "on track" if actual done ≥ expected − (this × total work). | 0.1 (10%) | Increase to be more lenient, decrease to be stricter. |
| **AT_RISK_DAYS_LEFT** | "At risk" if days left ≤ this (time-based). | 7 | Lower = warn later; higher = warn earlier. |
| **AT_RISK_PROGRESS_GAP** | "At risk" if actual done < expected − (this × total work). | 0.2 (20%) | Lower = more sensitive to being behind; higher = only warn when further behind. |

Edge cases (no questions, no deadline, no project start) are handled in code and either yield "outstanding" or "no deadline set" with an explanation.

---

## Files

- **Logic:** `app/(dashboard)/projects/[projectId]/submission/internalDeadlineWorkPlan.ts`  
  Parameters, weight/status helpers, and `computeInternalDeadlineLevel()` that returns level, subText, and explanation.

- **Usage:** `app/(dashboard)/projects/[projectId]/submission/page.tsx`  
  Computes `internalDeadlineResult` from project createdAt, internal deadline date, and questions (with weighting and status), then passes it to the checklist.

- **UI:** `app/(dashboard)/projects/[projectId]/submission/SubmissionChecklist.tsx`  
  Uses `internalDeadlineResult` for the Internal deadline row: status icon, subText, and (when details visible) the explanation paragraph.

- **Doc:** `docs/internal-deadline-work-plan.md` (in repo).

---

## Summary

- **Conclusion:** On track / At risk / Overdue (and "No deadline set" when applicable).  
- **Why in UI:** Explanation under Internal deadline when details are shown, describing time and/or progress so the user understands why the status is on track, at risk, or a problem (overdue).  
- **Measure:** Time from project create to internal deadline + question weighting + question status → total work, done work, expected work → level and explanation.  
- **Alignment:** All measure levels and thresholds are controlled by the parameters above and documented here for future alignment.
