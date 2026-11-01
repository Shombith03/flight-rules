---
name: credit
description: Determine who wrote what in a git repository where many commits are authored by an AI agent, and produce a traceable contribution report. Use when the user asks who wrote what, what their contributions are, to attribute commits, to measure one person's share of a repo, to build a contribution report, performance-review or invoicing evidence, or a portfolio or CV claim from git history.
---

# credit

Repositories built with Claude Code accumulate commits authored and committed as
`Claude <noreply@anthropic.com>`. Git records no link from those commits to the human
who directed them. This skill reconstructs that link from pull request provenance, and
reports plainly what cannot be reconstructed.

Never invent a number. Every figure in the report must come from a command you actually
ran in this session. If a step fails, say so in the output and drop the dependent figure.

## 0. Prepare the repository

```bash
git rev-parse --is-shallow-repository
git fetch --unshallow 2>/dev/null || git fetch --all --tags --prune
git fetch origin '+refs/pull/*/head:refs/remotes/origin/pr/*' 2>/dev/null || true
```

Count the full history across all refs, not the default branch:

```bash
git rev-list --all --count
git log --all --format=%cI | sort | sed -n '1p;$p'
```

Use `--all` in every later command that walks history. A report built from `HEAD` alone
undercounts and must not be presented as a repo-wide figure.

## 1. Establish the human's git identities

```bash
git log --all --format='%an <%ae>' | sort -u
git log --all --format='%cn <%ce>' | sort -u
```

Show the list to the user and ask which identities are theirs. Do not guess from name
similarity. People commonly have three to five: personal email, work email, a GitHub
`users.noreply.github.com` address, and one from a former employer. Also ask which
identities belong to AI agents (typically `Claude <noreply@anthropic.com>`, and any
bot accounts).

Record the confirmed identity set. All later counts are reported per identity-set, and
the report names the exact identities used.

Direct human commits:

```bash
git log --all --author='<email1>' --author='<email2>' --format=%H | sort -u | wc -l
```

## 2. Attribute AI commits by the PR that introduced them

Rule: an AI commit counts as person X's when it **first lands in the repository through a
pull request opened by X**.

Get the PR list, ordered by merge time:

```bash
gh pr list --state merged --limit 10000 \
  --json number,author,headRefOid,mergedAt,title,baseRefName \
  > /tmp/prs.json
gh pr list --state open --limit 2000 \
  --json number,author,headRefOid,title,baseRefName > /tmp/prs-open.json
```

If `gh` is missing, unauthenticated, or the repo has no remote on GitHub, skip to
section 6 (degraded mode).

Process PRs in ascending `mergedAt`. Unmerged PRs go last, after all merged ones, in
ascending PR number. For each PR in that order, its commits are those reachable from its
head SHA and not reachable from the head of any PR already processed:

```bash
git rev-list <headRefOid> --not <head1> <head2> ... <head_{n-1}>
```

Each commit is claimed by the first PR that brings it in; a commit is never counted
twice. Accumulate per PR author, splitting each PR's commits into AI-authored and
human-authored by the author identity of each commit.

Because the `--not` list grows long, write the processed heads to a file and use
`git rev-list <head> --stdin` with the earlier heads prefixed by `^`, or batch them.
Script this rather than running thousands of commands by hand; keep the script and name
its path in the report so the numbers are reproducible.

The script writes two files that later sections depend on:

- `/tmp/claimed.txt` — every SHA claimed by any PR, one per line.
- `/tmp/ai-attributed.txt` — the AI-authored SHAs claimed by PRs opened by the person
  being audited.

## 3. Order by merge time, never by PR number

Ordering PRs by number inflates attribution badly. A long-lived branch opened early but
merged late has, by merge time, absorbed commits that other PRs already merged; number
order hands all of those to the early PR. On the repository this method was validated
against, number order produced a 2.3x overcount compared with merge order.

Cross-check with an independent method before reporting. Base-tip method: for each PR,
take the merge base of its head with the base branch as it stood at merge time, and count
`git rev-list <base>..<head>`, then deduplicate across PRs in merge order.

```bash
git merge-base <baseSha> <headRefOid>
git rev-list <mergeBase>..<headRefOid> --count
```

If the two methods agree within a few percent, report the merge-order figure and note the
cross-check. If they disagree by more than a few percent, report both numbers and the
spread. Do not pick one silently.

## 4. Count and report what cannot be attributed

Commits that belong to no PR must be counted and excluded, never dropped silently:

```bash
# all commits minus the union of all claimed commits
git rev-list --all > /tmp/all.txt
# claimed.txt written by the section 2 script
comm -23 <(sort -u /tmp/all.txt) <(sort -u /tmp/claimed.txt) | wc -l
```

These are direct pushes to default branches, branches merged without a PR, imported
history, and branches never opened as PRs. In the validation case this was about 10% of
the repo. The report states this count, its share, and that it is
unattributable by this method — not assigned to anyone.

Also report AI commits inside PRs opened by other people as theirs, not as unknown.

## 5. Stated ambiguities

Include these verbatim in every report, as a section, not a footnote:

- A PR opened by X may carry commits written in another person's session, if X's branch
  merged that person's branch before X's PR merged. Those commits are credited to X.
- The reverse also happens: work from X's session can be credited to another person whose
  PR merged first.
- PR authorship is a proxy for direction, not proof of it. Pair sessions, branch handoffs
  and re-opened PRs all break the proxy.
- Commits outside any PR are excluded entirely.
- Line counts include generated or machine-written files unless separated.

## 6. Degraded mode (no `gh`)

