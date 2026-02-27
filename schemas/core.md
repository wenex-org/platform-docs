```yaml
---
title: "Wenex Documentation Architecture & Core Implementation"
author: "Principal Tech Lead & AI-Native Technical Writer"
context: "Platform Setup, MCP Architecture, Dual-Purpose Docs"
version: "1.0.0"
---
```


```
---
title: "Wenex Core Interface & Access Model"
module: "schemas"
dependencies: ["identity.users", "domain.clients"]
mcp-context: true
tags: ["core", "rbac", "abac", "multi-tenancy"]
---
```

```xml
<context>
The `Core` interface defines the standardized root schema shared by all domain entities across Wenex microservices. It guarantees deterministic identification, auditing, multi-tenant isolation, and an ABAC permission model.
</context>
```

## 1. Core Data Structure
```xml
<data-structure name="CoreEntity">
  <purpose>Defines mandatory fields for all database records ensuring auditing, access control, and optimistic concurrency.</purpose>
  
  <schema-mapping>
     primary_key: `id` (MongoDB ObjectId, hex string)
     versioning: `version` (semantic), `rand` (cache-busting), `timestamp` (ISO/Unix)
     isolation: `clients[]` (Mandatory multi-tenancy array)
     soft_delete: `deleted_at`, `deleted_by`, `deleted_in`
  </schema-mapping>
</data-structure>
```

### Field Specifications


The following documentation defines the foundational `Core` interface shared across all domain entities in Wenex and details the sophisticated, multi-layered Access Control Model. This model acts as an impenetrable shield, evaluating and sanitizing all REST, GraphQL, and MCP requests via the `Authority Interceptor` before they reach any service or database.

## Key Fields in Core Interface
```xml
<data-structure>
The Core interface defines a standardized set of fields shared by all domain entities across Wenex services. These fields ensure consistency in identification, ownership, access control, auditing, and extensibility.
</data-structure>
```

| Field          | Type               | Required | Description                                                                 | Example / Notes                                      |
|----------------|--------------------|----------|-----------------------------------------------------------------------------|------------------------------------------------------|
| `id`           | string             | yes      | Primary identifier (MongoDB ObjectId as hex string)                         | Auto-generated on creation                           |
| `ref`          | string             | no       | Optional external reference or business key used for integration with external systems, legacy databases, or internal cross-linking between Wenex entities/services. Must be unique within the scope where synchronization is required.| External system ID (e.g. CRM record ID), invoice number, order ID, SKU, purchase order number. Enables data mapping and bidirectional sync without relying on internal id. Indexed as unique + sparse.|
| `owner`        | string             | yes      | User ID (from `identity.users`) who owns the record                         | Single owner only — grants full "own" permissions    |
| `shares`       | string[]           | no       | List of user IDs explicitly granted share-level access                      | Permissions based on "share" actions                 |
| `groups`       | string[]           | no       | List of group identifiers (email domain, FQDN, or MongoID)                  | Matched against `groups` field in `identity.users`   |
| `clients`      | string[]           | yes      | List of allowed client IDs (from `domain.clients`)                          | Enforces application-level isolation                 |
| `created_at`   | Date               | yes      | Creation timestamp                                                          | Auto-generated on creation                           |
| `created_by`   | string             | yes      | User ID who created the record                                              | Auto-generated on creation                           |
| `created_in`   | string             | yes      | Client ID where creation occurred                                           | Auto-generated on creation                           |
| `updated_at`   | Date               | no       | Last update timestamp                                                       | Auto-generated on creation                           |
| `updated_by`   | string             | no       | User ID who last updated                                                    | Auto-generated on creation                           |
| `updated_in`   | string             | no       | Client ID where update occurred                                             | Auto-generated on creation                           |
| `deleted_at`   | Date               | no       | Soft-delete timestamp                                                       | Record hidden from normal queries                    |
| `deleted_by`   | string             | no       | User ID who performed soft-delete                                           | Auto-generated on creation                           |
| `deleted_in`   | string             | no       | Client ID where delete occurred                                             | Auto-generated on creation                           |
| `restored_at`  | Date               | no       | Restore timestamp (undo soft-delete)                                        | Auto-generated on creation                           |
| `restored_by`  | string             | no       | User ID who restored                                                        | Auto-generated on creation                           |
| `restored_in`  | string             | no       | Client ID where restore occurred                                            | Auto-generated on creation                           |
| `description`  | string             | no       | Optional free-form description                                              | Auto-generated on creation                           |
| `identity`     | string             | no       | Optional link to `identity.profiles` for display data                       | Avatar, full name, etc.                              |
| `props`        | Properties         | no       | Domain-specific extension properties (generic)                              | Varies per entity type                               |
| `tags`         | string[]           | no       | Free-form tags for filtering/search                                         | `["urgent", "europe", "q1-2026"]`                    |
| `version`      | string             | yes      | Semantic or incremental version                                             | Optimistic concurrency control                       |
| `rand`         | string             | yes      | Random string for cache-busting                                             | Changes on every update                              |
| `timestamp`    | string             | yes      | ISO string or Unix timestamp for ordering & change detection                | Auto-generated on creation                           |


## Access Control Model


The `Core` interface implements a **layered ABAC** model with four distinct access scopes. It operates on all REST, GraphQL, and MCP requests. Permissions are evaluated in order: **Owner → Shares → Groups → Clients**. The first matching layer that grants the requested action determines access. Every entity inheriting the `Core` interface automatically enforces these four concepts.


