---
name: check-performance
description: "Audit code for real performance problems — N+1 queries, unbounded result sets, missing indexes, blocking I/O on hot paths, accidental O(n^2), unbatched network calls, memory retention. Invoke when the user asks to check or audit performance, says something is slow, asks about optimizing queries or endpoints, or asks where the bottlenecks are. Publishes the audit as an Artifact and returns the link. Skip for pure correctness bugs, styling, or a full code review (use code-review)."
---

# Check Performance

Find the problems that would actually show up under load. Report evidence, not vibes.

## Scope

No target given → the working-tree diff if there is one, else the hot paths (request
handlers, jobs, loops over collections). A named file, directory, endpoint, or PR → just
that. State the scope in one line before the findings.

Detect the stack rather than assuming one:

```bash
ls package.json composer.json pyproject.toml Cargo.toml go.mod pom.xml 2>/dev/null
```

Then find the data layer (ORM, query builder, raw SQL) and the entry points (routes,
handlers, CLI commands, cron). Almost every real finding lives where those two meet.

## 1. Database access

Ranked by how often it matters in production.

- **N+1 queries** — a query inside a loop, or a lazy relation touched while iterating.
  The single most common real finding: look for a `for`/`map`/`foreach` whose body
  reaches a repository, model, or ORM client.
- **Unbounded result sets** — `findMany`/`SELECT` with no `LIMIT`, `take`, or pagination
  on a table that grows without bound.
- **Missing or unusable indexes** — a `WHERE`/`ORDER BY` on an unindexed column, or one
  wrapped in a function (`LOWER(name) LIKE …`) that defeats a B-tree. Read the
  schema/migrations before claiming an index is missing.
- **Over-fetching** — every column when the caller uses two; eager-loaded relations
  nobody reads.
- **Long-held locks** — queries inside a transaction that don't need to be, or
  `SELECT … FOR UPDATE` followed by slow work.
- **Repeated identical queries** in one request that should be hoisted or memoized.

## 2. Algorithmic complexity

Nested loops where both sides scale with input (O(n^2)) and a map/set lookup would do;
`includes`/`in`/`indexOf` against a large list inside a loop; sorting or rebuilding a
derived structure inside a loop instead of once outside; string concatenation in a hot
loop.

Only flag these when the collection can realistically be large. O(n^2) over a fixed
5-element config array is not a finding.

## 3. I/O and concurrency

- **Blocking sync I/O on a request path** — sync file reads, sync crypto, sync
  `child_process`, blocking HTTP in an async runtime.
- **Sequential awaits that are independent** and should be concurrent (`Promise.all`,
  `asyncio.gather`, goroutines + `WaitGroup`).
- **Unbatched external calls** — one round trip per item.
- **No timeout** on an outbound call, so one slow dependency stalls a worker pool.
- Work done inline that belongs in a background job (image processing, email, exports).

## 4. Caching and repeated work

Expensive pure computation recomputed per request with no cache; a cache with no TTL or
no invalidation path (a correctness risk as much as a performance one); config, compiled
regexes, or clients constructed per call instead of once at module or DI scope.

## 5. Memory and payload

Loading a whole table or file into memory where streaming would do; unbounded in-memory
maps/arrays that only grow (a leak); response payloads embedding large trees the client
discards; missing compression or pagination on a large listing endpoint.

## 6. Frontend (only if there is a UI)

Re-renders from unstable props (object/array/function literals recreated each render);
missing memoization on genuinely expensive subtrees, not blanket `memo`; large bundles
from a barrel import; layout thrash (reading a layout property and writing style in one
loop); unvirtualized lists rendering thousands of rows.

## Verify before reporting

A finding you cannot point at is noise. For each:

1. Cite `path/to/file.ts:123`.
2. Say **what the cost scales with** — "one query per friend, so 50 queries for a 50-item
   page", not "this could be slow".
3. Say what triggers it — which endpoint, job, or user action reaches the code.
4. Give the fix concretely, in this codebase's idiom.

Where profiling or query logging is available, use it rather than reasoning alone. Check
for an existing benchmark or load test before writing a new one.

## Do not

- Report micro-optimizations with no measurable effect (`++i` vs `i++`, `forEach` vs `for`
  on a 10-element array).
- Recommend a rewrite, a new framework, or a cache layer when a query fix would do.
- Call something slow without saying what makes it slow.
- Change code unless the user asked for fixes; this skill reports by default.

## Report

The deliverable is a published **Artifact link**, not findings pasted into the terminal.

In the document, group by severity, worst first:

- **Hot** — every request on a common path, or scales with data size.
- **Warm** — real but bounded, or on a less-travelled path.
- **Cold** — worth knowing, not worth stopping for.

Close with what you checked and found clean. If nothing real turned up, say so — do not
manufacture findings to fill the report.

### Publish

1. Load the `artifact-design` skill first — required for every Artifact, Markdown ones
   included. It also decides the format: Markdown suits a handful of findings, HTML earns
   its keep when there are many and grouping, anchors, or a before/after table help.
2. Write the file to the scratchpad with a stable basename derived from the target —
   `checkout-api-performance-audit.md`.
3. Call `Artifact` with a short noun-phrase `title` naming what was audited
   (`Checkout API Audit`, never `Performance Report`), a one-sentence `description`, and
   `favicon: ⚡` kept identical across redeploys.
4. Return the URL from the tool result. Never fabricate one.

Re-auditing the same target in the same session → call `Artifact` with the **same file
path** to redeploy to the same URL. An audit from an earlier session → recover the URL with
`action: "list"` or ask the user, then pass it as `url`. A different target → a new path.

If publishing fails, say so and fall back to a local file, reporting that path.

### Reply

Short: scope, the tally by severity, the worst offender named, then the link.

```text
Audited the order endpoints. 2 hot, 3 warm, 1 cold. The hot one is an N+1 in
`OrderController::index` — one query per line item, so ~200 queries on a full cart.

https://claude.ai/code/artifact/3e52452a-ac2b-45ed-9adf-0968da7288b6
```

Lead with the finding that costs the most — never make the user open the artifact to learn
what is hurting them now.
