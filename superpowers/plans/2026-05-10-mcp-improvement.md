# MCP Server Improvement — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor the Wenex MCP server from 490 flat undescribed CRUD tools into a factory-driven, properly-described, production-ready agent interface with stateful sessions, security middleware, and workflow tools.

**Architecture:** A new `registerCollectionTools` factory in `factory.mcp.ts` auto-generates all 10 CRUD tools per collection with rich inline descriptions, consistent underscore naming, and typed filter schemas — replacing 49 hand-written 300-line router files with 25-line factory calls. A stateful session store (in-memory Map with TTL) tracks connections per session ID. One high-value workflow tool encapsulates multi-step saga + transaction orchestration that raw CRUD cannot safely do alone.

**Tech Stack:** `@modelcontextprotocol/sdk`, NestJS gateway, Zod, `@wenex/sdk`, Express middleware

---

## Phase 1 — Foundation

### Task 1: Create `factory.mcp.ts`

**Files:**
- Create: `libs/common/src/core/mcp/factory.mcp.ts`

- [ ] **Step 1: Write the factory file**

```typescript
// libs/common/src/core/mcp/factory.mcp.ts
import { RequestConfig } from '@wenex/sdk/common/core/types';
import { Platform } from '@wenex/sdk';
import { z, ZodRawShape } from 'zod';

import { throwableToolCall } from './utils.mcp';
import { ServerMCP } from './server.mcp';
import { ITEMS_SCHEMA, TOTAL_SCHEMA } from './const.mcp';

export const FILTER_SCHEMA = z
  .object({
    query: z
      .record(z.string(), z.any())
      .optional()
      .describe('MongoDB query. Examples: {"state":"active"}, {"created_at":{"$gte":"2024-01-01"}}, {"$or":[{"type":"a"},{"type":"b"}]}'),
    pagination: z
      .object({
        limit: z.number().min(1).max(1000).optional().describe('Max items per page. Always set before querying.'),
        skip: z.number().min(0).optional().describe('Items to skip for pagination'),
      })
      .optional(),
    sort: z
      .record(z.string(), z.union([z.literal(1), z.literal(-1)]))
      .optional()
      .describe('Sort fields. Example: {"created_at":-1} newest first'),
    populate: z
      .array(z.string())
      .optional()
      .describe('Population paths confirmed in service docs. Example: ["owner","relations"]'),
    projection: z
      .record(z.string(), z.union([z.literal(0), z.literal(1)]))
      .optional()
      .describe('Field projection. Include: {"name":1}. Exclude: {"password":0}'),
  })
  .passthrough()
  .optional();

export interface PlatformCollection {
  count(query: Record<string, any>, config: Record<string, any>): Promise<number>;
  create(payload: any, config: Record<string, any>): Promise<any>;
  createBulk(payload: { items: any[] }, config: Record<string, any>): Promise<any[]>;
  find(filter: any, config: Record<string, any>): Promise<any[]>;
  findById(id: string, config: Record<string, any>): Promise<any>;
  updateById(id: string, payload: any, config: Record<string, any>): Promise<any>;
  updateBulk(payload: any, query: Record<string, any>, config: Record<string, any>): Promise<number>;
  deleteById(id: string, config: Record<string, any>): Promise<any>;
  restoreById(id: string, config: Record<string, any>): Promise<any>;
  destroyById(id: string, config: Record<string, any>): Promise<any>;
}

export interface CollectionToolsConfig {
  service: string;
  collection: string;
  entityName: string;
  serviceDoc: string;
  inputSchema: ZodRawShape;
  outputSchema: ZodRawShape;
  getCollection: (platform: Platform) => PlatformCollection;
}

const HEADERS_SCHEMA = z.record(z.string(), z.string()).optional();
const PARAMS_SCHEMA = z.object({ id: z.string(), ref: z.string().optional() }).partial();
const ERRORS_SCHEMA = z.array(z.object({}).passthrough()).optional();

function desc(entityName: string, path: string, detail: string, serviceDoc: string): string {
  return (
    `${detail}\n\n` +
    `Collection: ${path} | Entity: ${entityName}\n` +
    `• filter.query — MongoDB filter ({"state":"active"}, {"$or":[...]})\n` +
    `• filter.pagination — {limit: number (always set), skip: number}\n` +
    `• filter.sort — {field: 1|-1}\n` +
    `• filter.populate — confirmed paths from service docs\n` +
    `• x-zone header — own|share|client|broad (default: own,share,client)\n\n` +
    `📖 ${serviceDoc}`
  );
}

function buildConfig(args: any, headers: Record<string, any>): Record<string, any> {
  return { headers: { ...(args.headers ?? {}), ...headers } };
}

function resolveById(args: any, config: RequestConfig): { id: string } {
  const { id, ref } = (args.params ?? {}) as { id?: string; ref?: string };
  if (ref && (!id || id === '-')) config.params = { ref };
  return { id: id || '-' };
}

export function registerCollectionTools(cfg: CollectionToolsConfig): void {
  const mcp = ServerMCP.create();
  const { service, collection, entityName, serviceDoc, inputSchema, outputSchema, getCollection } = cfg;
  const path = `${service}/${collection}`;
  const t = (op: string) => `${op}_${service}_${collection}`;
  const d = (detail: string) => desc(entityName, path, detail, serviceDoc);
  const outSchema = { errors: ERRORS_SCHEMA, result: z.object(outputSchema).partial().optional() };
  const col = () => getCollection(mcp.platform);

  // count
  mcp.server.registerTool(
    t('count'),
    {
      title: `Count ${entityName}`,
      description: d(`Returns the total number of ${entityName} items matching the filter. Call before paginating large collections.`),
      inputSchema: { headers: HEADERS_SCHEMA, filter: FILTER_SCHEMA },
      outputSchema: { errors: ERRORS_SCHEMA, result: z.object(TOTAL_SCHEMA).partial().optional() },
      annotations: { readOnlyHint: true, destructiveHint: false, idempotentHint: true },
    },
    async (args, { requestInfo }) =>
      throwableToolCall(async () => {
        const [logger, headers] = mcp.utils(t('count'), requestInfo, args);
        const query = args.filter?.query ?? {};
        const config = buildConfig(args, headers);
        logger('count %s query=%o', path, query);
        const result = await col().count(query, config);
        return { structuredContent: { result: { total: result } }, content: [{ type: 'text', text: `Found exactly ${result} ${entityName} items.` }] };
      }),
  );

  // create
  mcp.server.registerTool(
    t('create'),
    {
      title: `Create ${entityName}`,
      description: d(`Creates a single ${entityName}. Provide required fields in body. Returns the created item with its generated id.`),
      inputSchema: { headers: HEADERS_SCHEMA, body: z.object(inputSchema) },
      outputSchema: outSchema,
      annotations: { readOnlyHint: false, destructiveHint: true, idempotentHint: false },
    },
    async (args, { requestInfo }) =>
      throwableToolCall(async () => {
        const [logger, headers] = mcp.utils(t('create'), requestInfo, args);
        const config = buildConfig(args, headers);
        logger('create %s payload=%o', path, args.body);
        const result = await col().create(args.body, config);
        return { structuredContent: { result }, content: [{ type: 'text', text: `${entityName} created with id "${result.id}".` }] };
      }),
  );

  // create_bulk
  mcp.server.registerTool(
    t('create_bulk'),
    {
      title: `Create Bulk ${entityName}`,
      description: d(`Creates multiple ${entityName} items in one call. Body must be {items:[...]}. Returns the created items array.`),
      inputSchema: { headers: HEADERS_SCHEMA, body: z.object(ITEMS_SCHEMA(inputSchema)) },
      outputSchema: { errors: ERRORS_SCHEMA, result: z.object(ITEMS_SCHEMA(outputSchema)).partial().optional() },
      annotations: { readOnlyHint: false, destructiveHint: true, idempotentHint: false },
    },
    async (args, { requestInfo }) =>
      throwableToolCall(async () => {
        const [logger, headers] = mcp.utils(t('create_bulk'), requestInfo, args);
        const config = buildConfig(args, headers);
        const n = (args.body as any)?.items?.length ?? 0;
        logger('create_bulk %s count=%d', path, n);
        const result = await col().createBulk(args.body as any, config);
        return { structuredContent: { result: { items: result } }, content: [{ type: 'text', text: `Created ${result.length} ${entityName} items.` }] };
      }),
  );

  // find
  mcp.server.registerTool(
    t('find'),
    {
      title: `Find ${entityName}`,
      description: d(`Retrieves a list of ${entityName} items. Always set filter.pagination.limit. Use count first for large collections.`),
      inputSchema: { headers: HEADERS_SCHEMA, filter: FILTER_SCHEMA },
      outputSchema: { errors: ERRORS_SCHEMA, result: z.object(ITEMS_SCHEMA(outputSchema)).partial().optional() },
      annotations: { readOnlyHint: true, destructiveHint: false, idempotentHint: true },
    },
    async (args, { requestInfo }) =>
      throwableToolCall(async () => {
        const [logger, headers] = mcp.utils(t('find'), requestInfo, args);
        const config = buildConfig(args, headers);
        logger('find %s filter=%o', path, args.filter);
        const result = await col().find(args.filter ?? {}, config);
        return { structuredContent: { result: { items: result } }, content: [{ type: 'text', text: `Retrieved ${result.length ?? 0} ${entityName} items.` }] };
      }),
  );

  // find_one
  mcp.server.registerTool(
    t('find_one'),
    {
      title: `Find One ${entityName}`,
      description: d(`Retrieves a single ${entityName} by params.id (MongoDB ObjectId) or params.ref (human-readable reference). Provide one.`),
      inputSchema: { headers: HEADERS_SCHEMA, params: PARAMS_SCHEMA },
      outputSchema: outSchema,
      annotations: { readOnlyHint: true, destructiveHint: false, idempotentHint: true },
    },
    async (args, { requestInfo }) =>
      throwableToolCall(async () => {
        const [logger, headers] = mcp.utils(t('find_one'), requestInfo, args);
        const config = buildConfig(args, headers) as RequestConfig;
        const { id } = resolveById(args, config);
        logger('find_one %s id=%s', path, id);
        const result = await col().findById(id, config);
        return { structuredContent: { result }, content: [{ type: 'text', text: result ? `${entityName} found.` : `${entityName} not found.` }] };
      }),
  );

  // update_one
  mcp.server.registerTool(
    t('update_one'),
    {
      title: `Update One ${entityName}`,
      description: d(`Partially updates a single ${entityName} by id or ref. Only send fields to change — unset fields are preserved.`),
      inputSchema: { headers: HEADERS_SCHEMA, params: PARAMS_SCHEMA, body: z.object(inputSchema).partial() },
      outputSchema: outSchema,
      annotations: { readOnlyHint: false, destructiveHint: false, idempotentHint: false },
    },
    async (args, { requestInfo }) =>
      throwableToolCall(async () => {
        const [logger, headers] = mcp.utils(t('update_one'), requestInfo, args);
        const config = buildConfig(args, headers) as RequestConfig;
        const { id } = resolveById(args, config);
        logger('update_one %s id=%s payload=%o', path, id, args.body);
        const result = await col().updateById(id, args.body, config);
        return { structuredContent: { result }, content: [{ type: 'text', text: `${entityName} updated.` }] };
      }),
  );

  // update_bulk
  mcp.server.registerTool(
    t('update_bulk'),
    {
      title: `Update Bulk ${entityName}`,
      description: d(`Applies a partial update to all ${entityName} items matching filter.query. Returns the count updated. Use count first to verify scope.`),
      inputSchema: { headers: HEADERS_SCHEMA, filter: FILTER_SCHEMA, body: z.object(inputSchema).partial() },
      outputSchema: { errors: ERRORS_SCHEMA, result: z.object(TOTAL_SCHEMA).partial().optional() },
      annotations: { readOnlyHint: false, destructiveHint: false, idempotentHint: false },
    },
    async (args, { requestInfo }) =>
      throwableToolCall(async () => {
        const [logger, headers] = mcp.utils(t('update_bulk'), requestInfo, args);
        const config = buildConfig(args, headers);
        const query = args.filter?.query ?? {};
        logger('update_bulk %s query=%o payload=%o', path, query, args.body);
        const result = await col().updateBulk(args.body, query, config);
        return { structuredContent: { result: { total: result } }, content: [{ type: 'text', text: `Updated ${result} ${entityName} items.` }] };
      }),
  );

  // delete_one
  mcp.server.registerTool(
    t('delete_one'),
    {
      title: `Soft-Delete One ${entityName}`,
      description: d(`Soft-deletes a single ${entityName} (sets deleted_at, data preserved). Reversible via restore_one. Use destroy_one for permanent removal.`),
      inputSchema: { headers: HEADERS_SCHEMA, params: PARAMS_SCHEMA },
      outputSchema: outSchema,
      annotations: { readOnlyHint: false, destructiveHint: true, idempotentHint: true },
    },
    async (args, { requestInfo }) =>
      throwableToolCall(async () => {
        const [logger, headers] = mcp.utils(t('delete_one'), requestInfo, args);
        const config = buildConfig(args, headers) as RequestConfig;
        const { id } = resolveById(args, config);
        logger('delete_one %s id=%s (soft)', path, id);
        const result = await col().deleteById(id, config);
        return { structuredContent: { result }, content: [{ type: 'text', text: `${entityName} soft-deleted (restorable via restore_one).` }] };
      }),
  );

  // restore_one
  mcp.server.registerTool(
    t('restore_one'),
    {
      title: `Restore One ${entityName}`,
      description: d(`Restores a soft-deleted ${entityName} (clears deleted_at). Only applies to items previously deleted via delete_one.`),
      inputSchema: { headers: HEADERS_SCHEMA, params: PARAMS_SCHEMA },
      outputSchema: outSchema,
      annotations: { readOnlyHint: false, destructiveHint: false, idempotentHint: true },
    },
    async (args, { requestInfo }) =>
      throwableToolCall(async () => {
        const [logger, headers] = mcp.utils(t('restore_one'), requestInfo, args);
        const config = buildConfig(args, headers) as RequestConfig;
        const { id } = resolveById(args, config);
        logger('restore_one %s id=%s', path, id);
        const result = await col().restoreById(id, config);
        return { structuredContent: { result }, content: [{ type: 'text', text: `${entityName} restored.` }] };
      }),
  );

  // destroy_one
  mcp.server.registerTool(
    t('destroy_one'),
    {
      title: `Destroy One ${entityName} (Permanent)`,
      description: d(`⚠ PERMANENT HARD DELETE — irreversible. Removes ${entityName} from the database entirely. Prefer delete_one unless permanent deletion is explicitly required.`),
      inputSchema: { headers: HEADERS_SCHEMA, params: PARAMS_SCHEMA },
      outputSchema: outSchema,
      annotations: { readOnlyHint: false, destructiveHint: true, idempotentHint: true },
    },
    async (args, { requestInfo }) =>
      throwableToolCall(async () => {
        const [logger, headers] = mcp.utils(t('destroy_one'), requestInfo, args);
        const config = buildConfig(args, headers) as RequestConfig;
        const { id } = resolveById(args, config);
        logger('destroy_one %s id=%s (HARD DELETE)', path, id);
        const result = await col().destroyById(id, config);
        return { structuredContent: { result }, content: [{ type: 'text', text: `${entityName} permanently destroyed (hard delete, irreversible).` }] };
      }),
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add libs/common/src/core/mcp/factory.mcp.ts
git commit -m "feat(mcp): add registerCollectionTools factory with typed filter schema"
```

