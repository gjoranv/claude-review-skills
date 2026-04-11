---
name: gh-review-respond
description: Read review comments on a GitHub PR, respond to them, fix code issues, and resolve conversations. Use when the user asks to "handle PR review comments", "fix review feedback", or "respond to PR reviews".
allowed-tools: Bash, Read, Grep, Glob, Edit
---

Handle review comments on a GitHub PR. $ARGUMENTS: first argument is a PR URL or `owner/repo#number`. Second argument (if present) is context: a GitHub issue URL or reference (with or without `--context` prefix). If no PR is given, use the PR referenced earlier in this conversation. If context is provided, use it to better judge whether reviewer comments are relevant to the overall goal.

1. **Read all comments** on the PR from all sources, grouped by file and conversation thread:
   - Review comments: `gh api repos/OWNER/REPO/pulls/NUMBER/comments` (inline comments on specific lines, both from reviews and standalone)
   - PR-level comments: `gh api repos/OWNER/REPO/issues/NUMBER/comments` (general comments not attached to code lines)
   Filter out already-resolved threads and only focus on unresolved comments. If there are no unresolved comments, tell the user and stop.
2. **Present a summary** of unresolved comments to the user: list each comment with the reviewer, file, line, and the feedback given.
3. **Categorize each comment**:
   - **Fix needed**: The reviewer pointed out a real issue — propose a code fix.
   - **Discussion**: The reviewer asked a question or raised a point that needs the user's input.
   - **Disagree/Won't fix**: If the user indicates a comment should not be addressed, draft a reply that acknowledges the reviewer's concern, explains the reasoning for the current approach, and offers to revisit if the reviewer feels strongly. Avoid dismissive language.
4. **Ask the user** to confirm which comments to fix, which to discuss, and which to decline.
5. **Apply fixes** for all confirmed items. Most fixes are local code changes, but some may require doc updates, test additions, or config changes. If a fix requires changes in a different repo, note it for the user rather than attempting it.
6. **Reply to each comment** on the PR:
   ```
   gh api repos/OWNER/REPO/pulls/NUMBER/comments \
     -X POST \
     -F in_reply_to=COMMENT_ID \
     -f body="Reply text"
   ```
   Use `-F` for numeric `in_reply_to`. Content guidelines:
   - For fixes: reply with what was changed (e.g., "Fixed, changed X to Y").
   - For discussion items: reply with the user's response.
   - For declined items: reply with the rationale.
   - For bot reviewers (e.g., Copilot): keep replies short and factual (e.g., "Fixed" or "Dismissed, false positive"). No diplomatic tone needed.
7. **Resolve all addressed conversations** in a single batched GraphQL mutation. Build one mutation that resolves all threads at once:
   ```
   gh api graphql -f query='mutation {
     t1: resolveReviewThread(input: {threadId: "ID1"}) { thread { isResolved } }
     t2: resolveReviewThread(input: {threadId: "ID2"}) { thread { isResolved } }
   }'
   ```
   Do not skip this step — every comment that has been replied to must be resolved.
8. **Stage and commit** the fixes:
   - **Bot reviewers** (e.g., Copilot): squash fixes into existing commits using `git commit --fixup` and `git rebase --autosquash`.
   - **Human reviewers**: ALWAYS create a NEW commit. NEVER squash, amend, fixup, or rebase for human reviews. The reviewer needs to see what changed.
   Tell the user to push when ready.
