---
name: check-security
description: "Audit a codebase against the OWASP Top Ten, naming the category ID for every finding. Invoke when the user asks to check/review/audit security, asks whether something is safe or exploitable, or when work touches authentication, authorization, sessions, input handling, data access, secrets, crypto, file uploads, outbound requests, dependencies, or error/log output. Skip for performance-only reviews (use check-performance) and for general correctness review (use code-review)."
---

# Check Security

Walk the OWASP Top Ten as a checklist and report what is actually at risk, with the
category ID attached so every finding is checkable.

## The rule that makes this useful

**Map findings to a category, or say plainly that they map to none.** A finding written
as "A01: `updateProfile` trusts the client-supplied `userId` instead of the session
subject — `src/profile/profile.controller.ts:44`" can be verified or refuted. "This
looks insecure" cannot.

Equally: **do not pad the report with categories that don't apply.** A review of a
CSS change should not carry an A10 section. Say which categories you checked and found
clean; that is the coverage statement, and it is short.

## Check the list is current

The Top Ten is versioned. The table below is **2021**. If a newer revision exists,
verify the current category IDs and names before leaning on it — the process holds
either way, but the numbering may have shifted. Say which version you audited against.

## Scope

- "check security" with no target → the working-tree diff if there is one, else the
  trust boundaries: every route/handler, every place user input is parsed, every place
  a secret or a credential is read.
- A named file, directory, endpoint, or PR → just that, plus whatever it calls into
  that carries the actual check.

State the scope in one line before the findings.

## The categories

### A01 — Broken Access Control

The most common source of real findings. For every handler that touches a resource:

- Does it derive the actor from the **session/token**, or from a client-supplied id,
  header, or body field? A request body naming its own `user_id` is the classic tell.
- Is the ownership check on the **server**, or only implied by what the UI renders?
- Can an id be swapped to reach another tenant's row (IDOR)? Sequential integer ids
  make this trivial to probe.
- Are admin/internal routes protected by more than obscurity or a path prefix?
- Does a list endpoint filter by the actor, or return everything and paginate?
- Is CORS permissive enough to let another origin ride the user's credentials?

Note the design that avoids the whole category: route keyed by the *other* party with
the actor taken from the token needs no extra check, because the caller cannot express
a request about someone else's side.

### A02 — Cryptographic Failures

- Passwords hashed with a **slow, salted** algorithm (bcrypt/scrypt/argon2), never
  MD5/SHA-1/SHA-256-bare. Check the cost factor is production-grade — test configs
  legitimately lower it, production configs must not.
- Secrets, tokens, PII in transit: TLS enforced, no downgrade path.
- At rest: is anything sensitive stored plaintext that shouldn't be?
- No homegrown crypto, no ECB mode, no static/reused IV, no `Math.random()` for
  anything security-bearing — use the CSPRNG.
- Comparisons of secrets/HMACs/API keys done in **constant time**, not `==`.
- Keys and secrets from env/secret manager, never committed. Grep for committed
  credentials.

### A03 — Injection

- SQL/NoSQL built by **parameterized query or tagged template**, never string
  concatenation or interpolation of user input. Raw SQL is fine — unparameterized
  raw SQL is not.
- LIKE patterns: is `%` / `_` in user input neutralized, or does a search for `50%`
  match everything?
- OS commands: no shell interpolation of user input; use argv-array APIs.
- Output encoding for XSS — `dangerouslySetInnerHTML`, `v-html`, `|raw`, direct DOM
  `innerHTML` with user data.
- Path traversal on any user-influenced file path (`../`).
- Template, LDAP, XPath, and header injection where those are in play.
- Prompt injection where untrusted text reaches an LLM that can call tools.

### A04 — Insecure Design

- **Rate limits** on auth, OTP/verification, password reset, search, and anything
  expensive. Check the key: per-user when authenticated, per-IP otherwise — and check
  guard/middleware **order**, since an auth-after-throttle chain silently keys
  everything by IP.
- Business logic that assumes a cooperative client: negative quantities, replayed
  requests, races between check and write.
- Missing lock/transaction where two concurrent requests could both win.
- Enumeration: does a "user not found" path differ observably from "wrong password" —
  in body, status, timing, *or* in a throttle/cooldown response?
