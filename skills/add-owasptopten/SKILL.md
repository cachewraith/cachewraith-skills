---
name: add-owasptopten
description: "Write a standing OWASP Top Ten instruction into the project's CLAUDE.md, so every later change that touches auth, sessions, input handling, data access, secrets, crypto, uploads, outbound requests, dependencies, or logging is checked against the category list before it ships. Invoke when the user asks to add, apply, or enforce OWASP rules in a project, or wants Claude to follow the Top Ten from now on. Skip when they want an actual security audit of existing code — that is check-security."
---

# Add OWASP Top Ten

One job: install a durable instruction. This skill configures the project; it does not
audit it.

## 1. Pick the target file

Default is `./CLAUDE.md` — the rule then applies to this repo and travels with it. Write
to `~/.claude/CLAUDE.md` only if the user asks for it everywhere.

Check first whether the rule is already in force:

```bash
grep -l 'owasp-top-ten:start' CLAUDE.md ~/.claude/CLAUDE.md 2>/dev/null
grep -ni 'owasp' CLAUDE.md ~/.claude/CLAUDE.md 2>/dev/null | head
```

Already in the global file? Say so and ask before adding a second copy — two OWASP
sections in context is pure duplication. Hand-written OWASP prose already there? Offer to
replace it with the managed block rather than stacking both.

## 2. Confirm the list is current

The table below is **OWASP Top 10:2021**. The list is periodically revised. If the model's
knowledge or a quick check says a newer edition exists, use its IDs and names instead —
the process is the same, only the labels move. Note the edition you wrote in your report.

## 3. Write the block

Managed markers, replaced in place so re-running never duplicates it:

```bash
F=CLAUDE.md
[ -f $F ] && sed -i '/<!-- owasp-top-ten:start -->/,/<!-- owasp-top-ten:end -->/d' $F
cat >> $F <<'MD'
<!-- owasp-top-ten:start -->
## Security: follow the OWASP Top Ten

Before calling any work done that touches authentication, authorization, sessions, input
handling, data access, secrets, crypto, file uploads, outbound requests, dependencies, or
error/log output, check it against the categories below. This is a checklist, not
background reading.

Apply it in both directions:

- **Writing code** — pick the design that avoids the category up front (parameterized
  queries, server-side authorization checks, allowlists for outbound URLs) rather than
  bolting a mitigation on afterward.
- **Reviewing code** — name the categories actually at risk and cite the ID with a
  `file:line`, so the finding is checkable: "A01: this handler trusts a client-supplied
  `userId`", never a vague "this looks insecure". Do not pad a review with categories that
  do not apply, and do not report a real concern as generic advice when it maps to one.

| ID | Category | Recurring failure mode |
|---|---|---|
| A01 | Broken Access Control | Access authorized from client-supplied IDs, or checked only in the UI |
| A02 | Cryptographic Failures | Secrets or PII unprotected in transit/at rest; homegrown crypto; fast hashes for passwords |
| A03 | Injection | SQL/NoSQL/OS/LDAP built by string concatenation; unescaped output (XSS) |
| A04 | Insecure Design | No rate limits, no trust boundary, logic that assumes a cooperative client |
| A05 | Security Misconfiguration | Debug mode on, permissive CORS, default credentials, missing headers, verbose errors |
| A06 | Vulnerable and Outdated Components | Known-CVE dependencies, unpinned or unaudited transitive deps |
| A07 | Identification and Authentication Failures | Weak session/token handling, no lockout, tokens that never expire or rotate |
| A08 | Software and Data Integrity Failures | Unsigned updates, insecure deserialization, untrusted CI/CD or plugin sources |
| A09 | Security Logging and Monitoring Failures | Security events unlogged, or logs leaking secrets/PII/stack traces |
| A10 | Server-Side Request Forgery | Server fetches a user-supplied URL without an allowlist |

The list is the floor, not the ceiling: a threat specific to this system still matters when
it maps to none of the ten.
<!-- owasp-top-ten:end -->
MD
```

The block lands at the end of the file; everything the user already wrote stays put. If
`CLAUDE.md` does not exist, this creates it.

## 4. Tell the user what changed

Name the file, whether it was created or updated, and the edition you wrote (`2021`, or
newer). Mention that it takes effect from the next session, since `CLAUDE.md` is loaded at
session start — and that `/check-security` is the skill for auditing what is already
there. Then stop — no commit.
