# GraphQL Reference

The gateway exposes a full GraphQL API at `http://localhost:3010/graphql` powered by Apollo Server. Every collection that has a REST controller also has a GraphQL resolver with identical business logic, guards, and caching behavior.

**Playground:** `http://localhost:3010/graphql`

## Naming Convention

GraphQL operation names follow a predictable pattern:

```
{verb}{ServiceName}{CollectionName}[ById|Bulk]
```

| Verb | REST equivalent | Type |
|---|---|---|
| `count` | `GET /count` | Query |
| `create` | `POST /` | Mutation |
| `createBulk` | `POST /bulk` | Mutation |
| `find` | `GET /` | Query |
| `findById` | `GET /:id` | Query |
| `updateById` | `PATCH /:id` | Mutation |
| `updateBulk` | `PATCH /bulk` | Mutation |
| `deleteById` | `DELETE /:id` | Mutation |
| `restoreById` | `PUT /:id/restore` | Mutation |
| `destroyById` | `DELETE /:id/destroy` | Mutation |

**Example** — `identity/users` collection:

| Operation | GraphQL name |
|---|---|
| Count | `countIdentityUser` |
| Create one | `createIdentityUser` |
| Create many | `createIdentityUserBulk` |
| Find list | `findIdentityUser` |
| Find by ID | `findIdentityUserById` |
| Update by ID | `updateIdentityUserById` |
| Update bulk | `updateIdentityUserBulk` |
| Soft delete | `deleteIdentityUserById` |
| Restore | `restoreIdentityUserById` |
| Hard delete | `destroyIdentityUserById` |

## Authentication

Pass the JWT in the `Authorization` HTTP header exactly as with REST:

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

In the Apollo Playground, use the **Headers** tab at the bottom.

## Query Examples — Identity Users

### Count

```graphql
query CountUsers($filter: QueryFilterDto!) {
  countIdentityUser(filter: $filter) {
    total
  }
}
```

Variables:

```json
{ "filter": { "query": {} } }
```

### Find List

```graphql
query FindUsers($filter: FilterDto!) {
  findIdentityUser(filter: $filter) {
    count
    data {
      id
      username
      email
      name
      created_at
    }
  }
}
```

Variables with pagination:

```json
{
  "filter": {
    "query": {},
    "pagination": {
      "limit": 10,
      "skip": 0,
      "sort": { "created_at": -1 }
    }
  }
}
```

### Find by ID

```graphql
query FindUserById($id: String!, $ref: String) {
  findIdentityUserById(id: $id, ref: $ref) {
    data {
      id
      username
      email
      name
      created_at
      updated_at
    }
  }
}
```

Variables:

```json
{ "id": "64a1b2c3d4e5f6a7b8c9d0e1" }
```

### Create One

```graphql
mutation CreateUser($data: CreateUserDto!) {
  createIdentityUser(data: $data) {
    data {
      id
      username
      email
      created_at
    }
  }
}
```

Variables:

```json
{
  "data": {
    "username": "jdoe",
    "email": "jdoe@example.com",
    "password": "Str0ng!Pass",
    "name": "John Doe"
  }
}
```

### Create Bulk

```graphql
mutation CreateUsersBulk($data: CreateUserItemsDto!) {
  createIdentityUserBulk(data: $data) {
    count
    data {
      id
      username
    }
  }
}
```

Variables:

```json
{
  "data": {
    "items": [
      { "username": "alice", "email": "alice@example.com" },
      { "username": "bob",   "email": "bob@example.com" }
    ]
  }
}
```

### Update by ID

```graphql
mutation UpdateUser($id: String!, $data: UpdateUserDto!, $ref: String) {
  updateIdentityUserById(id: $id, data: $data, ref: $ref) {
    data {
      id
      username
      name
      updated_at
    }
  }
}
```

Variables:

```json
{
  "id": "64a1b2c3d4e5f6a7b8c9d0e1",
  "data": { "name": "Jonathan Doe" }
}
```

### Update Bulk

```graphql
mutation UpdateUsersBulk($filter: QueryFilterDto!, $data: UpdateUserDto!) {
  updateIdentityUserBulk(filter: $filter, data: $data) {
    total
  }
}
```

Variables:

```json
{
  "filter": { "query": { "status": "inactive" } },
  "data": { "archived": true }
}
```

### Soft Delete

```graphql
mutation DeleteUser($id: String!, $ref: String) {
  deleteIdentityUserById(id: $id, ref: $ref) {
    data {
      id
      deleted_at
    }
  }
}
```

### Restore

```graphql
mutation RestoreUser($id: String!, $ref: String) {
  restoreIdentityUserById(id: $id, ref: $ref) {
    data {
      id
      deleted_at
    }
  }
}
```

### Hard Delete (Destroy)

```graphql
mutation DestroyUser($id: String!, $ref: String) {
  destroyIdentityUserById(id: $id, ref: $ref) {
    data {
      id
    }
  }
}
```

> Requires `manage:identity:users` scope.

## Using `curl` with GraphQL

```bash
curl -X POST http://localhost:3010/graphql \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { findIdentityUser(filter: { query: {} }) { count data { id username email } } }"
  }'
```

With variables:

```bash
curl -X POST http://localhost:3010/graphql \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query FindById($id: String!) { findIdentityUserById(id: $id) { data { id username email } } }",
    "variables": { "id": "64a1b2c3d4e5f6a7b8c9d0e1" }
  }'
```

