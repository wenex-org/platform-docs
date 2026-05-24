# Overview

Before diving into setup, it helps to understand the three layers that make up the Wenex world: the **Ecosystem** model that governs how applications coexist, the **Platform** that provides the shared data and API surface, and the **Client** pattern for building applications on top of it.

## In this section

- [Ecosystem](./overview/ecosystem) — How independent client applications share data, how identity and ownership work, and how the ABAC model enforces access control.
- [Platform](./overview/platform) — The distributed monorepo: gateway, domain microservices, workers, and the data layer they all share.
- [Client](./overview/client) — The canonical architecture for building a client application: gateway, services, and workers wired to the Platform via SDK and CQRS webhooks.
