---
name: check-performance
description: "Audit a codebase for real performance problems — N+1 queries, unbounded result sets, missing indexes, blocking I/O on hot paths, accidental O(n^2), unbatched network calls, memory retention. Invoke when the user asks to check/review/audit performance, says something is slow, asks about optimizing queries or endpoints, or asks where the bottlenecks are. Skip for pure correctness bugs, styling, or when the user wants a full code review (use code-review instead)."
---

# Check Performance

Audit code for performance problems that would actually show up under load. Report
evidence, not vibes.

## Scope first

Ask nothing; infer the target from the request:

- "check performance" with no target → the working-tree diff if there is one, else the
  hot paths (request handlers, jobs, loops over collections).
- A named file, directory, endpoint, or PR → just that.

State the scope you chose in one line before reporting findings.

## Orient before reading code

Detect the stack rather than assuming one:

```bash
ls package.json composer.json pyproject.toml requirements.txt Cargo.toml go.mod pom.xml 2>/dev/null
```

Then find the data layer — an ORM, a query builder, raw SQL — and the entry points
(routes, controllers, handlers, CLI commands, cron jobs). Those two together are where
almost every real finding lives.

## What to look for

Walk these in order. They are ranked by how often they matter in production.

### 1. Database access

- **N+1 queries** — a query inside a loop, or a lazy relation touched while iterating a
  collection. This is the single most common real finding. Look for a `for`/`map`/
  `foreach` whose body reaches a repository, model, or ORM client.
- **Unbounded result sets** — a `findMany`/`SELECT` with no `LIMIT`, `take`, or
  pagination on a table that grows without bound.
- **Missing or unusable indexes** — a `WHERE`/`ORDER BY` on a column with no index, or
  one wrapped in a function (`LOWER(name) LIKE …`) that defeats a plain B-tree index.
  Check the schema/migrations before claiming an index is missing.
- **Over-fetching** — selecting every column when the caller uses two, or eager-loading
  relations nobody reads.
- **Queries inside a transaction that don't need to be**, holding locks longer than
  necessary. Flag long-held row locks (`SELECT … FOR UPDATE` followed by slow work).
- **Repeated identical queries** in one request that should be hoisted or memoized.

### 2. Algorithmic complexity

- Nested loops over collections that both scale with input (O(n^2)) where a map/set
  lookup would be O(n).
- `Array.includes` / `in` / `indexOf` against a large list inside a loop.
- Sorting or rebuilding a derived structure inside a loop instead of once outside it.
- Repeated string concatenation in a hot loop.

Only flag these when the collection can realistically be large. An O(n^2) over a
fixed 5-element config array is not a finding.

### 3. I/O and concurrency

- **Blocking synchronous I/O** on a request path — sync file reads, sync crypto,
  sync `child_process`, blocking HTTP in an async runtime.
- **Sequential awaits that are independent** and should be concurrent
  (`Promise.all`, `asyncio.gather`, goroutines + `WaitGroup`).
- **Unbatched external calls** — one HTTP/RPC round trip per item.
- **No timeout** on an outbound call, so one slow dependency stalls a worker pool.
- Work done inline that belongs in a background job (image processing, email, exports).

### 4. Caching and repeated work

- Expensive pure computation recomputed per request with no cache.
- A cache with no TTL or no invalidation path (a correctness risk as much as a
  performance one).
- Config, compiled regexes, or clients constructed per call instead of once at module
  or DI scope.

### 5. Memory and payload

- Loading an entire table/file into memory where streaming or chunking would do.
- Unbounded in-memory maps/arrays that only ever grow (a leak).
- Response payloads that embed large nested trees the client discards.
- Missing compression or pagination on a large listing endpoint.

### 6. Frontend (only if the codebase has a UI)

- Re-renders from unstable props — object/array/function literals recreated each render.
- Missing memoization on genuinely expensive subtrees (not blanket `memo` everywhere).
- Large bundles from a barrel import pulling in a whole library.
- Layout thrash: reading a layout property and writing style in the same loop.
- Unvirtualized lists rendering thousands of rows.

## Verify before reporting

A finding you cannot point at is noise. For each one:

1. Cite `path/to/file.ts:123`.
2. Say what the cost scales with — "one query per friend in the page, so 50 queries for
   a 50-item page", not "this could be slow".
3. Say what triggers it — which endpoint, job, or user action reaches this code.
4. Give the fix concretely, in this codebase's idiom.

Where the project has profiling or query-logging available, use it rather than reasoning
alone. Check for an existing benchmark or load-test suite before writing a new one.

## Report

Group by severity, worst first:

- **Hot** — hit on every request to a common path, or scales with data size.
- **Warm** — real but bounded, or on a less-travelled path.
- **Cold** — worth knowing, not worth stopping for.

For each: file:line, what scales, the trigger, the fix.

Close with what you checked and found clean, so the user knows the coverage. If nothing
real turned up, say so plainly — do not manufacture findings to fill the report.

## Do not

- Do not report micro-optimizations with no measurable effect (`++i` vs `i++`, swapping
  a `forEach` for a `for` on a 10-element array).
- Do not recommend a rewrite, a new framework, or a cache layer when a query fix would do.
- Do not flag something as slow without saying what makes it slow.
- Do not change code unless the user asked for fixes; this skill reports by default.
