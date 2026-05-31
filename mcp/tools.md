# MCP Tools Reference

## Built-in MCP Tools

The gateway registers two core tools available to all agents.

### `auth_verify`

Verifies the agent's current APT (Auth Personal Token) and returns the decoded claims.

**Call this tool first** before any resource operation to confirm identity and understand what scopes are available.

**Input schema:**

```json
{
  "type": "object",
  "properties": {
    "headers": {
      "type": "object",
      "description": "Optional request headers to inspect"
    }
  }
}
```

**Example output:**

```json
{
  "sub": "64a1b2c3d4e5f6a7b8c9d0e1",
  "scope": "read:identity:users read:financial:transactions",
  "zone": "own",
  "exp": 1748908800,
  "client_id": "64a1b2c3d4e5f6a7b8c9d0e2"
}
```

### `read_documentations`

Loads MCP specification documentation by URI. This allows agents to self-serve context about the platform's capabilities, resources, and operations.

**Input schema:**

```json
{
  "type": "object",
  "required": ["uri"],
  "properties": {
    "uri": {
      "type": "string",
      "description": "Documentation URI, e.g. docs://core/specification?v=c"
    }
  }
}
```

**URI patterns:**

| URI | Content |
|---|---|
| `docs://core/specification?v=c` | Compact canonical MCP specification |
| `docs://core/specification?v=e` | Extended canonical MCP specification |
| `docs://core/resource-specification?v=c` | Service and collection catalog (compact) |
| `docs://core/auth-specification?v=c` | Authentication mechanics (compact) |
| `docs://core/agent-guidance?v=c` | Agent guidance and MongoDB patterns |
| `docs://core/cross-service-pattern?v=c` | Multi-service workflow patterns |
| `docs://service/identity?v=c` | Identity service documentation |
| `docs://service/financial?v=c` | Financial service documentation |
| `docs://service/career?v=c` | Career service documentation |

Version parameter: `v=c` (compact — fewer tokens) or `v=e` (extended — more detail).

## Service-Specific MCP Tools

Each gateway module can register additional MCP tools in its `*.router.ts` file. These tools expose the service's CRUD operations as named MCP tools.

Router files are located at:

```
apps/gateway/src/modules/{service}/crafts/{collection}/{collection}.router.ts
apps/gateway/src/app.router.ts  (core tools: auth_verify, read_documentations)
```

## MCP Documentation Spec Files

The `mcp/` directory contains the specification files served by `read_documentations`:

```
mcp/
├── readme.md                              # MCP routing guide (not a tool)
├── core/
│   ├── -specification.compact.md          # Canonical MCP rules (compact)
│   ├── -specification.extended.md         # Canonical MCP rules (extended)
│   ├── resource-specification.compact.md  # Service/collection catalog
│   ├── resource-specification.extended.md
│   ├── auth-specification.compact.md      # Auth mechanics
│   ├── auth-specification.extended.md
│   ├── agent-guidance.compact.md          # MongoDB query patterns for agents
│   ├── agent-guidance.extended.md
│   └── cross-service-pattern.compact.md   # Multi-service workflow patterns
└── service/
    ├── identity.compact.md
    ├── identity.extended.md
    ├── financial.compact.md
    ├── financial.extended.md
    └── ... (one pair per service)
```

**Compact vs Extended:**

| Variant | Use when |
|---|---|
| `?v=c` (compact) | Agent context is limited; decision-first, minimal prose |
| `?v=e` (extended) | Agent needs full rationale, richer examples, edge cases |
