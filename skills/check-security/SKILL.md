---
name: check-security
description: "Audit code against the OWASP Top Ten, naming the category ID for every finding. Invoke when the user asks to check, review, or audit security, asks whether something is safe or exploitable, or when work touches auth, sessions, input handling, data access, secrets, crypto, file uploads, outbound requests, dependencies, or log output. Publishes the audit as an Artifact and returns the link. Skip for performance-only reviews (use check-performance) and general correctness review (use code-review)."
---

# Check Security

Walk the Top Ten as a checklist and report what is actually at risk, with the category ID
attached so every finding is checkable.

**Map each finding to a category, or say plainly it maps to none.** "A01: `updateProfile`
trusts the client-supplied `userId` instead of the session subject —
`src/profile/profile.controller.ts:44`" can be verified or refuted; "this looks insecure"
cannot. Equally, **do not pad the report with categories that don't apply** — a CSS change
gets no A10 section.

The list is versioned; the table below is **2021**. If a newer revision exists, check the
current IDs before leaning on it and say which version you audited against.

## Scope

No target given → the working-tree diff if there is one, else the trust boundaries: every
route/handler, every place user input is parsed, every place a secret is read. A named
file, directory, endpoint, or PR → just that, plus what it calls into that carries the
actual check. State the scope in one line before the findings.

## A01 — Broken Access Control

Most real findings live here. Per handler:

- Is the actor derived from the **session/token**, or from a client-supplied id, header,
  or body field? A body naming its own `user_id` is the classic tell.
- Is the ownership check on the **server**, or only implied by what the UI renders?
- Can an id be swapped to reach another tenant's row (IDOR)? Sequential ids make it
  trivial to probe.
- Are admin/internal routes protected by more than a path prefix?
- Does a list endpoint filter by actor, or return everything and paginate?
- Is CORS permissive enough to let another origin ride the user's credentials?

The design that avoids the category: a route keyed by the *other* party, actor taken from
the token, needs no extra check — the caller cannot express a request about someone else.

## A02 — Cryptographic Failures

- Passwords hashed **slow and salted** (bcrypt/scrypt/argon2), never bare MD5/SHA. Check
  the cost factor is production-grade — test configs legitimately lower it.
- Secrets, tokens, PII: TLS enforced with no downgrade path; nothing sensitive stored
  plaintext.
- No homegrown crypto, no ECB, no static/reused IV, no `Math.random()` for anything
  security-bearing — use the CSPRNG.
- Secrets/HMACs/API keys compared in **constant time**, not `==`.
- Keys from env or a secret manager, never committed. Grep for committed credentials.

## A03 — Injection

- SQL/NoSQL **parameterized or tagged-template**, never concatenated user input. Raw SQL
  is fine; unparameterized raw SQL is not.
- LIKE patterns: is `%`/`_` in user input neutralized, or does a search for `50%` match
  everything?
- OS commands: argv-array APIs, no shell interpolation.
- XSS via `dangerouslySetInnerHTML`, `v-html`, `|raw`, direct `innerHTML`.
- Path traversal (`../`) on any user-influenced path.
- Template, LDAP, XPath, header injection where in play. Prompt injection where untrusted
  text reaches an LLM that can call tools.

## A04 — Insecure Design

- **Rate limits** on auth, OTP, password reset, search, anything expensive. Check the key
  (per-user when authenticated, per-IP otherwise) and the guard **order** — auth after
  throttle silently keys everything by IP.
- Logic assuming a cooperative client: negative quantities, replayed requests, races
  between check and write.
- Missing lock/transaction where two concurrent requests could both win.
- Enumeration: does "user not found" differ observably from "wrong password" — in body,
  status, timing, *or* in a throttle response?
- Is an object created before the operation that can fail, leaving junk behind?

## A05 — Security Misconfiguration

Debug mode, stack traces, or verbose errors reachable in production; default or sample
credentials; CORS `*` with credentials; missing security headers where a browser is
involved (CSP, HSTS, `X-Content-Type-Options`, frame-ancestors); buckets, directories, or
admin endpoints exposed by default; framework defaults left at "convenient".

## A06 — Vulnerable and Outdated Components

Run the ecosystem's audit tool — `npm audit`, `pip-audit`, `composer audit`,
`cargo audit`, `govulncheck` — and report only what is reachable from this code. Flag
unpinned versions on anything security-bearing, and abandoned packages doing crypto,
parsing, or auth. Report the CVE **and** whether the vulnerable path is actually used; a
dev-only transitive dep is not the auth path.

