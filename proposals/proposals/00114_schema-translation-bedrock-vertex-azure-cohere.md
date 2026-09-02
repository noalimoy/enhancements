---
discussion: https://github.com/orgs/praxis-proxy/discussions/1070
issue: https://github.com/praxis-proxy/enhancements/issues/XXX
status: proposed
repos:
  - ai
authors:
  - noalimoy
graduation_criteria:
  - How section with per-provider filter design reviewed
  - All four providers pass fixture-backed integration tests
  - Example config per provider in examples/configs/
stakeholders:
  - shaneutt
related:
  - https://github.com/praxis-proxy/ai/issues/114
  - https://github.com/praxis-proxy/ai/issues/363
---

# Schema Translation: Bedrock, Vertex AI, Azure, Cohere

## What?

Add request/response translation filters so that OpenAI Chat
Completions clients can transparently use AWS Bedrock, Google Vertex AI
(Gemini), Azure OpenAI, and Cohere backends through Praxis — the same
pattern already working for Anthropic Messages via `anthropic_to_openai`.

Each provider gets a dedicated `HttpFilter` under `apis/src/` that:

- Rewrites the OpenAI request body to the provider's native JSON shape.
- Rewrites the provider's response back to Chat Completions format.
- Handles streaming: SSE payload translation for Azure/Cohere/Vertex,
  binary eventstream decoding for Bedrock.
- Strips protocol-specific client headers that don't belong on the
  upstream hop (e.g. OpenAI `Authorization` when the gateway injects
  its own via `credential_inject` / `gcp_adc` / `aws_sigv4_sign`).

One filter set per provider, one example config per provider, registered
in `register.rs`, with fixture-backed tests.

### Goals

- Any OpenAI Chat Completions client works against Bedrock, Vertex AI,
  Azure OpenAI, and Cohere without application code changes.
- Streaming works end-to-end for all four providers.
- Each provider is independently toggleable via Cargo feature flags.
- Error responses are normalized to Chat Completions error shape.

### Non-goals

- OpenAI Responses ↔ Chat Completions translation (done,
  [praxis-proxy/ai#35]).
- Anthropic Messages ↔ OpenAI translation (done,
  [praxis-proxy/ai#103]).
- Credential injection or lifecycle management (existing filters:
  `credential_inject`, `gcp_adc`, `aws_sigv4_sign`, `azure_ad`).
- Token counting (existing filter, tracked in
  [praxis-proxy/ai#71]).
- Intelligent provider routing (tracked in
  [praxis-proxy/ai#74]).

## Why?

### Motivation

Praxis already sits in the request path doing classification, routing,
credential injection, token counting, and guardrails. The Anthropic
translation path proved the pattern works. But four major providers
remain uncovered — forcing operators to either:

1. Maintain provider SDKs in every application, or
2. Add a second translation hop (LiteLLM, etc.) alongside Praxis.

Adding translation filters eliminates the second hop and keeps all
AI gateway concerns in one place: one config file, one credential
boundary, one observability surface.

### User Stories

- As a platform engineer, I want to point my OpenAI-based apps at
  Praxis and switch the backend from OpenAI to Bedrock without
  changing application code.

- As an AI gateway operator, I want a single proxy that handles
  credentials, guardrails, and format translation so I don't need
  a second translation layer in the path.

- As a Praxis maintainer, I want each provider's translation to be
  a self-contained filter with its own tests and example config.

### Related

- [Schema Translation Epic]
- [API Translation Epic]
- [Anthropic Messages Implementation]

[Schema Translation Epic]: https://github.com/praxis-proxy/ai/issues/114
[API Translation Epic]: https://github.com/praxis-proxy/ai/issues/363
[Anthropic Messages Implementation]: https://github.com/praxis-proxy/ai/issues/103
[praxis-proxy/ai#35]: https://github.com/praxis-proxy/ai/issues/35
[praxis-proxy/ai#103]: https://github.com/praxis-proxy/ai/issues/103
[praxis-proxy/ai#71]: https://github.com/praxis-proxy/ai/issues/71
[praxis-proxy/ai#74]: https://github.com/praxis-proxy/ai/issues/74