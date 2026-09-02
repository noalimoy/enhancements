---
issue: https://github.com/praxis-proxy/praxis/issues/792
discussion: https://github.com/praxis-proxy/praxis/issues/792
status: hold
repos:
  - praxis
authors:
  - henschwartz
graduation_criteria:
  - How? section added after the What? and Why? direction is accepted
  - Open questions closed in Decisions before How?
  - Admin SSE endpoint streams live request metadata without restart
  - Server-side filters for method, path, status class, and cluster work as documented
  - Per-connection rate limiting via max_rps is enforced; events over the cap are dropped without backpressure on the request path
  - Zero measurable tap overhead when no clients are connected (request path limited to a cheap subscriber guard)
  - Integration test connects a tap client, sends traffic, and verifies each v1 event field (timestamp, request_id, method, path, status, duration_ms, cluster, upstream when known, body sizes)
  - Example config demonstrates enabling tap on the admin listener
  - Configurable max concurrent tap clients is enforced; excess connections receive HTTP 503 with a JSON error body
  - Slow-buffer tap clients receive an SSE error event before disconnect
  - When OpenTelemetry tracing is active, tap events include trace_id and span_id; omit both fields when tracing is inactive
stakeholders:
  - shaneutt
  - twghu
  - alexsnaps
epic: Observability Epic
related:
  - 00794
  - 00796
  - 00797
  - 00798
  - 00799
---

# On Hold

On-hold for now for priority reasons and not ready to move forward yet, though we will likely revisit this one later.
# Live Request Tap API (SSE Streaming)

## What?

Praxis today exposes health, metrics, pipeline inspection, and
runtime log-level controls on the admin listener, but operators
have no supported way to **watch live traffic** as requests
complete. Debugging a routing or upstream issue means tailing
access logs (often delayed or sampled) or attaching an external
proxy tap. Envoy and Linkerd both offer request tap surfaces;
Praxis needs an equivalent that fits the existing admin trust
model.