## A07 — Identification and Authentication Failures

- Token lifetime, expiry, rotation; is there any revocation path (blacklist, version
  claim, session store)?
- Do logout and refresh actually invalidate the old credential?
- Signature verification: algorithm pinned, `alg: none` rejected, secret not weak.
- Lockout/backoff on repeated failures; credential stuffing left open.
- Session fixation; cookie flags (`HttpOnly`, `Secure`, `SameSite`).
- OTP/reset codes hashed, single-use, expiring, attempt-limited.
- Is verification enforced on the login path, not just at signup?

## A08 — Software and Data Integrity Failures

Insecure deserialization of untrusted data (`pickle`, PHP `unserialize`, Java native,
unsafe YAML load); unsigned or unverified updates, plugins, downloaded artifacts; CI/CD
pulling unpinned actions or images, or exposing secrets to untrusted PR workflows;
webhooks accepted without signature verification.

## A09 — Security Logging and Monitoring Failures

Are auth failures, access-control denials, and privilege changes logged at all? Do logs
leak passwords, tokens, PII, or session ids? Do error responses expose internal class
names, paths, SQL, or stack traces to the client? Is there anything an operator could
alert on, or does everything fail silently?

## A10 — Server-Side Request Forgery

Any server-side fetch of a user-supplied URL — webhooks, avatar-by-URL, importers,
PDF/screenshot renderers, OAuth discovery. Is there an **allowlist** (scheme + host) or
only a blocklist? Are redirects followed without re-validating the destination? Is cloud
metadata (`169.254.169.254`) and RFC1918 space blocked, including via DNS rebinding?

## Beyond the ten

The checklist is the floor. A threat specific to this system still matters when it maps to
no category — a race on a domain invariant, a multi-tenant leak through a shared cache
key, a serializer exposing the wrong fields to the wrong viewer. Report those separately.

## Verify before reporting

Each finding carries: `path/to/file.ts:123`, the **category ID**, the **exploit path**
(who sends what, and what they get that they shouldn't), the **fix** in this codebase's
idiom, and **severity** — exploitable now / needs a precondition / defence-in-depth. If
you cannot write the exploit sentence, you have a smell, not a finding.

Read the surrounding code first. A check one layer up — guard, middleware, base class,
policy — is still a check; confirm it is absent rather than assuming from the handler
body. Check `README.md`, `CLAUDE.md`, `SECURITY.md`, or `MIGRATION.md` too: some
behaviour is a **documented, deliberate** trade-off, and reproducing a known issue for
wire compatibility is a decision, not an oversight.

## Report

The deliverable is a published **Artifact link**, not findings pasted into the terminal.

In the document: findings first, worst first, each in the form above. Then one line naming
the categories checked and found clean — that is the coverage statement. Then anything you
could not assess and why.

**Never put a live secret in the artifact.** If you find a committed credential, cite
`path:line` and describe it — never reproduce the value. Same for real tokens, keys, or
PII lifted from fixtures or logs.

### Publish

1. Load the `artifact-design` skill first — required for every Artifact, Markdown ones
   included. It also decides the format: Markdown suits a handful of findings, HTML earns
   its keep when there are many and severity badges and anchored navigation help.
2. Write the file to the scratchpad with a stable basename derived from the target —
   `auth-service-security-audit.md`.
3. Call `Artifact` with a short noun-phrase `title` naming what was audited
   (`Auth Service Audit`, never `Security Report`), a one-sentence `description`, and
   `favicon: 🔒` kept identical across redeploys.
4. Return the URL from the tool result. Never fabricate one.

Re-auditing the same target in the same session → call `Artifact` with the **same file
path** to redeploy to the same URL. An audit from an earlier session → recover the URL with
`action: "list"` or ask the user, then pass it as `url`. A different target → a new path.

If publishing fails, say so and fall back to a local file, reporting that path.

### Reply

Short: scope, the tally by severity, the one thing that needs acting on now, then the link.

```text
Audited the working-tree diff against OWASP 2021. 2 exploitable now (A01, A03),
1 needing a precondition (A07). The A01 IDOR on `GET /orders/:id` is reachable
unauthenticated.

https://claude.ai/code/artifact/3e52452a-ac2b-45ed-9adf-0968da7288b6
```

Lead with the worst finding — never make the user open the artifact to discover something
is exploitable right now. If nothing turned up, say so plainly and still publish the
coverage statement, so the audit is on record.

Do not change code unless the user asked for fixes. Do not run an exploit against
infrastructure you have not been authorized to test.
