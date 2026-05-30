# MCP Integration — Model Context Protocol

The Wenex Platform gateway exposes an MCP (Model Context Protocol) server at `GET /mcp`. This allows AI agents (Claude, GPT, Ollama-backed agents) to interact with the platform programmatically using the standardized tool-use protocol.

**Endpoint:** `http://localhost:3010/mcp`
**Transport:** Streamable HTTP (HTTP/1.1 chunked)
**Protocol:** MCP v1 (JSON-RPC over HTTP)

---

## What is MCP?

MCP is an open protocol that lets AI models communicate with external tools using a structured JSON-RPC interface. The platform acts as an MCP server, exposing tools that agents can call to query and manipulate data.

```mermaid
graph LR
    Agent["AI Agent\n(Claude / GPT / Ollama)"]
    MCP["MCP Client\n(SDK transport)"]
    GW["Gateway /mcp\n:3010"]
    TOOLS["Platform Tools\nauth_verify\nread_documentations\n+ resource tools"]

    Agent --> MCP --> GW --> TOOLS
```

---

## Agent Workflow

A typical agent interaction with the platform:

```mermaid
sequenceDiagram
    participant Agent as AI Agent
    participant GW as Gateway /mcp

    Agent->>GW: connect()
    GW-->>Agent: server startup context
    Agent->>GW: listTools()
    GW-->>Agent: [auth_verify, read_documentations, ...]

    Agent->>GW: callTool("auth_verify")
    GW-->>Agent: { sub, scope, exp }

    Agent->>GW: callTool("read_documentations", { uri: "docs://core/resource-specification?v=c" })
    GW-->>Agent: service catalog markdown

    loop Resource operations
        Agent->>GW: callTool("find_identity_users", { filter: { query: {} } })
        GW-->>Agent: [{ id, username, email }, ...]
    end
```

---

- See [Tools Reference](./tools) for available built-in and service-specific tools.
- See [Integration Guide](./integration) for SDK setup, authentication, and security.
