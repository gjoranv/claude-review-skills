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

## 2. Walk through the points — one at a time

This is the heart of the skill and the **inverse** of `/gh-review-pr`: do NOT present all points at once. Take **one point at a time**, and stop for the user's confirmation before advancing. For each point, in order:

1. **Restate the finding in plain (layman) language.** Say what the first review objected to, without assuming the reader remembers the original wording or the code. Avoid jargon; if a term is unavoidable, define it in a phrase.
2. **Give a concrete example or analogy** when it aids understanding — a tiny input/output, a "this is like…" comparison, or the specific line that was wrong. Skip if the point is already self-evident; don't pad.
3. **Show the current status, citing the commit.** State what the author changed since the first review and point to the commit (`abc1234`) or file+line where it changed. Distinguish what the author *claims* (from their reply) from what you *observed* in the diff.
4. **Verify — don't restate.** Read the changed code as it stands now; do not trust the author's "fixed it" reply or a single code read alone. When a claim is **empirically checkable, actually check it**:
   - Migrations / ordering / schema: apply the migrations to a **throwaway** database (e.g. a temp SQLite file) and inspect the result, rather than reasoning about the SQL by eye.
   - A cited test: run that specific test and report the outcome.
   - Behavior claims: exercise the smallest slice that demonstrates it.

   Prefer **engine-independent evidence**, and **state honestly what the check does and does not prove** (e.g. "this confirms ordering on SQLite; the production engine is Postgres, so it's strong evidence but not identical"). If you cannot verify empirically, say so and fall back to a careful code read — labelled as such.
5. **Give a verdict + assessment, then STOP.** Assign one of:
   - **resolved** — the point is fully addressed, backed by the evidence above.
   - **accepted-as-non-blocking** — not fully fixed, but you're choosing to accept it (a risk you'll tolerate, a follow-up you'll track).
   - **still-open** — not addressed, or the fix is wrong/incomplete.

   Then **ask whether the user is satisfied before moving to the next point.** Do not advance on your own. If the user wants to dig deeper or adjust, stay on this point (re-verify, gather more evidence) until they're satisfied. Only then move to the next point.

**The confirmation gate is mandatory.** Never batch points, never auto-advance, and never skip ahead to the summary because the remaining points "look fine." One point, one confirmation.

## 3. Close and submit

After the last point:

1. **Summary table** of every point → its verdict, so the whole second pass is visible at a glance:

   | # | Point | Verdict | Evidence |
   |---|---|---|---|
   | 1 | Case-fold on lookup | resolved | commit `abc1234`, migration applied to throwaway DB |
   | 2 | N+1 on the list endpoint | accepted-as-non-blocking | tracked as follow-up issue |

2. **Recommend the new review action.** Default to **`approve`** when every must-fix point is *resolved* (accepted-as-non-blocking risks and tracked follow-ups don't block). Recommend `comment` or `request-changes` if any must-fix point is *still-open*. State the recommendation and the reasoning.

3. **Show the proposed review body for editing.** Draft it from the summary (what was checked, what's resolved, what's accepted, what remains) and let the user edit before anything is posted.

4. **Submit only after explicit confirmation**, via the **two-step pending-review pattern** — never post comments individually:
   1. Create a PENDING review: `POST repos/OWNER/REPO/pulls/NUMBER/reviews` with `commit_id` and any `comments[][]` array.
   2. Submit it: `POST repos/OWNER/REPO/pulls/NUMBER/reviews/REVIEW_ID/events` with `event` (`APPROVE` / `COMMENT` / `REQUEST_CHANGES`) and `body`.

   Syntax pitfalls: `-f` for strings, `-F` for numbers; single-quote `comments[][]` params; `side=RIGHT` for added/modified lines, `LEFT` for deleted.

5. **Offer to resolve the addressed inline threads.** For points marked *resolved*, offer to resolve their inline review threads in a single batched GraphQL mutation:
   ```
   gh api graphql -f query='mutation {
     t1: resolveReviewThread(input: {threadId: "ID1"}) { thread { isResolved } }
     t2: resolveReviewThread(input: {threadId: "ID2"}) { thread { isResolved } }
   }'
   ```
   Only inline threads have resolution state. Do this only after the user confirms.

### Guardrails (carried over from `/gh-review-pr`)

- **If the current user is the PR author, do NOT post** — walk through the points and converse only; skip submission and thread resolution.
- **Never approve or request changes without the user's explicit confirmation.**
- Always submit via the two-step pending-review pattern; never post individual comments.

## Language

- **Default to pt-BR.** This skill is run in Portuguese by default, so pt-BR is the zero-friction default when no language is specified.
- **Override** via the `--lang <code>` argument (e.g. `--lang en`) or a free-text request in conversation ("in English", "faz em inglês"). A free-text request takes precedence over the argument if they conflict.
- The chosen language applies to **everything the user reads and everything posted**: the plain-language restatements, examples/analogies, the summary table, and the review body submitted to GitHub. Code, identifiers, and commit hashes stay as-is.
