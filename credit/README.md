# credit

A Claude Code skill that works out who wrote what in a repository where a large share of
commits are authored and committed by an AI agent.

## Why

Git records AI commits as `Claude <noreply@anthropic.com>`. Nothing in the commit links
that work to the human who directed it. That gap breaks performance reviews, CV and
portfolio claims, contractor invoicing, open-source credit, and the simple question of who
to ask about a subsystem.

## How it attributes

- Confirms the human's git identities first (people usually have three to five).
- Credits an AI commit to the person whose pull request first brought it into the
  repository.
- Processes PRs in merge-time order. Ordering by PR number inflates results — on the
  repo this method was validated against, it produced a 2.3x overcount.
- Cross-checks against an independent base-tip count and reports the spread when the two
  disagree.
- Counts commits that belong to no PR and excludes them explicitly rather than dropping
  them (about 10% of the repo in the validation case).

## Output

A markdown report covering scope, volume (with hand-written code separated from generated
or machine-written files), areas, signature work read from diffs, stack, and a timeline —
plus a stated-ambiguities section and the command behind every figure.

## Requirements

- `git`, with network access to unshallow the clone.
- GitHub CLI (`gh`), authenticated, for the PR list. Without it the skill degrades to
  identity-only attribution and says so in the report; it does not estimate the split.

## Usage

Install it from this repo's marketplace (see the [top-level README](../README.md)), or copy
this directory into `~/.claude/skills/credit/`. Then ask in a repository: "audit my
contributions", "who wrote what here", "build a contribution report for me".
