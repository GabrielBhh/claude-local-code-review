# API Design Rules Reference

## REST API

### HTTP Verb Semantics

| Issue | Violation | Fix |
|---|---|---|
| GET with side effects | `GET /users/delete?id=5` | `DELETE /users/5` |
| POST for idempotent read | `POST /search` when no body mutation | `GET /search?q=...` |
| PUT vs PATCH misuse | `PUT /users/5` with partial body | `PATCH /users/5` for partial update; `PUT` requires full replacement |
| Missing `405 Method Not Allowed` | Endpoint silently ignores wrong verb | Return `405` with `Allow` header |

### Status Codes

| Issue | Violation | Fix |
|---|---|---|
| 200 for created resource | `POST /users` returns `200 OK` | Return `201 Created` with `Location` header |
| 200 for validation error | Returns `200` with `{"error": "invalid"}` | `422 Unprocessable Entity` or `400 Bad Request` |
| 500 for client mistakes | Unvalidated input causes 500 | Validate input, return `400`/`422` |
| 200 for not-found | Returns `200` with empty body/null | `404 Not Found` |
| 403 vs 401 confusion | Unauthenticated request gets `403 Forbidden` | Unauthenticated → `401 Unauthorized`; authenticated but unauthorized → `403 Forbidden` |

### Response Shape

| Issue | Violation | Fix |
|---|---|---|
| Inconsistent error envelope | Some errors: `{"error": "..."}`, others: `{"message": "..."}` | Standardise: `{"error": {"code": "...", "message": "..."}}` |
| Exposing internal details | Stack traces or DB errors in response body | Log server-side; return generic message to client |
| Missing pagination on list endpoints | `GET /users` returns unbounded array | Add `limit`/`offset` or cursor-based pagination; include total count or `next` cursor |
| Returning 204 with body | `204 No Content` with JSON body | Strip body on 204 |
| Sensitive fields in response | `password_hash`, `internal_id`, PII returned unnecessarily | Use a response DTO / serializer that explicitly allowlists fields |

### Versioning & Backwards Compatibility

| Issue | Fix |
|---|---|
| No versioning strategy | Add `/v1/` prefix or `Accept: application/vnd.api+json;version=1` header |
| Breaking change in existing endpoint | Add new field or version; never remove/rename fields in a minor release |

### Authentication & Authorization

| Issue | Fix |
|---|---|
| Endpoint missing auth middleware | All non-public endpoints must require a valid token/session |
| IDOR: user-supplied ID not verified | `GET /orders/42` — verify `order.user_id == current_user.id` before returning |
| Admin-only endpoint without role check | Check role/permission, not just authentication |

---

## GraphQL

| Issue | Violation | Fix |
|---|---|---|
| N+1 queries | Resolver calls `db.find(user.id)` for each item in a list | Use DataLoader (batch + cache per request) |
| Unbounded query depth | No depth/complexity limit on schema | Add `graphql-depth-limit` / complexity analysis middleware |
| Missing field-level authorization | Any authenticated user can query `user { secretField }` | Enforce field-level permissions in resolvers or directives |
| Raw DB / internal errors returned | Exception message surfaced in `errors[].message` | Catch and return opaque error codes; log internally |
| `__schema` / introspection enabled in production | Schema fully exposed to all callers | Disable introspection in production environments |
| Mutations without idempotency key | Retry on network error causes duplicate writes | Accept a client-supplied `idempotencyKey` on mutations |

---

## gRPC / Protobuf

| Issue | Violation | Fix |
|---|---|---|
| Missing deadline on client call | `stub.DoThing(req)` with no timeout | `stub.withDeadlineAfter(5, TimeUnit.SECONDS).DoThing(req)` |
| Ignoring status code on client | Treating all non-exception responses as success | Check `StatusRuntimeException` and `response.getStatus()` |
| Streaming without cancellation | Server stream read in infinite loop | Use `context.cancel()` / `ClientCall.cancel()` when done |
| Sensitive data in proto field names | Field named `password`, `secret_key` in logged messages | Annotate with `[(google.api.field_behavior) = SENSITIVE]` or exclude from logs |
| Breaking proto change | Removing or renumbering a field | Never remove field numbers; deprecate with `reserved` instead |
