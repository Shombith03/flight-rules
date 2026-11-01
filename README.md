# flight-rules

Claude Code skills from nineteen months of running agents on a 4,700-file codebase.

NASA's flight rules are the book of "if X happens, do Y". Each rule exists because something went wrong once and nobody wanted to work out the answer again under pressure. These skills are the same idea for coding agents: each one encodes a failure that happened, and the procedure that stops it happening again.

| If… | Then | Skill |
|---|---|---|
| an agent is about to commit, and `git add -A` would sweep in build output, lockfile churn, debug leftovers or a `.env` | commit only the files the task was meant to change, and report the rest by name | [`stage`](stage/) |
| most commits are authored by `Claude <noreply@anthropic.com>` and nobody can say who directed what | reconstruct attribution from pull-request provenance, and state plainly what can't be attributed | [`credit`](credit/) |

These are general software skills. They assume `git`; `credit` also uses the GitHub CLI when it's available.

## Install

As a plugin, from inside Claude Code:

```
/plugin marketplace add shombith03/flight-rules
/plugin install flight-rules@flight-rules
```

Plugin skills are namespaced, so they're available as `/flight-rules:stage` and `/flight-rules:credit`. They also trigger on their own from their descriptions.

Or copy a single skill into your personal skills directory (or a project's `.claude/skills/`):

```bash
git clone https://github.com/shombith03/flight-rules.git
```

```bash
cp -r flight-rules/stage ~/.claude/skills/stage
```

Installed this way, it's `/stage`.

## Layout

```
.claude-plugin/marketplace.json   marketplace manifest
credit/SKILL.md                   who wrote what, in repos where agents write the commits
stage/SKILL.md                    commit only what the task changed
```

Each skill directory has its own README with the reasoning behind it.

## License

[MIT](LICENSE)