If the PR list is unavailable, attribute only by git identity:

```bash
git log --all --author='<email1>' --author='<email2>' --format=%H | sort -u | wc -l
```

Report the human's own commits, and report AI commits as a single unattributed total.
State explicitly at the top of the report that PR provenance was unavailable, name the
reason (`gh` not installed / not authenticated / no GitHub remote), and do not estimate
the split. An estimate here would be invented.

In this mode there is no `/tmp/ai-attributed.txt`. Section 7 runs over the person's direct
commits only, and the report says so.

## 7. Build the report

Every section below runs over the person's **attributed set**: their direct commits plus
the AI commits their PRs brought in. Filtering by `--author` alone would see only the
direct commits and leave out the AI work this skill exists to attribute.

```bash
git log --all --author='<email1>' --author='<email2>' --format=%H | sort -u > /tmp/direct.txt
touch /tmp/ai-attributed.txt   # empty in degraded mode
sort -u /tmp/direct.txt /tmp/ai-attributed.txt > /tmp/mine.txt
wc -l /tmp/direct.txt /tmp/ai-attributed.txt /tmp/mine.txt
```

The commands below read a set with `git log --no-walk --stdin`. Given empty input, git
falls back to `HEAD` and silently reports the wrong commits, so every command checks that
its set file is non-empty first. Where an empty set is legitimate (no AI commits, degraded
mode), report the figure as zero and say why.

Write a markdown report with these sections, in this order.

**Scope.** The identity set used (listed in full), commit counts per identity, the date
range from first to last commit, total repo commits, and the person's share as a
percentage. State the attribution method and the unattributable count from section 4.

**Volume.** Report direct and AI-attributed commits separately, then combined:

```bash
for set in direct ai-attributed mine; do
  [ -s "/tmp/$set.txt" ] || { echo "$set: empty"; continue; }
  printf '%s: ' "$set"
  git log --no-walk --stdin --numstat --format='---%H' < "/tmp/$set.txt" \
    | awk '/^[0-9]/ {a+=$1; r+=$2} END {print a+0, r+0}'
done
```

Report lines added and removed. Then report them again excluding generated or
machine-written files, with the exclusion list shown:

```bash
for set in direct ai-attributed mine; do
  [ -s "/tmp/$set.txt" ] || { echo "$set: empty"; continue; }
  printf '%s: ' "$set"
  git log --no-walk --stdin --numstat --format='---%H' < "/tmp/$set.txt" -- \
    . ':(exclude)*.lock' ':(exclude)package-lock.json' ':(exclude)*.min.*' \
    ':(exclude)dist/*' ':(exclude)build/*' ':(exclude)vendor/*' ':(exclude)*.svg' \
    ':(exclude)*.csv' ':(exclude)*.json' \
    | awk '/^[0-9]/ {a+=$1; r+=$2} END {print a+0, r+0}'
done
```

Include a warning in the report that raw line counts are dominated by generated files,
vendored dependencies, lockfiles, data blobs and bulk deletions, and that hand-written
code is the smaller, more meaningful figure. Report the two separately; never present raw
totals as code written.

**Areas.** Cluster the attributed commits by the paths they touch:

```bash
[ -s /tmp/mine.txt ] && git log --no-walk --stdin --name-only --format='' < /tmp/mine.txt \
  | grep -v '^$' | awk -F/ '{print (NF > 1 ? $1"/"$2 : $1)}' | sort | uniq -c | sort -rn | head -40
```

Group the top paths into named subsystems. State the heuristic you used (top two path
segments, by number of touching commits) and its limits: it rewards files that change
often rather than work that was hard, it misses single-file subsystems, and a repo-wide
rename or reformat skews it. Spot-check the top clusters by reading a few files before
naming a subsystem.

**Signature work.** Find the largest commits by lines changed, then read their diffs:

```bash
[ -s /tmp/mine.txt ] && git log --no-walk --stdin --no-merges --numstat \
    --format='@%H %cs %s' < /tmp/mine.txt \
  | awk '/^@/      {if (h) print t, h; h = substr($0, 2); t = 0; next}
         /^[0-9]/  {t += $1 + $2}
         END       {if (h) print t, h}' \
  | sort -rn | head -40
git show --stat <sha>
```

Each output line is `<lines changed> <sha> <date> <subject>`. Size is only a starting
point: the largest commits are often generated files or bulk moves.

Commit messages in AI-assisted repos are often thin or templated. Read the diff to say
what the commit actually did. Pick five to fifteen commits that introduced a system,
changed an interface, or fixed something structural. Describe each in one or two
sentences with its SHA and date, and whether it was a direct or an AI-attributed commit.

**Stack.** Report what the code and configuration actually contain, by inspection, not by
assumption: languages present, build and dependency manifests, CI configuration, test
framework, deployment and infrastructure config. Name the files you read.

**Timeline.**

```bash
[ -s /tmp/direct.txt ] && git log --no-walk --stdin --format=%cs < /tmp/direct.txt \
  | cut -c1-7 | sort | uniq -c
```

Report direct commits per month, and PR-attributed AI commits by the month their PR
merged (from `mergedAt` in the section 2 script, not the commit date). Note
periods of no activity rather than smoothing over them.

## Output rules

- Every number in the report carries the command or script that produced it, inline or in
  an appendix.
- Round nothing in a way that cannot be reversed; give raw counts.
- Mark any figure you could not compute as "not computed" and say why. Do not substitute
  an estimate.
- Keep the person's claims defensible: this report may be read by a reviewer, a client, or
  a hiring manager who can re-run the commands.
