---
name: generate-api-docs
description: "Generate frontend-ready API documentation from endpoint lines pasted after `/generate-api-docs`, especially forms such as `GET : https://api.example.com/...`, `POST : ...`, `PUT : ...`, `PATCH : ...`, and `DELETE : ...`. Treat each supplied URL as an API endpoint to investigate. For safe read-only methods such as GET, actively call/open the endpoint and inspect the real response, headers, status, and any available official OpenAPI/Swagger/API metadata to discover verified parameters and schemas. Produce frontend integration docs from observed or officially documented behavior, never from guesses. Publish the finished documentation as an Artifact and hand the user its `https://claude.ai/code/artifact/...` link as the result."
---

# Generate API Docs

Generate implementation-ready API documentation for frontend developers by investigating the API endpoints supplied after `/generate-api-docs`.

The deliverable is **an Artifact link**, not documentation pasted into the terminal. Publish the finished docs with the `Artifact` tool and return the resulting `https://claude.ai/code/artifact/<id>` URL.

## Expected Usage

The user normally invokes the skill like this:

```text
/generate-api-docs

GET : https://api.example.com/v1/scholarships
```

The user may provide several endpoints:

```text
/generate-api-docs

GET : https://api.example.com/v1/scholarships
POST : https://api.example.com/v1/applications
GET : https://api.example.com/v1/applications/{id}
```

Treat every HTTP method + URL line as an endpoint to investigate and document.

The URL is not merely text to reformat. It is the source to inspect whenever tools and access permit.

## Core Behavior

When the user provides:

```text
GET : <api_url>
```

actively inspect `<api_url>` before writing the documentation.

For GET endpoints:

1. Open or call the exact URL.
2. Record the actual HTTP status when available.
3. Inspect the response content type and useful response headers when available.
4. Inspect the response body, especially JSON structure.
5. Identify path parameters already visible in the URL or documented route template.
6. Identify query parameters only when they are present in the URL, exposed by official API metadata/docs, or otherwise directly verified.
7. Look for official OpenAPI, Swagger, schema, or API documentation associated with the endpoint when available.
8. Use those official sources to verify parameter names, types, required/optional state, defaults, enums, validation constraints, auth requirements, success responses, and documented errors.
9. Generate frontend-oriented API docs from the verified information.
10. Publish those docs as an Artifact and return the artifact link as the result.

Do not stop at simply rewriting the URL.

## Source Priority

Use evidence in this order:

1. The live response from the exact supplied endpoint.
2. Official OpenAPI or Swagger metadata for the same API.
3. Official API documentation for the same API/domain.
4. User-supplied notes, schemas, examples, headers, or credentials/context.
5. Project/backend source only when the user explicitly asks to inspect it or it is already the stated source for this request.

If sources conflict, prefer the live implementation for observed behavior and explicitly mention the mismatch with documentation.

## Web / HTTP Inspection Rules

Use available web, browser, or HTTP tools to inspect supplied public URLs.

Do not answer from memory when the endpoint can be checked directly.

For a full public URL:

```text
GET : https://api.example.com/v1/users
```

call/open that URL.

For a relative path:

```text
GET : /api/v1/users
```

use a base URL already provided in the same request or current context.

If no base URL is available, do not invent one. Document what can be determined and state that the live endpoint could not be called without a base URL.

If the endpoint is localhost, private-network-only, VPN-only, or otherwise inaccessible from available tools, do not pretend it was inspected.

State the access limitation and use any supplied schema/response details instead.

## Safe Request Policy

### GET, HEAD, OPTIONS

These methods may be called for documentation discovery when tools allow it.

For GET, inspect the real response.

HEAD or OPTIONS may be used when useful to verify metadata or allowed methods, but do not rely on them if the server does not support them.

### POST, PUT, PATCH, DELETE

Do not execute state-changing requests merely to discover documentation.

For these endpoints, inspect official OpenAPI/Swagger/API docs, supplied request/response examples, safe metadata, or explicitly provided backend source.

Only execute a state-changing request if the user explicitly asks for that request to be sent and supplies all required data.

Never invent a body just to probe an endpoint.

## Parameter Discovery

For every endpoint, determine parameters from verified evidence.

### Path Parameters

Document placeholders such as:

```text
/users/{id}
/users/:id
```

as path parameters.

Do not invent example IDs unless needed for a frontend example.

Use placeholders such as `<id>`.

### Query Parameters

A query parameter may be documented when at least one of these is true:

- It appears in the supplied URL.
- It appears in official OpenAPI/Swagger metadata.
- It appears in official API documentation.
- The server returns a clear validation/error message naming it.
- The user explicitly supplies it.

Do not guess query parameters from response field names or endpoint names.

For each verified query parameter, capture when available:

- Name
- Type
- Required or optional
- Default value
- Allowed enum values
- Minimum/maximum constraints
- Pagination meaning
- Search/filter/sort behavior
- Example value

### Request Body Parameters

For POST, PUT, PATCH, or body-carrying endpoints, document body fields only from verified schema/docs/user input/source.

Capture when available:

- Field name
- Type
- Required/optional
- Nullable status
- Enum values
- Validation constraints
- Nested object structure
- Array item structure
- File or multipart requirements