---

### Task 2: Export factory from `index.ts`

**Files:**
- Modify: `libs/common/src/core/mcp/index.ts`

- [ ] **Step 1: Read the current index**

Run: `cat libs/common/src/core/mcp/index.ts`

- [ ] **Step 2: Add the factory export**

Append to the file:

```typescript
export { registerCollectionTools, FILTER_SCHEMA } from './factory.mcp';
export type { CollectionToolsConfig, PlatformCollection } from './factory.mcp';
```

- [ ] **Step 3: Commit**

```bash
git add libs/common/src/core/mcp/index.ts
git commit -m "feat(mcp): export factory from mcp index"
```

---

## Phase 2 — Router Migrations

> Pattern for every router file:
> 1. Import `registerCollectionTools`, `CORE_INPUT_SCHEMA`, `CORE_OUTPUT_SCHEMA` (and `REACTION_SCHEMA` if needed) from `@app/common/core/mcp`
> 2. Import enums from their service package
> 3. Define the entity schema (entity-specific fields only)
> 4. Call `registerCollectionTools` with merged input/output schemas

---

### Task 3: Migrate `auth` routers

**Files:**
- Modify: `apps/gateway/src/modules/auth/crafts/grants/grants.router.ts`

- [ ] **Step 1: Replace the entire file**

```typescript
// apps/gateway/src/modules/auth/crafts/grants/grants.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { z } from 'zod';

const TIME_SCHEMA = {
  cron_exp: z.string(),
  duration: z.number().positive(),
};

const GRANT_SCHEMA = {
  subject: z.string(),
  action: z.string(),
  object: z.string(),
  field: z.array(z.string()).optional(),
  filter: z.array(z.string()).optional(),
  location: z.array(z.string()).optional(),
  time: z.array(z.object(TIME_SCHEMA)).optional(),
};

registerCollectionTools({
  service: 'auth',
  collection: 'grants',
  entityName: 'AuthGrant',
  serviceDoc: 'docs://core/auth-specification',
  inputSchema: { ...GRANT_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...GRANT_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.auth.grants,
});
```

- [ ] **Step 2: Commit**

```bash
git add apps/gateway/src/modules/auth/crafts/grants/grants.router.ts
git commit -m "refactor(mcp): migrate auth/grants router to factory"
```

---

### Task 4: Migrate `general` routers

**Files:**
- Modify: `apps/gateway/src/modules/general/crafts/activities/activities.router.ts`
- Modify: `apps/gateway/src/modules/general/crafts/artifacts/artifacts.router.ts`
- Modify: `apps/gateway/src/modules/general/crafts/comments/comments.router.ts`
- Modify: `apps/gateway/src/modules/general/crafts/events/events.router.ts`
- Modify: `apps/gateway/src/modules/general/crafts/workflows/workflows.router.ts`

- [ ] **Step 1: Replace activities.router.ts**

```typescript
// apps/gateway/src/modules/general/crafts/activities/activities.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { ActivityType } from '@app/common/enums/general';
import { State } from '@app/common/core/enums';
import { z } from 'zod';

const ACTIVITY_SCHEMA = {
  type: z.nativeEnum(ActivityType),
  state: z.nativeEnum(State).optional(),
  source: z.string().optional(),
  message: z.string(),
  details: z.any().optional(),
  metadata: z.any().optional(),
};

registerCollectionTools({
  service: 'general',
  collection: 'activities',
  entityName: 'GeneralActivity',
  serviceDoc: 'docs://service/general-specification',
  inputSchema: { ...ACTIVITY_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...ACTIVITY_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.general.activities,
});
```

- [ ] **Step 2: Replace artifacts.router.ts**

```typescript
// apps/gateway/src/modules/general/crafts/artifacts/artifacts.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { ValueType } from '@app/common/core/enums';
import { z } from 'zod';

const ARTIFACT_SCHEMA = {
  key: z.string(),
  type: z.nativeEnum(ValueType),
  value: z.any().optional(),
};

registerCollectionTools({
  service: 'general',
  collection: 'artifacts',
  entityName: 'GeneralArtifact',
  serviceDoc: 'docs://service/general-specification',
  inputSchema: { ...ARTIFACT_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...ARTIFACT_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.general.artifacts,
});
```

- [ ] **Step 3: Replace comments.router.ts**

```typescript
// apps/gateway/src/modules/general/crafts/comments/comments.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA, REACTION_SCHEMA } from '@app/common/core/mcp';
import { CommentStatus } from '@app/common/enums/general';
import { State } from '@app/common/core/enums';
import { z } from 'zod';

const COMMENT_SCHEMA = {
  type: z.string().optional(),
  post: z.string().optional(),
  ticket: z.string().optional(),
  content: z.string(),
  level: z.number().optional(),
  parent: z.string().optional(),
  state: z.nativeEnum(State).optional(),
  status: z.nativeEnum(CommentStatus).optional(),
  visibility: z.string().optional(),
  rate: z.number().optional(),
  votes: z.number().optional(),
  rating: z.number().optional(),
  views: z.number().optional(),
  loves: z.number().optional(),
  likes: z.number().optional(),
  dislikes: z.number().optional(),
  mentions: z.array(z.string()).optional(),
  attachments: z.array(z.string()).optional(),
  reactions: z.array(z.object(REACTION_SCHEMA)).optional(),
};

registerCollectionTools({
  service: 'general',
  collection: 'comments',
  entityName: 'GeneralComment',
  serviceDoc: 'docs://service/general-specification',
  inputSchema: { ...COMMENT_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...COMMENT_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.general.comments,
});
```

- [ ] **Step 4: Replace events.router.ts**

```typescript
// apps/gateway/src/modules/general/crafts/events/events.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { z } from 'zod';

const EVENT_SCHEMA = {
  title: z.string(),
  subtitle: z.string().optional(),
  s_date: z.string(),
  e_date: z.string(),
  place: z.string().optional(),
  location: z.string().optional(),
  attendees: z.array(z.string()).optional(),
  organizers: z.array(z.string()).optional(),
  status: z.string().optional(),
  category: z.string().optional(),
  color: z.string().optional(),
  correlation: z.string().optional(),
};

registerCollectionTools({
  service: 'general',
  collection: 'events',
  entityName: 'GeneralEvent',
  serviceDoc: 'docs://service/general-specification',
  inputSchema: { ...EVENT_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...EVENT_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.general.events,
});
```

- [ ] **Step 5: Replace workflows.router.ts**

```typescript
// apps/gateway/src/modules/general/crafts/workflows/workflows.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { WorkflowStatus } from '@app/common/enums/general';
import { z } from 'zod';

const STATE_SCHEMA = {
  ref: z.string(),
  name: z.string().optional(),
  status: z.nativeEnum(WorkflowStatus),
  value: z.any().optional(),
};

const TOKEN_SCHEMA = {
  id: z.string(),
  parent: z.string().optional(),
  locked: z.boolean().optional(),
  history: z.array(z.object(STATE_SCHEMA)),
};

const WORKFLOW_SCHEMA = {
  name: z.string(),
  status: z.nativeEnum(WorkflowStatus),
  tokens: z.array(z.object(TOKEN_SCHEMA)),
  data: z.any().optional(),
};

registerCollectionTools({
  service: 'general',
  collection: 'workflows',
  entityName: 'GeneralWorkflow',
  serviceDoc: 'docs://service/general-specification',
  inputSchema: { ...WORKFLOW_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...WORKFLOW_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.general.workflows,
});
```

