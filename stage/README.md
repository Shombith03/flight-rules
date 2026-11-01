# stage

A Claude Code skill that makes an agent commit only the files the current task was meant to change.

## Why

Agents reach for `git add -A` and `git commit -am`. That sweeps up build artifacts, regenerated files, lockfile churn from an incidental install, editor config, debug statements left in while exploring, unrelated edits, and sometimes `.env` files. The human then reviews a noisy diff, or doesn't. This skill removes that failure mode.

## What it does

- Forbids wildcard staging outright: `git add -A`, `git add .`, `git commit -a` and friends. Staging is by explicit path only.
- Records the expected paths and the pre-existing dirty state before the first edit, so prior uncommitted work is never swept into this task's commit — including changes the user had already staged.
- Keeps a ledger of every file the session writes, then reconciles it against `git status --porcelain` before committing. Anything changed but not in the ledger is named, explained in one line, and left unstaged.
- Screens the candidate set before commit: refuses secret-shaped paths and API-key-shaped strings in the diff, flags large binaries, generated or build output, unrequested lockfile changes, editor config, and leftover debug statements.
- Reports afterwards what was committed, what was left out, and why.
- Supports a dry-run mode that produces the plan and the exact commands, then stops.

## Usage

Install it from this repo's marketplace (see the [top-level README](../README.md)), or copy this directory into `~/.claude/skills/stage/`. Invoke it, or just let it trigger — the description fires on committing, staging, "commit this", "don't commit everything", and on tasks that will end in a commit.

For the plan without the commit:

```
/stage --dry-run
```

Installed through the marketplace, the command is namespaced: `/flight-rules:stage --dry-run`.

## Notes

Every git command whose output the skill parses uses NUL-delimited output, and every command that takes paths puts them after `--`, so paths with spaces and non-ASCII characters are handled correctly. The ledger lives inside `.git/`, so it never appears in `git status`. Nothing the skill excludes is discarded; excluded changes stay in the working tree.