## Authentication Discovery

Document authentication only when verified.

Evidence may include:

- Official OpenAPI security definitions.
- Official API documentation.
- A live `401 Unauthorized` or `403 Forbidden` response.
- Authentication-related response headers.
- User-supplied auth instructions.

If the endpoint returns 401/403, report the observed status and any useful safe error detail.

Do not expose user tokens, API keys, cookies, passwords, or secrets in generated documentation.

Replace them with placeholders such as:

- `<access-token>`
- `<api-key>`
- `<session-cookie>`

## Response Inspection

When a GET endpoint returns JSON, inspect the complete visible structure and generate a response schema from it.

For each response field, determine when possible:

- Exact JSON key
- Primitive type
- Object shape
- Array item shape
- Nullability observed in the response
- Nested structures

Distinguish observed sample types from formally documented schema types.

Example:

```json
{
  "id": 10,
  "name": "Example",
  "active": true,
  "tags": ["one", "two"]
}
```

Generate:

```ts
export interface ApiResponse {
  id: number;
  name: string;
  active: boolean;
  tags: string[];
}
```

If an array is empty or a value does not reveal enough information to determine a safe type, do not invent its internal schema.

Use `unknown` and explain why when necessary.

## Evidence Labels

Internally distinguish three kinds of information:

- **Observed** — directly returned by the live endpoint.
- **Documented** — defined in official API/OpenAPI/Swagger documentation.
- **Inferred from sample** — a type or shape inferred only from an observed example value.

Do not present inferred details as guaranteed API contract.

When an important detail is uncertain, say so briefly in the generated docs.

## Input Parsing

Recognize:

```text
GET : https://api.example.com/users
POST : https://api.example.com/users
PUT : https://api.example.com/users/{id}
PATCH : https://api.example.com/users/{id}
DELETE : https://api.example.com/users/{id}
```

Also recognize:

```text
GET https://api.example.com/users
POST https://api.example.com/users
```

Support:

- GET
- POST
- PUT
- PATCH
- DELETE
- OPTIONS
- HEAD

Treat content below an endpoint declaration as optional extra context for that endpoint until the next HTTP method declaration.

Example:

```text
GET : https://api.example.com/users
Auth: bearer token
Use this endpoint for the user list page.
```

Use supplied context together with verified live/documented information.

## Investigation Workflow

For each endpoint:

### 1. Parse the endpoint

Extract:

- HTTP method
- Full URL or relative path
- Query string already present
- Path placeholders
- Notes supplied beneath it

### 2. Inspect safely

For GET, HEAD, and OPTIONS, call/open the endpoint when possible.

Capture:

- HTTP status
- Response content type
- Relevant headers
- Response body
- Redirect behavior when relevant
- Error payload when the request fails

### 3. Find official contract metadata

When useful and available, inspect official API/OpenAPI/Swagger material associated with the same API.

Use it to discover details that cannot be proven from one response sample, especially:

- Optional query parameters
- Required parameters
- Request body schema
- Enum values
- Validation rules
- Authentication scheme
- Alternate success responses
- Error responses

Do not brute-force paths, fuzz parameters, or perform broad security probing.

### 4. Build the frontend contract

Combine verified evidence into:

- Endpoint purpose
- Authentication
- Headers
- Path parameters
- Query parameters
- Request body
- Success status and response
- Error responses
- TypeScript request/response types
- Frontend request example

### 5. Quality Check

Before returning docs, verify:

- The endpoint URL exactly matches the user's input unless a documented route template is shown separately.
- Parameters are evidence-backed.
- Types are either documented or clearly inferred from observed values.
- No secrets are exposed.
- No state-changing request was sent without explicit user instruction.
- Missing details are not fabricated.

### 6. Publish and return the link

Publish the documentation as an Artifact and give the user its link.

See **Deliverable — Artifact Link** below.

## Deliverable — Artifact Link

The result of this skill is a published Artifact URL of the form:

```text
https://claude.ai/code/artifact/3e52452a-ac2b-45ed-9adf-0968da7288b6
```

Do not dump the full documentation into the chat response. Publish it, then hand back the link.

### Publishing steps

1. Load the `artifact-design` skill before writing the file. This is required for every Artifact, Markdown ones included.
2. Write the documentation to a file in the scratchpad directory, for example `api-docs.md` or `api-docs.html`.
   Use a stable, descriptive basename derived from the API, such as `example-api-docs.md`.
3. Call the `Artifact` tool with that `file_path`.
   - `title`: a short noun-phrase name for the API being documented, such as `Scholarships API` — not a summary, and not `API Documentation`.
   - `description`: one sentence naming the API and the endpoints covered.
   - `favicon`: a fitting emoji, and keep it identical across redeploys of the same docs.
4. Return the artifact URL from the tool result as the answer.

### Choosing Markdown or HTML

Let `artifact-design` decide, based on the deliverable rather than on speed.

- Markdown suits a straightforward endpoint reference that is mostly prose, tables, and code blocks.
- HTML is worth the effort when the docs cover many endpoints and benefit from navigation, anchored sections, or styled method badges.

