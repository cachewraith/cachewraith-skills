---
name: ask
description: "Answer a question directly, after checking the real source of truth for it — the code, the config, the command output — instead of answering from memory. Deliberately cheap: a narrow lookup, then a short answer with the file:line that backs it. Invoke when the user asks a question about this project, this machine, or how something works and wants an answer rather than work. Skip when they want something built, changed, or committed, and skip when the question is open-ended enough to need a back-and-forth — that is discuss."
---

# Ask

Answer the question. Nothing else.

Two rules carry this skill, and they pull against each other on purpose: **check before
you answer**, and **spend as little as possible doing it**.

## 1. What kind of question is this

Decide before touching a tool:

| Question | What to do |
|---|---|
| The answer lives in this repo (what does X do, where is Y, is Z wired up) | Look it up. One targeted search, read only the lines that matter. |
| The answer lives on the machine (versions, branch, what is installed, why a command failed) | Run the one command that shows it. |
| General knowledge you actually hold, with no project-specific part | Answer straight, no tools. |
| The answer changed recently or depends on the outside world (a library's current API, a release, a CVE) | Say you'd have to look it up, and do one search — not five. |

Never answer a repo question from memory or from the file tree's shape. If you have not
seen the line, you do not know it.

## 2. Look, narrowly

Budget: **three tool calls.** Four or five when the question genuinely has parts. If you
are past that and still guessing, stop and say what you would need to check next — a wrong
answer delivered confidently costs the user more than a short "I'd need to look at X".

Cheap moves, in order of preference:

```bash
rg -n 'PATTERN' --glob '!node_modules' -m 20        # locate, don't dump
rg -n 'PATTERN' -l                                  # just the filenames first
sed -n '40,80p' path/to/file                        # read the range, not the file
git log --oneline -5 -- path/to/file                # when the question is "why"
```

Do **not**: read whole files when a range answers it, run a broad `find` over the tree,
open five files to confirm what the first one already said, or spawn a subagent. A subagent
starts cold and re-derives everything you already have — for one question it is the most
expensive way to get the cheapest answer.

Stop searching the moment you can answer. Confirmation beyond that is spending.

## 3. Answer

Lead with the answer. First sentence, no preamble.

- **Length matches the question.** A yes/no question gets a yes or no plus the reason. A
  "where is X" gets `file:line`. A "how does X work" gets a short paragraph or a few
  bullets — not a tour of the module.
- **Cite what you checked** — `src/auth/session.ts:42` — so the user can verify in one
  click. One or two citations, the load-bearing ones, not a bibliography.
- **Separate seen from inferred.** "The handler validates the token at `auth.ts:31`" is
  something you read. "So an expired token would 401 here" is inference — mark it as such
  when it matters.
- **Say when you don't know.** "Not in this repo — it's set by the deploy env" is a real
  answer. Guessing is not.

Do not add: a restatement of the question, a description of how you searched, a summary of
the answer you just gave, unrequested suggestions, or an offer to implement it. If the
answer implies obvious next work, one short line at the end is the whole allowance.

## 4. Do not act

This skill reads. It does not edit, create, commit, install, or run anything with a side
effect. If the answer is "that's broken", say it's broken and stop — the user asked what,
not fix it. They will ask for the fix if they want it.
