---
name: gh-review-pr
description: Review a GitHub PR as a reviewer. Use when the user asks to "review a PR", "review this pull request", or "give feedback on this PR".
argument-hint: "[PR-url] [context-issue]"
allowed-tools: Bash, Read, Grep, Glob
---

Review a GitHub PR as a code reviewer. $ARGUMENTS: first argument is a PR URL or `owner/repo#number`. Second argument (if present) is context: a GitHub plan issue, WP URL, or Jira key (with or without `--context` prefix). If no PR is given, check if the current branch has an open PR. If context is provided, use it to understand the overall goal and evaluate whether the PR achieves it.

## Review criteria

Think as a software architect first: before looking at individual lines, evaluate the overall design. Is this the right abstraction? Are the boundaries in the right place? Is the approach the simplest that solves the problem? What will be hard to change later?

Then check for:
- **Bugs**: Logic errors, off-by-one, null/undefined handling, race conditions
- **Security**: Injection, hardcoded secrets, missing input validation
- **Design**: Does the approach make sense? Are there simpler alternatives? Flag unnecessary abstraction or indirection.
- **Consistency**: Does it follow existing patterns and conventions in the codebase?
- **Language conventions**: For Java code, check the JDK version in pom.xml (including parent poms) and verify the code uses modern idioms for that version (e.g. records, sealed classes, pattern matching, text blocks, switch expressions).
- **Edge cases**: Are boundary conditions handled?
- **Tests**: Are changes adequately tested? Are there missing test cases?

## Reviewer persona

Before starting the review, check for a `reviewer-profile.md` file in the same directory as this skill. If found, read it and adopt its tone, focus areas, and signature moves throughout the review. All review comments and the final review body MUST reflect this persona.

## Core steps

Do all of these in one pass. Do NOT stop for an overview or ask the user to confirm before reviewing. Review silently and present findings once.

1. **Gather**: Fetch PR description, commits, full diff, and author (`gh pr view --json author`). Read linked issue(s) if any for intent. If the current user is the author, do NOT post comments or reviews — present findings in conversation only, skip steps 3-4.
2. **Review**: Apply the review criteria above to the full diff. For large PRs (30+ files), prioritize files with the most changes, new files, and critical paths (auth/data/API boundaries); note what was skimmed vs. reviewed in depth.
3. **Present findings** in one message. No preamble, no "let me first summarize", no confirmation gates. Format:
   - **Verdict**: `approve` / `comment` / `request-changes`
   - **One-line summary**: what the PR does
   - **Must fix**: bugs, security, correctness
   - **Should fix**: design, missing tests, inconsistencies
   - **Nit**: style, naming
   - **Risks**: blind spots the author may not have flagged
   - **Manual review**: files the user should review themselves (security-sensitive: auth/crypto/access control/secrets; or high-impact logic)
   - **Praise**: only if genuinely warranted — skip the section if not

   Every finding: one line, cite `file_path:line_number`, state the issue. No restating the code. Skip empty sections entirely.
4. **Ask** which findings to post and the action (approve/comment/request-changes). Then submit via the two-step pending review pattern. Never post comments individually.

   Two-step submission: (1) create PENDING review via `POST .../pulls/NUMBER/reviews` with `commit_id` and `comments[][]` array, (2) submit via `POST .../pulls/NUMBER/reviews/REVIEW_ID/events` with `event` and `body`.

   Syntax pitfalls: use `-f` for strings and `-F` for numbers; single-quote `comments[][]` params; use `side=RIGHT` for added/modified lines, `LEFT` for deleted; for code suggestions, triple backticks with `suggestion` in the comment body.

## Conditional steps (apply when relevant)

- **Re-review**: If you have already submitted a review on this PR, focus on changes since the last review. Check which previous comments were addressed. Only review new or changed code in detail.
- **Local checkout**: For non-trivial changes, run `gh pr checkout <NUMBER>` for full codebase context. Warn the user first if there are uncommitted local changes.
- **Prior art**: Search the repo (and org if relevant) for similar patterns. Note whether the PR should follow existing patterns or vice versa.

Rules:
- Be constructive and respectful in all comments.
- Do not nitpick excessively. Focus on what matters.
- If there are no issues, say so and recommend approval.
- Never approve or request changes without the user's explicit confirmation.
