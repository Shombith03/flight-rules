---
name: stage
description: Commit only the files the current task was meant to change. Use whenever work will end in a git commit, or when asked to commit, stage, "commit this", "commit my changes", "don't commit everything", "only commit X", or to review what is about to be committed. Keeps a ledger of files the session wrote, reconciles it against the working tree, screens staged paths for secrets, generated output and binaries, and reports what was left out.
---

# stage

Commit only what this task was meant to change. Everything else stays unstaged and is reported to the user by name.

## The forbidden commands

Never run any of these. There is no exception and no "just this once":

```
git add -A
git add .
git add *
git add --all
git add -u
git add :/
git commit -a
git commit --all
git commit -am "..."
```

Also never `git stash -u` to "clean up" before committing, and never `git checkout .` or `git restore .` to discard unexpected changes. Unexpected changes are reported, not erased.

Stage only by explicit path, one path per argument, always after `--`:

```bash
git add -- "src/parser.ts" "docs/my notes.md"
```

If a tool, alias, or script would stage by wildcard on your behalf, do not use it.

## Where the ledger lives

The ledger files live inside the repository's git directory, so they never show up in `git status` and can never be committed by accident:

```bash
LEDGER_DIR="$(git rev-parse --absolute-git-dir)/stage"
```

Shell variables do not survive between separate shell invocations. Put that line at the top of every command block below that uses `$LEDGER_DIR`.

## Listing changed paths

Several steps need the list of paths git sees as changed or untracked. Always build it this way:

```bash
git status --porcelain=v1 -z --untracked-files=all | \
  while IFS= read -r -d '' entry; do
    printf '%s\n' "${entry:3}"
    # With -z, a rename or copy is followed by its original path as a separate field.
    case "${entry:0:2}" in
      R?|C?|?R|?C) IFS= read -r -d '' orig; printf '%s\n' "$orig" ;;
    esac
  done | LC_ALL=C sort -u
```

- `-z` keeps paths with spaces, quotes and non-ASCII characters intact. Both sides of a rename are listed as paths.
- `--untracked-files=all` lists each new file. Without it, git reports a new directory as `dir/` and it will never match the ledger.
- `LC_ALL=C` makes `sort` and `comm` agree on ordering.

Below, this pipeline is called **list-changed**.

## 1. Record intent before touching anything

At the start of the task, before the first edit:

```bash
LEDGER_DIR="$(git rev-parse --absolute-git-dir)/stage"
mkdir -p "$LEDGER_DIR"
: > "$LEDGER_DIR/ledger.txt"
git rev-parse --abbrev-ref HEAD
# list-changed, written to the pre-existing file:
git status --porcelain=v1 -z --untracked-files=all | \
  while IFS= read -r -d '' entry; do
    printf '%s\n' "${entry:3}"
    case "${entry:0:2}" in
      R?|C?|?R|?C) IFS= read -r -d '' orig; printf '%s\n' "$orig" ;;
    esac
  done | LC_ALL=C sort -u > "$LEDGER_DIR/pre-existing.txt"
```

Write down, in your own working notes:

- **Expected paths**: the files or directories you believe this task will change, derived from the user's request. Be specific; a directory is acceptable when the task genuinely spans one.
- **Pre-existing dirty state**: every path already modified, staged, or untracked at task start, from the command above.

Pre-existing changes are not yours. Never stage them as part of this task. If one of them sits in a file you must also edit, say so before you edit it and ask how to proceed.

## 2. Keep a ledger of what you write

Every time you create, edit, move, or delete a file, append its path to the ledger. For a move, append both the old and the new path. One path per line, verbatim, no quoting or escaping, so paths with spaces survive:

```bash
LEDGER_DIR="$(git rev-parse --absolute-git-dir)/stage"
printf '%s\n' "src/components/date picker.tsx" >> "$LEDGER_DIR/ledger.txt"
```

The ledger is the authority on what you changed. Do not reconstruct it afterwards from the diff — that defeats the purpose.

## 3. Reconcile before committing

Build the actual change set and compare it to the ledger.

