# Logger

**Port:** `:4060`  
**Source:** `apps/workers/logger/`

The logger persists audit records to PostgreSQL. It consumes audit log events emitted by all platform services via Kafka and writes them to a durable relational store for compliance, debugging, and traceability.

## Responsibilities

- Consume `LoggerInterface` payloads from Kafka.
- Persist each payload as an `Audit` record in PostgreSQL via `AuditsService`.

## Audit Record

The `Audit` entity (defined in `modules/audits/entities/audit.entity.ts`) stores the structured log emitted by services when handling requests. Fields typically include the requesting user, client, action performed, target resource, and timestamps.

## Infrastructure Dependencies

| Dependency | Usage |
|---|---|
| Kafka | Consumes audit log events from all services |
| PostgreSQL | Stores `Audit` records for durable retention |

## Retention

Expired audit records are purged by the [cleaner](./cleaner) worker based on the `CLEANER_AUDIT_LOGS_TTL` environment variable (defaults to 4 years).

## Key Files

| File | Purpose |
|---|---|
| `app.service.ts` | `audit()` — delegates to `AuditsService.create()` |
| `app.controller.ts` | `/status`, `/metrics` |
| `modules/audits/audits.service.ts` | Persists `LoggerInterface` payload to PostgreSQL |
| `modules/audits/entities/audit.entity.ts` | TypeORM `Audit` entity definition |
