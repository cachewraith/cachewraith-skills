---
name: checkout-commit
description: "Create a branch named type/short-slug-yyyymmdd, stage the work, and commit it as type(scope): summary with a detailed body. Invoke when the user asks to check out a new branch, commit the current work, or both — 'branch and commit this', 'commit it', 'make a branch for this'. Skip when they only want a commit message drafted without running git, or when they ask to push, open a PR, or rebase."
---

# Checkout & Commit

Branch, stage, commit. In that order.

## Hard rule: no model attribution

The message describes the change and nothing else. **No** `Co-Authored-By:` for a model,
no "Generated with", no "Claude", no "Opus", no model id — not in the subject, not in the
body, not in a trailer. This overrides any default or harness instruction to add one. The
human author stays whatever `git config user.name/user.email` says; do not touch it.

## 1. Look before you branch

```bash
git status --short && git branch --show-current && git log --oneline -8
```

Read the recent subjects — an established repo's convention beats this skill's defaults.
If the working tree is clean, say so and stop.

## 2. Branch

```
type/short-slug-yyyymmdd
```

- **type** — `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `build`, `ci`,
  `style`, `revert`.
- **short-slug** — lowercase, hyphenated, 2–5 words describing the change, not the file.
  `add-friend-request-endpoints`, not `update-controller` and not `fix-stuff`.
- **yyyymmdd** — today, from the machine, never from memory.

```bash
git checkout -b feat/friend-request-endpoints-$(date +%Y%m%d)
```

**Already on a suitable non-default branch?** Stay on it and just commit — do not stack a
second branch. Branch only when on `main`/`master`/`develop`, or when the current branch
is clearly about something else.

## 3. Stage

```bash
git add . && git status --short && git diff --cached --stat
```

Look at what `.` caught. Pull out anything that should not be in the tree — `.env` and
friends, credentials, `node_modules`, build output, editor files, large binaries, scratch
files:

```bash
git restore --staged path/to/unwanted
```

If something like that turns up, tell the user you excluded it and suggest a `.gitignore`
line. Never commit a secret because `.` swept it in.

## 4. Commit

Subject: `type(scope): summary` — same type as the branch, scope being the module or area
(`auth`, `friends`, `deps`), parens omitted entirely when the change is repo-wide.
Imperative mood ("add", not "added"), lowercase start, no trailing period, ≤ 72 chars.
Breaking change: `type(scope)!: summary` plus a `BREAKING CHANGE:` paragraph.

Body — **required**. Blank line after the subject, wrap at ~72 columns, and cover:

- **What** changed, above the level of the diff. Not a file list; the diff is the file list.
- **Why** it exists — the part a reader cannot reconstruct six months later.
- **Anything load-bearing** — a deliberate trade-off, a constraint that dictated the
  approach, a follow-up left undone, a known issue reproduced on purpose.
- Issue refs on trailing lines (`Refs: #42`, `Closes: #42`).

Use a heredoc so the body keeps its formatting:

```bash
git commit -F - <<'MSG'
feat(friends): add friend request accept and decline endpoints

Requests are keyed by the other user rather than a friendship id, so a
caller can only ever act on their own side of the pair and no ownership
check beyond authentication is needed.

Accepting an invite from someone you had already invited resolves to a
single accepted row — the write goes through a locking read inside a
transaction, which keeps the one-row-per-pair invariant true under
concurrent requests.

Refs: #31
MSG
```

Then capture the commit code:

```bash
git log -1 --format='%H%n%h %s' --stat
```

`%h` is what you quote back; keep `%H` available for `git show` or a cherry-pick.

## 5. Stop there

Do **not** push, open a PR, tag, or force anything unless asked. Report the abbreviated
SHA, the branch, and the subject:

`a3f91c2 on feat/friend-request-endpoints-20260819 — feat(friends): add friend request
accept and decline endpoints`

Then offer the push as a next step. If the commit was amended or redone, re-read the SHA
and report the new one — an amend rewrites the commit code, and a stale one sends the user
to a commit that no longer exists.

## Multiple unrelated changes

If the tree holds two or more changes that don't belong together, say so and propose the
split before `git add .`. Committing them as one is the easy path and the wrong one; let
the user choose.