```bash
LEDGER_DIR="$(git rev-parse --absolute-git-dir)/stage"

# list-changed, written to the actual file:
git status --porcelain=v1 -z --untracked-files=all | \
  while IFS= read -r -d '' entry; do
    printf '%s\n' "${entry:3}"
    case "${entry:0:2}" in
      R?|C?|?R|?C) IFS= read -r -d '' orig; printf '%s\n' "$orig" ;;
    esac
  done | LC_ALL=C sort -u > "$LEDGER_DIR/actual.txt"

LC_ALL=C sort -u "$LEDGER_DIR/ledger.txt" > "$LEDGER_DIR/ledger.sorted"

# Changed on disk but never written by this session:
LC_ALL=C comm -13 "$LEDGER_DIR/ledger.sorted" "$LEDGER_DIR/actual.txt" > "$LEDGER_DIR/unexpected.txt"

# ...of which the user's own prior work:
LC_ALL=C comm -12 "$LEDGER_DIR/unexpected.txt" "$LEDGER_DIR/pre-existing.txt"

# ...and what appeared during this task without being written by you:
LC_ALL=C comm -23 "$LEDGER_DIR/unexpected.txt" "$LEDGER_DIR/pre-existing.txt"
```

Git prints paths relative to the repository root, so record ledger paths relative to the root too.

For each path in the unexpected list:

1. If it is in `pre-existing.txt`, it is the user's own prior work; leave it alone and mention it once.
2. For what remains, state the path and one line on why it is plausibly there. Common causes: a build or test run regenerated it; a formatter or linter ran on save; a package manager rewrote a lockfile; an editor wrote local config; a debug edit was left in.
3. Leave it unstaged. Do not stage it because it "looks related".

Also run the reverse check — ledger entries that show no change on disk:

```bash
LEDGER_DIR="$(git rev-parse --absolute-git-dir)/stage"
LC_ALL=C comm -23 "$LEDGER_DIR/ledger.sorted" "$LEDGER_DIR/actual.txt"
```

These mean an edit was reverted, overwritten, or never saved. Investigate before committing.

### When a needed change falls outside the expected paths

This happens legitimately: a fix requires touching a shared helper, a type definition, a config key. Do not silently include it, and do not drop it. Stop and tell the user:

> The fix also requires editing `src/shared/http client.ts` — the retry option this task needs does not exist there yet. That path was outside the expected set. Include it in this commit, commit it separately, or leave it out?

Then follow the answer.

## 4. Screen what is about to be committed

Screening happens in two passes. Each check is a stop-and-ask, not an automatic exclusion — except secrets, which are a refusal. Nothing is committed until both passes are clean.

### Pass 1: paths, before staging

Run these checks on your candidate path list.

**Secret-shaped paths — refuse to stage.** Do not stage a path matching any of:

```
.env  .env.*  *.pem  *.key  *.p12  *.pfx  id_rsa  id_rsa.*  id_ed25519*
credentials.json  service-account*.json  *.keystore  .npmrc  .netrc
*secret*  *credential*
```

**Generated and build output.** Flag paths under or matching, unless the task was explicitly about them:

```
dist/  build/  out/  target/  bin/  obj/  coverage/  node_modules/  .next/  .cache/
*.min.js  *.min.css  *.map  *.o  *.pyc  __pycache__/  *.log
```

If such a path is tracked and keeps reappearing, suggest a `.gitignore` entry rather than committing it.

**Lockfile churn.** `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Cargo.lock`, `poetry.lock`, `go.sum`, `Gemfile.lock`, `composer.lock`. If the task did not involve adding or upgrading a dependency, a changed lockfile is almost certainly an incidental install. Report it and leave it out unless told otherwise.

**Editor and local config.** `.vscode/`, `.idea/`, `*.swp`, `.DS_Store`, `.direnv/`, local `*.local.*` overrides. Leave out.

### Pass 2: content, after staging

Content checks need the staged content, because new files have no `git diff` until they are staged. Stage the paths that survived pass 1, then check them. Always pass the candidate paths to these commands; the index may also hold changes the user staged before the task.

```bash
git add -- "path one" "path two"
```

**Secrets in content — refuse to commit.** Check the added lines:

```bash
git diff --cached -U0 -- "path one" "path two" | grep '^+' | grep -nEi \
  'AKIA[0-9A-Z]{16}|gh[pousr]_[A-Za-z0-9]{20,}|sk-[A-Za-z0-9]{20,}|-----BEGIN [A-Z ]*PRIVATE KEY-----|(api[_-]?key|secret|token|passwd|password).{0,20}[:=].{0,4}[A-Za-z0-9/+_-]{16,}'
```

