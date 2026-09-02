---
issue: https://github.com/praxis-proxy/praxis/issues/1051
discussion: https://github.com/praxis-proxy/praxis/issues/1051 
status: proposed
repos:
  - praxis
  - ai
authors:
  - VaishnaviHire
  - cdoern
graduation_criteria:
  - How? section with requirements and design
  - Request-scoped trace context model reviewed by stakeholders
  - Forwarded upstream requests propagate x-request-id and W3C traceparent when enabled
  - Subrequests propagate x-request-id and W3C traceparent transparently through the default subrequest path when enabled
  - Default/easy subrequest API cannot accidentally bypass trace propagation
  - AI delegated callouts that use subrequests are covered without AI-specific trace header stitching
  - Unit and integration tests cover inbound, generated, malformed, and subrequest propagation cases
stakeholders:
  - shaneutt
  - alexsnaps
  - aslakknutsen
  - leseb
related:
  - https://github.com/praxis-proxy/ai/pull/654
---

# Trace Context Propagation

## What?

Add core support for request correlation and W3C trace context
propagation across Praxis request execution paths.

Praxis already has a `request_id` observability filter that can
forward or generate `X-Request-ID`, and it has OpenTelemetry/span
export plumbing. What is missing is a shared, request-scoped trace
context that can be used consistently by both the normal forwarded
upstream request and by subrequests created during request processing.

This proposal introduces that missing correlation model in core. When
trace propagation is enabled, Praxis should resolve or generate the
correlation values for a request once, store them in request-scoped
context, and propagate them through the framework-owned request and
subrequest paths.

The propagated headers are:

- `x-request-id` — request correlation ID used primarily for logs and
  support/debugging.
- `traceparent` — W3C Trace Context header used to continue a
  distributed trace across services.

The default subrequest path should be context-aware. Filters and
call sites should not need to know whether tracing is enabled, and they
should not manually add regular tracing headers. If a filter creates a
subrequest through the supported subrequest subsystem, propagation
should happen transparently when enabled.

### Goals

- Add request-scoped trace context handling in core.
- Reuse an existing request ID when present, including values produced
  by the existing `request_id` filter, or generate one when absent.
- Accept and continue valid inbound W3C `traceparent` values.
- Generate a valid W3C trace context when inbound trace context is
  absent or malformed.
- Propagate `x-request-id` and `traceparent` to the forwarded upstream
  request when enabled.
- Propagate `x-request-id` and `traceparent` to subrequests
  transparently through the default subrequest path when enabled.
- Keep tracing/correlation propagation in core plumbing rather than in
  individual filters.
- Make the context-aware subrequest path the default API so new
  callouts do not accidentally bypass propagation.
- Allow AI request flows such as file resolve and file search to rely
  on the core mechanism rather than manually stitching correlation
  headers into delegated callouts.
- Validate propagation behavior with unit and integration coverage.

### Non-Goals

- Defining an opt-out path before a concrete use case exists.
- Adding AI-specific trace parsing or header construction.
- Implementing OGX-to-vector-backend propagation as part of the core
  Praxis change.

## Why?

### Motivation

A single downstream request can fan out into multiple upstream or
subrequest operations.  Without consistent correlation propagation, 
the services involved see unrelated HTTP requests. That makes it difficult
to answer operational questions such as:

- Which backend or delegated callout belonged to this user request?
- Was latency caused by the forwarded model request or by retrieval?
- Which logs across Praxis, AI callouts, OGX, or another backend belong
  to the same request?
- Did a subrequest bypass the normal tracing path?

The current AI-side prototype in
[praxis-proxy/ai#654](https://github.com/praxis-proxy/ai/pull/654)
proves the need for propagating `x-request-id` and W3C `traceparent`
across AI request legs. Review feedback on that PR pointed out that the
mechanism is not AI-specific. Request ID resolution, trace context
validation, request-scoped context storage, and propagation through
subrequests are framework concerns.

Keeping this logic in individual filters can cause difficulty in debugging.
Praxis should avoid that by making propagation as core as possible.

The subrequest abstraction already implies a child relationship to the
request that created it. Trace propagation should reflect that
relationship. If a subrequest is made through the supported subrequest
subsystem, it should inherit the parent request's correlation context by
default when propagation is enabled.

### User Stories

- As a platform operator, I want all forwarded requests and subrequests for one
  downstream request to share correlation identifiers so that I can
  debug failures and latency without manually stitching unrelated logs
  together.
- As a platform operator, I want Praxis to propagate W3C trace context
  through framework-owned paths so that distributed traces can connect
  client requests, forwarded upstream requests, and delegated
  subrequests.
- As a filter author, I want the default subrequest API to handle
  tracing headers for me so that I do not need to know about or
  manually add `x-request-id` and `traceparent`.
- As a maintainer, I want the easy subrequest path to be context-aware
  so that new filters and callout paths do not accidentally bypass
  observability plumbing.
- As a support engineer, I want `x-request-id` to remain available for
  log search while `traceparent` carries the distributed trace identity
  across services.
