# MCP Refactor Design — Wenex Platform

**Date:** 2026-05-07  
**Status:** Approved  
**Scope:** `mcp-client.ts`, `apps/gateway/src/**/*.router.ts`, `mcp/**/*.md`

---

## Goal

Produce a production-ready MCP server and client for the Wenex platform where:

1. `mcp-client.ts` is a **generic, standard-conformant MCP client** with zero Wenex-specific references — any MCP server can be connected to it
2. The **server pushes startup context** to connecting agents automatically via MCP session-init notification, so no client needs to know platform internals
3. **Core/utility tools** (`auth_verify`, `read_documentations`) carry rich inline descriptions that make the documentation system self-bootstrapping from the tool list alone
4. **All 13 service specs** are systematically correct against runtime sources of truth
5. **Agent guidance** (MongoDB + Mermaid) is expanded to production depth
6. **Cross-service workflow patterns** are documented for the first time

---

## Sources of Truth (precedence order)

1. `apps/gateway/src` — controllers, routers, resolvers (external surface only)
2. `libs` — enum files + `common/src/schemas/map.ts` (enums, types, population paths)
3. Existing MCP spec prose — `mcp/readme.md`, `mcp/core/`, `mcp/service/`

Internal-only methods, classes, and interfaces are ignored unless they directly define externally visible runtime behavior.

---

## Constraints (carry through all implementation)

- Enums: only values confirmed by platform-libs enum files
- Population paths: only paths confirmed by `map.ts`; label others as "raw ID – not a confirmed population path"
- Mark fields as write-only / platform-managed / serializer-hidden where applicable; never expose them in examples
- Do not invent operations, enums, population paths, headers, or tool names not confirmed by runtime sources
- Do not remove safety warnings
- Compact files: shorter, decision-first, safety-first, low-token; defer detail to extended
- Extended files: cover the same knowledge as compact, with fuller rationale, richer examples, stronger agent guidance
- Do not merge compact and extended into one file

---

## Section 1 — Client Architecture (`mcp-client.ts`)

### What Gets Removed

| Current | Reason |
| --- | --- |
| Hardcoded `wenex-startup` prompt fetch by name | Client must not know server internals |
| `wenex-mcp-client` client identity string | Platform-specific branding |
| Wenex branding in console output | Not part of a generic client |

The default `mcpServerUrl` value (`http://127.0.0.1:3010/mcp`) is kept as a convenience default but is not treated as Wenex-specific — it is a local address that any server could occupy.

### What Gets Added

- `sampling: {}` declared in client capabilities during `initialize` — signals to any MCP server that this client supports server-sent messages
- A `notifications/message` handler: any message the server pushes on session open is captured and prepended to the conversation as a `system`-role message before the first user turn
- `listPrompts()` called generically on connect as a fallback: if no `notifications/message` arrives within 2 seconds of the `initialized` acknowledgement, the client calls `listPrompts()`, picks up whatever the server exposes, and applies them as conversation prefix — this preserves compatibility with servers that use prompt discovery instead of push

### Class Shape

```typescript
interface ClientMCPConfig {
  mcpServerUrl: string;
  ollamaHost?: string;
  defaultModel?: string;
  maxToolRounds?: number;
  maxHistoryMessages?: number;
}

class ClientMCP {
  connect(serverUrl?: string, authToken?: string): Promise<void>
  processQuery(query: string, model?: string): Promise<string>
  chatLoop(model?: string): Promise<void>
  disconnect(): Promise<void>
}
```

Auth token is passed explicitly to `connect()` — not read from env inside the class. The `main()` entry-point wrapper at the bottom of the file reads `MCP_CLIENT_APT_TOKEN` and `MCP_SERVER_URL` from env and passes them to `connect()`. This is the only place platform-specific config appears, and it is in the run-time wrapper, not the class.

### Entry-Point Wrapper

```typescript
// Only Wenex-specific knowledge lives here — outside the class
(async () => {
  const token = process.env.MCP_CLIENT_APT_TOKEN;
  const serverUrl = process.env.MCP_SERVER_URL;
  if (!token) { console.error('MCP_CLIENT_APT_TOKEN required'); process.exit(1); }

  const client = new ClientMCP();
  await client.connect(serverUrl, token);
  await client.chatLoop();
})();
```

