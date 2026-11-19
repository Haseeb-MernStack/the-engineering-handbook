# API Design

> An API is a contract between teams that you cannot easily take back, so its shape, error behavior, and evolution rules deserve more care than the code behind it.

## Overview

An API is the surface other people build on. Once a client depends on a field name, a status code, or an ordering guarantee, that detail is part of your contract whether you documented it or not. Internal code can be refactored in an afternoon. A public or widely consumed API can take years to change.

Good API design is therefore mostly about three things:

- **Clarity.** A client developer can predict behavior from the shape of the interface.
- **Safety.** Retries, partial failures, and concurrent writes do not corrupt data.
- **Evolvability.** You can add capabilities without breaking existing consumers.

The style you choose (REST, gRPC, GraphQL) matters less than the discipline you apply within it. A well-designed REST API beats a sloppy GraphQL schema, and the reverse is also true.

## Core Concepts

### Resource-oriented design (REST-style HTTP)

REST-style APIs model nouns, not verbs. Resources have stable identifiers, and HTTP methods express intent.

| Method | Meaning | Safe | Idempotent |
|--------|---------|------|------------|
| GET | Read a resource | Yes | Yes |
| POST | Create or trigger processing | No | No (by default) |
| PUT | Replace a resource | No | Yes |
| PATCH | Partially modify a resource | No | Depends on the patch format |
| DELETE | Remove a resource | No | Yes |

"Safe" means the request has no side effects the client is responsible for. "Idempotent" means that sending the request N times leaves the server in the same state as sending it once. The definitions are in RFC 9110.

```http
POST /v1/orders HTTP/1.1
Host: api.example.com
Content-Type: application/json
Idempotency-Key: 7c1f0a52-9d3e-4b6e-8a41-2f5d0b9e6c11

{"customer_id": "cus_123", "items": [{"sku": "A-100", "qty": 2}]}
```

```http
HTTP/1.1 201 Created
Location: /v1/orders/ord_9981
Content-Type: application/json

{"id": "ord_9981", "status": "pending", "items": [{"sku": "A-100", "qty": 2}]}
```

### RPC with gRPC

gRPC defines services and messages in a Protocol Buffers schema and generates client and server code. It runs over HTTP/2 and supports streaming in both directions.

```proto
service Orders {
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  rpc WatchOrder(WatchOrderRequest) returns (stream OrderEvent);
}
```

The schema is the contract. Field numbers, not names, identify fields on the wire, so you must never reuse or renumber them.

### GraphQL

GraphQL exposes a typed graph. The client asks for exactly the fields it needs in one request. The server resolves each field with its own function.

```graphql
query {
  order(id: "ord_9981") {
    status
    items { sku qty }
  }
}
```

### Comparing the styles

| Concern | REST over HTTP | gRPC | GraphQL |
|---------|----------------|------|---------|
| Primary fit | Public APIs, CRUD-heavy domains | Internal service-to-service calls | Many client types with varied data needs |
| Contract | OpenAPI (optional but recommended) | Protobuf (required) | Schema (required) |
| Caching | Native HTTP caching | Application-level | Hard at the HTTP layer; usually application-level |
| Streaming | Awkward (SSE, WebSocket) | First-class | Subscriptions, implementation-specific |
| Browser access | Direct | Needs a proxy or gRPC-Web | Direct |
| Typical risk | Chatty clients, inconsistent conventions | Tight coupling to generated stubs | Expensive queries, N+1 resolvers |

### Pagination

Never return unbounded collections. Two common strategies:

- **Offset/limit.** `?limit=50&offset=100`. Simple. Skips rows, so results shift if data is inserted or deleted during paging. Deep offsets get slow because the database still walks the skipped rows.
- **Cursor (keyset).** `?limit=50&after=eyJpZCI6MTAwfQ`. The cursor encodes the position of the last item seen. Stable under concurrent writes and efficient on indexed columns. The cost is that clients cannot jump to page N.

Treat cursors as opaque. If clients parse them, you cannot change them.

### Idempotency

Networks fail after the server has acted but before the client has seen the response. The client then retries. Without protection, a retried `POST /payments` charges the customer twice.

The common pattern is an idempotency key:

1. The client generates a unique key per logical operation and sends it in a header.
2. The server stores the key with the result of the first execution.
3. A repeat request with the same key returns the stored result and does not run again.
4. A repeat with the same key but a different body is rejected as a client error.

The key check and the business write must be atomic, usually in one database transaction or protected by a unique constraint.

### Versioning and compatibility

The best versioning strategy is to avoid needing a new version. Prefer additive changes.

| Change | Compatible? |
|--------|-------------|
| Add an optional request field | Yes |
| Add a response field | Yes, if clients ignore unknown fields |
| Add a new enum value | Risky; clients with exhaustive switches may fail |
| Remove or rename a field | No |
| Change a field's type or meaning | No |
| Tighten validation | No |

When you must break compatibility, you can version in the URL (`/v2/`), in a header, or through media types. URL versioning is the most visible and easiest to route. Whatever you choose, publish a deprecation policy, announce the sunset date, and measure who still calls the old version before you remove it.

### Errors

