---
name: gh-rereview-guided
description: Guided second-pass review of a PR you already reviewed — walk through
  the first review's points one at a time, in plain language with examples,
  confirming satisfaction before each next point. Use for "second review",
  "re-review", "segunda revisão".
argument-hint: "[PR-url] [--lang xx]"
allowed-tools: Bash, Read, Grep, Glob
---

Guided **second-pass** review of a PR you already reviewed. Where `/gh-review-pr` does a first pass silently and presents all findings once, this skill does the inverse: it walks through the points raised by the **first review** one at a time, in plain language, and confirms you're satisfied with each point before moving to the next. Use it after the author has pushed fixes and you need to decide whether to change your verdict (typically `request-changes` → `approve`).

$ARGUMENTS: first argument is a PR URL or `owner/repo#number`. An optional `--lang <code>` sets the narration language (see **Language**). If no PR is given, check whether the current branch has an open PR.

## 1. Gather and decompose the first review

1. **Resolve the PR**. Use the first argument (PR URL or `owner/repo#number`). If none is given, check the current branch's open PR (`gh pr view --json number,url,author`). Record `OWNER`, `REPO`, `NUMBER`, and the PR author's login.

2. **Locate the "first review"** — the prior review to walk through:
   - Fetch all reviews: `gh api repos/OWNER/REPO/pulls/NUMBER/reviews`.
   - Default: the **current user's most recent** review (`gh api user --jq .login` to get your login; pick your latest review by `submitted_at`).
   - Fall back to the PR's most recent review if you have none.
   - **If there is no prior review at all → STOP.** This skill is a second pass; there's nothing to walk through. Suggest `/gh-review-pr` for a first-pass review.
   - Note the reviewed `commit_id` (the commit the first review was submitted against) — you'll diff against it to see what the author changed since.

3. **Fetch the material** needed to judge each point:
   - **Review body**: the `body` of the first review (structured buckets: must-fix / should-fix / nit / risks / follow-ups).
   - **Inline comments** from that review: `gh api repos/OWNER/REPO/pulls/NUMBER/comments`, keyed by review or by `pull_request_review_id`. Record `(id, file, line, body)` for each.
   - **Author replies**: replies threaded under those inline comments, plus any PR-level or review-level responses (`gh api repos/OWNER/REPO/issues/NUMBER/comments`). These say what the author *claims* to have done — treat as claims to verify, not as truth.
   - **Changes since the reviewed commit**: `gh api repos/OWNER/REPO/pulls/NUMBER/commits` and the diff since `commit_id` (e.g. `gh pr diff NUMBER`, or check out with `gh pr checkout NUMBER` for non-trivial verification). This is the evidence you check points against.

4. **Decompose the first review into an ordered list of discrete "points."** A *point* is one item to walk through. Sources:
   - Each entry in the review body's buckets (must-fix / should-fix / nit / risks / follow-ups).
   - Each inline review comment.

   Merge duplicates (an inline comment and a body bullet describing the same issue are one point). Keep this decomposition **prose-guided, not a rigid parser** — review bodies vary in format, so read for intent rather than matching a fixed structure. Order points by importance (must-fix first), and present the ordered list before starting the walkthrough so the user knows what's coming.
