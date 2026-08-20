# cachewraith-skills

Six skills for Claude Code that I use daily, packaged as an installable plugin.

Each one is opinionated about a single job: audit security against a checklist, find
performance problems that survive contact with load, use the project's own test runner,
commit with a consistent branch and message convention, and turn a list of endpoints
into documentation a frontend developer can build against.

## Install

```
/plugin marketplace add cachewraith/cachewraith-skills
/plugin install cachewraith-skills@cachewraith-skills
```

Restart Claude Code, and the six skills load automatically. Claude invokes them when a
request matches, or you can call one by name — `/check-security`, `/testing`, and so on.

To update later:

```
/plugin update cachewraith-skills
```

To remove:

```
/plugin uninstall cachewraith-skills
```

## The skills

| Skill | What it does |
|---|---|
| **check-security** | Walks the OWASP Top 10:2021 as a checklist and reports what is actually at risk. Every finding carries its category ID and a file:line, so it can be verified or refuted — `A01: updateProfile trusts the client-supplied userId` rather than "this looks insecure". Categories that don't apply are not padded in. |
| **check-performance** | Hunts the problems that show up under load: N+1 queries, unbounded result sets, missing indexes, blocking I/O on hot paths, accidental O(n²), unbatched network calls, memory retention. Reports evidence, not vibes. |
| **testing** | Finds the project's existing test setup before writing a line — runner, config, conventions — then runs, writes, or debugs tests in it. Works with Jest, Vitest, Pytest, PHPUnit, Go test, cargo test, RSpec, and JUnit. |
| **checkout-commit** | Branches as `type/short-slug-yyyymmdd`, stages, and commits as `type(scope): summary` with a real body. The message describes the change and nothing else — no model attribution, no "generated with" trailer. |
| **generate-api-docs** | Paste endpoints as `GET : https://api.example.com/...` and it investigates them for real: calls safe read-only methods, inspects actual responses and headers, and reads any OpenAPI/Swagger metadata it can find. Publishes frontend integration docs as an Artifact and hands back the link. Documented from observed behavior, never guesses. |
| **release-version** | Bumps the version everywhere it actually lives — manifest, lockfile, `__version__`, README badge — using the ecosystem's own tool so lockfiles stay in sync. Works across Node, Python, Rust, Go, PHP, Ruby, Java, .NET, Dart, Elixir, and Claude Code plugins, and greps for stragglers in anything it doesn't know. Picks the semver bump from the real diff since the last tag, writes the changelog entry in the format already there, tags, then pushes the commit and tag — stopping short of a registry publish. |

## Using them without installing

Any single skill works on its own — copy its directory into `~/.claude/skills/` for
personal use, or into a project's `.claude/skills/` to share it with a repo:

```bash
git clone https://github.com/cachewraith/cachewraith-skills.git
cp -r cachewraith-skills/skills/check-security ~/.claude/skills/
```

## Layout

```
.claude-plugin/
  marketplace.json    # makes the repo installable as a marketplace
  plugin.json         # the plugin manifest
skills/
  check-security/SKILL.md
  check-performance/SKILL.md
  testing/SKILL.md
  checkout-commit/SKILL.md
  generate-api-docs/SKILL.md
  release-version/SKILL.md
```

Validate any change before pushing:

```bash
claude plugin validate . --strict
claude plugin validate ./skills --strict
```

## Contributing

Issues and pull requests are welcome. If you add a skill, keep the frontmatter
`description` specific about **when to invoke and when to skip** — that text is the only
thing Claude sees when deciding whether the skill is relevant, so a vague description
means the skill never fires.

## License

MIT — see [LICENSE](LICENSE).