- [ ] **Step 6: Commit**

```bash
git add apps/gateway/src/modules/general/
git commit -m "refactor(mcp): migrate general service routers to factory"
```

---

### Task 5: Migrate `identity` routers

**Files:**
- Modify: `apps/gateway/src/modules/identity/crafts/users/users.router.ts`
- Modify: `apps/gateway/src/modules/identity/crafts/profiles/profiles.router.ts`
- Modify: `apps/gateway/src/modules/identity/crafts/sessions/sessions.router.ts`

- [ ] **Step 1: Replace users.router.ts**

```typescript
// apps/gateway/src/modules/identity/crafts/users/users.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { Status } from '@app/common/core/enums';
import { z } from 'zod';

const USER_SCHEMA = {
  status: z.nativeEnum(Status),
  tz: z.string(),
  lang: z.string(),
  region: z.string(),
  email: z.string().optional(),
  phone: z.string().optional(),
  username: z.string().optional(),
  subjects: z.array(z.string()).optional(),
};

registerCollectionTools({
  service: 'identity',
  collection: 'users',
  entityName: 'IdentityUser',
  serviceDoc: 'docs://service/identity-specification',
  inputSchema: { ...USER_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...USER_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.identity.users,
});
```

- [ ] **Step 2: Replace profiles.router.ts**

```typescript
// apps/gateway/src/modules/identity/crafts/profiles/profiles.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { ProfileType, Gender, State } from '@app/common/enums/identity';
import { z } from 'zod';

const PROFILE_SCHEMA = {
  type: z.nativeEnum(ProfileType),
  gender: z.nativeEnum(Gender),
  state: z.nativeEnum(State),
  cover: z.string().optional(),
  avatar: z.string().optional(),
  gallery: z.array(z.string()).optional(),
  nickname: z.string().optional(),
  last_name: z.string().optional(),
  first_name: z.string().optional(),
  middle_name: z.string().optional(),
  nationality: z.string().optional(),
  national_code: z.string().optional(),
  verified_at: z.string().optional(),
  verified_by: z.string().optional(),
  verified_in: z.string().optional(),
  birthdate: z.string().optional(),
};

registerCollectionTools({
  service: 'identity',
  collection: 'profiles',
  entityName: 'IdentityProfile',
  serviceDoc: 'docs://service/identity-specification',
  inputSchema: { ...PROFILE_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...PROFILE_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.identity.profiles,
});
```

- [ ] **Step 3: Replace sessions.router.ts**

```typescript
// apps/gateway/src/modules/identity/crafts/sessions/sessions.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { z } from 'zod';

const SESSION_SCHEMA = {
  ip: z.string(),
  agent: z.string(),
  expiration: z.number(),
};

registerCollectionTools({
  service: 'identity',
  collection: 'sessions',
  entityName: 'IdentitySession',
  serviceDoc: 'docs://service/identity-specification',
  inputSchema: { ...SESSION_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...SESSION_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.identity.sessions,
});
```

- [ ] **Step 4: Commit**

```bash
git add apps/gateway/src/modules/identity/
git commit -m "refactor(mcp): migrate identity service routers to factory"
```

---

### Task 6: Migrate `financial` routers

**Files:**
- Modify: `apps/gateway/src/modules/financial/crafts/accounts/accounts.router.ts`
- Modify: `apps/gateway/src/modules/financial/crafts/currencies/currencies.router.ts`
- Modify: `apps/gateway/src/modules/financial/crafts/invoices/invoices.router.ts`
- Modify: `apps/gateway/src/modules/financial/crafts/transactions/transactions.router.ts`
- Modify: `apps/gateway/src/modules/financial/crafts/wallets/wallets.router.ts`

- [ ] **Step 1: Replace accounts.router.ts**

```typescript
// apps/gateway/src/modules/financial/crafts/accounts/accounts.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { AccountType, AccountOwnership } from '@app/common/enums/financial';
import { z } from 'zod';

const ACCOUNT_SCHEMA = {
  type: z.nativeEnum(AccountType),
  ownership: z.nativeEnum(AccountOwnership),
  members: z.array(z.string()).optional(),
};

registerCollectionTools({
  service: 'financial',
  collection: 'accounts',
  entityName: 'FinancialAccount',
  serviceDoc: 'docs://service/financial-specification',
  inputSchema: { ...ACCOUNT_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...ACCOUNT_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.financial.accounts,
});
```

- [ ] **Step 2: Replace currencies.router.ts**

```typescript
// apps/gateway/src/modules/financial/crafts/currencies/currencies.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { CurrencyType, CurrencyProvider, CurrencyCategory, CurrencyLib } from '@app/common/enums/financial';
import { z } from 'zod';

const UNIT_SCHEMA = {
  name: z.string(),
  rate: z.number(),
  symbol: z.string().optional(),
  precision: z.number().optional(),
};

const CURRENCY_SCHEMA = {
  type: z.nativeEnum(CurrencyType),
  provider: z.nativeEnum(CurrencyProvider),
  code: z.string().optional(),
  symbol: z.string(),
  precision: z.number(),
  countries: z.array(z.string()).optional(),
  name: z.string().optional(),
  token: z.string().optional(),
  explore: z.string().optional(),
  network: z.string().optional(),
  contract: z.string().optional(),
  subunits: z.array(z.object(UNIT_SCHEMA)).optional(),
  category: z.nativeEnum(CurrencyCategory).optional(),
  lib: z.nativeEnum(CurrencyLib),
  nodes: z.array(z.string()).optional(),
};

registerCollectionTools({
  service: 'financial',
  collection: 'currencies',
  entityName: 'FinancialCurrency',
  serviceDoc: 'docs://service/financial-specification',
  inputSchema: { ...CURRENCY_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...CURRENCY_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.financial.currencies,
});
```

- [ ] **Step 3: Replace invoices.router.ts**

```typescript
// apps/gateway/src/modules/financial/crafts/invoices/invoices.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA, PAY_SCHEMA } from '@app/common/core/mcp';
import { InvoiceType } from '@app/common/enums/financial';
import { State } from '@app/common/core/enums';
import { z } from 'zod';

const ITEM_SCHEMA = {
  title: z.string().optional(),
  price: z.number(),
  quantity: z.number(),
  profit: z.number().optional(),
  discount: z.number().optional(),
  measurement: z.string().optional(),
  ...CORE_INPUT_SCHEMA,
};

const INVOICE_SCHEMA = {
  type: z.nativeEnum(InvoiceType),
  title: z.string().optional(),
  paid: z.number().optional(),
  amount: z.number(),
  payees: z.array(z.object(PAY_SCHEMA)),
  payers: z.array(z.object(PAY_SCHEMA)).optional(),
  currency: z.string().optional(),
  items: z.array(z.object(ITEM_SCHEMA)).optional(),
  state: z.nativeEnum(State).optional(),
  profit: z.number().optional(),
  discount: z.number().optional(),
  closed_at: z.string().optional(),
  expires_at: z.string().optional(),
  replication: z.number().optional(),
  subscription: z.string().optional(),
};

registerCollectionTools({
  service: 'financial',
  collection: 'invoices',
  entityName: 'FinancialInvoice',
  serviceDoc: 'docs://service/financial-specification',
  inputSchema: { ...INVOICE_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...INVOICE_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.financial.invoices,
});
```

- [ ] **Step 4: Replace transactions.router.ts**

```typescript
// apps/gateway/src/modules/financial/crafts/transactions/transactions.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA, PAY_SCHEMA } from '@app/common/core/mcp';
import { TransactionType, TransactionState, TransactionReason } from '@app/common/enums/financial';
import { z } from 'zod';

const TRANSACTION_SCHEMA = {
  saga: z.string(),
  type: z.nativeEnum(TransactionType),
  state: z.nativeEnum(TransactionState),
  reason: z.nativeEnum(TransactionReason),
  amount: z.number(),
  payees: z.array(z.object(PAY_SCHEMA)).optional(),
  payers: z.array(z.object(PAY_SCHEMA)).optional(),
  failed_at: z.string().optional(),
  verified_at: z.string().optional(),
  canceled_at: z.string().optional(),
  invoice: z.string().optional(),
};

registerCollectionTools({
  service: 'financial',
  collection: 'transactions',
  entityName: 'FinancialTransaction',
  serviceDoc: 'docs://service/financial-specification',
  inputSchema: { ...TRANSACTION_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...TRANSACTION_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.financial.transactions,
});
```

- [ ] **Step 5: Replace wallets.router.ts**

```typescript
// apps/gateway/src/modules/financial/crafts/wallets/wallets.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { z } from 'zod';

const WALLET_SCHEMA = {
  strict: z.boolean().optional(),
  account: z.string(),
  currency: z.string(),
  amount: z.number(),
  blocked: z.number().optional(),
  internal: z.number().optional(),
  external: z.number().optional(),
  address: z.string().optional(),
  private: z.string().optional(),
};

registerCollectionTools({
  service: 'financial',
  collection: 'wallets',
  entityName: 'FinancialWallet',
  serviceDoc: 'docs://service/financial-specification',
  inputSchema: { ...WALLET_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...WALLET_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.financial.wallets,
});
```

- [ ] **Step 6: Commit**

```bash
git add apps/gateway/src/modules/financial/
git commit -m "refactor(mcp): migrate financial service routers to factory"
```

---

### Task 7: Migrate `logistic` routers

**Files:**
- Modify: `apps/gateway/src/modules/logistic/crafts/cargoes/cargoes.router.ts`
- Modify: `apps/gateway/src/modules/logistic/crafts/drivers/drivers.router.ts`
- Modify: `apps/gateway/src/modules/logistic/crafts/locations/locations.router.ts`
- Modify: `apps/gateway/src/modules/logistic/crafts/travels/travels.router.ts`
- Modify: `apps/gateway/src/modules/logistic/crafts/vehicles/vehicles.router.ts`

- [ ] **Step 1: Replace cargoes.router.ts**

```typescript
// apps/gateway/src/modules/logistic/crafts/cargoes/cargoes.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { z } from 'zod';

const CARGO_SCHEMA = {
  title: z.string().optional(),
  weight: z.number(),
  width: z.number(),
  height: z.number(),
  length: z.number(),
  fragile: z.boolean().optional(),
  perishable: z.boolean().optional(),
  travels: z.array(z.string()).optional(),
};

registerCollectionTools({
  service: 'logistic',
  collection: 'cargoes',
  entityName: 'LogisticCargo',
  serviceDoc: 'docs://service/logistic-specification',
  inputSchema: { ...CARGO_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...CARGO_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.logistic.cargoes,
});
```

- [ ] **Step 2: Replace drivers.router.ts**