## Operation Name Reference

### Auth Service

> Note: The `/auth` controller does not follow the standard CRUD pattern and has no resolver.

### Identity Service

| Collection | Prefix | Example operation |
|---|---|---|
| Users | `IdentityUser` | `findIdentityUser` |
| Profiles | `IdentityProfile` | `createIdentityProfile` |
| Sessions | `IdentitySession` | `deleteIdentitySessionById` |

### Financial Service

| Collection | Prefix | Example operation |
|---|---|---|
| Accounts | `FinancialAccount` | `findFinancialAccount` |
| Wallets | `FinancialWallet` | `updateFinancialWalletById` |
| Invoices | `FinancialInvoice` | `createFinancialInvoice` |
| Transactions | `FinancialTransaction` | `findFinancialTransaction` |
| Currencies | `FinancialCurrency` | `createFinancialCurrency` |

### Career Service

| Collection | Prefix | Example operation |
|---|---|---|
| Businesses | `CareerBusiness` | `findCareerBusiness` |
| Branches | `CareerBranch` | `createCareerBranch` |
| Employees | `CareerEmployee` | `updateCareerEmployeeById` |
| Products | `CareerProduct` | `deleteCareerProductById` |
| Services | `CareerService` | `findCareerService` |
| Stocks | `CareerStock` | `createCareerStock` |
| Stores | `CareerStore` | `findCareerStore` |
| Customers | `CareerCustomer` | `createCareerCustomer` |

### Domain Service

| Collection | Prefix | Example operation |
|---|---|---|
| Apps | `DomainApp` | `findDomainApp` |
| Clients | `DomainClient` | `createDomainClient` |

### Essential Service

| Collection | Prefix | Example operation |
|---|---|---|
| Sagas | `EssentialSaga` | `findEssentialSaga` |
| Saga Stages | `EssentialSagaStage` | `findEssentialSagaStageById` |

### Context Service

| Collection | Prefix | Example operation |
|---|---|---|
| Configs | `ContextConfig` | `createContextConfig` |
| Settings | `ContextSetting` | `updateContextSettingById` |

### General Service

| Collection | Prefix | Example operation |
|---|---|---|
| Activities | `GeneralActivity` | `findGeneralActivity` |
| Artifacts | `GeneralArtifact` | `createGeneralArtifact` |
| Comments | `GeneralComment` | `findGeneralComment` |
| Events | `GeneralEvent` | `createGeneralEvent` |
| Workflows | `GeneralWorkflow` | `findGeneralWorkflow` |

### Special Service

| Collection | Prefix | Example operation |
|---|---|---|
| Files | `SpecialFile` | `findSpecialFile` |
| Stats | `SpecialStat` | `findSpecialStat` |

### Touch Service

| Collection | Prefix | Example operation |
|---|---|---|
| Emails | `TouchEmail` | `createTouchEmail` |
| Notices | `TouchNotice` | `findTouchNotice` |
| Pushes | `TouchPush` | `createTouchPush` |
| SMS | `TouchSms` | `createTouchSms` |
| Histories | `TouchHistory` | `findTouchHistory` |

### Content Service

| Collection | Prefix | Example operation |
|---|---|---|
| Notes | `ContentNote` | `createContentNote` |
| Posts | `ContentPost` | `findContentPost` |
| Tickets | `ContentTicket` | `createContentTicket` |

### Logistic Service

| Collection | Prefix | Example operation |
|---|---|---|
| Locations | `LogisticLocation` | `findLogisticLocation` |
| Drivers | `LogisticDriver` | `createLogisticDriver` |
| Vehicles | `LogisticVehicle` | `findLogisticVehicle` |
| Travels | `LogisticTravel` | `createLogisticTravel` |
| Cargoes | `LogisticCargo` | `findLogisticCargo` |

### Conjoint Service

| Collection | Prefix | Example operation |
|---|---|---|
| Accounts | `ConjointAccount` | `findConjointAccount` |
| Channels | `ConjointChannel` | `createConjointChannel` |
| Contacts | `ConjointContact` | `findConjointContact` |
| Members | `ConjointMember` | `createConjointMember` |
| Messages | `ConjointMessage` | `findConjointMessage` |

### Thing Service

| Collection | Prefix | Example operation |
|---|---|---|
| Devices | `ThingDevice` | `createThingDevice` |
| Sensors | `ThingSensor` | `findThingSensor` |
| Metrics | `ThingMetric` | `createThingMetric` |

## Filter Input Types

GraphQL reuses the same filter system as REST. The input types map to:

| Input Type | Fields | Usage |
|---|---|---|
| `QueryFilterDto` | `query` | For `count` and `updateBulk` |
| `FilterDto` | `query`, `populate`, `projection`, `pagination` | For `find` and `updateBulk` |
| `FilterOneDto` | `query`, `populate`, `projection` | For single-entity operations |

See [Filtering & Pagination](./filtering.md) for the complete field reference.

## Response Types

GraphQL returns typed serializers. The shape mirrors the REST response envelope:

| Return type | Shape |
|---|---|
| `TotalSerializer` | `{ total: Int }` |
| `UserDataSerializer` | `{ data: User }` |
| `UserItemsSerializer` | `{ data: [User], count: Int }` |

All entity types include the [common platform fields](./rest-reference.md#common-fields-on-every-document).