### 1. Clients (`clients`)
- **Concept:** Represents the highest level of isolation and multi-tenancy in the platform. A record belongs exactly to one (or multiple) clients. 
- **Enforcement:** Mandatory array of allowed client IDs (MongoId) from `domain.clients`. Upon creation, this field is automatically populated based on the client ID present in the request's access token.
- **Security Constraint:** For any non-read operation, the system forcibly injects the current user's `clientId` into the query. A user can **never** create, update, or delete data belonging to another client. If the request `Zone` includes `client`, the user can access all data within their client (provided they possess the required Grant).
- Supported actions: `create:client`, `read:client`, `update:client`, `delete:client`, `restore:client`, `destroy:client`

### 2. Owner (`owner`)
- **Concept:** The primary and individual owner of the record.
- **Enforcement:** Single user ID (MongoId) from `identity.users`. Upon creation, the system automatically sets this field to the `uid` extracted from the access token. Even if a different owner is submitted in the DTO, the system ignores it and enforces the current user's ID (unless the user has explicit elevated permissions to assign ownership).
- **Security Constraint:** Implicitly receives full permissions on **own** resources based on their assigned Grants.
- Supported actions: `create:own`, `read:own`, `update:own`, `delete:own`, `restore:own`, `destroy:own`

### 3. Shares (`shares`)
- **Concept:** An explicit and precise sharing mechanism. A record can be shared with specific users or other clients.
- **Enforcement:** Explicit array of user IDs. Records shared via this array become accessible to the target user when they query using the `share` Zone.
- Supported actions: `create:share`, `read:share`, `update:share`, `delete:share`, `restore:share`, `destroy:share`

### 4. Groups (`groups`)
- **Concept:** Mechanism for collective access management. A record can be attached to one or multiple groups.
- **Enforcement:** Array of group identifiers. The `Authority Interceptor` always verifies the current user's group memberships (from `identity.users.groups`) and automatically appends the necessary filters to the query.
- Supported actions: `create:group`, `read:group`, `update:group`, `delete:group`, `restore:group`, `destroy:group`

---

## The `Zone` Concept

The **`Zone`** is a key parameter used to define the user's scope of visibility over the data. It is extracted directly from the HTTP Request Headers.


Supported Zones:
- **`own`** *(Default)*: Strictly records directly owned by the user (`owner` equals current user ID).
- **`share`** *(Default)*: Records explicitly shared with the current user via the `shares` field.
- **`group`**: Records belonging to groups the user is an active member of (determined via `identity.users.groups`).
- **`client`**: Full access to all data within the current client boundary (Requires specific permission Grants).

---


**Pipeline Steps Explained:**
1. **Extract Zone:** Reads the intended data scope (`own`, `share`, `group`, `client`) from the headers.
2. **Pre-cleansing Query:** Removes all unauthorized, dangerous, or extraneous fields from the incoming payload that must not reach MongoDB.
3. **Grant & Scope Verification:** Validates if the user possesses the necessary Grants and Scopes (often identical to Client Scopes extracted from the token) to perform the action on the requested resource.
4. **Apply Zone & Ownership Rules:** The interceptor branches logic based on the operation type (See *Read vs. Non-Read* below).
5. **Apply Group Memberships:** Verifies the user's group memberships and injects the corresponding filters.

---

## Read vs. Non-Read Operations

<security-rules>
Both operation categories require an explicit Permission Grant. However, the `Authority Interceptor` handles them with vastly different strictness levels regarding multi-tenancy.
</security-rules>

- **Read Operations (Flexible):** If the developer/user does not explicitly request the `client` Zone, the `clients` filter is **not** injected. Consequently, the query defaults to evaluating the `own` + `share` zones, making it perfectly possible to access records shared from *other clients* (Cross-tenant sharing).
- **Non-Read Operations (Strict):** Operations like Create, Update, and Delete are highly restricted. The `clients` filter is **always forcibly injected** into the query, regardless of whether the `client` Zone was requested. A user can *only* mutate data belonging to their current client context.



## 2. Access Control Model

<access-model>
  <type>Layered ABAC</type>
  <enforcement>Auth Service (Grants + APT/JWT Claims)</enforcement>
  <resolution-order>
    1. Owner (Highest privilege, intrinsic)
    2. Shares (Explicit discrete delegation)
    3. Groups (Attribute-based logical grouping)
    4. Clients (Tenant-wide base access)
  </resolution-order>
</access-model>


<llm-reasoning>
MCP Agent Decision Rules for Permissions
When an AI Agent or MCP tool evaluates permissions on a Wenex entity, it MUST follow these deterministic rules:

1. **Rule of Least Privilege**: Never assume `owner` scope. Always verify if the `actor.id` exactly matches `entity.owner`.
2. **Tenant Isolation**: If the request's client context is NOT present in the `entity.clients[]` array, immediately reject the action (Multi-tenancy violation).
3. **Soft Deletion Protocol**: If `deleted_at` is populated, the entity is considered destroyed for standard `read` actions. Only `restore:*` or `destroy:*` (hard delete) actions are permitted based on scope.
4. **Scope Mapping**:
   - `create/read/update/delete:own` -> requires `actor.id === owner`
   - `create/read/update/delete:share` -> requires `actor.id IN shares[]`
</llm-reasoning>