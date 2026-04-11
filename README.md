# claude-review-skills

Custom [Claude Code](https://claude.ai/code) skills for the GitHub PR review cycle.

## The problem

AI coding tools make writing code faster, but the review cycle stays human-speed. Giving feedback, handling comments, resolving threads, committing fixes: the mechanical parts of reviewing eat up time that should go to design judgment.

## The solution

Four skills handle the review cycle:

| Skill | What it does |
|---|---|
| `/gh-review-pr` | Reviews a PR with full codebase context, searches for prior art across the repo and org, presents findings grouped by severity, and submits a batched review |
| `/gh-review-respond` | Reads unresolved review comments, categorizes them (fix / discuss / disagree), applies fixes, replies, resolves threads, and commits |
| `/gh-review-cross-pr` | Reviews multiple related PRs across repos for consistency: naming, data contracts, deployment ordering, missing changes |
| `/gh-respond-cross-pr` | Handles review comments across related PRs as a set, with bot noise detection for cross-PR false positives |

All skills accept an optional `--context` argument (a GitHub issue URL) to evaluate changes against the overall goal. All keep the developer in control: you confirm what to post, what to fix, and what to push back on.

## Installation

Copy the skill directories into your Claude Code skills folder:

```bash
cp -r gh-review-pr gh-review-respond gh-review-cross-pr gh-respond-cross-pr ~/.claude/skills/
```

Or clone this repo and symlink:

```bash
git clone https://github.com/gjoranv/claude-review-skills ~/git/claude-review-skills
for skill in gh-review-pr gh-review-respond gh-review-cross-pr gh-respond-cross-pr; do
  ln -s ~/git/claude-review-skills/$skill ~/.claude/skills/$skill
done
```

## Usage

```
/gh-review-pr owner/repo#42                    # Review someone else's PR
/gh-review-pr owner/repo#42 --context org/planning#8  # Review with context
/gh-review-respond owner/repo#42               # Handle review feedback on your PR
/gh-review-cross-pr org/frontend#45 org/backend#112   # Review related PRs together
/gh-respond-cross-pr org/frontend#45 org/backend#112  # Handle feedback across PRs
```

The skills accept PR URLs or `owner/repo#number`. If a PR was referenced earlier in the conversation, the argument can be omitted.

## Requirements

- [Claude Code](https://claude.ai/code)
- [GitHub CLI (`gh`)](https://cli.github.com/) authenticated

## Related

These skills are described in [I Stopped Reading Diffs. Here's What I Review Instead.](https://gjoranv.hashnode.dev/i-stopped-reading-diffs-here-s-what-i-review-instead)
