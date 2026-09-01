# API Reference

Complete reference for the Wenex Platform HTTP API — authentication, authorization, REST and GraphQL endpoints, filtering, and real-time streaming.

## In this section

- [Authentication](./authentication) — Token types (JWT and APT), grant flows, the `strict` flag, `x-api-key`, and client integration examples.
- [Authorization](./authorization) — ABAC model, policies, grants, scopes, and the `AuthorityInterceptor`.
- [REST Reference](./rest-reference) — All REST endpoints, request/response shapes, common headers, and error codes.
- [GraphQL Reference](./graphql-reference) — Schema overview, queries, mutations,.
- [Filtering & Pagination](./filtering) — Query filter syntax, operators, sorting, and limit/skip pagination (corrected 2026-09-02).
- [Streaming (SSE)](./streaming) — Server-Sent Events: subscribing to resource streams and handling reconnection.
- [Realtime Data (MQTT)](./realtime) — Near-real-time change notifications over MQTT and MQTT-over-WebSocket: topics, message schema, the `mqtt` client, and EMQX authn/authz.
- [Request Headers](./headers) — canonical list of every recognized request header (linked 2026-09-02).
