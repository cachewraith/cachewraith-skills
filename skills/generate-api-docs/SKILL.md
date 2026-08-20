---
name: generate-api-docs
description: "Investigate the API endpoints pasted after `/generate-api-docs` — lines such as `GET : https://api.example.com/v1/users` — and publish frontend-ready documentation as an Artifact, returning its link. Calls read-only endpoints for real and reads any official OpenAPI/Swagger metadata, documenting only verified behaviour. Invoke when the user supplies endpoints to document. Skip when they want an endpoint merely called, or docs written from local source alone."
---

# Generate API Docs

Investigate each endpoint the user pastes, then publish the documentation as an Artifact
and hand back its `https://claude.ai/code/artifact/<id>` link.

Two failure modes define this skill: reformatting the user's URL into Markdown without
inspecting anything, and leaving finished docs in terminal scrollback. Avoid both.

## Input

Any line carrying an HTTP method and a URL is an endpoint to document — `GET : <url>`
and `GET <url>` both count. Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS, HEAD.
Text below an endpoint is context for it until the next method line:

```text
GET : https://api.example.com/v1/users
Auth: bearer token
Used by the user list page.
```

A relative path (`GET : /api/v1/users`) needs a base URL from the same request or the
current context. Never invent one — document what you can and say the live call was not
possible.

## Investigate

**GET, HEAD, OPTIONS** — call the real URL. Record status, content type, useful headers,
body, redirects, and any error payload. Do not answer from memory when the endpoint can
be checked directly.

**POST, PUT, PATCH, DELETE** — never send one to discover documentation. Work from
OpenAPI/Swagger metadata, official docs, supplied examples, or backend source the user
pointed at. Send one only if the user explicitly asks and supplies the data.

Never fuzz, brute-force paths, or security-probe an API as part of documenting it.

Evidence, in order of authority:

1. Live response from the exact supplied endpoint
2. Official OpenAPI/Swagger metadata for the same API
3. Official API documentation
4. User-supplied notes, schemas, examples, headers
5. Project source — only when the user asks, or it is already the stated source

When live behaviour and documentation disagree, document the live behaviour and note the
mismatch. When an endpoint is unreachable (localhost, VPN, private network), say so
plainly and continue from docs alone — never claim a URL was called when it was not.

## What counts as verified

| Label | Means |
|---|---|
| Observed | Returned by the live endpoint |
| Documented | Defined in official API/OpenAPI/Swagger material |
| Inferred from sample | A type or shape read off one example value |

Mark inferred details as inferred; they are not a guaranteed contract.

**Document a query parameter only when** it appears in the supplied URL, in official
metadata or docs, in a server validation error naming it, or the user supplied it.
Never infer one from a response field or from the endpoint's name. Body fields likewise
come only from schema, docs, user input, or source.

Never invent parameters, request fields, status codes, or auth requirements. Never
silently replace the user's endpoint with a different one.

Auth is documented only from a security definition, official docs, an observed 401/403,
auth-related headers, or the user's own instructions.

## Secrets

Replace every token, key, cookie, or password from supplied headers and URLs with a
placeholder — `<access-token>`, `<api-key>`, `<session-cookie>`. Nothing secret reaches
the published artifact.

## TypeScript

Generate types whenever the contract carries enough information.

- Exact JSON keys; concrete types over `any`; `unknown` where nothing safe can be
  established, with a one-line reason.
- Optional only where the contract proves it, `null` only where observed or documented.
- Enum values preserved exactly; shared shapes reused across endpoints.
- Empty array or ambiguous value → do not invent the item schema.

## Document content

Markdown, headed `# API Documentation`, with a summary table when there are several
endpoints. Per endpoint, include only the sections that carry something:

```text
## GET `https://api.example.com/v1/users`

### Overview
One or two lines, written for the frontend caller.

### Live Verification
- Status: `200 OK`
- Content-Type: `application/json`
- Verified from: live endpoint

### Authentication / Headers / Path Parameters / Query Parameters / Request Body
Tables. Query params carry type, required, default, description.

### Success Response
Real JSON, secrets replaced.

### Response Type
TypeScript interface.

### Error Responses
| Status | Meaning | Evidence |

### Frontend Example
`fetch` in TypeScript, or the client the user asked for.

### Notes
Uncertainty, sample-only inference, pagination behaviour.
```

Do not include undocumented query parameters in an example just to make it look
complete.

## Publish

1. Load the `artifact-design` skill first — required for every Artifact, Markdown ones
   included. It also decides Markdown vs HTML: Markdown for a straightforward reference,
   HTML when many endpoints benefit from navigation and styled method badges.
2. Write the file to the scratchpad with a stable, descriptive basename
   (`scholarships-api-docs.md`).
3. Call `Artifact` with a short noun-phrase `title` naming the API (`Scholarships API`,
   never `API Documentation`), a one-sentence `description`, and a `favicon` kept
   identical across redeploys.
4. Return the URL from the tool result. Never fabricate one.

Updating: same session, same API → call `Artifact` with the **same file path**. Docs from
an earlier session → recover the URL with `action: "list"` or ask the user, then pass it
as `url`. A genuinely different API → a new file path.

If publishing fails, say so and fall back to a local file, reporting that path.

## Reply

Short. What was verified, then the link:

```text
Documented 3 endpoints against the live API. `GET /v1/scholarships` and
`GET /v1/applications/{id}` returned `200`; `POST /v1/applications` is documented
from the published OpenAPI schema and was not called.

https://claude.ai/code/artifact/3e52452a-ac2b-45ed-9adf-0968da7288b6
```

Call out what the docs alone would not surface — an unreachable endpoint, a contract that
disagreed with live behaviour, auth that blocked verification. Never paste the full
documentation into the chat instead of publishing it.
