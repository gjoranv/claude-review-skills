---
name: gh-review-respond
description: Read review comments on a GitHub PR, respond to them, fix code issues, and resolve conversations. Use when the user asks to "handle PR review comments", "fix review feedback", or "respond to PR reviews".
argument-hint: "[PR-url] [context-issue]"
allowed-tools: Bash, Read, Grep, Glob, Edit
---

Handle review comments on a GitHub PR. $ARGUMENTS: first argument is a PR URL or `owner/repo#number`. Second argument (if present) is context: a GitHub plan issue, WP URL, or Jira key (with or without `--context` prefix). If no PR is given, use the PR referenced earlier in this conversation. If context is provided, use it to better judge whether reviewer comments are relevant to the overall goal.

1. **Read all comments** on the PR from all three sources. Each has different resolution semantics — do NOT skip any source:
   - **Inline review comments**: `gh api repos/OWNER/REPO/pulls/NUMBER/comments` (comments on specific lines of code). Filter out threads marked resolved via GraphQL (`isResolved: true`).
   - **Review-level comments**: `gh api repos/OWNER/REPO/pulls/NUMBER/reviews` (body text submitted with each review, e.g. approval or request-changes summary). No resolution state — include all that contain substantive feedback not already addressed.
   - **PR-level comments**: `gh api repos/OWNER/REPO/issues/NUMBER/comments` (general comments not attached to code or reviews). **No resolution state** — these are plain issue comments. Include ALL that the user has not already responded to. A PR-level comment must never be silently dropped because it "isn't resolvable".

   If all three sources return nothing actionable, tell the user and stop. Otherwise proceed even if only PR-level comments are unanswered.
2. **Present a summary** grouped by source (inline / review-level / PR-level), with reviewer, file+line for inline, and the feedback. Never omit a source from the summary — if a source has nothing, say so explicitly ("no PR-level comments to address").
3. **Categorize each comment**:
   - **Fix needed**: The reviewer pointed out a real issue — propose a code fix.
   - **Discussion**: The reviewer asked a question or raised a point that needs the user's input.
   - **Disagree/Won't fix**: If the user indicates a comment should not be addressed, draft a reply that acknowledges the reviewer's concern, explains the reasoning for the current approach, and offers to revisit if the reviewer feels strongly. Avoid dismissive language.
4. **Ask the user** to confirm which comments to fix, which to discuss, and which to decline.
5. **Apply fixes** for all confirmed items. Most fixes are local code changes, but some may require doc updates, test additions, or config changes. If a fix requires changes in a different repo, note it for the user rather than attempting it.
6. **Stage and commit** the fixes:
   - If the PR is already approved, STOP and ask the user before committing. New commits invalidate the approval and require re-review.
   - **Bot reviewers** (e.g., Copilot): squash fixes into existing commits using `git commit --fixup` and `git rebase --autosquash`.
   - **Human reviewers**: ALWAYS create a NEW commit. NEVER squash, amend, fixup, or rebase for human reviews. The reviewer needs to see what changed.
7. **Confirm with the user** before taking any externally-visible action. Show the commit summary and ask permission to push, reply to comments, and resolve conversations. If declined, stop.
8. **Push** the branch: `git push`. Do this BEFORE replying so the reviewer sees the updated code when they re-visit the conversations.
9. **Reply to each comment**. The API differs by source:

   **Inline review comments** (reply into the existing thread):
   ```
   gh api repos/OWNER/REPO/pulls/NUMBER/comments \
     -X POST \
     -F in_reply_to=COMMENT_ID \
     -f body="Reply text"
   ```
   Use `-F` for numeric `in_reply_to`.

   **PR-level comments** (no reply mechanism — post a new issue comment that references the original):
   ```
   gh api repos/OWNER/REPO/issues/NUMBER/comments \
     -X POST \
     -f body="@reviewer Reply text"
   ```
   Start with `@<reviewer-login>` so they get notified; PR-level comments don't thread like inline ones.

   **Review-level comments** (the body of a review): reply via a PR-level comment addressed to the reviewer, same pattern as above. Don't try to reply to the review itself — that API creates a new review, not a reply.

   Content guidelines (all sources):
   - For fixes: reply with what was changed (e.g., "Fixed, changed X to Y").
   - For discussion items: reply with the user's response.
   - For declined items: reply with the rationale.
   - For bot reviewers (e.g., Copilot): keep replies short and factual (e.g., "Fixed" or "Dismissed, false positive"). No diplomatic tone needed.
10. **Resolve inline review threads** in a single batched GraphQL mutation:
    ```
    gh api graphql -f query='mutation {
      t1: resolveReviewThread(input: {threadId: "ID1"}) { thread { isResolved } }
      t2: resolveReviewThread(input: {threadId: "ID2"}) { thread { isResolved } }
    }'
    ```
    Only inline threads can be resolved — PR-level and review-level comments have no resolution state, so replying to them is enough. Every inline thread that has been replied to must be resolved.