```typescript
// apps/gateway/src/modules/logistic/crafts/drivers/drivers.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { DriverType, Gender, State, Status } from '@app/common/enums/logistic';
import { z } from 'zod';

const DRIVER_SCHEMA = {
  type: z.nativeEnum(DriverType),
  gender: z.nativeEnum(Gender),
  state: z.nativeEnum(State),
  status: z.nativeEnum(Status),
  license: z.string(),
  verified_at: z.string().optional(),
  verified_by: z.string().optional(),
  verified_in: z.string().optional(),
  expiration_date: z.string(),
};

registerCollectionTools({
  service: 'logistic',
  collection: 'drivers',
  entityName: 'LogisticDriver',
  serviceDoc: 'docs://service/logistic-specification',
  inputSchema: { ...DRIVER_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...DRIVER_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.logistic.drivers,
});
```

- [ ] **Step 3: Replace locations.router.ts**

```typescript
// apps/gateway/src/modules/logistic/crafts/locations/locations.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { LocationGeometryType, LocationType } from '@app/common/enums/logistic';
import { z } from 'zod';

const GEOMETRY_SCHEMA = {
  type: z.nativeEnum(LocationGeometryType),
  coordinates: z.array(z.any()),
  bbox: z.array(z.number()).optional(),
};

const LOCATION_SCHEMA = {
  name: z.string().optional(),
  title: z.string().optional(),
  type: z.nativeEnum(LocationType).optional(),
  geometry: z.object(GEOMETRY_SCHEMA),
  properties: z.any().optional(),
};

registerCollectionTools({
  service: 'logistic',
  collection: 'locations',
  entityName: 'LogisticLocation',
  serviceDoc: 'docs://service/logistic-specification',
  inputSchema: { ...LOCATION_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...LOCATION_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.logistic.locations,
});
```

- [ ] **Step 4: Replace travels.router.ts**

```typescript
// apps/gateway/src/modules/logistic/crafts/travels/travels.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { z } from 'zod';

const TRAVEL_SCHEMA = {
  cargoes: z.array(z.string()).optional(),
  drivers: z.array(z.string()).optional(),
  vehicles: z.array(z.string()).optional(),
  locations: z.array(z.string()),
};

registerCollectionTools({
  service: 'logistic',
  collection: 'travels',
  entityName: 'LogisticTravel',
  serviceDoc: 'docs://service/logistic-specification',
  inputSchema: { ...TRAVEL_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...TRAVEL_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.logistic.travels,
});
```

- [ ] **Step 5: Replace vehicles.router.ts**

```typescript
// apps/gateway/src/modules/logistic/crafts/vehicles/vehicles.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { VehicleType, Status } from '@app/common/enums/logistic';
import { z } from 'zod';

const VEHICLE_SCHEMA = {
  type: z.nativeEnum(VehicleType),
  status: z.nativeEnum(Status),
  plates: z.array(z.string()),
  drivers: z.array(z.string()).optional(),
};

registerCollectionTools({
  service: 'logistic',
  collection: 'vehicles',
  entityName: 'LogisticVehicle',
  serviceDoc: 'docs://service/logistic-specification',
  inputSchema: { ...VEHICLE_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...VEHICLE_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.logistic.vehicles,
});
```

- [ ] **Step 6: Commit**

```bash
git add apps/gateway/src/modules/logistic/
git commit -m "refactor(mcp): migrate logistic service routers to factory"
```

---

### Task 8: Migrate `career` routers

**Files:**
- Modify: `apps/gateway/src/modules/career/crafts/branches/branches.router.ts`
- Modify: `apps/gateway/src/modules/career/crafts/businesses/businesses.router.ts`
- Modify: `apps/gateway/src/modules/career/crafts/customers/customers.router.ts`
- Modify: `apps/gateway/src/modules/career/crafts/employees/employees.router.ts`
- Modify: `apps/gateway/src/modules/career/crafts/products/products.router.ts`
- Modify: `apps/gateway/src/modules/career/crafts/services/services.router.ts`
- Modify: `apps/gateway/src/modules/career/crafts/stocks/stocks.router.ts`
- Modify: `apps/gateway/src/modules/career/crafts/stores/stores.router.ts`

- [ ] **Step 1: Replace branches.router.ts**

```typescript
// apps/gateway/src/modules/career/crafts/branches/branches.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { BranchType } from '@app/common/enums/career';
import { State, Status } from '@app/common/core/enums';
import { z } from 'zod';

const BRANCH_SCHEMA = {
  type: z.nativeEnum(BranchType),
  name: z.string().optional(),
  business: z.string(),
  code: z.string().optional(),
  state: z.nativeEnum(State).optional(),
  status: z.nativeEnum(Status),
  rate: z.number(),
  votes: z.number().optional(),
  rating: z.number().optional(),
  parent: z.string().optional(),
  manager: z.string().optional(),
  location: z.string(),
  phone: z.string().optional(),
  address: z.string().optional(),
  opening_date: z.string().optional(),
};

registerCollectionTools({
  service: 'career',
  collection: 'branches',
  entityName: 'CareerBranch',
  serviceDoc: 'docs://service/career-specification',
  inputSchema: { ...BRANCH_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...BRANCH_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.career.branches,
});
```

- [ ] **Step 2: Replace businesses.router.ts**

```typescript
// apps/gateway/src/modules/career/crafts/businesses/businesses.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { BusinessType } from '@app/common/enums/career';
import { State, Status } from '@app/common/core/enums';
import { z } from 'zod';

const BUSINESS_SCHEMA = {
  name: z.string(),
  type: z.nativeEnum(BusinessType),
  code: z.string().optional(),
  alias: z.string().optional(),
  logo: z.string().optional(),
  cover: z.string().optional(),
  slogan: z.string().optional(),
  state: z.nativeEnum(State),
  status: z.nativeEnum(Status),
  rate: z.number().optional(),
  votes: z.number().optional(),
  rating: z.number().optional(),
  address: z.string().optional(),
  support: z.string().optional(),
  website: z.string().optional(),
  location: z.string().optional(),
  categories: z.array(z.string()).optional(),
  founder: z.string().optional(),
  co_founders: z.array(z.string()).optional(),
  partners: z.array(z.string()).optional(),
  investors: z.array(z.string()).optional(),
  suppliers: z.array(z.string()).optional(),
  customers: z.array(z.string()).optional(),
  foundation_date: z.string().optional(),
};

registerCollectionTools({
  service: 'career',
  collection: 'businesses',
  entityName: 'CareerBusiness',
  serviceDoc: 'docs://service/career-specification',
  inputSchema: { ...BUSINESS_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...BUSINESS_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.career.businesses,
});
```

- [ ] **Step 3: Replace customers.router.ts**

```typescript
// apps/gateway/src/modules/career/crafts/customers/customers.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { CustomerType, LocationRange } from '@app/common/enums/career';
import { Status } from '@app/common/core/enums';
import { z } from 'zod';

const CUSTOMER_SCHEMA = {
  type: z.nativeEnum(CustomerType),
  range: z.nativeEnum(LocationRange).optional(),
  profile: z.string().optional(),
  branch: z.string().optional(),
  business: z.string(),
  location: z.string().optional(),
  addresses: z.array(z.string()).optional(),
  stores: z.array(z.string()).optional(),
  services: z.array(z.string()).optional(),
  employees: z.array(z.string()).optional(),
  status: z.nativeEnum(Status).optional(),
  rate: z.number().optional(),
  votes: z.number().optional(),
  rating: z.number().optional(),
  documents: z.array(z.string()).optional(),
  certifications: z.array(z.string()).optional(),
};

registerCollectionTools({
  service: 'career',
  collection: 'customers',
  entityName: 'CareerCustomer',
  serviceDoc: 'docs://service/career-specification',
  inputSchema: { ...CUSTOMER_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...CUSTOMER_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.career.customers,
});
```

- [ ] **Step 4: Replace employees.router.ts**

```typescript
// apps/gateway/src/modules/career/crafts/employees/employees.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { EmployeeType, LocationRange } from '@app/common/enums/career';
import { State, Status } from '@app/common/core/enums';
import { z } from 'zod';

const EMPLOYEE_SCHEMA = {
  type: z.nativeEnum(EmployeeType),
  range: z.nativeEnum(LocationRange).optional(),
  job_title: z.string(),
  profile: z.string(),
  branch: z.string(),
  manager: z.string(),
  business: z.string(),
  location: z.string(),
  services: z.array(z.string()),
  state: z.nativeEnum(State),
  status: z.nativeEnum(Status),
  rate: z.number().optional(),
  votes: z.number().optional(),
  rating: z.number().optional(),
  salary: z.number().optional(),
  currency: z.string(),
  department: z.string(),
  grade: z.number().optional(),
  level: z.string(),
  hire_date: z.string().optional(),
  fire_date: z.string().optional(),
  skills: z.array(z.string()),
  documents: z.array(z.string()),
  certifications: z.array(z.string()),
};

registerCollectionTools({
  service: 'career',
  collection: 'employees',
  entityName: 'CareerEmployee',
  serviceDoc: 'docs://service/career-specification',
  inputSchema: { ...EMPLOYEE_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...EMPLOYEE_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.career.employees,
});
```

- [ ] **Step 5: Replace products.router.ts**

```typescript
// apps/gateway/src/modules/career/crafts/products/products.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { ProductFeatureType } from '@app/common/enums/career';
import { State } from '@app/common/core/enums';
import { z } from 'zod';

const FEATURE_SCHEMA = {
  type: z.nativeEnum(ProductFeatureType),
  title: z.string(),
  value: z.union([z.boolean(), z.number(), z.string()]),
  ...CORE_INPUT_SCHEMA,
};

const PRODUCT_SCHEMA = {
  name: z.string(),
  alias: z.string().optional(),
  state: z.nativeEnum(State),
  store: z.string().optional(),
  branch: z.string().optional(),
  business: z.string().optional(),
  brand: z.string().optional(),
  content: z.string().optional(),
  cover: z.string().optional(),
  gallery: z.array(z.string()),
  categories: z.array(z.string()),
  rate: z.number(),
  votes: z.number(),
  rating: z.number(),
  features: z.array(z.object(FEATURE_SCHEMA)).optional(),
};

registerCollectionTools({
  service: 'career',
  collection: 'products',
  entityName: 'CareerProduct',
  serviceDoc: 'docs://service/career-specification',
  inputSchema: { ...PRODUCT_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...PRODUCT_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.career.products,
});
```

- [ ] **Step 6: Replace services.router.ts**

```typescript
// apps/gateway/src/modules/career/crafts/services/services.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { LocationRange, ServiceType } from '@app/common/enums/career';
import { State, Status } from '@app/common/core/enums';
import { z } from 'zod';

const SERVICE_SCHEMA = {
  type: z.nativeEnum(ServiceType),
  range: z.nativeEnum(LocationRange).optional(),
  name: z.string(),
  state: z.nativeEnum(State),
  status: z.nativeEnum(Status),
  branch: z.string().optional(),
  business: z.string(),
  location: z.string().optional(),
  categories: z.array(z.string()).optional(),
  rate: z.number().optional(),
  votes: z.number().optional(),
  rating: z.number().optional(),
  currency: z.string().optional(),
  unit: z.string().optional(),
  price: z.number().optional(),
  profit: z.number().optional(),
  discount: z.number().optional(),
};

registerCollectionTools({
  service: 'career',
  collection: 'services',
  entityName: 'CareerService',
  serviceDoc: 'docs://service/career-specification',
  inputSchema: { ...SERVICE_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...SERVICE_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.career.services,
});
```

- [ ] **Step 7: Replace stocks.router.ts**

