---
name: gh-review-guided
description: First-pass review of a PR followed by a guided walkthrough — produce
  the findings, then walk through them one at a time in plain language with examples,
  confirming satisfaction before each next point, then post. Use for "revisão guiada",
  "revisar guiado", "review guiada", "guided review", "guided PR review".
argument-hint: "[PR-url] [--context issue] [--lang xx]"
allowed-tools: Bash, Read, Grep, Glob
---

**First-pass guided** review of a PR. Where `/gh-review-pr` does a first pass silently and presents every finding at once, this skill produces the same first-pass findings and then walks you through them one at a time — in plain language, with examples, confirming you're satisfied with each point before advancing — and finally assembles and posts the review. Use it for a fresh PR you want to *understand*, point by point, before deciding what to post.

This is a **first** pass over findings produced now, in this session. It is deliberately distinct from `/gh-rereview-guided`, which is a *second* pass that re-checks a prior review's points against the author's later fixes.

$ARGUMENTS: first argument is a PR URL or `owner/repo#number`. Optional `--context <issue>` (a GitHub issue URL/ref, WP URL, or Jira key) frames the overall goal. Optional `--lang <code>` sets the narration language (see **Language** in section 4). If no PR is given, check whether the current branch has an open PR.

## 1. Review (first pass)

Do this section silently — gather, review, and form findings before narrating anything. The walkthrough (section 4) is where you go slow; here you work like `/gh-review-pr`.

1. **Resolve the PR and parse arguments.** Take the PR from the first argument (PR URL or `owner/repo#number`); if none is given, use the current branch's open PR (`gh pr view --json number,url,author`). Record `OWNER`, `REPO`, `NUMBER`, and the author's login. Pull `--context <issue>` and `--lang <code>` out of the arguments if present.

2. **Gather the material.** Fetch the PR description, commits, full diff, and author (`gh pr view NUMBER --json title,body,commits,author`; `gh pr diff NUMBER`). Read any linked issue(s) and the `--context` issue for intent — use them to judge whether the PR actually achieves its goal.

3. **Author guardrail.** Get your login (`gh api user --jq .login`). **If you are the PR author, do NOT post** — run the walkthrough and converse only; skip the posting in section 5.

4. **Review against the `gh-review-pr` criteria.** Think as a software architect first: is this the right abstraction, are the boundaries right, is it the simplest approach, what will be hard to change later? Then check for:
   - **Bugs**: logic errors, off-by-one, null/undefined handling, race conditions.
   - **Security**: injection, hardcoded secrets, missing input validation.
   - **Design**: does the approach make sense? Flag unnecessary abstraction or indirection.
   - **Consistency**: does it follow existing patterns and conventions in the codebase? For Java, check the JDK version in pom.xml (including parent poms) and verify modern idioms for that version (records, sealed classes, pattern matching, text blocks, switch expressions).
   - **Edge cases**: are boundary conditions handled?
   - **Tests**: are changes adequately tested? Any missing cases?

   For large PRs (30+ files), prioritize files with the most changes, new files, and critical paths (auth/data/API boundaries); note what was skimmed vs. reviewed in depth.

5. **Check out locally and verify when checkable.** For non-trivial PRs, `gh pr checkout NUMBER` for full codebase context (warn first if there are uncommitted local changes). When a finding is empirically checkable, actually run the check on the checked-out branch (e.g. `ruff`, `pytest`, or the smallest slice that demonstrates the behavior) rather than reasoning by eye. Prefer engine-independent evidence and note what the check does and does not prove.

6. **Reviewer persona.** Check for `~/.claude/skills/gh-review-pr/reviewer-profile.md`. If it exists, read it and adopt its tone, focus areas, and signature moves throughout the walkthrough and the posted review.

## 2. Show the map

Before the point-by-point walkthrough, present the whole shape **once** so the reviewer sees where it's going:

- **Verdict**: `approve` / `comment` / `request-changes` (a recommendation — confirmed later in section 5, never final here).
- **One-line summary**: what the PR does.
- **The ordered list of points** you'll walk through, most important first: must-fix (bugs, security, correctness) → should-fix (design, missing tests, inconsistencies) → nit (style, naming). One line each, citing `file_path:line_number`. Merge duplicates into a single point.

Present this list once and then stop — the walkthrough itself starts in section 4. Do not expand any point into detail here; the map is just the table of contents.

## 3. Problem/solution context

Before point 1, ground the reviewer in what the PR is even about — in plain (layman) language, no assumption that they've read the diff:

- **The problem** the PR sets out to solve: what was wrong, missing, or painful before it.
- **The solution** it chose: the approach the author took, and any notable alternative they *didn't* take.

