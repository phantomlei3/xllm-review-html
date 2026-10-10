---
name: review-feedback-export
description: Export merge-request review discussions to a Markdown handoff, with optional reviewer and resolution filters. Use when someone needs review feedback collected for another agent or collaborator.
---

# Review Feedback Export

Turn the review content currently published on a merge-request page into a faithful, portable Markdown handoff. Preserve enough context for another agent to act on the feedback without access to the review system.

## Workflow

1. **Inspect the destination and its conventions.** Check repository instructions, the working tree, and nearby handoff files. Reuse an established format and filename pattern when appropriate. Do not overwrite unrelated user work.

2. **Read the live review page through an authorized session.** Prefer the authenticated browser session or an available review-system connector. A raw HTTP request may return only the JavaScript application shell; if the content is absent, switch to the logged-in UI rather than treating the shell as an empty review. Refresh the page before extraction to avoid stale comments, wait for loading to finish, and expand collapsed comment groups and discussions. Check for pagination, “load more” controls, and outdated diff threads. Do not approve, resolve, submit, or otherwise mutate the review.

3. **Build a comment inventory before drafting.** For every visible item, identify its author, timestamp, root or reply relationship, inline file/line anchor, and actual discussion state. Distinguish reviewer-authored findings from author replies, automated summaries, and general review summaries. Record the MR title, source/target branches, current head if shown, review state, and extraction time.

4. **Apply the user's filters to discussions, not just visible words.**
   - If asked to omit resolved discussions, omit the resolved thread and all its replies. Determine resolution from the thread's explicit state or a clear status record in that thread. A button labeled “Resolved” may be an available action; its presence alone does not prove the thread is resolved. Likewise, “No Processing Required” may be a control, not evidence of thread state.
   - If asked for one or more reviewers, include only the requested authors' comments. Include another author's reply only when the user asks for reply context. Keep a reviewer-authored summary if it is a separate, non-resolved comment, even when some findings it references have their own resolved threads; describe it as a summary and do not recreate excluded thread content.
   - When a status or parent/reply relationship is genuinely ambiguous, inspect the containing thread and its surrounding controls or metadata. Do not guess from a global page label or a neighboring discussion. If it remains ambiguous, preserve the item with an explicit uncertainty note or ask a focused clarification when that choice changes the requested output.

5. **Write a self-contained Markdown handoff.** Include concise source metadata, the applied filters, and a short action index when there are multiple findings. Follow with the selected comments, preserving original wording, author, timestamp, reply relationship, severity, code location, and useful code excerpts. Keep reviewer claims distinct from any synthesized index; label the index as a summary. Do not present reviewer claims as independently verified facts. If the user asked to omit a category, do not include its comment text in the Markdown.

6. **Check completeness and the result.** Compare the document against the inventory: each in-scope comment appears once, excluded comments do not appear, and the stated counts/status match the page snapshot. Mention any inaccessible, collapsed, or otherwise unverified content as a scope limitation. Run `git diff --check` on the artifact and inspect the diff. Tests are unnecessary for a prose-only export unless requested.

7. **Commit and push only within the user's requested scope.** Stage only the handoff and skill files relevant to the request. Commit when asked; push only when asked. Check `git status` before and after. If Git author identity is missing, inspect repository-local configuration and recent history; reuse an identity only when repository convention clearly establishes it, otherwise ask before inventing one.

## Lessons from review-page extraction

- Reload before exporting. A previously open merge-request tab can show an older comment count and omit reviews added later.
- Expand grouped comments before counting. A page may display a single “similar comments hidden” row while holding several substantial review summaries or findings.
- The page's conversation count is not a reliable count of actionable findings: one root comment can contain many numbered issues, and one inline thread can contain multiple replies.
- “Resolved” in an action button is not necessarily a resolved-state marker. Inspect the thread itself. An explicit resolved event/state is stronger evidence than a control available to the current user.
- Keep resolved filtering and reviewer filtering separate. A requested reviewer's unresolved summary can remain in scope even when it discusses separate findings whose threads are resolved; do not copy those findings from the summary into a new standalone list unless requested.
- If the review page changes during extraction, refresh and reconcile the final document against the latest visible state before committing.

## Expected output

Use a descriptive file such as `<project>-mr-<id>-review.md` when the repository has no established naming convention. The handoff should make clear which comments were included and why, while remaining useful when forwarded without the source page.