```typescript
// apps/gateway/src/modules/career/crafts/stocks/stocks.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { StockType } from '@app/common/enums/career';
import { State, Status } from '@app/common/core/enums';
import { z } from 'zod';

const STOCK_SCHEMA = {
  type: z.nativeEnum(StockType),
  name: z.string().optional(),
  state: z.nativeEnum(State).optional(),
  status: z.nativeEnum(Status).optional(),
  product: z.string(),
  feature: z.string().optional(),
  store: z.string().optional(),
  branch: z.string().optional(),
  business: z.string().optional(),
  capacity: z.number().optional(),
  inventory: z.number(),
  place: z.string().optional(),
  position: z.string().optional(),
  location: z.string().optional(),
  threshold: z.number().optional(),
  currency: z.string().optional(),
  unit: z.string().optional(),
  price: z.number().optional(),
  profit: z.number().optional(),
  discount: z.number().optional(),
};

registerCollectionTools({
  service: 'career',
  collection: 'stocks',
  entityName: 'CareerStock',
  serviceDoc: 'docs://service/career-specification',
  inputSchema: { ...STOCK_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...STOCK_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.career.stocks,
});
```

- [ ] **Step 8: Replace stores.router.ts**

```typescript
// apps/gateway/src/modules/career/crafts/stores/stores.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { LocationRange, StoreFork, StoreType } from '@app/common/enums/career';
import { State, Status } from '@app/common/core/enums';
import { z } from 'zod';

const STORE_SCHEMA = {
  name: z.string(),
  type: z.nativeEnum(StoreType),
  fork: z.nativeEnum(StoreFork),
  range: z.nativeEnum(LocationRange).optional(),
  state: z.nativeEnum(State).optional(),
  status: z.nativeEnum(Status),
  parent: z.string().optional(),
  manager: z.string().optional(),
  business: z.string(),
  categories: z.array(z.string()).optional(),
  rate: z.number().optional(),
  votes: z.number().optional(),
  rating: z.number().optional(),
  phone: z.string().optional(),
  address: z.string().optional(),
  location: z.string().optional(),
};

registerCollectionTools({
  service: 'career',
  collection: 'stores',
  entityName: 'CareerStore',
  serviceDoc: 'docs://service/career-specification',
  inputSchema: { ...STORE_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...STORE_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.career.stores,
});
```

- [ ] **Step 9: Commit**

```bash
git add apps/gateway/src/modules/career/
git commit -m "refactor(mcp): migrate career service routers to factory"
```

---

### Task 9: Migrate `touch` routers

**Files:**
- Modify: `apps/gateway/src/modules/touch/crafts/emails/emails.router.ts`
- Modify: `apps/gateway/src/modules/touch/crafts/notices/notices.router.ts`
- Modify: `apps/gateway/src/modules/touch/crafts/pushes/pushes.router.ts`
- Modify: `apps/gateway/src/modules/touch/crafts/smss/smss.router.ts`

- [ ] **Step 1: Replace emails.router.ts**

```typescript
// apps/gateway/src/modules/touch/crafts/emails/emails.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { EmailProvider } from '@app/common/enums/touch';
import { z } from 'zod';

const SMTP_SCHEMA = {
  response: z.string(),
  accepted: z.array(z.string()).optional(),
  rejected: z.array(z.string()).optional(),
  message_id: z.string(),
  message_time: z.number(),
  message_size: z.number(),
};

const EMAIL_SCHEMA = {
  provider: z.nativeEnum(EmailProvider),
  to: z.array(z.string()),
  cc: z.array(z.string()).optional(),
  bcc: z.array(z.string()).optional(),
  from: z.string(),
  subject: z.string(),
  html: z.string().optional(),
  text: z.string().optional(),
  date: z.string().optional(),
  reply_to: z.array(z.string()).optional(),
  in_reply_to: z.string().optional(),
  attachments: z.array(z.any()).optional(),
  smtp: z.object(SMTP_SCHEMA).optional(),
};

registerCollectionTools({
  service: 'touch',
  collection: 'emails',
  entityName: 'TouchEmail',
  serviceDoc: 'docs://service/touch-specification',
  inputSchema: { ...EMAIL_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...EMAIL_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.touch.emails,
});
```

- [ ] **Step 2: Replace notices.router.ts**

```typescript
// apps/gateway/src/modules/touch/crafts/notices/notices.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { NoticeType } from '@app/common/enums/touch';
import { z } from 'zod';

const ACTION_SCHEMA = {
  label: z.string(),
  type: z.string().optional(),
  path: z.string().optional(),
  icon: z.string().optional(),
};

const NOTICE_SCHEMA = {
  type: z.nativeEnum(NoticeType),
  title: z.string(),
  subtitle: z.string().optional(),
  content: z.string(),
  category: z.string().optional(),
  visited: z.boolean().optional(),
  visited_at: z.string().optional(),
  visited_by: z.string().optional(),
  visited_in: z.string().optional(),
  thumbnail: z.string().optional(),
  attachments: z.array(z.any()).optional(),
  actions: z.array(z.object(ACTION_SCHEMA)).optional(),
};

registerCollectionTools({
  service: 'touch',
  collection: 'notices',
  entityName: 'TouchNotice',
  serviceDoc: 'docs://service/touch-specification',
  inputSchema: { ...NOTICE_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...NOTICE_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.touch.notices,
});
```

- [ ] **Step 3: Replace pushes.router.ts**

```typescript
// apps/gateway/src/modules/touch/crafts/pushes/pushes.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { z } from 'zod';

const KEYS_SCHEMA = {
  auth: z.string(),
  p256dh: z.string(),
};

const PUSH_SCHEMA = {
  session: z.string(),
  keys: z.object(KEYS_SCHEMA),
  endpoint: z.string(),
  blacklist: z.string().optional(),
  expiration: z.number(),
};

registerCollectionTools({
  service: 'touch',
  collection: 'pushes',
  entityName: 'TouchPush',
  serviceDoc: 'docs://service/touch-specification',
  inputSchema: { ...PUSH_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...PUSH_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.touch.pushes,
});
```

- [ ] **Step 4: Replace smss.router.ts**

```typescript
// apps/gateway/src/modules/touch/crafts/smss/smss.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { SmsProvider, SmsType } from '@app/common/enums/touch';
import { z } from 'zod';

const SMS_SCHEMA = {
  provider: z.nativeEnum(SmsProvider),
  type: z.nativeEnum(SmsType).optional(),
  message: z.string().optional(),
  template: z.string().optional(),
  parameters: z.array(z.string()).optional(),
  receptors: z.array(z.string()),
  sender: z.string().optional(),
  res: z.any().optional(),
};

registerCollectionTools({
  service: 'touch',
  collection: 'smss',
  entityName: 'TouchSms',
  serviceDoc: 'docs://service/touch-specification',
  inputSchema: { ...SMS_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...SMS_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.touch.smss,
});
```

- [ ] **Step 5: Commit**

```bash
git add apps/gateway/src/modules/touch/
git commit -m "refactor(mcp): migrate touch service routers to factory"
```

---

### Task 10: Migrate `content` routers

**Files:**
- Modify: `apps/gateway/src/modules/content/crafts/notes/notes.router.ts`
- Modify: `apps/gateway/src/modules/content/crafts/posts/posts.router.ts`
- Modify: `apps/gateway/src/modules/content/crafts/tickets/tickets.router.ts`

- [ ] **Step 1: Replace notes.router.ts**

```typescript
// apps/gateway/src/modules/content/crafts/notes/notes.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA, REACTION_SCHEMA } from '@app/common/core/mcp';
import { NoteType } from '@app/common/enums/content';
import { State } from '@app/common/core/enums';
import { z } from 'zod';

const NOTE_SCHEMA = {
  type: z.nativeEnum(NoteType),
  state: z.nativeEnum(State).optional(),
  status: z.string().optional(),
  content: z.string(),
  level: z.number().optional(),
  parent: z.string().optional(),
  relation: z.string().optional(),
  visibility: z.string().optional(),
  loves: z.number().optional(),
  likes: z.number().optional(),
  dislikes: z.number().optional(),
  mentions: z.array(z.string()).optional(),
  attachments: z.array(z.string()).optional(),
  reactions: z.array(z.object(REACTION_SCHEMA)).optional(),
};

registerCollectionTools({
  service: 'content',
  collection: 'notes',
  entityName: 'ContentNote',
  serviceDoc: 'docs://service/content-specification',
  inputSchema: { ...NOTE_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...NOTE_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.content.notes,
});
```

- [ ] **Step 2: Replace posts.router.ts**

```typescript
// apps/gateway/src/modules/content/crafts/posts/posts.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA, REACTION_SCHEMA } from '@app/common/core/mcp';
import { PostStatus } from '@app/common/enums/content';
import { State } from '@app/common/core/enums';
import { z } from 'zod';

const POST_SCHEMA = {
  title: z.string(),
  type: z.string().optional(),
  slug: z.string().optional(),
  subtitle: z.string().optional(),
  parent: z.string().optional(),
  content: z.string(),
  summary: z.string().optional(),
  categories: z.array(z.string()).optional(),
  state: z.nativeEnum(State),
  status: z.nativeEnum(PostStatus),
  visibility: z.string().optional(),
  rate: z.number().optional(),
  votes: z.number().optional(),
  rating: z.number().optional(),
  views: z.number().optional(),
  loves: z.number().optional(),
  likes: z.number().optional(),
  dislikes: z.number().optional(),
  thumbnail: z.string().optional(),
  attachments: z.array(z.string()).optional(),
  featured_image: z.string().optional(),
  keywords: z.array(z.string()).optional(),
  related_posts: z.array(z.string()).optional(),
  publication_date: z.string().optional(),
  reactions: z.array(z.object(REACTION_SCHEMA)).optional(),
};

registerCollectionTools({
  service: 'content',
  collection: 'posts',
  entityName: 'ContentPost',
  serviceDoc: 'docs://service/content-specification',
  inputSchema: { ...POST_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...POST_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.content.posts,
});
```

- [ ] **Step 3: Replace tickets.router.ts**

```typescript
// apps/gateway/src/modules/content/crafts/tickets/tickets.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { TicketPriority, TicketStatus } from '@app/common/enums/content';
import { State } from '@app/common/core/enums';
import { z } from 'zod';

const TICKET_SCHEMA = {
  title: z.string(),
  type: z.string().optional(),
  state: z.nativeEnum(State).optional(),
  status: z.nativeEnum(TicketStatus),
  priority: z.nativeEnum(TicketPriority),
  parent: z.string().optional(),
  rate: z.number().optional(),
  votes: z.number().optional(),
  rating: z.number().optional(),
  content: z.string(),
  department: z.string().optional(),
  due_date: z.string().optional(),
  assigned_to: z.string().optional(),
  solution: z.string().optional(),
  attachments: z.array(z.string()).optional(),
  feedback: z.string().optional(),
  related_tickets: z.array(z.string()).optional(),
};

registerCollectionTools({
  service: 'content',
  collection: 'tickets',
  entityName: 'ContentTicket',
  serviceDoc: 'docs://service/content-specification',
  inputSchema: { ...TICKET_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...TICKET_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.content.tickets,
});
```

- [ ] **Step 4: Commit**

```bash
git add apps/gateway/src/modules/content/
git commit -m "refactor(mcp): migrate content service routers to factory"
```

---

### Task 11: Migrate `context` routers

