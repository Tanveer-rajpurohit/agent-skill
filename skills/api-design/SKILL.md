---
name: api-design
description: >
  Design clean, consistent, evolvable HTTP/REST (and pragmatic GraphQL/gRPC) APIs.
  Trigger for: designing endpoints and resource models, choosing status codes,
  shaping request/response bodies, error response format, pagination, filtering,
  sorting, versioning, idempotency, rate limiting at the API layer, authn/authz
  (JWT, API keys, scopes), input validation, webhooks, and API contracts/OpenAPI.
  Also trigger for "design an API", "what status code", "how to paginate",
  "REST endpoint naming", "version my API", "API error format", "should this be
  POST or PUT". Focuses on contracts that are consistent and safe to evolve.
---

# API Design Skill — Consistent, Evolvable Contracts

You design APIs a client team would thank you for: **predictable, consistent,
hard to misuse, safe to evolve.** The contract is the product. Consistency across
endpoints matters more than any single clever choice.

## Resource modeling & naming

```
Nouns, plural, lowercase, hyphenated. Hierarchy via nesting (max ~2 levels).
  GET    /users
  POST   /users
  GET    /users/{id}
  PATCH  /users/{id}
  DELETE /users/{id}
  GET    /users/{id}/orders            // sub-resource
  GET    /orders?user_id={id}          // prefer flat + filter when not strictly owned

Actions that aren't CRUD → POST a sub-resource or a verb sub-path:
  POST /orders/{id}/cancel             // not GET /cancelOrder
  POST /users/{id}/password-reset
```
Rules: no verbs in resource paths (the HTTP method is the verb). Keep one casing
convention for JSON fields (`snake_case` or `camelCase`) and never mix. IDs are
opaque strings (UUIDs/ULIDs), not exposed auto-increment ints.

## Methods & status codes (use them precisely)

```
GET    safe, idempotent, cacheable, NO body          200
POST   create / non-idempotent action                201 (created, return Location) | 200
PUT    replace whole resource, idempotent            200 | 204
PATCH  partial update                                200 | 204
DELETE remove, idempotent                            204 (no body) | 200

2xx  200 OK · 201 Created · 202 Accepted (async) · 204 No Content
3xx  304 Not Modified (ETag)
4xx  400 Bad Request (malformed)      401 Unauthorized (not authenticated)
     403 Forbidden (authed, no access) 404 Not Found
     409 Conflict (dup / version)      422 Unprocessable (valid JSON, bad values)
     429 Too Many Requests (rate limit, + Retry-After)
5xx  500 Internal · 502/503/504 upstream/unavailable/timeout

Don't return 200 with {"error": ...}. The status code IS the signal. Clients,
proxies, and monitoring all depend on it.
```

## One error format, everywhere

```jsonc
// 422
{
  "error": {
    "code": "validation_failed",        // stable, machine-readable, snake_case
    "message": "Email is invalid.",     // human-readable, safe to show
    "details": [                         // optional, field-level
      { "field": "email", "code": "invalid_format" }
    ],
    "request_id": "req_01H...”           // echo for support/tracing
  }
}
```
Rules: same envelope on **every** error across the whole API. `code` is the
contract (clients switch on it); `message` can change freely. Never leak stack
traces, SQL, or internal paths to clients. Always include a `request_id` that also
appears in your logs.

## Request/response body design

```
- Responses return the full resource representation after writes (so client doesn't refetch).
- Wrap collections so you can add metadata without breaking clients:
    { "data": [...], "page": { "next_cursor": "...", "has_more": true } }
- Single resource: return the object directly OR { "data": {...} } — pick one, be consistent.
- Timestamps: ISO-8601 UTC strings ("2026-06-24T10:00:00Z"). Never epoch-vs-ISO mixed.
- Money: integer minor units (cents) + currency code. NEVER floats for money.
- Booleans named affirmatively (is_active, not is_not_disabled).
- Don't expose internal enums raw if they may change; map to stable API values.
```

## Pagination, filtering, sorting

