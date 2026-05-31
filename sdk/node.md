# Node SDK — `@wenex/sdk`

The `@wenex/sdk` package is the official TypeScript/JavaScript client for the Wenex Platform. It wraps the REST API with typed methods, handles Brotli compression, and exposes RxJS-compatible patterns.

**npm:** [`@wenex/sdk`](https://www.npmjs.com/package/@wenex/sdk)
**Version:** 1.3.4
**Repository:** [wenex-org/platform-sdk](https://github.com/wenex-org/platform-sdk)

---

## Installation

```bash
npm install @wenex/sdk axios
# or
pnpm add @wenex/sdk axios
```

Optional: Brotli compression support (reduces network payload):

```bash
npm install @wenex/sdk axios brotli-wasm
```

---

## Quick Start

```typescript
import axios from 'axios';
import { Platform } from '@wenex/sdk';

// 1. Create an axios instance configured for the gateway
const http = axios.create({
  baseURL: 'http://localhost:3010',
  headers: {
    Authorization: `Bearer ${process.env.WENEX_TOKEN}`,
  },
});

// 2. Build the Platform client
const platform = Platform.build(http);

// 3. Use any service
const users = await platform.identity.users.find({ query: {} });
console.log(users); // User[]
```

---

## Platform Class

`Platform` is the root client. It lazily instantiates service clients on first access.

```typescript
class Platform {
  // Service clients (lazily initialized)
  get auth():      Auth.Client
  get identity():  Identity.Client
  get financial(): Financial.Client
  get career():    Career.Client
  get domain():    Domain.Client
  get essential(): Essential.Client
  get context():   Context.Client
  get general():   General.Client
  get special():   Special.Client
  get touch():     Touch.Client
  get content():   Content.Client
  get logistic():  Logistic.Client
  get conjoint():  Conjoint.Client
  get thing():     Thing.Client
  get graphql():   GraphqlService

  static build(axios: AxiosInstance, prefix?: string): Platform
}
```

The optional `prefix` parameter is prepended to all request paths (useful when the API is served under a sub-path).

---

## Authentication

### With a JWT

```typescript
const http = axios.create({
  baseURL: 'http://localhost:3010',
  headers: { Authorization: `Bearer ${jwtToken}` },
});
const platform = Platform.build(http);
```

### With an APT (long-lived token)

```typescript
const http = axios.create({
  baseURL: 'http://localhost:3010',
  headers: { Authorization: `Bearer ${aptToken}` },
});
const platform = Platform.build(http);
```

### Obtaining a token via the SDK

```typescript
const platform = Platform.build(axios.create({ baseURL: 'http://localhost:3010' }));

const { access_token } = await platform.auth.auths.token({
  username: 'admin@example.com',
  password: 'Str0ng!Pass',
  grant_type: 'password',
});

// Re-build with the token
const authedPlatform = Platform.build(
  axios.create({
    baseURL: 'http://localhost:3010',
    headers: { Authorization: `Bearer ${access_token}` },
  }),
);
```

---

## RestfulService Methods

Every collection (users, accounts, products, etc.) extends `RestfulService`, which provides these typed methods:

```typescript
class RestfulService<T extends Core, D extends Dto<Core>> {
  // Count documents matching a query
  count(query: Query<T>, config?: RequestConfig<T>): Promise<number>

  // Create one document
  create(data: D, config?: RequestConfig<T>): Promise<Serializer<T>>

  // Create many documents
  createBulk(data: Items<D>, config?: RequestConfig<T>): Promise<Serializer<T>[]>

  // Stream documents via SSE
  cursor(filter: FilterOne<T>, options: FetchEventSourceInit, config?: RequestConfig<T>): Promise<void>

  // Find documents with full filter support
  find(filter: Filter<T>, config?: RequestConfig<T>): Promise<Serializer<T>[]>

  // Find one document by ID
  findById(id: string, config?: RequestConfig<T>): Promise<Serializer<T>>

  // Update one document by ID
  updateById(id: string, data: Optional<D>, config?: RequestConfig<T>): Promise<Serializer<T>>

  // Update many documents matching a query
  updateBulk(data: Optional<D>, query: Query<T>, config?: RequestConfig<T>): Promise<number>

  // Soft-delete by ID
  deleteById(id: string, config?: RequestConfig<T>): Promise<Serializer<T>>

  // Restore a soft-deleted document
  restoreById(id: string, config?: RequestConfig<T>): Promise<Serializer<T>>

  // Hard-delete by ID (permanent)
  destroyById(id: string, config?: RequestConfig<T>): Promise<Serializer<T>>
}
```

---

## RequestConfig Options

The `config` parameter is an extension of Axios `AxiosRequestConfig`:

```typescript
interface RequestConfig<T> extends AxiosRequestConfig {
  params?: {
    zone?:  'own' | 'share' | 'group' | 'client' | string; // comma-separated
    skip?:  number;
    limit?: number;
    sort?:  Pagination<T>['sort'];
    [key: string]: any;
  };
  headers?: {
    [k in keyof Metadata]?: string | number | boolean | null;
  };
  fullResponse?: boolean; // If true, returns the full axios response instead of unwrapped data
  brotli?: { quality: number } | boolean; // Enable Brotli compression
}
```

---

## CRUD Examples

### Identity — Users

```typescript
const users = platform.identity.users;

// Count all users
const total = await users.count({});
console.log(`Total users: ${total}`);

// Count with filter
const activeCount = await users.count({ status: 'active' });

// Create a user
const newUser = await users.create({
  username: 'jdoe',
  email: 'jdoe@example.com',
  password: 'Str0ng!Pass',
  name: 'John Doe',
});
console.log(newUser.id);

// Create many users
const created = await users.createBulk({
  items: [
    { username: 'alice', email: 'alice@example.com' },
    { username: 'bob',   email: 'bob@example.com' },
  ],
});

// Find with filter and pagination
const page1 = await users.find({
  query: { status: 'active' },
  pagination: { limit: 20, skip: 0, sort: { created_at: -1 } },
  projection: { username: 1, email: 1 },
});

// Find one by ID
const user = await users.findById('64a1b2c3d4e5f6a7b8c9d0e1');

// Update
const updated = await users.updateById('64a1b2c3d4e5f6a7b8c9d0e1', {
  name: 'Jonathan Doe',
});

// Bulk update
const count = await users.updateBulk(
  { status: 'inactive' },
  { archived: true },
);

// Soft delete
const deleted = await users.deleteById('64a1b2c3d4e5f6a7b8c9d0e1');

// Restore
const restored = await users.restoreById('64a1b2c3d4e5f6a7b8c9d0e1');

// Hard delete (permanent — requires manage scope)
await users.destroyById('64a1b2c3d4e5f6a7b8c9d0e1');
```

### Financial — Transactions

```typescript
const txns = platform.financial.transactions;

// Find completed transactions over $100 with account populated
const results = await txns.find({
  query: { status: 'completed', amount: { $gte: 100 } },
  populate: [{ path: 'account', select: ['name', 'balance'] }],
  pagination: { limit: 50, skip: 0, sort: { created_at: -1 } },
});
```

### Auth — Token and Verify

```typescript
const auth = platform.auth.auths;

// Get token
const { access_token } = await auth.token({
  username: 'admin@example.com',
  password: 'secret',
  grant_type: 'password',
});

// Verify token
const claims = await auth.verify();
console.log(claims.sub, claims.scope);

// Check permission
const { can } = await auth.can({ action: 'read', resource: 'identity:users' });

// Logout
await auth.logout();
```

### Auth — APTs

```typescript
const apts = platform.auth.apts;

// Create an APT
const apt = await apts.create({
  name: 'my-bot',
  scope: 'read:identity:users',
});
console.log(apt.token); // Store this — only shown once

// List APTs
const myApts = await apts.find({ query: {} });

// Revoke an APT
await apts.deleteById(apt.id);
```

---

## Streaming (Cursor)

```typescript
import { fetchEventSource } from '@microsoft/fetch-event-source';

await platform.identity.users.cursor(
  { query: { status: 'active' } },
  {
    onmessage(event) {
      if (event.event === 'end') {
        console.log('Stream complete');
        return;
      }
      const user = JSON.parse(event.data);
      console.log('User:', user.id, user.username);
    },
    onerror(err) {
      console.error(err);
      throw err; // Re-throw to stop retries
    },
  },
);
```

---

## GraphQL via SDK

The SDK exposes a `graphql` client for executing raw GraphQL operations:

```typescript
const gql = platform.graphql;

const result = await gql.query(`
  query {
    findIdentityUser(filter: { query: {} }) {
      count
      data { id username email }
    }
  }
`);

// With variables
const result2 = await gql.query(
  `query FindById($id: String!) {
    findIdentityUserById(id: $id) { data { id username email } }
  }`,
  { id: '64a1b2c3d4e5f6a7b8c9d0e1' },
);
```

---

## Zone Filtering

Pass `zone` in `config.params`:

```typescript
// My own documents only
const mine = await platform.identity.users.find(
  { query: {} },
  { params: { zone: 'own' } },
);

// Shared with me
const shared = await platform.identity.users.find(
  { query: {} },
  { params: { zone: 'own,share' } },
);
```

---

## Full Response Mode

By default the SDK unwraps the response envelope and returns the data directly. Set `fullResponse: true` to receive the raw Axios response:

```typescript
const response = await platform.identity.users.find(
  { query: {} },
  { fullResponse: true },
);
// response.data = { data: [...], count: N }
// response.headers['etag'] = '"abc123"'
```

---

## Brotli Compression

Enable Brotli to compress request bodies (requires `brotli-wasm` peer dependency):

```typescript
await platform.identity.users.create(
  { username: 'alice', email: 'alice@example.com' },
  { brotli: { quality: 6 } }, // quality 1–11
);

// Boolean shorthand (uses default quality)
await platform.identity.users.create(data, { brotli: true });
```

---

## Multi-Tenant Usage

Pass the `x-domain` header to scope requests to a specific tenant:

```typescript
const tenantHttp = axios.create({
  baseURL: 'http://localhost:3010',
  headers: {
    Authorization: `Bearer ${token}`,
    'x-domain': 'tenant-a.example.com',
  },
});
const tenantPlatform = Platform.build(tenantHttp);
```

Or override per-request:

```typescript
await platform.identity.users.find(
  { query: {} },
  { headers: { 'x-domain': 'tenant-b.example.com' } },
);
```

---

## TypeScript Tips

The `Platform` class accepts a generic `Properties` type parameter for extending entity types with custom properties:

```typescript
interface MyUserProps {
  preferredLanguage: string;
  tier: 'free' | 'pro' | 'enterprise';
}

const platform = Platform.build<MyUserProps>(http);

// platform.identity.users.find() returns User<MyUserProps>[]
const users = await platform.identity.users.find({ query: {} });
users[0].props?.preferredLanguage; // typed
```

---

## Service Client Reference

| `platform.X` | Collections available |
| --- | --- |
| `platform.auth` | `.auths`, `.apts`, `.grants` |
| `platform.identity` | `.users`, `.profiles`, `.sessions` |
| `platform.financial` | `.accounts`, `.wallets`, `.invoices`, `.transactions`, `.currencies` |
| `platform.career` | `.businesses`, `.branches`, `.employees`, `.products`, `.services`, `.stocks`, `.stores`, `.customers` |
| `platform.domain` | `.apps`, `.clients` |
| `platform.essential` | `.sagas`, `.sagaStages` |
| `platform.context` | `.configs`, `.settings` |
| `platform.general` | `.activities`, `.artifacts`, `.comments`, `.events`, `.workflows` |
| `platform.special` | `.files`, `.stats` |
| `platform.touch` | `.emails`, `.notices`, `.pushes`, `.sms`, `.histories` |
| `platform.content` | `.notes`, `.posts`, `.tickets` |
| `platform.logistic` | `.locations`, `.drivers`, `.vehicles`, `.travels`, `.cargoes` |
| `platform.conjoint` | `.accounts`, `.channels`, `.contacts`, `.members`, `.messages` |
| `platform.thing` | `.devices`, `.sensors`, `.metrics` |

---

## Customizing the SDK

`Platform` is designed to be extended. You can build a custom client that adds domain-specific services, overrides existing ones with extra methods, and sets a path prefix for all requests.

### Extending Platform

```typescript
import { Platform } from '@wenex/sdk';
import type { AxiosInstance } from 'axios';

export class CustomClient<Properties extends object = object> extends Platform<Properties> {
  constructor(axios: AxiosInstance) {
    super(axios, '/prefix/'); // optional — prepended to every request path
  }

  protected _orders?: OrdersService;

  get orders() {
    return (this._orders ??= OrdersService.build(this.axios));
  }

  static override build<Properties extends object = object>(axios: AxiosInstance) {
    return new CustomClient<Properties>(axios);
  }
}
```

### Adding a New Service

A service wraps one or more HTTP endpoints using the shared axios instance:

```typescript
import type { AxiosInstance } from 'axios';
import { RequestConfig } from '@wenex/sdk/common/core/types';

export class OrdersService {
  protected readonly url = (path?: string) => `/orders${path ? `/${path}` : ''}`;

  constructor(protected axios: AxiosInstance) {}

  list(config?: RequestConfig): Promise<Order[]> {
    return this.axios.get<Order[]>(this.url(), config).then(r => r.data);
  }

  create(data: CreateOrderDto, config?: RequestConfig): Promise<Order> {
    return this.axios.post<Order>(this.url(), data, config).then(r => r.data);
  }

  static build(axios: AxiosInstance) {
    return new OrdersService(axios);
  }
}
```

### Extending an Existing Collection

Extend a built-in service class to add extra methods while keeping all standard CRUD operations:

```typescript
import { UsersService } from '@wenex/sdk/services/identity';
import { RequestConfig } from '@wenex/sdk/common/core/types';

export class CustomUsersService<Properties extends object = object> extends UsersService<Properties> {
  constructor(axios: AxiosInstance) {
    super(axios, '/prefix/');
  }

  search(query: string, config?: RequestConfig): Promise<User[]> {
    return this.post(this.url('search'), { query }, config);
  }

  static override build<Properties extends object = object>(axios: AxiosInstance) {
    return new CustomUsersService<Properties>(axios);
  }
}
```

Then override the collection inside a custom domain client:

```typescript
import { IdentityClient } from '@wenex/sdk';

export class CustomIdentityClient<Properties extends object = object> extends IdentityClient<Properties> {
  protected override _users?: CustomUsersService<Properties>;

  constructor(axios: AxiosInstance) {
    super(axios, '/prefix/');
  }

  override get users() {
    return (this._users ??= CustomUsersService.build<Properties>(this.axios));
  }

  static override build<Properties extends object = object>(axios: AxiosInstance) {
    return new CustomIdentityClient<Properties>(axios);
  }
}
```

Wire the custom domain client back into the top-level client:

```typescript
export class CustomClient<Properties extends object = object> extends Platform<Properties> {
  protected override _identity?: CustomIdentityClient<Properties>;

  override get identity() {
    return (this._identity ??= CustomIdentityClient.build<Properties>(this.axios));
  }
  // ...
}
```

### Usage

```typescript
const client = CustomClient.build(http);

// All platform services work as normal
const users = await client.identity.users.find({ query: {} });

// Extended method available
const results = await client.identity.users.search('jane');

// Custom service available
const orders = await client.orders.list();
```
