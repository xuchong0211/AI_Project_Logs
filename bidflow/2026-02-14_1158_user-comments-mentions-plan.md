# UserComments @-Mention Feature — Plan

## 1. Overview

Add the ability to mention other users in comments by typing `@`. Mentionable users are **project team members** (users assigned to the same project). The feature is enabled only in project-scoped comment contexts. In the input, mentions appear as **tags** (chips); stored and submitted content uses a plain-text format.

## 2. Current State

- **UserComments** (`components/UserComments.tsx`) has props: `category`, `entityId`, `entityType`, `projectId`, `refreshTrigger`.
- Three input surfaces: **new comment**, **reply**, and **edit** — each is a **contenteditable** div (not textarea) so mentions can be rendered as inline tags.
- Label: *"Add a comment. You can reference a team member by using '@'..."* when `projectId` is set; otherwise *"Add a comment."*
- **CommentAndActivityTabs** is the only consumer; it forwards props including `projectId` to UserComments.

### Usages of CommentAndActivityTabs

| Location | category | entityId | entityType | Project context |
|----------|----------|----------|------------|-----------------|
| SubscriptionsClient | `subscription` | planId or none | `plan` or none | No project |
| submission page | `submission` | projectId | `project` | Yes |
| documents list page | `documents` | projectId | `project` | Yes |
| document detail page | `documents` | documentId | `document` | Yes (projectId in URL) |
| question detail page | `question` | questionId | `question` | Yes (projectId in URL) |
| clarifications page | `clarifications` | projectId | `project` | Yes |

- Comments stored in `user_comments` with `userId` (Clerk ID). Project members in `project_team_members`; users have `clerkId`, name, email.
- **Existing API:** `GET /api/projects/[projectId]/team` returns team members (userId, userName, userEmail, clerkId, etc.).

## 3. Scope

- **Mentionable users** = users assigned to the **same project** (project team members).
- **Where enabled:** Only in project-scoped comment contexts (when `projectId` is passed).
- **Subscription (no project):** @ hint is hidden and mention dropdown is disabled.
- **Already-mentioned users** are excluded from the dropdown (no duplicate mentions in the same message).
- **Mention all:** A special “All team members” option appears in the dropdown (matches query “all”, “everyone”, “team” or empty); stored as `@[All](all)` and rendered as “All team members”.

## 4. Resolving “current project”

- When **entityType is `"project"`**: `entityId` is the projectId.
- When **entityType is `"document"` or `"question"`**: parent pages pass **`projectId`** from route params.

**Implementation:** Optional `projectId` prop on CommentAndActivityTabs and UserComments; all project-scoped parents pass it.

## 5. Backend

- **Reuse** `GET /api/projects/[projectId]/team` for mention suggestions. No new endpoint or schema changes.
- **Mention format in content:** Stored in comment `content` as plain text: `@[Display Name](clerkId)` (e.g. `@[Jane Doe](user_abc123)`, `@[All](all)` for “mention all”).
- **Regex for parsing:** `@\[([^\]]+)\]\(([^)]+)\)` to get display name and clerkId for rendering.

## 6. Frontend (UserComments)

### 6.1 Input: contenteditable with mention tags

- **Inputs** are contenteditable divs (not textareas). Stored value is still the serialized string `@[Name](clerkId)`; the DOM is built from that and serialized back on input.
- **Mention tags in input:** When the user selects someone from the dropdown, a **non-editable span** (chip) is inserted: `contenteditable="false"`, `data-mention` (clerkId), `data-mention-name` (display name), styled as a blue tag. A non-breaking space after each tag allows the cursor to sit after the mention.
- **Placeholder:** A single overlay div shows “Add a comment...” or “Write a reply...” when the serialized value is empty (`!newComment.trim()` / `!replyText.trim()`). No duplicate CSS placeholder.

### 6.2 @ detection and dropdown

- **When to show @ UI:** Only when `projectId` is present.
- **Data:** When `projectId` is set, fetch once from `GET /api/projects/[projectId]/team` and cache in state.
- **@ detection:** On input, “text before cursor” is taken from the contenteditable (serialized like stored format). Last `@` and the substring after it form the **query**. If the query contains a space, the dropdown is closed.
- **Dropdown:** Filter team members by name/email against the query. Exclude users (and “All”) already present in the current message (parse serialized value for `getMentionedClerkIds`). Show “All team members” first when the query matches “all”/“everyone”/“team” or is empty and “All” is not already mentioned. Limit to 8 individual members.
- **Keyboard:** **Arrow Up/Down** move the highlighted option; **Enter** inserts the highlighted mention; **Escape** closes the dropdown. Selected item is scrolled into view.

### 6.3 Insertion and deletion

- **On select (click or Enter):** The **“@” and the typed query** (e.g. `@j`) are **removed** from the contenteditable (range is expanded backward by `1 + query.length` characters and deleted), then the mention tag is inserted at that position. State is updated by serializing the contenteditable.
- **Backspace:** If the node immediately before the cursor is a mention tag (or the space we inserted after it), that tag (and the following space) is removed in one Backspace; default behaviour is prevented.

### 6.4 Clearing after submit

- **New comment:** On submit, state is cleared (`setNewComment("")`) and the new-comment contenteditable is cleared via `setEditableContentFromMentionString(newCommentRef.current, "")`. On submit error, both state and contenteditable are restored.
- **Reply:** Before clearing state and closing the reply form, the reply contenteditable is cleared. When the reply form is opened again (or restored after error), its content is set from `replyText` in a `useEffect` when `replyingTo` changes.

### 6.5 Rendering (read-only)

- When **displaying** comment/reply content, `renderContentWithMentions(content)` parses the stored string and renders mention tokens as styled spans (`@Display Name` with blue styling). For `clerkId === "all"`, display text is “All team members”.

## 7. Utilities (UserComments and exports)

- **Parse/serialize:** `parseMentionStringToSegments`, `serializeContentEditableToMentionString`, `getTextBeforeCursorFromEditable`, `setEditableContentFromMentionString`.
- **Insert/delete:** `insertMentionTagAtCursor(container, name, clerkId, deleteCharsBeforeCursor)`, `moveRangeStartBackward`, `getMentionTagBeforeCursor`, `isMentionTag`.
- **Mention list:** `getMentionedClerkIds(text)`, `MENTION_ALL_CLERK_ID`, `MENTION_ALL_OPTION`, `getMentionInsertText`, `renderContentWithMentions`, `MENTION_REGEX`.

## 8. Parent Pages

- **Submission page:** Pass `projectId={projectId}`.
- **Documents list page:** Pass `projectId={projectId}`.
- **Document detail page:** Pass `projectId={projectId}`.
- **Question detail page:** Pass `projectId={projectId}`.
- **Clarifications page:** Pass `projectId={projectId}`.
- **SubscriptionsClient:** Do not pass `projectId`; mentions disabled, @ hint hidden.

## 9. Optional / Later

- **Notifications:** Parse content for mentions and notify mentioned users (separate task).
- **Subscription/organisation mentions:** If needed, define “mentionable” (e.g. org members) and a separate endpoint; keep this iteration project-only.

## 10. Mention Format (reference)

- **Stored in DB:** `@[Display Name](clerkId)` e.g. `@[Jane Doe](user_abc123)`, `@[All](all)` for “mention all”.
- **Regex:** `@\[([^\]]+)\]\(([^)]+)\)` for parsing.
