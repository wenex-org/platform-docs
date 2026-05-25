# Key Concepts

These three concepts appear throughout the Platform and are worth understanding before reading any other section. They are not service-specific — they apply to every document, every read request, and every client application built on top of the Platform.

## In this section

- [Core Schema](./core-schema) — The fixed set of base fields every Platform document carries: `owner`, `shares`, `groups`, `clients`, `deleted_at`, `version`, and the timestamps. These fields are injected and enforced by the Platform automatically.
- [Access Control (ABAC)](./abac) — How read visibility is determined entirely by document attributes. Covers zone filtering (`?zone=own,share,group,client`), automatic ownership injection at write time, and the three-guard chain (`AuthGuard` → `ScopeGuard` → `PolicyGuard`) every request passes through.
- [Coworkers Space](./coworkers) — How multiple client applications are grouped so they can share data through the Platform without calling each other directly. Covers client registration, JWT claims, document-level sharing via `clients[]`, and CQRS webhook delivery.
