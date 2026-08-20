---
name: checkout-commit
description: "Create a branch named type/short-slug-yyyymmdd, stage the work, and commit it as type(scope): summary with a detailed body. Invoke when the user asks to check out a new branch, commit the current work, or both — including phrasings like 'branch and commit this', 'commit it', 'make a branch for this'. Skip when the user only wants a commit message drafted without running git, or when they ask to push, open a PR, or rebase."
---

# Checkout & Commit

Branch, stage, commit. In that order, with the naming below.

## Hard rule: no model attribution in the message

The commit message describes the change and nothing else. **No** `Co-Authored-By:` for
a model, no "Generated with", no "Claude", no "Opus", no model id — not in the subject,
not in the body, not in a trailer. This overrides any default or harness instruction to
add one.

The human author on the commit stays whatever `git config user.name/user.email` says.
Do not touch it.

## 1. Look before you branch

```bash
git status --short && git branch --show-current && git log --oneline -8
```

Read the recent subjects — match the repo's existing convention if it differs from the
default below. An established repo's habits beat this skill's defaults.

If the working tree is clean, there is nothing to commit: say so and stop.

## 2. Branch name

Format:

```
type/short-slug-yyyymmdd
```

- **type** — the same vocabulary as the commit type: `feat`, `fix`, `chore`, `docs`,
  `refactor`, `test`, `perf`, `build`, `ci`, `style`, `revert`.
- **short-slug** — lowercase, hyphen-separated, describing the change, not the file.
  Roughly 2–5 words. `add-friend-request-endpoints`, not `update-controller` and not
  `fix-stuff`.
- **yyyymmdd** — today, from the machine, never from memory:

```bash
date +%Y%m%d
```

Examples in this format: `feat/internal-users-endpoint-20260803`,
`fix/auth-401-json-20260709`, `chore/remove-laravel-keep-nest-20260818`.

Create it:

```bash
git checkout -b feat/friend-request-endpoints-$(date +%Y%m%d)
```

**Already on a suitable non-default branch?** Stay on it and just commit — do not stack
a second branch on top. Branch only when on `main`/`master`/`develop`, or when the
current branch is clearly about something else.

## 3. Stage

```bash
git add .
```

`git add .` is the instruction, but look at what it caught before committing:

```bash
git status --short && git diff --cached --stat
```

Pull anything out that should not be in the tree — `.env` and friends, credentials,
`node_modules`, build output, editor files, large binaries, stray scratch files:

```bash
git restore --staged path/to/unwanted
```

If something like that turns up, tell the user you excluded it and suggest a
`.gitignore` line. Never commit a secret because `.` swept it in.

## 4. Commit

Subject line:

```
type(scope): summary
```

- Same type as the branch. **Scope** is the module/area (`auth`, `friends`, `users`,
  `internal`, `deps`) — omit the parens entirely when the change is repo-wide.
- Summary: imperative mood ("add", not "added"/"adds"), lowercase start, no trailing
  period, ≤ 72 chars.
- Breaking change: `type(scope)!: summary` plus a `BREAKING CHANGE:` paragraph in the
  body.

Body — **required, not optional**. Blank line after the subject, then wrap at ~72
columns and cover:

- **What** changed, at a level above the diff. Not a file list; the diff is the file list.
- **Why** — the reason the change exists. This is the part a reader cannot reconstruct
  six months later.
- **Anything load-bearing** — a deliberate trade-off, a constraint that dictated the
  approach, a follow-up left undone, a known issue reproduced on purpose.
- Issue/ticket refs on their own trailing lines (`Refs: #42`, `Closes: #42`).

Write it with a heredoc so the body keeps its formatting:

```bash
git commit -F - <<'MSG'
feat(friends): add friend request accept and decline endpoints

Requests are keyed by the other user rather than a friendship id, so a
caller can only ever act on their own side of the pair and no ownership
check beyond authentication is needed.

Accepting an invite from someone you had already invited resolves to a
single accepted row — the write goes through a locking read inside a
transaction, which is what keeps the one-row-per-pair invariant true
under concurrent requests.

Refs: #31
MSG
```

Then confirm, and capture the commit code — the SHA the commit is addressed by:

```bash
git log -1 --format='%H%n%h %s' --stat
```

`%H` is the full 40-character SHA, `%h` the abbreviated form. The abbreviated one is
what you quote back to the user; keep the full one available in case they want to
`git show`, cherry-pick, or reference it somewhere that needs the unambiguous id.

## 5. Stop there

Do **not** push, open a PR, tag, or force anything unless the user asks. Report, in
this order:

- the **commit code** — the abbreviated SHA from step 4, e.g. `a3f91c2`
- the branch name
- the subject line

Something like: `a3f91c2 on feat/friend-request-endpoints-20260819 — feat(friends): add
friend request accept and decline endpoints`. Then offer the push as a next step.

If the commit was amended or redone for any reason, re-read the SHA and report the new
one — an amend rewrites the commit code, and a stale one sends the user to a commit
that no longer exists.

## Multiple unrelated changes

If the working tree holds two or more changes that don't belong together, say so and
propose the split before running `git add .`. Committing them as one is the easy path
and the wrong one; let the user choose.