```
Cursor pagination (default — stable under inserts, scales):
  GET /orders?limit=20&cursor=eyJpZCI6...
  → { "data": [...], "page": { "next_cursor": "...", "has_more": true } }
  cursor encodes the last sort key (e.g. base64 of created_at+id). Opaque to client.

Offset pagination (only for small/admin lists — skips/duplicates under writes):
  GET /orders?limit=20&offset=40

Filtering:  GET /orders?status=paid&created_after=2026-01-01
Sorting:    GET /orders?sort=-created_at   (- prefix = desc)
Cap limit (e.g. max 100) and validate — never let a client request unbounded rows.
```

## Idempotency (for unsafe, retryable operations)

```
POST /payments
Idempotency-Key: 9f8c...               // client-generated unique key per logical op

Server: first time → process, store {key → response}. Retry with same key →
return the stored response, do NOT reprocess. Prevents double-charge on network
retry. Essential for payments, order creation, any side-effecting POST.
```
(See `system-design` PATTERNS.md for the Redis-backed implementation.)

## Versioning & evolution

```
Version in the URL path: /v1/users  (simplest, most visible, cache-friendly).
Evolve WITHOUT a new version when changes are additive & backward-compatible:
  ✅ add a new optional field, add a new endpoint, add a new enum value (if clients tolerate)
  ❌ remove/rename a field, change a type, change status-code semantics, tighten validation
Breaking change → new version (/v2). Keep /v1 running with a deprecation timeline + Sunset header.
Be a conservative producer, liberal consumer (Postel): ignore unknown request fields,
never break on extra response fields.
```

## Auth & security at the API boundary

```
- AuthN: Bearer JWT (short-lived access + refresh) or API keys (server-to-server).
  Validate signature + expiry on EVERY request, at the gateway/middleware.
- AuthZ: check the caller may act on THIS resource (object-level), not just "is logged in".
  The classic hole (IDOR/BOLA): GET /users/{id} returning anyone's data — verify ownership/scope.
- Validate every input against a schema (zod/pydantic/struct tags) at the edge. Reject unknown→422.
- Rate limit per key/IP (429 + Retry-After). Set max body size. Set timeouts.
- HTTPS only, HSTS. CORS allowlist (no wildcard with credentials).
- Never put secrets/PII in URLs (they're logged). Use headers/body.
- Validate Content-Type; parse defensively; cap array/string sizes.
```

## Webhooks (if you emit events)

```
- POST to subscriber URL with a signed payload (HMAC over body + timestamp header).
- Subscriber verifies signature + rejects old timestamps (replay protection).
- At-least-once delivery → include an event id; consumers must be idempotent.
- Retry with exponential backoff; expose a redelivery endpoint + event log.
```

## REST vs GraphQL vs gRPC (pick deliberately)

```
REST   → default. Public APIs, CRUD, caching, broad client compatibility.
GraphQL→ many client-driven view shapes, avoid over/under-fetch, one flexible endpoint.
         Cost: caching, rate-limiting, query-depth/complexity limits are on YOU.
gRPC   → internal service-to-service, low latency, streaming, strict contracts (protobuf).
         Not browser-native without a proxy. Great inside a microservice mesh.
```

## Anti-patterns

```
❌ Verbs in paths (/getUser, /createOrder)        → method + noun
❌ 200 on errors with an error body               → real status code
❌ Inconsistent field casing/error shapes         → one convention, everywhere
❌ Offset pagination on large hot tables          → cursor
❌ Floats for money                               → integer minor units
❌ Auth check = "is logged in" only               → object-level authorization
❌ Breaking changes without a version bump         → additive or /v2
❌ Unbounded list endpoints (no limit cap)         → enforce a max
❌ Leaking internal errors/stack traces            → generic message + request_id + log internally
```

## Response format

1. Show the endpoint table (method, path, status codes) first.
2. Give example request + response JSON for the main cases incl. one error.
3. Call out idempotency/pagination/auth decisions explicitly.
4. Note the one evolution risk (what would break clients) and how you avoided it.
