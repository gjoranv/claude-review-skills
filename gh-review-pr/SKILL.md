---
name: gh-review-pr
description: Review a GitHub PR as a reviewer. Use when the user asks to "review a PR", "review this pull request", or "give feedback on this PR".
argument-hint: "[PR-url] [context-issue]"
allowed-tools: Bash, Read, Grep, Glob
---

Review a GitHub PR as a code reviewer. $ARGUMENTS: first argument is a PR URL or `owner/repo#number`. Second argument (if present) is context: a GitHub issue URL or reference (with or without `--context` prefix). If no PR is given, check if the current branch has an open PR. If context is provided, use it to understand the overall goal and evaluate whether the PR achieves it.

## Review criteria

Think as a software architect first: before looking at individual lines, evaluate the overall design. Is this the right abstraction? Are the boundaries in the right place? Is the approach the simplest that solves the problem? What will be hard to change later?

Then check for:
- **Bugs**: Logic errors, off-by-one, null/undefined handling, race conditions
- **Security**: Injection, hardcoded secrets, missing input validation
- **Design**: Does the approach make sense? Are there simpler alternatives? Flag unnecessary abstraction or indirection.
- **Consistency**: Does it follow existing patterns and conventions in the codebase?
- **Edge cases**: Are boundary conditions handled?
- **Tests**: Are changes adequately tested? Are there missing test cases?

## Reviewer persona

Before starting the review, check for a `reviewer-profile.md` file in the same directory as this skill. If found, read it and adopt its tone, focus areas, and signature moves throughout the review. All review comments and the final review body MUST reflect this persona.

## Core steps

1. **Read the PR**: Fetch the PR description, commits, and full diff. Check if the current user is the PR author (`gh pr view --json author`).
   - If the user is the author, do NOT post any comments or reviews on the PR. Only present findings in the conversation for the user to act on themselves. Skip steps 6-7.
2. **Understand the context**: Read the linked issue(s) if any, to understand the intent behind the changes.
3. **Present an overview**: Briefly summarize what the PR does, which areas it touches, and any concerns at a high level. Confirm understanding with the user before the detailed review.
4. **Review the diff** file by file, using the review criteria above. For large PRs (30+ files), prioritize: files with the most changes, new files, and files touching critical paths (auth, data, API boundaries). Tell the user which files were reviewed in depth and which were skimmed.
5. **Present findings** to the user, grouped by severity:
   - **Must fix**: Bugs, security issues, correctness problems
   - **Should fix**: Design concerns, missing tests, inconsistencies
   - **Nit**: Style, naming, minor suggestions
   - **Praise**: Call out things that are well done
   - **Manual review recommended**: Flag files with security-sensitive changes (auth, crypto, access control, secrets, data handling) or high-impact logic changes, and tell the user to review the diff themselves.
6. **Ask the user** which comments to post and choose a review action:
   - Only offer "Request changes" if someone else has already approved the PR (check with `gh pr view --json reviews`). Otherwise choose between "Approve" and "Comment".
7. **Submit the review** (skip if user is the author) using a two-step pending review pattern. Never post comments individually.

   Two-step flow: (1) create PENDING review via `POST .../pulls/NUMBER/reviews` with `commit_id` and `comments[][]` array, (2) submit via `POST .../pulls/NUMBER/reviews/REVIEW_ID/events` with `event` and `body`.

   **Footer**: When submitting a Body to the review, always append the following footer at the end of it (separated by `---`):

   ```
   ---
   *AI-assisted review · Findings and comments are verified and approved by a human reviewer before submission*
   ```

   Key syntax pitfalls:
   - Use `-f` for strings, `-F` for numbers (line numbers)
   - Always single-quote `comments[][]` parameters
   - Use `side=RIGHT` for added/modified lines, `LEFT` for deleted lines
   - For code suggestions, use triple backticks with `suggestion` in the comment body

## Conditional steps (apply when relevant)

- **Re-review**: If you have already submitted a review on this PR, focus on changes since the last review. Check which previous comments were addressed. Only review new or changed code in detail.
- **Local checkout**: For non-trivial changes, run `gh pr checkout <NUMBER>` for full codebase context. Warn the user first if there are uncommitted local changes.
- **Prior art**: Search the repo (and org if relevant) for similar patterns. Note whether the PR should follow existing patterns or vice versa.

Rules:
- Be constructive and respectful in all comments.
- Do not nitpick excessively. Focus on what matters.
- If there are no issues, say so and recommend approval.
- Never approve or request changes without the user's explicit confirmation.