Reach for an analogy when it makes the shape click ("this is like adding a coat-check so you don't carry every bag yourself"); skip it when the change is self-evident. Keep this short — it's the on-ramp to the first point, not a second review. This block is **mandatory**: always set the context before the walkthrough begins.

## 4. Guided walkthrough — one point at a time

This is the heart of the skill. Do NOT present all points at once (that's `/gh-review-pr`; you already showed the *map* in section 2). Take **one point at a time**, in the order from the map, and stop for the user's confirmation before advancing. For each point:

1. **Restate the finding in plain (layman) language.** Say what the concern is without assuming the reader remembers the code or the map wording. Avoid jargon; if a term is unavoidable, define it in a phrase.
2. **Give a concrete example or analogy** when it aids understanding — a tiny input/output, a "this is like…" comparison, or the specific line at issue. Skip if the point is already self-evident; don't pad.
3. **Show the current status, citing file/line.** Point to the `file_path:line_number` the finding is about and show what the code does there now.
4. **Verify — don't restate.** Read the code as it stands; when the finding is empirically checkable, **actually check it** (run the specific test, `ruff`/`pytest`, or the smallest slice that demonstrates the behavior) rather than reasoning by eye. Prefer engine-independent evidence, and **state honestly what the check does and does not prove** (e.g. "this confirms it on SQLite; production is Postgres, so it's strong evidence, not identical"). If you can't verify empirically, say so and fall back to a careful code read — labelled as such.
5. **Decide: inline comment or chat-only.** Assign each point one disposition:
   - **inline comment** — worth posting on the PR at its `file_path:line_number`.
   - **chat-only** — design discussion, a non-blocking risk, or a trivial nit the linter already enforces: talk it through but don't post it.
6. **Confirmation gate (mandatory).** Ask whether the user is satisfied before moving to the next point. Do NOT auto-advance. If they want to dig deeper or adjust the disposition, stay on this point (re-verify, gather more evidence) until they're satisfied. Only then move on.

**Never batch points, never auto-advance, and never skip ahead to the summary because the remaining points "look fine."** One point, one confirmation.

### Language

- **Default to pt-BR.** This skill is run in Portuguese by default, so pt-BR is the zero-friction default when no language is specified.
- **Override** via the `--lang <code>` argument (e.g. `--lang en`) or a free-text request in conversation ("in English", "faz em inglês"). A free-text request takes precedence over the argument if they conflict.
- The chosen language applies to **everything the user reads and everything posted**: the map, the problem/solution context, the plain-language restatements, examples/analogies, the summary table, and the review body submitted to GitHub. Code, identifiers, and commit hashes stay verbatim.

## 5. Close and post

After the last point:

1. **Summary table** of every point → its disposition and verdict, so the whole pass is visible at a glance:

   | # | Point | Disposition | Verdict |
   |---|---|---|---|
   | 1 | Missing null check on `user.email` | inline comment | must-fix |
   | 2 | Prefer a record over the DTO class | chat-only | nit |

2. **Assemble comments and body.** Draft the inline comments for **only** the points flagged *inline comment* (each at its `file_path:line_number`), and the review body (the summary that accompanies the review). Let the user edit both before anything is posted.

3. **Footer.** Check for `~/.claude/skills/gh-review-pr/reviewer-footer.md`. If it exists, append its content to the review body, separated by `---`. Replace `{{model}}` with the model name powering this session (e.g. "Claude Opus 4.6").

4. **Confirm the action.** State the recommended review action (`approve` / `comment` / `request-changes`) and the reasoning. **Never `approve` or `request-changes` without the user's explicit confirmation.** (If you are the PR author — the section 1 guardrail — stop here: do not post.)

5. **Submit via the two-step pending-review pattern** — never post comments individually:
   1. Create a PENDING review: `POST repos/OWNER/REPO/pulls/NUMBER/reviews` with `commit_id` and the `comments[][]` array.
   2. Submit it: `POST repos/OWNER/REPO/pulls/NUMBER/reviews/REVIEW_ID/events` with `event` (`APPROVE` / `COMMENT` / `REQUEST_CHANGES`) and `body`.

   Syntax pitfalls: `-f` for strings, `-F` for numbers; single-quote `comments[][]` params; `side=RIGHT` for added/modified lines, `LEFT` for deleted; for code suggestions, triple backticks with `suggestion` in the comment body.

### Guardrails

- **If the current user is the PR author, do NOT post** — walk through the points and converse only; skip submission.
- **Never approve or request changes without the user's explicit confirmation.**
- Always submit via the two-step pending-review pattern; never post individual comments.
- Apply `reviewer-profile.md` / `reviewer-footer.md` if present, exactly as `/gh-review-pr` does.
