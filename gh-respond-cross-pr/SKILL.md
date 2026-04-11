---
name: gh-respond-cross-pr
description: Respond to review comments across multiple related PRs. Use when the user asks to "handle review comments across PRs", "respond to cross-PR reviews", or "fix feedback on all PRs".
allowed-tools: Bash, Read, Grep, Glob, Edit
---

Handle review comments across multiple related PRs that belong together.

**Arguments**: `<PR1> <PR2> [PR3...] [--context <issue-url>]`
Example: `/gh-respond-cross-pr org/frontend#45 org/backend#112 --context org/planning#8`

The `--context` argument is optional and can be a GitHub issue URL or reference. If provided, use it to understand the overall goal and better distinguish real issues from bot noise.

## Steps

1. **Read all unresolved comments** across all PRs, from both review and inline comment endpoints.
2. **Identify cross-PR context**: Understand how the PRs relate. A comment on one PR may be best addressed by a change in another PR.
3. **Filter bot noise**: Bot reviewers (e.g. Copilot) review each PR in isolation and may flag things that only make sense in the context of the full change set. Identify and flag these:
   - Missing imports/definitions that exist in a sibling PR
   - "Unused" code that is consumed by a sibling PR
   - Naming suggestions that would break consistency with sibling PRs
   Mark these as dismissible with a short explanation referencing the related PR.
4. **Present a unified summary** to the user: group comments by PR, showing reviewer, file, line, and feedback. Flag bot noise separately.
5. **Categorize each comment** (same as `gh-review-respond`):
   - **Fix needed**: Real issue, propose a fix. Note which PR the fix belongs in.
   - **Discussion**: Needs user input.
   - **Disagree/Won't fix**: Acknowledge concern, explain reasoning, offer to revisit.
   - **Bot noise**: Cross-PR false positive, dismiss with explanation.
6. **Ask the user** to confirm which comments to fix, discuss, decline, or dismiss.
7. **Apply fixes** in the correct repos. Some fixes from a comment on one PR may need to be applied in another PR's repo.
8. **Reply to each comment** on the appropriate PR:
   ```
   gh api repos/OWNER/REPO/pulls/NUMBER/comments \
     -X POST \
     -F in_reply_to=COMMENT_ID \
     -f body="Reply text"
   ```
   - For bot noise: reply with a brief explanation (e.g. "This is defined in the sibling PR org/backend#112").
9. **Resolve all addressed conversations** in a single batched GraphQL mutation per PR.
10. **Stage and commit** fixes in each repo:
    - **Bot reviewers**: squash into existing commits with `git commit --fixup` and `git rebase --autosquash`.
    - **Human reviewers**: ALWAYS create a NEW commit. NEVER squash for human reviews.
    Tell the user to push each repo when ready.

Rules:
- Cross-PR awareness is the primary value. Always consider the full change set before acting on a comment.
- Be constructive and respectful.
