# Ecosystem

The Wenex ecosystem is the organizational model that lets multiple independent applications share a single Platform instance without direct coupling.

## In this section

- **[Platform](./platform)** — the backend core: API gateway, 15 domain microservices, and 7 event-driven workers. Understand the request pipeline, consistent CRUD surface, and how all components connect.

- **[Client App](./client-app)** — an OAuth-registered application that proxies calls through the Platform, syncs data locally via CQRS webhooks, and adds custom business logic on top. Covers the three-layer architecture (gateway, services, workers) and key patterns like saga transactions and the platform proxy hook.
