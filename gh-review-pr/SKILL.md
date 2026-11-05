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
- **Leaked references**: For PRs to public/OSS repos, flag any internal issue tracker references or internal URLs in the diff, commit messages, or PR description
- **Design**: Does the approach make sense? Are there simpler alternatives? Flag unnecessary abstraction or indirection.
- **Consistency**: Does it follow existing patterns and conventions in the codebase?
- **Language conventions**: For Java code, check the JDK version in pom.xml (including parent poms) and verify the code uses modern idioms for that version (e.g. records, sealed classes, pattern matching, text blocks, switch expressions).
- **Edge cases**: Are boundary conditions handled?
- **Tests**: Are changes adequately tested? Are there missing test cases? Flag setup or assertion blocks duplicated across test methods; they belong in shared helpers or a test-support class so each test shows only what it tests.

## Reviewer persona

Before starting the review, check for `~/.claude/skills/gh-review-pr/reviewer-profile.md`. If it exists, read it and adopt its tone, focus areas, and signature moves throughout the review. All review comments and the final review body MUST reflect this persona.

## Core steps

Do all of these in one pass. Do NOT stop for an overview or ask the user to confirm before reviewing. Review silently and present findings once.

1. **Gather**: Fetch PR description, commits, full diff, author, and CI status (`gh pr view --json author,statusCheckRollup`). Read linked issue(s) if any for intent. If the current user is the author, do NOT post comments or reviews — present findings in conversation only, skip steps 3-4. If CI has passed, do not run tests locally.
2. **Review**: Apply the review criteria above to the full diff. For large PRs (30+ files), prioritize files with the most changes, new files, and critical paths (auth/data/API boundaries); note what was skimmed vs. reviewed in depth.
3. **Present findings** in one message. No preamble, no "let me first summarize", no confirmation gates. Format:
   - **Verdict**: `approve` / `comment` / `request-changes`
   - **One-line summary**: what the PR does
   - **Must fix**: bugs, security, correctness
   - **Should fix**: design, missing tests, inconsistencies
   - **Nit**: style, naming
   - **Risks**: blind spots the author may not have flagged
   - **Manual review**: files the user should review themselves (security-sensitive: auth/crypto/access control/secrets; or high-impact logic)
   - **Praise**: only for extraordinary work — a clever solution, an unusually clean abstraction, or something that teaches the reviewer. Skip for code that's merely correct.

   Choose each finding's section by its impact, not its category. Set the cost against the scale of what it affects: how often the code runs, how long the operation already takes, how many users or how much data it touches, and how hard the damage is to undo. A 10-second delay in a job that already takes hours is minor; the same delay on every request is serious. State that scale in the finding so the reader can check the weighting.

   Every finding: one line, cite `file_path:line_number`, state the issue. No restating the code. Skip empty sections entirely.

   If the changes introduce complex flow, new abstractions, or non-obvious component interactions, suggest the user run `/explain-code` on the key areas for a visual diagram.
4. **Post the inline comments as a pending draft**, right after presenting the findings. A pending review is visible only to the user until it is submitted, so the user edits, drops, or adds comments next to the diff in GitHub. If there are no inline comments, skip the draft: it would show nothing.
   - GitHub allows one pending review per user per PR. Check first: `gh api repos/OWNER/REPO/pulls/NUMBER/reviews --jq '.[] | select(.state == "PENDING") | .id'`. If one exists, stop and ask the user to submit or discard it.
   - Create the review via `POST .../pulls/NUMBER/reviews` with `commit_id` and the `comments[][]` array, without `event` (that leaves it PENDING) and without `body`: GitHub's submit dialog does not show a body stored on the draft. Never post comments individually.
   - Hand over the review body (below), give the user the link `https://github.com/OWNER/REPO/pull/NUMBER/files` and the number of draft comments, and recommend an action (approve / comment / request-changes). Say that they can submit in GitHub by pasting the body, which this session copies to the clipboard on request, or tell this session to submit.

   **Review body**: what the reviewer wants the author to take away beyond the inline comments: the overall verdict in a sentence, and any point that does not belong on a single line. It is written to the author, who knows what the PR does, so never restate the changes, the PR description, or the inline comments. If nothing remains after that, the body is just the footer. **Footer**: check for `~/.claude/skills/gh-review-pr/reviewer-footer.md`. If it exists, append its content to the review body, separated by `---`. Replace `{{model}}` with the model family, version, and variant without extras like context window size (e.g. "Opus 4.8", "GPT-5.6 Sol"). Write the body to a temp file and show it in a fenced code block, never as a block quote, for reading and editing. Never touch the user's clipboard on your own: offer to copy the body, since pasting from the terminal carries its indentation, line wrapping, and quote markers, and copy it (`pbcopy < <file>` on macOS) only when the user asks. If copying fails, for example in a sandbox, give the file path so the user can copy it themselves. If the user edits the body in the session, update the file.

   **Submitting**: only when the user says so and names the action (e.g. "submit as comment"). Submit with the body from the file and `--arg event <APPROVE|COMMENT|REQUEST_CHANGES>`, building the JSON with `jq -n --rawfile body <file> --arg event ... '{body: $body, event: $event}'` and piping it to `gh api ... --input -`:
   - With a draft: first check that the pending review still exists, since the user may have submitted or discarded it in GitHub, then post to `repos/OWNER/REPO/pulls/NUMBER/reviews/REVIEW_ID/events`. The user's edits to the draft comments in GitHub are kept.
   - Without a draft: post to `repos/OWNER/REPO/pulls/NUMBER/reviews`, adding `commit_id` to the JSON.

   Report the review's URL.

   **In-session walkthrough**, only when the user asks for it (e.g. "walk me through the comments"): before creating the draft, show each proposed comment and let the user approve, edit, or drop it, applying edits before moving on. Then show the review body for editing. The draft is still created the same way.

   Syntax pitfalls: use `-f` for strings and `-F` for numbers; single-quote `comments[][]` params; use `side=RIGHT` for added/modified lines, `LEFT` for deleted; for code suggestions, triple backticks with `suggestion` in the comment body.

## Conditional steps (apply when relevant)

- **Re-review**: If you have already submitted a review on this PR, focus on changes since the last review. Check which previous comments were addressed. Only review new or changed code in detail.
- **Local checkout**: For non-trivial changes, read the PR's code in a temporary worktree, never by switching the user's checkout: `git fetch origin pull/<NUMBER>/head` and `git worktree add --detach /tmp/review-<REPO>-<NUMBER> FETCH_HEAD`. This creates no branch and leaves the user's branch and uncommitted changes alone. When the review is done, remove it with `git worktree remove --force /tmp/review-<REPO>-<NUMBER>`, even if the review stopped early.
- **Prior art**: Search the repo (and org if relevant) for similar patterns. Note whether the PR should follow existing patterns or vice versa.
- **External prior art**: For changes touching security boundaries, cryptography, concurrency, or novel infrastructure patterns, web-search for how similar problems have been solved or have failed elsewhere. Report relevant findings (known pitfalls, established patterns, CVEs) as part of the review.

Rules:
- Be constructive and respectful in all comments.
- Do not nitpick excessively. Focus on what matters.
- No low-value PR-level comments ("Clean setup.", "Looks good overall.", "Nice work."). Either say something substantive or say nothing.
- If there are no issues, say so and recommend approval.
- Never approve or request changes without the user's explicit confirmation.