This proposal, from
[#792](https://github.com/praxis-proxy/praxis/issues/792),
adds **`GET /api/tap`** on the **existing admin listener** as a
**Server-Sent Events (SSE)** stream of per-request metadata.
Each completed request may emit one tap event. Operators connect
with ordinary HTTP clients (browser `EventSource`, `curl -N`,
future `praxisctl tap` in
[#793](https://github.com/praxis-proxy/praxis/issues/793)).

The tap bus must be **inactive when nobody is listening**: no
per-request serialization, channel send, or filter evaluation
unless at least one connected client would receive events.
On the request path, that means a **cheap subscriber guard** only
(for example a broadcast receiver count or equivalent atomic check)
when idle — not building tap JSON for every request. Implementation
details belong in How?, but this constraint shapes request-path
integration and is not automatic for naive SSE designs.
When clients are connected, events are filtered **on the server**
before they are written to the SSE stream.

### Operator-facing surface

Illustrative query parameters (exact names may be refined; behavior
is what this proposal locks):

| Parameter | Example | Purpose |
| --- | --- | --- |
| `method` | `POST` | Emit only matching HTTP methods |
| `path` | `/api/*` | Prefix match (`*` suffix wildcard only) |
| `status` | `5xx` | Filter by status class (`1xx`–`5xx` or exact code) |
| `cluster` | `backend` | Filter by resolved upstream cluster name |
| `max_rps` | `100` | Cap events per second for this connection; excess events are **dropped** |

Tap event fields (minimum v1 set):

- `timestamp` (request completion time)
- `request_id`
- `method`, `path`
- `status`, `duration_ms`
- `cluster`, upstream address when known
- Request and response body sizes when available
- When OpenTelemetry tracing is active:
  `trace_id` and `span_id` (see
  [#301](https://github.com/praxis-proxy/praxis/issues/301)); **omit both
  fields** when tracing is inactive

Optional header capture (request/response header maps) is a
**configuration knob**, not required for the first delivery,
because headers may contain secrets.

The endpoint uses the **same admin exposure policy** as
`/healthy`, `/metrics`, `/api/pipelines`, and `/api/log-level`:
loopback bind by default; reachable on the data-plane interface
only when `insecure_options.allow_public_admin: true`. No new
listener, port, or authentication mechanism in this change.

### Goals

- Stream live request/response **metadata** over SSE on the admin
  listener
- Apply **server-side filters** (method, path prefix, status class,
  cluster) before writing events
- Guarantee **zero tap overhead** when no clients are connected
  (request path limited to a cheap subscriber guard, not per-request
  tap serialization)
- Support **per-connection rate limiting** via `max_rps`; events over
  the cap are **dropped** (no queue, no request-path backpressure)
- Cap **concurrent tap clients** via configuration
- Emit JSON events suitable for `serde_json` consumers and a future
  CLI ([#793](https://github.com/praxis-proxy/praxis/issues/793))
- Cover the surface with an integration test and an example config
- Sit under
  [Epic #160 Observability](https://github.com/praxis-proxy/praxis/issues/160)

### Non-Goals

- Capturing or replaying full request/response **bodies** on the
  tap stream (metadata and sizes only in v1)
- Replacing access logs
  ([#799](https://github.com/praxis-proxy/praxis/issues/799),
  [#126](https://github.com/praxis-proxy/praxis/issues/126))
- Changing metrics cardinality or Prometheus scrape format
  ([#794](https://github.com/praxis-proxy/praxis/issues/794))
- Building `praxisctl tap` itself
  ([#793](https://github.com/praxis-proxy/praxis/issues/793))
- Pushing or mutating live configuration
  ([#785](https://github.com/praxis-proxy/praxis/issues/785))
- A standalone HTML dashboard
  ([#125](https://github.com/praxis-proxy/praxis/issues/125))
- Backpressure on the data-plane request path to guarantee tap delivery
- Unbounded in-memory queues for slow tap clients
- New authentication beyond the existing admin bind policy

### Open Questions

1. **Status filter grammar.** Does `status=5xx` mean any 5xx only,
   or also support exact codes (`404`) and ranges?
2. **Path glob semantics.** Is `*` suffix-only (prefix match) or
   full glob (`**`, `?`)?
3. **Header redaction.** When optional header capture is enabled,
   which hop-by-hop or sensitive names are stripped by default?
4. **Client limits.** Default max concurrent tap clients and default
   `max_rps` when the query parameter is omitted?
5. **Event timing.** Emit on response completion only, or also
   mid-request milestones (e.g. upstream connect)?
6. **SSE keepalive.** Heartbeat interval and behavior when the
   client buffer is slow?
7. **`max_rps` overflow.** When filtered traffic exceeds a client's
   `max_rps`, are events dropped, queued, or does the client disconnect?

## Why?

### Motivation

Metrics show aggregates; access logs show history. Neither answers
"what is happening **right now** on this listener?" during an
active incident. Operators debugging misroutes, upstream timeouts,
or filter regressions need a **low-latency, selective** view of
completed requests without restarting the proxy or enabling
verbose logging globally
([#798](https://github.com/praxis-proxy/praxis/issues/798)).

SSE on the admin listener matches how Praxis already exposes
operational APIs: same trust boundary, same loopback-first posture,
no data-plane filter changes. Server-side filtering and rate limits
keep tap useful under load—slow consumers must not stall request
processing or allocate unbounded memory.

The design deliberately avoids tap work when nobody is listening.
Production proxies should not pay a per-request tax for a debugging
feature that is almost always off.

This work complements pipeline inspection
([#796](https://github.com/praxis-proxy/praxis/issues/796)) and
metrics expansion
([#794](https://github.com/praxis-proxy/praxis/issues/794)):
pipelines explain **what should run**; metrics summarize **how much**;
tap shows **what just happened**.

### User Stories

These are stakeholder needs derived from
[#792](https://github.com/praxis-proxy/praxis/issues/792);
they are not separate tracked issues.

- As an SRE, I want to watch live requests through the proxy during
  an incident so that I can see routing and upstream behavior without
  restarting or enabling global trace logging.
- As a platform operator, I want to filter tap output by path and
  status class so that I am not flooded by health-check traffic.
- As an operator on a shared admin host, I want rate limits on tap
  streams so that a stuck client cannot exhaust proxy resources.
- As a security-conscious operator, I want tap on the admin listener
  only so that live traffic metadata is not exposed on the data plane
  by default.
- As a future CLI author, I want stable JSON event shapes so that
  `praxisctl tap` can consume the same stream as `curl`.

## Decisions

Proposed design choices for each Open Question above. Confirm during
proposal review before implementation begins.

- **Status filter grammar.** Support **status class** tokens
  (`1xx`–`5xx`) and **exact three-digit codes** (`404`, `502`).
  No range syntax in v1.
- **Path glob semantics.** **Suffix `*` only** in v1: `path=/api/*`
  matches paths with that prefix. No `**` or `?`.
- **Header redaction.** Optional header capture is **off by default**.
  When enabled, strip `Authorization`, `Proxy-Authorization`, `Cookie`,
  `Set-Cookie`, and other sensitive names documented in How?; never emit
  full bodies on the tap stream in v1.
- **Client limits.** Configurable **`max_clients`** (default **4**,
  provisional — enough for a few concurrent debug sessions without
  unbounded admin connections; tunable in config) and default
  **`max_rps` per connection** of **100** when the query parameter is
  omitted (provisional starting point to cap serialization cost for one
  slow consumer; operators tighten filters or raise the limit via query
  param). Reject new connections over **`max_clients`** with **HTTP 503
  Service Unavailable** and a JSON body indicating the tap client limit
  (for example `{"error":"tap_client_limit","max_clients":4}`). This is
  connection-slot exhaustion, not per-event rate limiting — clients should
  not treat it like the data-plane **`rate_limit`** filter's **429**.
- **`max_rps` overflow.** Events that would exceed a connection's
  **`max_rps`** after server-side filtering are **dropped** for that
  connection. Tap does **not** queue unbounded events, apply
  request-path backpressure, or stall proxy processing to preserve tap
  delivery. Operators who need full fidelity must narrow filters or
  raise `max_rps`. How? may expose a dropped-event counter on the SSE
  stream or in admin metrics; v1 contract is drop-only.
- **Event timing.** Emit **one event per request on response
  completion** (after upstream response headers are known and
  duration is measured). No mid-request events in v1.
- **OTel fields.** Include `trace_id` and `span_id` only when the
  request has an active trace context; **omit both keys** from the
  JSON event when tracing is inactive (no `"-"` sentinel).
- **SSE keepalive.** Send a comment heartbeat every **15 seconds**
  when idle. When a client's kernel send buffer stays full beyond a
  configured threshold (default and exact seconds in How?), the server
  emits one SSE **`event: error`** line with JSON
  `{"error":"buffer_full","retry":"narrow_filters_or_raise_max_rps"}`
  **before** closing the connection. Clients distinguish this from network
  failure or process restart by reading the error event; both are
  retriable by reconnecting with tighter filters or higher `max_rps`.
  No separate warning event in v1.