---

## Section 2 — Server-Side Session-Init Push (`core.router.ts`)

### Mechanism

After the MCP `initialize` handshake completes, `core.router.ts` sends a `notifications/message` over the open SSE stream using `server.sendLoggingMessage()`. The content is loaded from the `wenex-startup` registered MCP prompt in the libs submodule:

```typescript
// content loaded from the registered wenex-startup MCP prompt
server.sendLoggingMessage({
  level: 'info',
  data: startupPromptContent,
})
```

This uses the MCP protocol's `notifications/message` pathway — the only server→client push that Streamable HTTP reliably supports without requiring full bidirectional sampling.

A generic client that handles `notifications/message` events captures `level: 'info'` messages on connect and injects them as system context. The client never needs to know the content in advance.

### Backward Compatibility

The `wenex-startup` MCP prompt **remains registered** in the libs submodule as a standard named prompt. Claude Desktop and other clients that do explicit prompt discovery via `listPrompts()` + `getPrompt()` continue to work unchanged. The session-init push is additive.

### What Does Not Change

All 47 domain CRUD routers are untouched in this section. Tool description updates are covered in Section 3.

---

## Section 3 — Two-Tier Tool Descriptions

### Tier 1: Core/Utility Tools (rich inline descriptions)

`auth_verify` and `read_documentations` in `core.router.ts` receive full inline guidance. Agents encounter these tools before loading any documentation — they must be self-explanatory from the tool list alone.

**`auth_verify` description additions:**

- When to call it: before any resource operation, once per session
- What the decoded token fields mean at a glance: subject, scope, zone, expiry
- What to do on `401`: load `docs://core/auth-specification` via `read_documentations`

**`read_documentations` description additions:**

- Complete URI catalog inline: all `docs://core/*` and `docs://service/*` URIs with one-line purpose each
- Version rule: `?v=c` (compact, default) vs `?v=e` (extended, for troubleshooting / onboarding)
- Minimum startup sequence: `specification` → `resource-specification` → `auth-specification` (when auth matters)
- When to escalate to extended: ambiguity, `401`/`403`, complex filters, schema authoring

**Effect:** An agent with no prior context can read the tool list alone and know exactly what to load first and in what order. The documentation system is self-bootstrapping.

### Tier 2: Domain CRUD Tools (minimal doc pointers, unchanged)

All 47 service tools keep the current pattern:

```
description: `Read "docs://service/<service>-specification"`
```

This is intentional — these tools are called after the agent has already loaded the service spec. Rich inline descriptions on 47 tools would bloat the tool list and consume tokens on every request.

**No new tools are added.** The two core tools plus domain routers cover the full surface. The session-init push handles onboarding context without requiring a dedicated `platform_context` tool.

---

## Section 4 — Spec Documentation Overhaul

### Track 1: Service Specs (all 13 services, both compact + extended)

Each service spec gets a systematic pass against all three sources of truth.

**Update template per service:**

| Item | Rule |
| --- | --- |
| Field schemas | Confirmed fields only; write-only / platform-managed / serializer-hidden marked explicitly |
| Enum values | Only values confirmed by platform-libs enum files |
| Population paths | Only paths from `map.ts`; others labelled "raw ID – not a confirmed population path" |
| Special operations | Non-CRUD endpoints documented with input/output shapes |
| Safety warnings | Preserved and promoted to top of each spec |

**Priority order** (by agent usage frequency):

1. `identity` — users, profiles, sessions, subjects
2. `financial` — accounts, currencies, invoices, wallets, transactions
3. `essential` — sagas, saga stages, orchestration
4. `general` — activities, artifacts, comments, events, workflows
5. `special` — files, uploads, share links, stats
6. `domain` — clients, apps, scopes
7. `content` — notes, posts, tickets
8. `context` — configs, settings, RBAC
9. `touch` — emails, notices, pushes, SMS
10. `conjoint` — messaging, channels, contacts
11. `career` — businesses, branches, employees, products
12. `logistic` — locations, drivers, vehicles, travels
13. `thing` — devices, sensors, metrics, telemetry

**Compact** stays decision-first, safety-first, low-token. **Extended** covers the same knowledge with fuller rationale, edge cases, and worked examples. Both files remain separate.

