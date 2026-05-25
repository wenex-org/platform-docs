# Coworkers Space

A **Coworkers Space** is the organizational concept that groups multiple independent client applications so they can share data through the Platform. It is not a standalone Platform entity — it is expressed as a `coworkers[]` array on each OAuth client registration and injected as a claim into every JWT token issued to users of that client.

---

## What It Is

A Coworkers Space represents a company, team, or group of developers who collaborate and build multiple applications on top of the same Platform instance. Applications within the same space can read each other's documents without calling each other directly — all data exchange flows through the Platform.

```mermaid
graph TB
    subgraph CW["Coworkers Space"]
        subgraph CA["Client A"]
            BEA[Backend]
            DBA[(Own DB)]
        end
        subgraph CB["Client B"]
            BEB[Backend]
            DBB[(Own DB)]
        end
    end

    GW["Platform Gateway :3010"]

    BEA -->|writes & reads| GW
    BEB -->|writes & reads| GW
    GW -->|CQRS webhook| BEA
    GW -->|CQRS webhook| BEB
```

---

## How It Is Represented

### On the Client Registration

Each OAuth client in `/domain/clients` carries a `coworkers` array listing the IDs of the other clients in its space:

```json
{
  "_id": "68fc7a456e8fa60ae29c3d02",
  "name": "Client A",
  "coworkers": ["68fc7a456e8fa60ae29c3d02", "71ab2b789f1ea71bf30d4e13"]
}
```

The client always includes its own ID in `coworkers[]`.

### In JWT Tokens

At token issuance time the Platform reads the client's `coworkers[]` and embeds it in the JWT:

```json
{
  "sub": "user_id",
  "client_id": "68fc7a456e8fa60ae29c3d02",
  "coworkers": ["68fc7a456e8fa60ae29c3d02", "71ab2b789f1ea71bf30d4e13"],
  "scope": "read:identity:users write:content:notes ...",
  "exp": 1748908800
}
```

---

## How Data Sharing Works

Data is shared at the document level via the `clients[]` field. A client adds a coworker's `client_id` to `clients[]` when creating or updating a document. The Platform's `publisher` worker then delivers the document to every client listed in `clients[]` via their registered CQRS webhook.

```mermaid
sequenceDiagram
    participant CA as Client A
    participant GW as Platform Gateway
    participant PUB as Publisher Worker
    participant CB as Client B Worker

    CA->>GW: POST /content/notes\n{ clients: ["clientA_id", "clientB_id"] }
    GW-->>CA: 201 { id: "doc1", ... }
    GW-)PUB: Kafka event (doc1 created)

    PUB->>CA: POST /cqrs { event: "create", data: { id: "doc1", ... } }
    PUB->>CB: POST /cqrs { event: "create", data: { id: "doc1", ... } }

    CA->>CA: store doc1 in Client A DB
    CB->>CB: store doc1 in Client B DB
```

**Rules:**

- Clients never call each other. All data exchange goes through the Platform.
- Sharing is opt-in per document — adding a coworker's ID to `clients[]` is an explicit act by the writing client.
- Both clients end up with an identical local copy of the document (same field names and structure as the Platform schema).

---

## Reading Coworker Data

To query documents that a coworker has shared with your client, use the `client` zone:

```
GET /content/notes?zone=client
```

This returns all documents where `token.client_id` is listed in `clients[]` — which includes documents created by your own client and any documents other clients have explicitly shared with you.

Combine zones to broaden the query:

```
GET /content/notes?zone=own,client
```

---

## Wenex Client

The **Wenex Client** is the official first-party application built by the Wenex team. It operates as a normal OAuth client with no special Platform privileges — it participates in the same Coworkers model as any other client.

---

## Summary

| Concept | Representation |
| --- | --- |
| Coworkers Space | `coworkers[]` array on `/domain/clients` + `coworkers` JWT claim |
| Joining a space | Add partner `client_id` to your own `coworkers[]` registration |
| Sharing a document | Add partner `client_id` to `clients[]` at create or update time |
| Reading shared data | `?zone=client` — matches token's `client_id` against `clients[]` |
| Delivery mechanism | Platform `publisher` worker → CQRS webhook POST |

See [ABAC](./abac) for how `clients[]` is evaluated during reads, and [Core Schema](./core-schema) for the full set of document ownership fields.