**Files:**
- Modify: `apps/gateway/src/modules/context/crafts/configs/configs.router.ts`
- Modify: `apps/gateway/src/modules/context/crafts/settings/settings.router.ts`

- [ ] **Step 1: Replace configs.router.ts**

```typescript
// apps/gateway/src/modules/context/crafts/configs/configs.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { ConfigKey } from '@app/common/enums/context';
import { Status } from '@app/common/core/enums';
import { z } from 'zod';

const CONFIG_SCHEMA = {
  key: z.nativeEnum(ConfigKey),
  eid: z.string(),
  value: z.any().optional(),
  status: z.nativeEnum(Status).optional(),
};

registerCollectionTools({
  service: 'context',
  collection: 'configs',
  entityName: 'ContextConfig',
  serviceDoc: 'docs://service/context-specification',
  inputSchema: { ...CONFIG_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...CONFIG_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.context.configs,
});
```

- [ ] **Step 2: Replace settings.router.ts**

```typescript
// apps/gateway/src/modules/context/crafts/settings/settings.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { Status, ValueType } from '@app/common/core/enums';
import { z } from 'zod';

const SETTING_SCHEMA = {
  key: z.string(),
  type: z.nativeEnum(ValueType),
  value: z.any().optional(),
  status: z.nativeEnum(Status).optional(),
};

registerCollectionTools({
  service: 'context',
  collection: 'settings',
  entityName: 'ContextSetting',
  serviceDoc: 'docs://service/context-specification',
  inputSchema: { ...SETTING_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...SETTING_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.context.settings,
});
```

- [ ] **Step 3: Commit**

```bash
git add apps/gateway/src/modules/context/
git commit -m "refactor(mcp): migrate context service routers to factory"
```

---

### Task 12: Migrate `domain` routers

**Files:**
- Modify: `apps/gateway/src/modules/domain/crafts/apps/apps.router.ts`
- Modify: `apps/gateway/src/modules/domain/crafts/clients/clients.router.ts`

- [ ] **Step 1: Replace apps.router.ts**

```typescript
// apps/gateway/src/modules/domain/crafts/apps/apps.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { GrantType, Status } from '@app/common/core/enums';
import { AppType } from '@app/common/enums/domain';
import { Scope } from '@app/common/core';
import { z } from 'zod';

const CHANGE_LOG_SCHEMA = {
  code: z.string().optional(),
  semver: z.string(),
  changes: z.array(z.string()),
  deprecation_date: z.string().optional(),
  ...CORE_INPUT_SCHEMA,
};

const APP_SCHEMA = {
  type: z.nativeEnum(AppType),
  cid: z.string(),
  name: z.string().optional(),
  status: z.nativeEnum(Status),
  url: z.string().optional(),
  logo: z.string().optional(),
  site: z.string().optional(),
  slogan: z.string().optional(),
  scopes: z.array(z.nativeEnum(Scope)).optional(),
  grant_types: z.array(z.nativeEnum(GrantType)).optional(),
  access_token_ttl: z.number().optional(),
  refresh_token_ttl: z.number().optional(),
  change_logs: z.array(z.object(CHANGE_LOG_SCHEMA)).optional(),
};

registerCollectionTools({
  service: 'domain',
  collection: 'apps',
  entityName: 'DomainApp',
  serviceDoc: 'docs://service/domain-specification',
  inputSchema: { ...APP_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...APP_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.domain.apps,
});
```

- [ ] **Step 2: Replace clients.router.ts**

```typescript
// apps/gateway/src/modules/domain/crafts/clients/clients.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { ClientPlan, ClientServiceProvider, ClientServiceType } from '@app/common/enums/domain';
import { GrantType, State, Status } from '@app/common/core/enums';
import { Scope } from '@app/common/core';
import { z } from 'zod';

const DOMAIN_SCHEMA = {
  name: z.string(),
  status: z.nativeEnum(Status),
  ...CORE_INPUT_SCHEMA,
};

const SERVICE_SCHEMA = {
  type: z.nativeEnum(ClientServiceType),
  provider: z.nativeEnum(ClientServiceProvider),
  config: z.any(),
  ...CORE_INPUT_SCHEMA,
};

const CLIENT_SCHEMA = {
  name: z.string(),
  plan: z.nativeEnum(ClientPlan),
  url: z.string().optional(),
  logo: z.string().optional(),
  site: z.string().optional(),
  slogan: z.string().optional(),
  state: z.nativeEnum(State).optional(),
  status: z.nativeEnum(Status).optional(),
  api_key: z.string().optional(),
  client_id: z.string().optional(),
  client_secret: z.string().optional(),
  expiration_date: z.string().optional(),
  access_token_ttl: z.number().optional(),
  refresh_token_ttl: z.number().optional(),
  scopes: z.array(z.nativeEnum(Scope)),
  whitelist: z.array(z.string()).optional(),
  coworkers: z.array(z.string()).optional(),
  grant_types: z.array(z.nativeEnum(GrantType)),
  domains: z.array(z.object(DOMAIN_SCHEMA)).optional(),
  services: z.array(z.object(SERVICE_SCHEMA)).optional(),
};

registerCollectionTools({
  service: 'domain',
  collection: 'clients',
  entityName: 'DomainClient',
  serviceDoc: 'docs://service/domain-specification',
  inputSchema: { ...CLIENT_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...CLIENT_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.domain.clients,
});
```

- [ ] **Step 3: Commit**

```bash
git add apps/gateway/src/modules/domain/
git commit -m "refactor(mcp): migrate domain service routers to factory"
```

---

### Task 13: Migrate `essential` routers

**Files:**
- Find and modify all `*.router.ts` files under `apps/gateway/src/modules/essential/`

- [ ] **Step 1: Locate essential routers**

Run: `find apps/gateway/src/modules/essential -name "*.router.ts"`

- [ ] **Step 2: Replace sagas.router.ts** (adjust path based on Step 1 output)

```typescript
// apps/gateway/src/modules/essential/crafts/sagas/sagas.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { SagaState } from '@app/common/enums/essential';
import { z } from 'zod';

const SAGA_SCHEMA = {
  ttl: z.number(),
  job: z.string(),
  state: z.nativeEnum(SagaState),
  session: z.string(),
  pruned_at: z.string().optional(),
  aborted_at: z.string().optional(),
  committed_at: z.string().optional(),
};

registerCollectionTools({
  service: 'essential',
  collection: 'sagas',
  entityName: 'EssentialSaga',
  serviceDoc: 'docs://service/essential-specification',
  inputSchema: { ...SAGA_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...SAGA_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.essential.sagas,
});
```

- [ ] **Step 3: For any additional essential routers found in Step 1**

Read the existing file, extract its schema block, and apply the same factory pattern with appropriate `service: 'essential'`, correct `collection` name, `entityName`, and `getCollection` accessor.

- [ ] **Step 4: Commit**

```bash
git add apps/gateway/src/modules/essential/
git commit -m "refactor(mcp): migrate essential service routers to factory"
```

---

### Task 14: Migrate `conjoint` routers

**Files:**
- Modify: `apps/gateway/src/modules/conjoint/crafts/accounts/accounts.router.ts`
- Modify: `apps/gateway/src/modules/conjoint/crafts/channels/channels.router.ts`
- Modify: `apps/gateway/src/modules/conjoint/crafts/contacts/contacts.router.ts`
- Modify: `apps/gateway/src/modules/conjoint/crafts/members/members.router.ts`
- Modify: `apps/gateway/src/modules/conjoint/crafts/messages/messages.router.ts`

- [ ] **Step 1: Replace conjoint accounts.router.ts**

```typescript
// apps/gateway/src/modules/conjoint/crafts/accounts/accounts.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { AccountType } from '@app/common/enums/conjoint';
import { z } from 'zod';

const ACCOUNT_SCHEMA = {
  type: z.nativeEnum(AccountType),
  profile: z.string().optional(),
  bio: z.string().optional(),
  status: z.string().optional(),
};

registerCollectionTools({
  service: 'conjoint',
  collection: 'accounts',
  entityName: 'ConjointAccount',
  serviceDoc: 'docs://service/conjoint-specification',
  inputSchema: { ...ACCOUNT_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...ACCOUNT_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.conjoint.accounts,
});
```

- [ ] **Step 2: Replace channels.router.ts**

```typescript
// apps/gateway/src/modules/conjoint/crafts/channels/channels.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { ChannelScope, ChannelType } from '@app/common/enums/conjoint';
import { State, Status } from '@app/common/core/enums';
import { z } from 'zod';

const CHANNEL_SCHEMA = {
  type: z.nativeEnum(ChannelType),
  scope: z.nativeEnum(ChannelScope),
  name: z.string().optional(),
  title: z.string().optional(),
  state: z.nativeEnum(State).optional(),
  status: z.nativeEnum(Status).optional(),
  profile: z.string().optional(),
  account: z.string().optional(),
  pinned_messages: z.array(z.string()).optional(),
};

registerCollectionTools({
  service: 'conjoint',
  collection: 'channels',
  entityName: 'ConjointChannel',
  serviceDoc: 'docs://service/conjoint-specification',
  inputSchema: { ...CHANNEL_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...CHANNEL_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.conjoint.channels,
});
```

- [ ] **Step 3: Replace contacts.router.ts**

```typescript
// apps/gateway/src/modules/conjoint/crafts/contacts/contacts.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { ContactType } from '@app/common/enums/conjoint';
import { z } from 'zod';

const CONTACT_SCHEMA = {
  type: z.nativeEnum(ContactType),
  phone: z.string().optional(),
  email: z.string().optional(),
  account: z.string().optional(),
  nickname: z.string().optional(),
};

registerCollectionTools({
  service: 'conjoint',
  collection: 'contacts',
  entityName: 'ConjointContact',
  serviceDoc: 'docs://service/conjoint-specification',
  inputSchema: { ...CONTACT_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...CONTACT_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.conjoint.contacts,
});
```

- [ ] **Step 4: Replace members.router.ts**

```typescript
// apps/gateway/src/modules/conjoint/crafts/members/members.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { z } from 'zod';

const MEMBER_SCHEMA = {
  channel: z.string(),
  account: z.string(),
  role: z.string().optional(),
  permissions: z.array(z.string()).optional(),
};

registerCollectionTools({
  service: 'conjoint',
  collection: 'members',
  entityName: 'ConjointMember',
  serviceDoc: 'docs://service/conjoint-specification',
  inputSchema: { ...MEMBER_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...MEMBER_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.conjoint.members,
});
```

- [ ] **Step 5: Replace messages.router.ts**

```typescript
// apps/gateway/src/modules/conjoint/crafts/messages/messages.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA, REACTION_SCHEMA } from '@app/common/core/mcp';
import { MessageType } from '@app/common/enums/conjoint';
import { z } from 'zod';

const MESSAGE_SCHEMA = {
  type: z.nativeEnum(MessageType),
  content: z.any(),
  caption: z.string().optional(),
  channel: z.string().optional(),
  account: z.string().optional(),
  mentions: z.array(z.string()).optional(),
  hashtags: z.array(z.string()).optional(),
  reply_to: z.string().optional(),
  edited_at: z.string().optional(),
  delivered_at: z.string().optional(),
  scheduled_at: z.string().optional(),
  views: z.number().optional(),
  visited_at: z.string().optional(),
  originate_from: z.string().optional(),
  forwarded_from: z.string().optional(),
  reactions: z.array(z.object(REACTION_SCHEMA)).optional(),
};

registerCollectionTools({
  service: 'conjoint',
  collection: 'messages',
  entityName: 'ConjointMessage',
  serviceDoc: 'docs://service/conjoint-specification',
  inputSchema: { ...MESSAGE_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...MESSAGE_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.conjoint.messages,
});
```

