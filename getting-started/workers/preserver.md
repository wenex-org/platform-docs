# Preserver

**Port:** `:4030`  
**Source:** `apps/workers/preserver/`

The preserver is the EMQX ExHook provider. It implements a gRPC server that EMQX calls for every MQTT lifecycle event — authentication, authorization, connection, disconnection, and message publishing. This lets the platform control which clients can connect to the MQTT broker and what topics they can access.

## Responsibilities

- **Authentication** (`onClientAuthenticate`) — validate MQTT client credentials against the Auth service. Rejects unknown or expired tokens.
- **Authorization** (`onClientAuthorize`) — enforce topic-level ACLs. Called before a client can publish or subscribe.
- **Connection tracking** (`onClientConnected` / `onClientDisconnected`) — log connect/disconnect events for presence tracking.
- **Message filtering** (`onMessagePublish`) — inspect outbound messages before delivery; can drop or modify them.
- **Provider lifecycle** (`onProviderLoaded` / `onProviderUnloaded`) — called by EMQX when the ExHook plugin starts or stops.

## ExHook gRPC Protocol

EMQX calls the preserver over gRPC using the ExHook spec. The proto definition lives at:

```
apps/workers/preserver/src/app.proto
apps/workers/preserver/src/protobuf/auth.proto
```

The `EmqxController` maps each ExHook method to a `@GrpcMethod`:

```
onProviderLoaded       → provider registration
onProviderUnloaded     → provider teardown
onClientAuthenticate   → MQTT CONNECT packet validation
onClientAuthorize      → publish/subscribe ACL check
onClientConnected      → post-connect hook
onClientDisconnected   → post-disconnect hook
onMessagePublish       → pre-deliver message hook
```

## Auth Integration

The preserver uses `AuthProviderModule` to verify tokens and check ABAC policies. Credentials are cached in Redis to reduce round-trips to the auth service on every MQTT packet.

## Infrastructure Dependencies

| Dependency | Usage |
|---|---|
| Redis | Token/session cache for auth lookups |
| Auth service | Token verification and ABAC policy evaluation |
| EMQX | Calls this worker via gRPC ExHook on every MQTT event |

## Key Files

| File | Purpose |
|---|---|
| `modules/emqx/emqx.controller.ts` | `@GrpcService` — all ExHook gRPC method handlers |
| `modules/emqx/emqx.service.ts` | Business logic for each hook event |
| `modules/emqx/interfaces/` | TypeScript interfaces for all ExHook request/response types |
| `modules/emqx/enums/` | `AuthorizeReqType`, `ResponseType` enums |
| `app.proto` | ExHook gRPC service definition |
