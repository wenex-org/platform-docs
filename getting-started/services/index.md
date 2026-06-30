# Services

Wenex Platform consists of 15 domain microservices. All services are NestJS applications that expose a REST API and a gRPC server.

## Architecture Summary

```mermaid
graph LR
    subgraph Core Services
        AUTH["auth<br/>:3020/:5020"]
        DOM["domain<br/>:3030/:5030"]
        CTX["context<br/>:3040/:5040"]
        ESS["essential<br/>:3050/:5050"]
        ID["identity<br/>:3080/:5080"]
    end

    subgraph Business Services
        FIN["financial<br/>:3060/:5060"]
        CAR["career<br/>:3140/:5140"]
        SPE["special<br/>:3090/:5090"]
        TCH["touch<br/>:3100/:5100"]
        CNT["content<br/>:3110/:5110"]
        LOG["logistic<br/>:3120/:5120"]
        CON["conjoint<br/>:3130/:5130"]
        GEN["general<br/>:3070/:5070"]
        THG["thing<br/>:3150/:5150"]
        EDU["education<br/>:3160/:5160"]
    end

    GW[Gateway :3010] --> Core Services
    GW --> Business Services
    Core Services --> Kafka[(Kafka)]
    Business Services --> Kafka
```

## Service Index

| Service | REST Port | gRPC Port | Domain |
| --- | :---: | :---: | --- |
| [Auth](./auth) | 3020 | 5020 | Core |
| [Domain](./domain) | 3030 | 5030 | Core |
| [Context](./context) | 3040 | 5040 | Core |
| [Essential](./essential) | 3050 | 5050 | Core |
| [Identity](./identity) | 3080 | 5080 | Core |
| [Financial](./financial) | 3060 | 5060 | Business |
| [Career](./career) | 3140 | 5140 | Business |
| [Special](./special) | 3090 | 5090 | Business |
| [Touch](./touch) | 3100 | 5100 | Business |
| [Content](./content) | 3110 | 5110 | Business |
| [Logistic](./logistic) | 3120 | 5120 | Business |
| [Conjoint](./conjoint) | 3130 | 5130 | Business |
| [General](./general) | 3070 | 5070 | Business |
| [Thing](./thing) | 3150 | 5150 | Business |
| [Education](./education) | 3160 | 5160 | Business |

## Standard Service Internals

Every microservice follows the same internal structure:

```
apps/services/<name>/src/
├── main.ts                    # Starts REST listener + gRPC server
├── app.module.ts              # Root module: DB connection, imported modules
├── app.service.ts             # Health checks, initialization
├── modules/
│   └── <entity>/
│       ├── <entity>.module.ts
│       ├── <entity>.controller.ts   # REST (also consumed by gateway via gRPC)
│       ├── <entity>.service.ts      # Business logic + gRPC handler
│       ├── <entity>.repository.ts   # Typegoose (MongoDB) queries
│       ├── <entity>.schema.ts       # Mongoose schema definition
│       └── dto/
│           ├── create-<entity>.dto.ts
│           ├── update-<entity>.dto.ts
│           └── <entity>.serializer.ts
└── protobuf/                  # Generated gRPC TypeScript stubs
```

## Scope Naming Convention

Required scopes follow the pattern `{action}:{service}:{collection}`:

```
read:identity:users       → ReadIdentityUsers
write:financial:accounts  → WriteFinancialAccounts
manage:auth:apts          → ManageAuthApts
```

The `manage:` prefix grants all read, write, and destructive actions including `destroy` and bulk operations.

## Health Checks

Every service exposes `GET /status` with checks for its dependencies:

```bash
curl http://localhost:3020/status    # auth
curl http://localhost:3080/status    # identity
curl http://localhost:3060/status    # financial
```

Example response:

```json
{
  "status": "ok",
  "info": {
    "mongodb": { "status": "up" },
    "redis": { "status": "up" },
    "kafka": { "status": "up" }
  }
}
```