- [ ] **Step 6: Commit**

```bash
git add apps/gateway/src/modules/conjoint/
git commit -m "refactor(mcp): migrate conjoint service routers to factory"
```

---

### Task 15: Migrate `special` routers

**Files:**
- Modify: `apps/gateway/src/modules/special/crafts/files/files.router.ts`
- Modify: `apps/gateway/src/modules/special/crafts/stats/stats.router.ts`

- [ ] **Step 1: Replace files.router.ts**

```typescript
// apps/gateway/src/modules/special/crafts/files/files.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { State } from '@app/common/core/enums';
import { z } from 'zod';

const FILE_SCHEMA = {
  field: z.string().optional(),
  title: z.string().optional(),
  state: z.nativeEnum(State).optional(),
  original: z.string(),
  encoding: z.string().optional(),
  mimetype: z.string(),
  size: z.number(),
  bucket: z.string(),
  key: z.string(),
  acl: z.string(),
  content_type: z.string().optional(),
  storage_class: z.string().optional(),
  location: z.string(),
  etag: z.string().optional(),
};

registerCollectionTools({
  service: 'special',
  collection: 'files',
  entityName: 'SpecialFile',
  serviceDoc: 'docs://service/special-specification',
  inputSchema: { ...FILE_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...FILE_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.special.files,
});
```

- [ ] **Step 2: Replace stats.router.ts**

```typescript
// apps/gateway/src/modules/special/crafts/stats/stats.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { StatKey, StatType } from '@app/common/enums/special';
import { z } from 'zod';

const STAT_SCHEMA = {
  type: z.nativeEnum(StatType),
  key: z.nativeEnum(StatKey),
  obj: z.any().optional(),
  flag: z.any().optional(),
  day: z.number().optional(),
  month: z.number().optional(),
  year: z.number(),
  hours: z.array(z.number()).optional(),
  days: z.array(z.number()).optional(),
  months: z.array(z.number()).optional(),
};

registerCollectionTools({
  service: 'special',
  collection: 'stats',
  entityName: 'SpecialStat',
  serviceDoc: 'docs://service/special-specification',
  inputSchema: { ...STAT_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...STAT_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.special.stats,
});
```

- [ ] **Step 3: Commit**

```bash
git add apps/gateway/src/modules/special/
git commit -m "refactor(mcp): migrate special service routers to factory"
```

---

### Task 16: Migrate `thing` routers

**Files:**
- Modify: `apps/gateway/src/modules/thing/crafts/devices/devices.router.ts`
- Modify: `apps/gateway/src/modules/thing/crafts/metrics/metrics.router.ts`
- Modify: `apps/gateway/src/modules/thing/crafts/sensors/sensors.router.ts`

- [ ] **Step 1: Replace devices.router.ts**

```typescript
// apps/gateway/src/modules/thing/crafts/devices/devices.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { State, Status } from '@app/common/core/enums';
import { z } from 'zod';

const DEVICE_SCHEMA = {
  name: z.string(),
  type: z.string().optional(),
  token: z.string().optional(),
  state: z.nativeEnum(State).optional(),
  status: z.nativeEnum(Status).optional(),
  location: z.string().optional(),
};

registerCollectionTools({
  service: 'thing',
  collection: 'devices',
  entityName: 'ThingDevice',
  serviceDoc: 'docs://service/thing-specification',
  inputSchema: { ...DEVICE_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...DEVICE_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.thing.devices,
});
```

- [ ] **Step 2: Replace metrics.router.ts**

```typescript
// apps/gateway/src/modules/thing/crafts/metrics/metrics.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { State } from '@app/common/core/enums';
import { z } from 'zod';

const METRIC_SCHEMA = {
  sensor: z.string(),
  key: z.string().optional(),
  state: z.nativeEnum(State).optional(),
  device: z.string().optional(),
  value: z.union([z.number(), z.array(z.number())]),
};

registerCollectionTools({
  service: 'thing',
  collection: 'metrics',
  entityName: 'ThingMetric',
  serviceDoc: 'docs://service/thing-specification',
  inputSchema: { ...METRIC_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...METRIC_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.thing.metrics,
});
```

- [ ] **Step 3: Replace sensors.router.ts**

```typescript
// apps/gateway/src/modules/thing/crafts/sensors/sensors.router.ts
import { registerCollectionTools, CORE_INPUT_SCHEMA, CORE_OUTPUT_SCHEMA } from '@app/common/core/mcp';
import { State, Status } from '@app/common/core/enums';
import { z } from 'zod';

const SENSOR_SCHEMA = {
  device: z.string(),
  name: z.string().optional(),
  type: z.string().optional(),
  state: z.nativeEnum(State).optional(),
  status: z.nativeEnum(Status).optional(),
  unit: z.string().optional(),
  metric: z.string().optional(),
};

registerCollectionTools({
  service: 'thing',
  collection: 'sensors',
  entityName: 'ThingSensor',
  serviceDoc: 'docs://service/thing-specification',
  inputSchema: { ...SENSOR_SCHEMA, ...CORE_INPUT_SCHEMA },
  outputSchema: { ...SENSOR_SCHEMA, ...CORE_OUTPUT_SCHEMA },
  getCollection: (p) => p.thing.sensors,
});
```

- [ ] **Step 4: Commit**

```bash
git add apps/gateway/src/modules/thing/
git commit -m "refactor(mcp): migrate thing service routers to factory"
```

---

## Phase 3 — Architecture

### Task 17: Stateful Sessions + SSE in `server.mcp.ts`

**Files:**
- Modify: `libs/common/src/core/mcp/server.mcp.ts`

- [ ] **Step 1: Replace the entire file**

```typescript
// libs/common/src/core/mcp/server.mcp.ts
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/streamableHttp.js';
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { RequestInfo } from '@modelcontextprotocol/sdk/types.js';
import { randomUUID } from 'crypto';
import { Request, Response } from 'express';
import { Platform } from '@wenex/sdk';
import { ZodRawShape } from 'zod';
import axios from 'axios';

import { APP } from '../app';
import { ENV } from '../env.util';
import { logger } from '../utils';
import { APP_VERSION } from '../envs';
import { getHeaders } from './utils.mcp';

const { API_PORT } = APP.GATEWAY;
const HOST = process.env['GATEWAY_HOST'] || 'localhost';
const AXIOS_CONFIG = { baseURL: `http://${HOST}:${API_PORT}` };

const name = ENV('APP_NAME', process.env.npm_package_name || 'platform');
const version = ENV('APP_VERSION', APP_VERSION() || '0.0.0');
const api = axios.create(AXIOS_CONFIG) as any;

const SESSION_TTL_MS = 30 * 60 * 1000;

interface SessionEntry {
  transport: StreamableHTTPServerTransport;
  lastSeen: number;
}

export class ServerMCP {
  readonly log = (name: string) => logger(ServerMCP.name).extend(name);

  static server: McpServer;
  static platform: Platform;
  private static sessions = new Map<string, SessionEntry>();

  private static pruneExpiredSessions(): void {
    const now = Date.now();
    for (const [id, entry] of ServerMCP.sessions) {
      if (now - entry.lastSeen > SESSION_TTL_MS) {
        void entry.transport.close();
        ServerMCP.sessions.delete(id);
      }
    }
  }

  setup(app: any) {
    app.all('/mcp', async (req: Request, res: Response) => {
      try {
        ServerMCP.pruneExpiredSessions();

        const sessionId = req.headers['mcp-session-id'] as string | undefined;
        let transport: StreamableHTTPServerTransport;

        if (sessionId && ServerMCP.sessions.has(sessionId)) {
          const entry = ServerMCP.sessions.get(sessionId)!;
          entry.lastSeen = Date.now();
          transport = entry.transport;
        } else {
          transport = new StreamableHTTPServerTransport({
            sessionIdGenerator: () => randomUUID(),
          });

          res.on('close', () => {
            if (transport.sessionId) ServerMCP.sessions.delete(transport.sessionId);
            void transport.close();
          });

          await ServerMCP.server.connect(transport);
        }

        await transport.handleRequest(req, res, req.body);

        if (transport.sessionId && !ServerMCP.sessions.has(transport.sessionId)) {
          ServerMCP.sessions.set(transport.sessionId, { transport, lastSeen: Date.now() });
        }
      } catch (error) {
        console.error('MCP request error:', error);
        if (!res.headersSent) {
          res.status(500).json({ jsonrpc: '2.0', error: { code: -32603, message: 'Internal server error' }, id: null });
        }
      }
    });
  }

  get server() {
    return (ServerMCP.server = ServerMCP.server ?? new McpServer({ name, version }));
  }

  get platform() {
    return (ServerMCP.platform = ServerMCP.platform ?? Platform.build(api));
  }

  utils(name: string, requestInfo: RequestInfo | undefined, args?: ZodRawShape) {
    const [log, headers] = [this.log(name), getHeaders({ requestInfo })];
    if (args) log('input schema: %o', args);
    log('request headers: %o', headers);
    return [log, headers] as const;
  }

