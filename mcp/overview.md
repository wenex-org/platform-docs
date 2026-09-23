---
title: "MCP Overview: Model Context Protocol"
description: "What the Wenex MCP server at /mcp offers AI agents like Claude, GPT and Ollama: the Streamable HTTP transport and the agent workflow."
---

# MCP Integration — Model Context Protocol

The Wenex Platform gateway exposes an MCP (Model Context Protocol) server at `/mcp` — a Streamable HTTP transport mounted with `app.all`, so it answers `POST` (the JSON-RPC calls), `GET` (the event stream) and `DELETE` (session close) alike. This allows AI agents (Claude, GPT, Ollama-backed agents) to interact with the platform programmatically using the standardized tool-use protocol.

**Endpoint:** `http://localhost:3010/mcp`
**Transport:** Streamable HTTP (HTTP/1.1 chunked)
**Protocol:** MCP v1 (JSON-RPC over HTTP)

## What is MCP?

MCP is an open protocol that lets AI models communicate with external tools using a structured JSON-RPC interface. The platform acts as an MCP server, exposing tools that agents can call to query and manipulate data.

```mermaid
graph LR
    accTitle: MCP connection path
    accDescr: An AI agent such as Claude, GPT or an Ollama model uses an MCP client to reach the gateway's /mcp endpoint on port 3010, which exposes the platform tools.
    Agent["AI Agent\n(Claude / GPT / Ollama)"]
    MCP["MCP Client\n(SDK transport)"]
    GW["Gateway /mcp\n:3010"]
    TOOLS["Platform Tools\nauth_verify\nread_documentations\n+ resource tools"]

    Agent --> MCP --> GW --> TOOLS
```

## Agent Workflow

A typical agent interaction with the platform:

```mermaid
sequenceDiagram
    accTitle: MCP agent workflow
    accDescr: The agent connects and receives the startup context, lists the tools, verifies its token with auth_verify, reads the documentation with read_documentations, then loops over resource operations such as find.
    participant Agent as AI Agent
    participant GW as Gateway /mcp

    Agent->>GW: connect()
    GW-->>Agent: server startup context
    Agent->>GW: listTools()
    GW-->>Agent: [auth_verify, read_documentations, ...]

    Agent->>GW: callTool("auth_verify")
    GW-->>Agent: { uid, cid, subject, scope, exp, … }  (the JwtToken shape — no `sub` claim)

    Agent->>GW: callTool("read_documentations", { uri: "docs://core/resource-specification" })
    GW-->>Agent: service catalog markdown

    loop Resource operations
        Agent->>GW: callTool("find", { resource: "identity/users", filter: { query: {} } })
        GW-->>Agent: [{ id, username, email }, ...]
    end
```

- See [Tools Reference](./tools) for available built-in and service-specific tools.
- See [Integration Guide](./integration) for SDK setup, authentication, and security.
