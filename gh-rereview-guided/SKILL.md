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