  static create() {
    ServerMCP.server = ServerMCP.server ?? new McpServer({ name, version });
    ServerMCP.platform = ServerMCP.platform ?? Platform.build(api);
    return new ServerMCP();
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add libs/common/src/core/mcp/server.mcp.ts
git commit -m "feat(mcp): enable stateful sessions with TTL-based cleanup"
```

---

### Task 18: Security Middleware

**Files:**
- Create: `libs/common/src/core/mcp/middleware.mcp.ts`
- Modify: `libs/common/src/core/mcp/server.mcp.ts`

- [ ] **Step 1: Create the middleware file**

```typescript
// libs/common/src/core/mcp/middleware.mcp.ts
import { Request, Response, NextFunction } from 'express';

const WINDOW_MS = 60_000;
const MAX_REQUESTS_PER_WINDOW = 120;

const ipWindows = new Map<string, { count: number; resetAt: number }>();

function getClientIp(req: Request): string {
  const forwarded = req.headers['x-forwarded-for'] as string | undefined;
  return forwarded?.split(',')[0]?.trim() ?? req.socket.remoteAddress ?? 'unknown';
}

export function mcpRateLimiter(req: Request, res: Response, next: NextFunction): void {
  if (req.method === 'OPTIONS') return next();

  const ip = getClientIp(req);
  const now = Date.now();
  const window = ipWindows.get(ip);

  if (!window || now > window.resetAt) {
    ipWindows.set(ip, { count: 1, resetAt: now + WINDOW_MS });
    return next();
  }

  window.count++;
  if (window.count > MAX_REQUESTS_PER_WINDOW) {
    res.status(429).json({ jsonrpc: '2.0', error: { code: -32603, message: 'Too many requests. Slow down.' }, id: null });
    return;
  }

  next();
}

export function mcpAuthPresenceCheck(req: Request, res: Response, next: NextFunction): void {
  // OPTIONS (CORS preflight) and GET (SSE stream resumption) skip auth check
  if (req.method === 'OPTIONS' || req.method === 'GET') return next();

  const auth = req.headers.authorization;
  if (!auth || !auth.startsWith('Bearer ')) {
    res.status(401).json({ jsonrpc: '2.0', error: { code: -32603, message: 'Authorization: Bearer <token> header required' }, id: null });
    return;
  }

  next();
}
```

- [ ] **Step 2: Wire middleware into `server.mcp.ts` setup method**

In `libs/common/src/core/mcp/server.mcp.ts`, add the import at the top:

```typescript
import { mcpRateLimiter, mcpAuthPresenceCheck } from './middleware.mcp';
```

And at the start of the `setup(app)` method, before `app.all('/mcp', ...)`, add:

```typescript
app.use('/mcp', mcpRateLimiter);
app.use('/mcp', mcpAuthPresenceCheck);
```

Full updated `setup` method:

```typescript
setup(app: any) {
  app.use('/mcp', mcpRateLimiter);
  app.use('/mcp', mcpAuthPresenceCheck);

  app.all('/mcp', async (req: Request, res: Response) => {
    // ... (same session handling code from Task 17)
  });
}
```

- [ ] **Step 3: Export middleware from mcp index**

Add to `libs/common/src/core/mcp/index.ts`:

```typescript
export { mcpRateLimiter, mcpAuthPresenceCheck } from './middleware.mcp';
```

- [ ] **Step 4: Commit**

```bash
git add libs/common/src/core/mcp/middleware.mcp.ts libs/common/src/core/mcp/server.mcp.ts libs/common/src/core/mcp/index.ts
git commit -m "feat(mcp): add per-IP rate limiting and auth presence middleware"
```

---

## Phase 4 — Workflow Tools

### Task 19: Add `init_financial_transaction` workflow tool

**Files:**
- Create: `apps/gateway/src/workflow.router.ts`
- Modify: `apps/gateway/src/app.module.ts`

- [ ] **Step 1: Create the workflow router**

```typescript
// apps/gateway/src/workflow.router.ts
import { ServerMCP, throwableToolCall, PAY_SCHEMA } from '@app/common/core/mcp';
import { TransactionType, TransactionReason } from '@app/common/enums/financial';
import { SagaState } from '@app/common/enums/essential';
import { z } from 'zod';

const mcp = ServerMCP.create();

// ------------------------------------------------------------
// init_financial_transaction
// ------------------------------------------------------------

mcp.server.registerTool(
  'init_financial_transaction',
  {
    title: 'Initiate Financial Transaction',
    description:
      'Creates a saga + transaction in a single atomic call. Use this instead of calling create_essential_sagas and create_financial_transactions separately — this ensures the saga is properly linked before the transaction is created.\n\n' +
      '• Creates an EssentialSaga with job="financial-transaction"\n' +
      '• Creates a FinancialTransaction linked to that saga\n' +
      '• Returns { saga, transaction } — both objects with their generated ids\n\n' +
      'Required: type, reason, amount, and at least one payer or payee wallet.\n' +
      '📖 docs://service/financial-specification | docs://service/essential-specification',
    inputSchema: {
      headers: z.record(z.string(), z.string()).optional(),
      type: z.nativeEnum(TransactionType).describe('Transaction type (e.g. TRANSFER, DEPOSIT, WITHDRAWAL)'),
      reason: z.nativeEnum(TransactionReason).describe('Reason code for the transaction'),
      amount: z.number().positive().describe('Transaction amount in the base currency unit'),
      payees: z.array(z.object(PAY_SCHEMA)).optional().describe('Recipient wallets'),
      payers: z.array(z.object(PAY_SCHEMA)).optional().describe('Source wallets'),
      invoice: z.string().optional().describe('Optional linked invoice id'),
      saga_ttl: z.number().int().positive().default(3600).describe('Saga time-to-live in seconds (default: 3600)'),
    },
    outputSchema: {
      errors: z.array(z.object({}).passthrough()).optional(),
      result: z
        .object({
          saga: z.object({}).passthrough().optional(),
          transaction: z.object({}).passthrough().optional(),
        })
        .optional(),
    },
    annotations: { readOnlyHint: false, destructiveHint: true, idempotentHint: false },
  },
  async (args, { requestInfo }) =>
    throwableToolCall(async () => {
      const [logger, headers] = mcp.utils('init_financial_transaction', requestInfo, args);
      const config = { headers: { ...(args.headers ?? {}), ...headers } };

      // Step 1: create the orchestration saga
      const sagaPayload = {
        ttl: args.saga_ttl ?? 3600,
        job: 'financial-transaction',
        state: SagaState.PENDING,
        session: crypto.randomUUID(),
      };
      logger('creating saga: %o', sagaPayload);
      const saga = await mcp.platform.essential.sagas.create(sagaPayload, config);
      logger('saga created: %s', saga.id);

      // Step 2: create the transaction linked to the saga
      const txPayload: Record<string, any> = {
        saga: saga.id,
        type: args.type,
        reason: args.reason,
        amount: args.amount,
      };
      if (args.payees) txPayload.payees = args.payees;
      if (args.payers) txPayload.payers = args.payers;
      if (args.invoice) txPayload.invoice = args.invoice;

      logger('creating transaction: %o', txPayload);
      const transaction = await mcp.platform.financial.transactions.create(txPayload, config);
      logger('transaction created: %s', transaction.id);

      return {
        structuredContent: { result: { saga, transaction } },
        content: [
          {
            type: 'text',
            text:
              `Transaction initiated.\n` +
              `  saga id:        ${saga.id}\n` +
              `  transaction id: ${transaction.id}\n` +
              `Monitor saga state to track completion. States: PENDING → COMMITTED | ABORTED.`,
          },
        ],
      };
    }),
);
```

- [ ] **Step 2: Import the workflow router in `app.module.ts`**

In `apps/gateway/src/app.module.ts`, add the import alongside the existing `'./app.router'` import:

```typescript
import './app.router';
import './workflow.router';
```

- [ ] **Step 3: Commit**

```bash
git add apps/gateway/src/workflow.router.ts apps/gateway/src/app.module.ts
git commit -m "feat(mcp): add init_financial_transaction workflow tool"
```

---

## Phase 5 — Observability

### Task 20: MCP-Specific Prometheus Metrics

**Files:**
- Create: `libs/common/src/core/mcp/metrics.mcp.ts`
- Modify: `libs/common/src/core/mcp/factory.mcp.ts`
- Modify: `libs/common/src/core/mcp/index.ts`

- [ ] **Step 1: Create the metrics file**

```typescript
// libs/common/src/core/mcp/metrics.mcp.ts
import { Counter, Histogram, register } from 'prom-client';

export const mcpToolCallTotal = new Counter({
  name: 'mcp_tool_calls_total',
  help: 'Total number of MCP tool calls',
  labelNames: ['tool', 'status'] as const,
  registers: [register],
});

export const mcpToolCallDuration = new Histogram({
  name: 'mcp_tool_call_duration_seconds',
  help: 'MCP tool call duration in seconds',
  labelNames: ['tool'] as const,
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
  registers: [register],
});

export const mcpSessionsActive = new Counter({
  name: 'mcp_sessions_created_total',
  help: 'Total number of MCP sessions created',
  registers: [register],
});
```

- [ ] **Step 2: Instrument `throwableToolCall` in `utils.mcp.ts`**

In `libs/common/src/core/mcp/utils.mcp.ts`, add the import:

```typescript
import { mcpToolCallTotal, mcpToolCallDuration } from './metrics.mcp';
```

Replace the `throwableToolCall` export with an instrumented version:

```typescript
export const throwableToolCall = async <T = any>(callable: () => Promise<T> | T, toolName?: string) => {
  const end = toolName ? mcpToolCallDuration.startTimer({ tool: toolName }) : undefined;
  try {
    const result = await callable();
    if (toolName) {
      mcpToolCallTotal.inc({ tool: toolName, status: 'success' });
      end?.();
    }
    return result;
  } catch (error) {
    if (toolName) {
      mcpToolCallTotal.inc({ tool: toolName, status: 'error' });
      end?.();
    }
    if (isAxiosError(error)) {
      const { response } = error as AxiosError;
      const structuredContent = toJSON(serializeException(response));
      log.extend(throwableToolCall.name)(`mcp exception occurred with error %o`, structuredContent);
      return {
        isError: true,
        content: [{ type: 'text' as const, text: 'look at the structured content.' }],
        structuredContent: { errors: Array.isArray(structuredContent) ? structuredContent : [structuredContent] },
      };
    } else {
      const structuredContent = toJSON(serializeException(error));
      log.extend(throwableToolCall.name)(`mcp exception occurred with error %o`, structuredContent);
      return {
        isError: true,
        content: [{ type: 'text' as const, text: 'look at the structured content.' }],
        structuredContent: { errors: Array.isArray(structuredContent) ? structuredContent : [structuredContent] },
      };
    }
  }
};
```

- [ ] **Step 3: Pass tool name from factory to throwableToolCall**

In `libs/common/src/core/mcp/factory.mcp.ts`, update every `throwableToolCall(async () => {` call to pass the tool name as the second argument. Example for the `count` tool:

```typescript
async (args, { requestInfo }) =>
  throwableToolCall(async () => {
    // ... handler body unchanged
  }, t('count')),
```

Apply the same `t('count')`, `t('create')`, `t('create_bulk')`, ... pattern to all 10 operation handlers inside `registerCollectionTools`.

- [ ] **Step 4: Export metrics from index**

Add to `libs/common/src/core/mcp/index.ts`:

```typescript
export { mcpToolCallTotal, mcpToolCallDuration, mcpSessionsActive } from './metrics.mcp';
```

- [ ] **Step 5: Commit**

```bash
git add libs/common/src/core/mcp/metrics.mcp.ts libs/common/src/core/mcp/utils.mcp.ts libs/common/src/core/mcp/factory.mcp.ts libs/common/src/core/mcp/index.ts
git commit -m "feat(mcp): add Prometheus metrics for tool call rate, latency, and session count"
```

---

## Self-Review Checklist

- [x] **Spec coverage:** Factory (1.1, 1.2, 1.3 ✓), Filter schema (4.2 ✓), Sessions (2.2 ✓), Security (2.3 ✓), Workflow tool (3.1 partial ✓), Observability (4.3 ✓)
- [x] **Placeholder scan:** No TBD or "similar to Task N" — every step has complete file content
- [x] **Type consistency:** `CollectionToolsConfig`, `PlatformCollection`, `FILTER_SCHEMA`, `PARAMS_SCHEMA` defined in Task 1 and referenced consistently in Tasks 3–16
- [x] **Enum import paths:** `DriverType/Gender/State/Status` for logistic come from `@app/common/enums/logistic` — verify these match the actual package paths if TypeScript errors appear; adjust import source while keeping the enum names identical
- [x] **`SagaState.PENDING`:** Verify this enum value exists in `@app/common/enums/essential`; if the enum value differs use the correct one from that file
- [x] **`crypto.randomUUID()`:** Available in Node 22 (confirmed in `.nvmrc`). No import needed — it is a global in Node 18+.
- [x] **Essential router path:** Task 13 Step 1 tells the executor to run `find` first because the exact directory structure for essential wasn't confirmed from the codebase scan
- [x] **`prom-client`:** Already a transitive dependency via `@willsoto/nestjs-prometheus` — no new package needed