---

### Track 2: Agent-Guidance Expansion (`mcp/core/agent-guidance.compact.md` + `extended.md`)

**MongoDB additions:**

| Addition | Purpose |
| --- | --- |
| `$elemMatch` | Query array-of-object fields (e.g. items in an order) |
| `$size` | Check array length |
| `$expr` | Compare two fields within a document |
| Geospatial deep-dive | `$near` vs `$geoWithin` with coordinate order reminder (`[lng, lat]`) |
| Pagination patterns | `limit` + `skip` + `sort` for deterministic pages |
| Anti-patterns table | Queries known to produce wrong results on Wenex collections |

Key anti-patterns to document:

- Querying `deleted_at` directly instead of using the `x-exclude-soft-delete-query` metadata header
- Using `$search` outside `$text`
- Sending unbounded list requests without `filter.pagination.limit`
- Querying high-volume append-only collections (`thing/metrics`, `financial/transactions`) without a count-first + paginate pattern

**Mermaid additions:**

| New type | When to use |
| --- | --- |
| `journey` | User journey maps across Wenex services |
| `timeline` | Project/event timelines, saga stage sequences |
| `xychart-beta` | Bar/line charts for metrics and telemetry data |

Each diagram type gets a worked example using Wenex domain data (e.g. transaction state machine, saga flow, device metric trend, user onboarding journey).

A "choose your diagram" decision table is added mapping user intent → diagram type — replacing the current simple list.

---

### Track 3: Cross-Service Patterns (new document pair)

New files: `mcp/core/cross-service-patterns.compact.md` + `mcp/core/cross-service-patterns.extended.md`

**Patterns to document:**

| Pattern | Services | Key guidance |
| --- | --- | --- |
| Financial transaction saga | `financial` + `essential` | Use `init_financial_transactions`; saga stages are backend-driven |
| IoT with physical location | `thing` + `logistic` | Device metrics are append-only; GeoJSON coordinate order |
| User onboarding flow | `identity` + `domain` + `context` | Subject resolution order; client secret is platform-managed |
| Content with reactions/comments | `content` + `general` | Comments live in `general`, not `content` |
| Notification dispatch | `touch` + `essential` | Create/send triggers real-world communication; wrap in saga for reliability |
| File upload + share link | `special` + `identity` | `files` stores metadata only; binary via upload endpoint; share requires subject |

Each pattern documents:

- Recommended tool call sequence
- Which service to call first and why
- Data dependencies between calls (what ID from service A feeds into service B)
- Failure / rollback considerations

**`mcp/readme.md` update:** A new row is added to the routing table:

```
| Multi-service workflow | docs://core/cross-service-patterns | Load when intent spans 2+ services |
```

The deterministic loading flow diagram gains a new branch: "Does the task span multiple services? → Load docs://core/cross-service-patterns"

---

## File Change Summary

| File | Change type |
| --- | --- |
| `mcp-client.ts` | Refactor — remove Wenex refs, add notification handler, generic auth |
| `apps/gateway/src/modules/core/core.router.ts` | Enhance — session-init push + rich tool descriptions |
| `mcp/core/agent-guidance.compact.md` | Expand — MongoDB + Mermaid additions |
| `mcp/core/agent-guidance.extended.md` | Expand — same additions with full rationale and examples |
| `mcp/core/cross-service-patterns.compact.md` | New — 6 cross-service workflow patterns |
| `mcp/core/cross-service-patterns.extended.md` | New — same patterns with full detail and examples |
| `mcp/readme.md` | Update — cross-service row in routing table + loading flow diagram |
| `mcp/service/*-specification.compact.md` (×13) | Systematic update per template above |
| `mcp/service/*-specification.extended.md` (×13) | Systematic update per template above |

**Total files:** ~34 files changed or created.

---

## Out of Scope

- Auth service endpoints — not MCP-callable, no changes
- Worker services — no MCP surface
- gRPC / Kafka / BullMQ internals — no MCP surface
- New MCP tools beyond `auth_verify` and `read_documentations` for core
- OpenAI / Anthropic client adapters — client stays Ollama-based; generic architecture allows future addition
- Removing the `wenex-startup` named prompt from the libs submodule — kept for backward compatibility with standard MCP clients
