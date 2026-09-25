# Wenex Platform — Developer Documentation

**Version:** see [the changelog](./changelog.md) (the newest tag is the release — `platform/package.json` may lag it) | **Node:** 22.x | **Package Manager:** pnpm 10.5.2

Wenex Platform is a large-scale distributed microservices system built with NestJS. It exposes a unified REST, GraphQL, and MCP (Model Context Protocol) gateway in front of 14 domain microservices backed by MongoDB, PostgreSQL, Redis, Kafka, Elasticsearch, MinIO, and MQTT.

## Table of Contents

| Document | Description |
| --- | --- |
| [Getting Started](./getting-started/index.md) | Clone, install, seed, and run the platform |
| [**Ecosystem & ABAC Model**](./getting-started/overview/ecosystem/index.md) | **How Clients, Coworkers, and the Platform relate — read this first** |
| [Platform Architecture](./getting-started/overview/ecosystem/platform.md) | System design, request flow, communication topology, design principles |
| **API Reference** | |
| [Authentication](./api/authentication.md) | Tokens, APTs, scopes, RBAC — with curl examples |
| [REST Reference](./api/rest-reference.md) | All REST endpoints, headers, responses, curl examples |
| [GraphQL Reference](./api/graphql-reference.md) | All queries and mutations with examples |
| [Filtering & Pagination](./api/filtering.md) | `query`, `populate`, `projection`, `pagination`, `zone` |
| [Streaming (SSE)](./api/streaming.md) | Cursor / Server-Sent Events endpoint |
| **SDK** | |
| [SDK Guide](./sdk/index.md) | `@wenex/sdk` installation, configuration, and examples |
| **Services** | |
| [Service Catalog](./getting-started/services/index.md) | All 14 microservices — purpose, ports, collections |
| **MCP** | |
| [MCP Integration](./mcp/overview.md) | MCP tools, AI agent usage, `mcp-client.ts` |
| **Client Development** | |
| [Client Development Guide](./getting-started/overview/ecosystem/client-app.md) | Building a client app — canonical structure, patterns, best practices |

## Quick Reference

### Gateway Endpoints

| Path | Purpose |
| --- | --- |
| `http://localhost:3010/api` | Swagger UI (OpenAPI) |
| `http://localhost:3010/api-json` | OpenAPI JSON spec |
| `http://localhost:3010/graphql` | Apollo GraphQL Playground |
| `http://localhost:3010/mcp` | MCP streamable HTTP transport |
| `http://localhost:3010/metrics` | Prometheus metrics |
| `http://localhost:3010/status` | Liveness / readiness health check |

### URL Pattern

Every REST endpoint follows:

```text
/{service}/{collection}/{operation}
```

Examples:

```text
GET  /identity/users
POST /financial/accounts
GET  /auth/apts/count
```

### Request Headers

| Header | Type | Purpose |
| --- | --- | --- |
| `Authorization` | `Bearer <jwt>` | Required for all protected routes |
| `x-request-id` | string | Trace ID (auto-generated if omitted) |

## C4 Diagrams

Architecture diagrams live in the platform repository's [`diagrams/`](https://github.com/wenex-org/platform/tree/main/diagrams) directory:

- [Context Diagram](https://github.com/wenex-org/platform/blob/main/diagrams/c4/context-diagram.svg) — system-level actors
- [Container Diagram](https://github.com/wenex-org/platform/blob/main/diagrams/c4/container-diagram.svg) — platform internals
- [Client Container Diagram](https://github.com/wenex-org/platform/blob/main/diagrams/c4/container-diagram.client.svg) — client-side view
- [Request Flow](https://github.com/wenex-org/platform/blob/main/diagrams/rflow/request-flow-diagram.svg) — end-to-end request path