Either way the content rules in this skill still apply: evidence-backed parameters, no invented fields, no exposed secrets.

### Updating existing docs

- Re-running the skill in the same session for the same API: call `Artifact` again with the **same file path** to redeploy to the same URL.
- The user asks to update docs published in an earlier session: recover the URL with `action: "list"`, or ask the user for the link, then pass it as `url` so the existing artifact is updated instead of a new link being created.
- Documenting a genuinely different API: use a new file path so it gets its own artifact.

### Response format

Keep the chat reply short. State what was verified, then give the link.

```text
Documented 3 endpoints against the live API. `GET /v1/scholarships` and
`GET /v1/applications/{id}` returned `200`; `POST /v1/applications` is
documented from the published OpenAPI schema and was not called.

https://claude.ai/code/artifact/3e52452a-ac2b-45ed-9adf-0968da7288b6
```

Call out anything the user needs to know that the docs alone would not surface — an endpoint that could not be reached, a documented contract that disagreed with live behavior, or auth that blocked verification.

If publishing the Artifact fails, say so plainly and fall back to writing the documentation to a local file, then report that path.

## Output Format

This section describes the content written into the Artifact file.

Use Markdown.

Start with:

```text
# API Documentation
```

For multiple endpoints, include an endpoint summary.

Example:

```text
## Endpoint Summary

| Method | Endpoint | Status | Auth | Purpose |
|---|---|---|---|---|
| GET | `/v1/users` | `200` | Not verified | Retrieve users |
```

Only include status/auth values that were actually observed or documented.

## Endpoint Template

Use sections that are useful for the endpoint.

Omit empty sections that add no value.

Example structure:

```text
## GET `https://api.example.com/v1/users`

### Overview

Short frontend-focused description.

### Live Verification

- Status: `200 OK`
- Content-Type: `application/json`
- Verified from: live endpoint

### Authentication

Bearer token required.

### Headers

| Header | Required | Description |
|---|---|---|
| Authorization | Yes | `Bearer <access-token>` |

### Path Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| id | string | Yes | Resource identifier |

### Query Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| page | number | No | 1 | Page number |

### Request Body

None

### Success Response

JSON response here.

### Response Type

TypeScript interface here.

### Error Responses

| Status | Meaning | Evidence |
|---|---|---|
| 401 | Authentication required | Observed |

### Frontend Example

TypeScript fetch example here.

### Notes

Important uncertainty, sample-only inference, pagination behavior, or frontend integration detail.
```

## TypeScript Rules

Generate frontend TypeScript types whenever the contract contains enough information.

- Preserve exact JSON keys.
- Prefer concrete types over `any`.
- Use `unknown` when a safe type cannot be established.
- Mark properties optional only when the contract/docs prove they are optional.
- Include `null` only when nullability is observed or documented.
- Preserve enum values exactly.
- Reuse shared types when several endpoints return the same documented structure.

## Frontend Request Examples

Prefer the HTTP client already requested by the user.

If none is specified, use TypeScript `fetch`.

For example:

```ts
const response = await fetch(
  "https://api.example.com/v1/users",
);

if (!response.ok) {
  throw new Error(`API request failed: ${response.status}`);
}

const data: UsersResponse = await response.json();
```

For GET requests, include verified query parameters when useful:

```ts
const params = new URLSearchParams({
  page: "1",
});

const response = await fetch(
  `https://api.example.com/v1/users?${params.toString()}`,
);
```

Do not include undocumented query parameters merely to make the example look complete.

## Failure and Access Cases

### Endpoint returns 404

Document the observed 404.

Do not invent a response contract.

### Endpoint returns 401 or 403

Document the auth-related response.

If official schema/docs are available, use them to document the protected endpoint without exposing credentials.

### Endpoint returns 500

Document the observed server error and avoid treating the error payload as the normal response schema.

### Endpoint is inaccessible

State that live verification was not possible and why, if known.

Continue only with official docs or user-supplied information.

### Response is HTML instead of API JSON

Do not pretend it is a JSON API response.

Identify what was actually returned and check whether the URL points to:

- API documentation
- Login page
- Error page
- Frontend route

## Never Do These

- Never invent parameters.
- Never invent request fields.
- Never invent status codes.
- Never invent auth requirements.
- Never infer query parameters solely from response fields.
- Never claim a URL was called when it was not.
- Never send POST/PUT/PATCH/DELETE requests just to probe an API.
- Never expose secrets from supplied headers or URLs.
- Never fuzz, brute-force, or security-scan the API as part of documentation generation.
- Never silently replace the user's endpoint with a different endpoint.
- Never paste the full documentation into the chat reply instead of publishing it and returning the artifact link.
- Never fabricate an artifact URL. The link must come from an actual `Artifact` tool result.

## Final Quality Standard

The finished documentation must tell a frontend developer what is actually known about the endpoint from live behavior and authoritative API metadata.

For:

```text
GET : <api_url>
```

the default behavior is:

**Investigate the endpoint first, generate the documentation, publish it as an Artifact, and return the link.**

Do not merely transform the user's line into Markdown, and do not leave the finished docs sitting in terminal scrollback.
