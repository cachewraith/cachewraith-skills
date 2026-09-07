---
name: discuss
description: "Think a decision through with the user: scan the parts of the project the decision actually touches, look up anything that depends on the outside world, then come back with a position — the options that survive, the one you'd pick, and what would change your mind. Kept cheap on purpose: a scoped scan, at most a couple of searches, no subagents. Invoke when the user wants to weigh an approach, choose between options, or pressure-test an idea before building it. Skip when they want a direct factual answer (that is ask) or want the work done."
---

# Discuss

The user wants a thinking partner, not a report and not a build. Come back with an
opinion they can push against.

## 1. Find the real question first

Most "what do you think about X" questions hide a decision. Name it before spending
anything: *is this "which of these two", or "is this a good idea at all", or "what am I
not seeing"?* If the request is genuinely ambiguous and the readings lead somewhere
different, ask one question and stop — do not scan the project to answer both.

Write the decision down in a line to yourself. Everything below is scoped to it.

## 2. Scan what the decision touches

Not the project. The **part of the project the decision touches**.

Budget: **four or five tool calls**, before any web lookup.

```bash
rg -n 'PATTERN' -l                                # where does this concern live
sed -n '1,60p' path/to/the/file                   # read the one that matters
cat package.json 2>/dev/null | head -40           # what's already a dependency
git log --oneline -10                             # what direction is this moving
```

What you are looking for, in this order:

1. **What already exists** — the codebase may have solved this once already, and a second
   spelling of the same idea is a cost the user has not priced in.
2. **The local vocabulary** — the patterns, naming, and libraries already here. Consistency
   with them usually beats a marginally better idea.
3. **The constraints** — the runtime, the versions, the thing that cannot change.

Do not read the whole tree, do not enumerate every file, and do not spawn a subagent: a
subagent starts cold, re-derives context you already hold, and returns prose you then have
to re-read. For a discussion it is the expensive path with the worse result.

## 3. Look outward only when the answer lives outside

Search the web when — and only when — the decision turns on something you cannot see from
here and cannot safely hold in memory:

- how a library actually behaves in the version this project pins
- whether an approach is still the current one, or has a known failure mode
- a benchmark, a CVE, a deprecation, a breaking change

**Two searches, maximum.** Query the specific thing (`"<library> <version> <behavior>"`),
read the one result that answers it, stop. Do not open a page to confirm what its own
snippet already said. If the question turns on nothing external — a naming choice, a
structural call, a tradeoff inside code you have already read — skip this step entirely.
Most discussions do.

Anything you bring back from the web, say where it came from and how current it is.

## 4. Have the discussion

Structure, short:

- **Where you land, first.** One or two sentences. A recommendation, not a menu. "I'd do
  B" — then the reason.
- **The options that survive.** Two, occasionally three. Each gets the cost that actually
  decides it, not a symmetric pros/cons table. Options you rejected get one clause, not a
  paragraph.
- **What it costs.** The part that bites later: the migration, the coupling, the thing that
  gets hard at 10x, the second implementation of something already here.
- **What would change your mind.** Name the fact that flips the recommendation — "if these
  rows ever exceed a few thousand, A stops working". This is the most useful line in the
  whole answer, because it tells the user which of their facts matters.
- **One question back**, when one genuinely blocks the call. One, and only if it changes
  the answer.

Tone: a colleague with a view. Disagree with the premise when it is wrong — say so in a
sentence, then engage with what they are actually trying to do rather than stopping at the
objection. Do not hedge everything into "it depends"; when it does depend, say on what.

Ground it in what you read: `src/queue/worker.ts:88 already retries` is worth more than a
general principle about retries.

## 5. Stay a discussion

No files written, no commands with side effects, no artifact, no implementation — unless
the user asks. A code sketch is fine when it is the clearest way to show a difference; keep
it to a few lines in the message, not a file on disk.

End on the open thread, not a summary. The user came to think; leave them somewhere to
reply.
