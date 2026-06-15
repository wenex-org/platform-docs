# MCP Integration Guide

## Connecting an Agent

### Using the MCP TypeScript SDK

```typescript
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js';
import { Client } from '@modelcontextprotocol/sdk/client/index.js';

const transport = new StreamableHTTPClientTransport(
  new URL('http://localhost:3010/mcp'),
  {
    requestInit: {
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${process.env.WENEX_APT_TOKEN}`,
      },
    },
  },
);

const mcp = new Client({ name: 'my-agent', version: '1.0.0' });
await mcp.connect(transport);

// List available tools
const { tools } = await mcp.listTools();
console.log(tools.map((t) => t.name));

// Call a tool
const result = await mcp.callTool({
  name: 'auth_verify',
  arguments: {},
});
console.log(result);
```

## Using `mcp-client.ts`

The platform ships a ready-to-use interactive MCP client at [`mcp-client.ts`](../../mcp-client.ts) that connects to the gateway and uses Ollama as the LLM backend.

### Prerequisites

```bash
# Start a local Ollama instance (the client's default model)
ollama run qwen3.6:35b

# Or tunnel from a remote GPU
ssh -L 11434:localhost:11434 user@gpu-host
```

### Running the Client

```bash
# Set your auth token
export MCP_CLIENT_APT_TOKEN="apt_..."

# Run the interactive client
npx ts-node mcp-client.ts
```

The client:
1. Connects to `http://127.0.0.1:3010/mcp`
2. Loads all available tools
3. Captures the server's startup context prompt
4. Enters an interactive chat loop where the LLM can call platform tools

### Client Configuration

```typescript
const DEFAULT_CONFIG = {
  maxToolRounds: 10,            // Max tool calls per user turn
  maxHistoryMessages: 50,       // Sliding context window
  defaultModel: 'qwen3.6:35b', // Ollama model
  ollamaHost: 'http://localhost:11434',
  mcpServerUrl: 'http://127.0.0.1:3010/mcp',
};
```

Override via constructor:

```typescript
const client = new ClientMCP({
  mcpServerUrl: 'https://api.yourdomain.com/mcp',
  defaultModel: 'llama3.1:70b',
  maxToolRounds: 20,
});
```

## Authentication for MCP

MCP connections require an APT (Auth Personal Token) passed as a Bearer token in the HTTP headers:

```typescript
headers: {
  Authorization: `Bearer ${aptToken}`,
}
```

Create an APT with the minimum scopes required for your agent:

```bash
curl -X POST http://localhost:3010/auth/apts \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-ai-agent",
    "scopes": ["read:identity:users", "read:financial:transactions"],
    "subjects": ["analytics-agent@example.com"]
  }'
```

Store the returned `token` — it is shown only once.

## Observability

MCP requests flow through the same middleware pipeline as REST requests:

- Prometheus metrics at `/metrics` include MCP tool call counts
- OpenTelemetry traces capture tool execution spans
- `X-Request-ID` header propagates through MCP calls for correlation
- Rate limiting applies to MCP connections per-token

## Security Considerations

- Always use an APT with minimal required scopes — not an admin JWT
- APTs can be revoked at any time via `DELETE /auth/apts/:id`
- `auth_verify` lets agents self-check their token before making calls
- All MCP tool calls are subject to the same `AuthGuard`, `ScopeGuard`, and `PolicyGuard` as REST requests
- MCP connections do not bypass rate limits