- Does an object get created before the operation that can fail, leaving junk behind?

### A05 — Security Misconfiguration

- Debug mode, stack traces, or verbose errors reachable in production.
- Default or sample credentials still present.
- CORS `*` combined with credentials; overly broad allowed origins/methods/headers.
- Security headers where a browser is involved (CSP, HSTS, `X-Content-Type-Options`,
  frame-ancestors).
- Storage buckets, directories, or admin endpoints exposed by default.
- Dependency and framework defaults left at "convenient" rather than "safe".

### A06 — Vulnerable and Outdated Components

- Run the ecosystem's audit tool and report only what is reachable from this code:
  `npm audit`, `pip-audit`, `composer audit`, `cargo audit`, `govulncheck`.
- Unpinned or floating versions on anything security-bearing.
- Abandoned packages doing crypto, parsing, or auth.

Report the CVE and whether the vulnerable path is actually used. An advisory in a
dev-only transitive dep is not the same as one in the auth path — say which it is.

### A07 — Identification and Authentication Failures

- Token lifetime, expiry, and rotation; is there any revocation path (a blacklist,
  a version claim, a session store)?
- Are logout and refresh actually invalidating the old credential?
- Signature verification: algorithm pinned, `alg: none` rejected, secret not weak.
- Lockout / backoff on repeated failures; credential-stuffing left open.
- Session fixation, cookie flags (`HttpOnly`, `Secure`, `SameSite`).
- Are OTP/reset codes hashed, single-use, expiring, and attempt-limited?
- Is a verification step actually enforced on the login path, not just at signup?

### A08 — Software and Data Integrity Failures

- Insecure deserialization of untrusted data (`pickle`, PHP `unserialize`,
  Java native, YAML unsafe load).
- Unsigned or unverified updates, plugins, or downloaded artifacts.
- CI/CD pulling unpinned actions/images; secrets exposed to untrusted PR workflows.
- Webhooks accepted without signature verification.

### A09 — Security Logging and Monitoring Failures

- Are auth failures, access-control denials, and privilege changes logged at all?
- Do logs leak passwords, tokens, full card/PII, or session ids?
- Do error responses expose internal class names, file paths, SQL, or stack traces
  to the client?
- Is there anything an operator could alert on, or does everything fail silently?

### A10 — Server-Side Request Forgery

- Any server-side fetch of a user-supplied URL — webhooks, avatar-by-URL, importers,
  PDF/screenshot renderers, OAuth discovery.
- Is there an **allowlist** (scheme + host), or only a blocklist?
- Are redirects followed without re-validating the destination?
- Is cloud metadata (`169.254.169.254`) and RFC1918 space blocked, including via
  DNS rebinding?

## Beyond the ten

The checklist is the floor, not the ceiling. A threat specific to this system still
matters when it maps to no category — race conditions on a domain invariant, a
multi-tenant leak through a shared cache key, a privacy problem in what a resource
serializer exposes to which viewer. Report those in their own section.

## Verify before reporting

For every finding:

1. **Cite** `path/to/file.ts:123`.
2. **Category ID** and name.
3. **The exploit path** — concrete: who sends what, and what they get that they
   shouldn't. If you cannot write that sentence, you have a code smell, not a finding.
4. **The fix**, in this codebase's idiom.
5. **Severity** — exploitable now / needs a precondition / defence-in-depth.

Read the surrounding code before calling something vulnerable. A check one layer up —
in a guard, a middleware, a base class, a policy — is still a check. Confirm it is
absent rather than assuming it from the handler body alone.

Check `MIGRATION.md`, `README.md`, `CLAUDE.md`, or `SECURITY.md` before reporting:
some behaviour is a **documented, deliberate** trade-off. Reproducing a known issue on
purpose for wire compatibility is a decision, not an oversight — reference it and move
on rather than re-reporting it as new.

## Report

Findings first, worst first, each in the form above. Then one short line naming the
categories you checked and found clean. Then anything you could not assess and why
(no access to prod config, a dependency you cannot read).

Do not change code unless the user asked for fixes. Do not write or run an exploit
against infrastructure you have not been authorized to test.
