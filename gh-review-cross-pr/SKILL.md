---
name: gh-review-cross-pr
description: Review multiple related PRs across repos for consistency. Use when the user asks to "review these PRs together", "cross-review PRs", or "check consistency across PRs".
argument-hint: "[PR1] [PR2] [--context issue]"
allowed-tools: Bash, Read, Grep, Glob
---

Review multiple related PRs across repos for cross-repo consistency.

**Arguments**: `<PR1> <PR2> [PR3...] [--context <issue-url>]`
Example: `/gh-review-cross-pr org/frontend#45 org/backend#112 --context org/planning#8`

The `--context` argument is optional and can be a GitHub issue URL or reference. If provided, use it to understand the overall goal and verify the PRs implement it correctly.

## Steps

1. **Read context** (if provided): Fetch the issue to understand the overall goal and expected data flow.
2. **Read all PRs** in parallel: Fetch descriptions, commits, and full diffs for each PR.
3. **Map the interfaces**: Identify how the PRs connect:
   - Shared field/variable names, API endpoints, config keys
   - Data that flows from one repo to another (e.g. a Terraform output consumed by application code)
   - Shared constants, enums, or type definitions
4. **Check for consistency**:
   - **Naming**: Same concept should use the same name across repos (e.g. not `operatorService` in one and `operator_service` in another)
   - **Data contracts**: Types, field names, and formats must match at integration points
   - **Deployment ordering**: Note if one PR must be deployed before another to avoid breakage
   - **Missing changes**: Check if any repo is missing changes needed to complete the feature (e.g. a new field added in one repo but not consumed in another)
5. **Review each PR individually** for bugs, security, and design using the same criteria as `gh-review-pr`.
6. **Present findings** to the user:
   - **Cross-repo issues**: Naming mismatches, contract violations, missing changes, deployment ordering
   - **Per-PR issues**: Grouped by severity (must fix, should fix, nit). Praise only for extraordinary work; skip for merely correct code.
   - **Manual review recommended**: Flag files with security-sensitive changes (auth, crypto, access control, secrets, data handling) or high-impact logic changes, and tell the user to review the diff themselves.
7. **Self-review check**: If the current user is the author of all PRs, skip this step entirely. Present findings in conversation only; do not post comments or reviews.

   **Walk through each comment** with the user before posting. For each finding:
   - Show the proposed comment text and which PR it targets
   - Let the user approve, edit, or drop it

   After all comments are reviewed, show the proposed **review body** for each PR. Let the user edit them. Then confirm the action per PR (approve/comment/request-changes) and submit.

Rules:
- Be constructive and respectful in all comments.
- Focus on cross-repo consistency first; that's the primary value of this skill.
- When posting comments on individual PRs, reference the related PR(s) so reviewers have context.
- Never approve or request changes without the user's explicit confirmation.