Any hit: unstage the path (`git restore --staged -- "<path>"`), report it, and ask. Never commit it and fix it later — the value is in history from that point on.

**Large binaries.** Flag any staged file over ~1 MB or without a text diff:

```bash
git diff --cached --numstat -z -- "path one" "path two"
# a "-\t-\t" pair marks a binary file
git diff --cached --name-only -z --diff-filter=d -- "path one" "path two" | xargs -0 -I{} du -h -- "{}"
```

Report size and path; ask before committing.

**Debug leftovers.** Grep your own staged diff for what you added while exploring:

```bash
git diff --cached -- "path one" "path two" | grep -nE '^\+.*(console\.log|debugger|printf\("DEBUG|binding\.pry|pdb\.set_trace|TODO: remove|XXX|FIXME: temp)'
```

Remove them, re-stage the affected paths by name, and re-run pass 2.

## 5. Commit, report

Run from the repository root, since the paths are root-relative. Name the paths on the commit itself:

```bash
git diff --cached --stat -- "src/parser.ts" "src/parser.test.ts"
git commit -m "Fix off-by-one in header offset parsing" -- "src/parser.ts" "src/parser.test.ts"
git show --stat HEAD   # confirm exactly the screened paths went in
```

A plain `git commit` commits everything in the index, including anything the user had already staged before the task started. `git commit -- <paths>` commits only the named paths and leaves the rest of the index as it was.

Then report, every time, in this shape:

```
Committed (3 files):
  src/parser.ts
  src/parser.test.ts
  docs/changelog.md

Left unstaged, not part of this task:
  dist/bundle.min.js   — regenerated by the test run
  package-lock.json    — npm install side effect; no dependency change was requested
  .vscode/settings.json — editor-local, was already modified before this task started

Refused:
  .env.local — contains an API-key-shaped value

Nothing above was discarded; all of it is still in your working tree.
```

Keep it factual. No path appears in the report without a reason.

## Dry-run mode

If the user says `--dry-run`, "show me the plan", "what would you commit", or "don't commit yet": do steps 1 through 4 and the report, and stop. Do not run `git add`, `git commit`, or `git push`.

Without staging, run pass 2 against the working tree instead. Tracked files go through `git diff`; untracked files have no diff, so grep the files themselves:

```bash
git diff -U0 -- "<tracked candidate paths>" | grep '^+' | grep -nEi '<patterns above>'
grep -nEi '<patterns above>' -- "<untracked candidate paths>"
```

End with the exact commands you would run, so the user can inspect or run them:

```
Would run:
  git add -- "src/parser.ts" "src/parser.test.ts"
  git commit -m "Fix off-by-one in header offset parsing" -- "src/parser.ts" "src/parser.test.ts"
```

## Path handling

Paths with spaces, quotes, or non-ASCII characters are normal and must work.

- Always `--` before paths, always one shell-quoted argument per path.
- Parse git output only in `-z` / NUL-delimited form (`git status --porcelain=v1 -z`, `git diff --name-only -z`), read with `read -r -d ''` or `xargs -0`. Never pipe newline-delimited git output into `for` loops or word splitting.
- `git -c core.quotePath=false status --porcelain` gives readable non-ASCII output for showing to the user. Never parse it.
- Never build a path list by string concatenation into a single `git add` string.
- The ledger is one path per line, so a path containing a newline cannot be recorded. If you meet one, stop and ask rather than staging it.

## Edge cases

- **Nothing in the ledger changed on disk.** Report it and commit nothing.
- **Partial-file changes.** If only part of a file belongs to this task, do not reach for `git add -p` interactively. Tell the user which hunks are in scope and ask whether to split the file's change or commit it whole.
- **Deletions.** `git add -- "<path>"` stages a deletion correctly. Do not use `git rm -r`.
- **Renames.** After a plain `mv`, `git add -- "<old>" "<new>"` stages the rename. After `git mv`, both sides are already staged: do not `git add` the old path. Git rejects it, and a rejected path makes git stage none of the paths in that call. Do name both sides on `git commit --`.
- **Submodules.** A changed submodule pointer is never incidental and never implicit. Report it and ask.
- **Amending.** Apply the same screening to `git commit --amend`, and never amend a commit that is already pushed unless the user asks.
- **Merge or rebase in progress.** Stop. Report the state and ask; do not stage into it.