Return a consistent, machine-readable error body. RFC 9457 (Problem Details for HTTP APIs) defines a standard shape.

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/invalid-quantity",
  "title": "Invalid quantity",
  "status": 422,
  "detail": "qty must be between 1 and 100.",
  "instance": "/v1/orders"
}
```

Use status codes for the class of failure (client error versus server error) and the body for detail. Never leak stack traces or internal hostnames.

## Trade-offs

- **REST versus RPC.** REST gives uniformity and cacheability. It struggles with operations that are not naturally resources (for example, "recalculate prices"). Forcing them into nouns produces awkward designs. Modeling such actions as sub-resources (`POST /orders/{id}/cancellations`) is usually acceptable.
- **gRPC.** Compact, fast, strongly typed, and good for streaming. The cost is tooling friction for browsers and humans (you cannot just `curl` it), and a build-time dependency on generated code across teams.
- **GraphQL.** Removes over-fetching and under-fetching for clients. The cost moves to the server: query cost control, per-field authorization, resolver batching, and harder HTTP caching and observability.
- **Offset versus cursor pagination.** Offset is easy to build and supports jumping. Cursor is correct under change but more work for you and less flexible for clients.
- **Strict versus lenient validation.** Strict input validation catches bugs early. Lenient parsing ("be liberal in what you accept") makes later tightening a breaking change.
- **Idempotency storage.** Safe retries cost storage, a lookup on every write, and a retention policy for keys.

## Failure Modes

- **Double execution on retry.** A timeout leads to a client retry, which leads to a duplicate order or charge. Detect by looking for duplicate business records with near-identical timestamps. Prevent with idempotency keys on every non-idempotent write.
- **Silent breaking change.** A developer renames a field "for clarity" and a mobile app that cannot be force-updated crashes. Detect with contract tests and schema diff checks in CI (for example, comparing OpenAPI or Protobuf files against the last release). Prevent with a review rule: no removals without a deprecation window.
- **Unbounded responses.** A list endpoint works with 100 rows in staging and times out with 10 million in production. Prevent with mandatory limits and a server-enforced maximum page size.
- **GraphQL query abuse.** A deeply nested query fans out into thousands of database calls. Prevent with depth and complexity limits, persisted queries, and batching loaders. Detect with per-query cost metrics.
- **Inconsistent errors.** Some endpoints return `200` with `{"error": ...}`, others return `500` for bad input. Clients write brittle special cases. Prevent with a shared error library and a lint rule.
- **Leaky abstraction.** Database column names and internal enums appear in the response. Every schema migration becomes an API migration. Prevent by mapping domain objects to explicit response types.
- **Authorization gaps.** An endpoint checks that the caller is logged in but not that the caller owns the resource. See [Authentication and Authorization](./authentication-and-authorization.md).
- **Unknown enum or field crash.** A client library fails on an unrecognized value. Prevent by documenting that clients must tolerate unknown fields and values, and by testing with such values.

## In Practice

### Design checklist

- Who are the consumers, and can you reach them to coordinate a change?
- Is every collection paginated with an enforced maximum?
- Is every non-idempotent write safe to retry?
- Are error shapes, status codes, and field naming consistent across endpoints?
- Are timestamps in a single format and time zone (for example, RFC 3339 in UTC)?
- Are identifiers opaque strings and not leaked auto-increment integers?
- Is there a written deprecation policy?
- Is the contract (OpenAPI, Protobuf, or GraphQL schema) stored in the repo and checked in CI?
- Are rate limits and payload size limits documented? See [Service Resilience](./service-resilience.md).

### Worked example: adding a field safely

You need to split `name` into `first_name` and `last_name`.

1. Add the two new response fields. Keep `name`. This is additive.
2. Accept both forms on input. Define which wins when both are present.
3. Mark `name` deprecated in the documentation and, where supported, in the schema.
4. Instrument which clients still read or send `name`.
5. Contact the remaining consumers. Set a removal date.
6. Remove `name` in the next major version only after traffic to it is effectively zero.

### Developer vs. engineer

A developer builds the endpoint that satisfies today's screen. An engineer asks what happens when the request is retried, when the list has a million rows, when a second client appears, and when the field must change in a year. The engineer also writes the contract before the handler.

### When AI generates API code

AI tools produce plausible CRUD handlers quickly. Verify:

- Pagination exists and has an upper bound.
- Writes are idempotent where retries are possible.
- Status codes match the actual failure class.
- Authorization is checked per resource, not just per session.
- The response type is explicit and does not serialize an entire database row.
- Input is validated against a schema, not trusted.

## Further Reading

- RFC 9110, HTTP Semantics (IETF). Defines methods, safety, and idempotency.
- RFC 9457, Problem Details for HTTP APIs (IETF).
- Roy Fielding, "Architectural Styles and the Design of Network-based Software Architectures" (doctoral dissertation, 2000). The origin of REST.
- Google, API Improvement Proposals (AIPs). A public, opinionated set of resource-oriented API design rules.
- Protocol Buffers and gRPC official documentation. Covers schema evolution rules.
- The GraphQL Specification (GraphQL Foundation) and the official documentation on graphql.org.
