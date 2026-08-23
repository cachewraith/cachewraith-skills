---
name: add-designpattern
description: "Write a standing design-pattern instruction into the project's CLAUDE.md, so before building any non-trivial unit Claude weighs the candidate patterns against the shape of the problem, names the one it picked and why, and defaults to the simplest construct when none earns its place. Invoke when the user asks to add, apply, or enforce design patterns in a project, or wants Claude to consider patterns before writing code. Skip when they want an existing design reviewed or refactored — that is code-review or simplify."
---

# Add Design Pattern

One job: install a durable instruction. This skill configures the project; it does not
refactor it.

The instruction is written to bite in both directions — it makes Claude consider the
catalog *and* stops it from reaching for a pattern where a function would do. A block that
only said "use design patterns" would make the codebase worse.

## 1. Pick the target file and check for duplication

Default `./CLAUDE.md`, so the rule travels with the repo. `~/.claude/CLAUDE.md` only if
the user wants it everywhere.

```bash
grep -l 'design-patterns:start' CLAUDE.md ~/.claude/CLAUDE.md 2>/dev/null
```

Already present? Say so and ask before adding a second copy.

## 2. Read what the project already uses

```bash
git grep -lE 'Factory|Strategy|Repository|Builder|Adapter|Observer|Decorator' -- src app lib 2>/dev/null | head
```

If the codebase has an established vocabulary, say so in your report — the block's
"match what is already here" rule then has something concrete to match.

## 3. Write the block

```bash
F=CLAUDE.md
[ -f $F ] && sed -i '/<!-- design-patterns:start -->/,/<!-- design-patterns:end -->/d' $F
cat >> $F <<'MD'
<!-- design-patterns:start -->
## Design patterns: choose deliberately

Before writing any non-trivial unit — a new class, module, or a branch point that will
grow — state in one or two lines: the shape of the problem, the candidate patterns, the
one chosen, and why. Not an essay, and not silence either.

Match the problem shape, not the pattern name:

| The problem | Candidates |
|---|---|
| Construction is conditional, or the concrete type varies | Factory Method, Abstract Factory |
| An object needs many optional parts, or must be built step by step | Builder |
| One instance must be shared | container-scoped singleton — never a static global |
| Two incompatible interfaces must meet | Adapter, Bridge |
| Behavior must be added without touching the original | Decorator, Proxy |
| A subsystem needs one simple entry point | Facade |
| Part and whole must be treated alike | Composite |
| One algorithm, several interchangeable variants | Strategy |
| The steps are fixed, the details vary | Template Method |
| Something must react to change elsewhere | Observer, Mediator |
| An action must be queued, logged, or undone | Command, Memento |
| Behavior depends on which state the object is in | State |
| An input passes through ordered, optional handlers | Chain of Responsibility |
| Persistence must be swappable or testable | Repository, Unit of Work |
| A failure path must be explicit, not thrown | Result / Either, Null Object |
| A remote dependency can fail or stall | Circuit Breaker, Retry with backoff |
| Reads and writes have diverging models | CQRS |

Rules
- **The simplest construct that works, wins.** A function, a plain class, a language
  feature, or a `match` beats a pattern. A pattern earns its place when there are already
  two real variants, or a known axis of change — never on one hypothetical future one.
- **Write the language's idiom, not the 1994 diagram.** A first-class function is Strategy
  in most languages; a decorator, a context manager, an enum with behavior, or a
  discriminated union may be the local spelling. Do not build an interface hierarchy the
  language does not need.
- **Match the vocabulary already in this codebase** over introducing a new one. Consistency
  beats a marginally better fit.
- **Name it where it lands** — class or module name, or one line of doc — so the next
  reader sees the pattern without inferring it.
- **Do not retrofit** patterns into working code that nobody asked you to change.
- Say when a pattern was considered and rejected, and why. That is a design decision worth
  one line in the commit body or the PR.
<!-- design-patterns:end -->
MD
```

The block lands at the end; anything already in the file stays put. Creates `CLAUDE.md` if
it does not exist.

## 4. Report

Name the file, whether it was created or updated, and any pattern vocabulary you found
already in use. Say that it takes effect next session, since `CLAUDE.md` loads at session
start. Then stop — no refactor, no commit.
