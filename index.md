---
layout: home

hero:
  name: Wenex Platform
  text: Application Production Factory
  tagline: A well-structured, open-source microservice platform built with TypeScript and NestJS — designed to make your enterprise fast and efficient.
  actions:
    - theme: brand
      text: Get Started
      link: /getting-started
    - theme: alt
      text: GitHub
      link: https://github.com/wenex-org/platform

features:
  - icon: 🗄️
    title: Object Storage and NoSQL
    details: MongoDB as the primary entity store, MinIO for object storage, and Redis for caching and sessions — all integrated and production-ready out of the box.
    link: /getting-started/overview/ecosystem/platform

  - icon: 🔗
    title: RESTful, GraphQL, and gRPC
    details: A unified gateway exposing REST, GraphQL, and gRPC across 14 domain microservices with a consistent CRUD surface on every collection.
    link: /api/rest-reference

  - icon: 📊
    title: Observability and Monitoring
    details: Prometheus metrics, OpenTelemetry distributed traces, Elasticsearch log aggregation, and health check endpoints on every service and worker.
    link: /getting-started/overview/ecosystem/platform

  - icon: 🤖
    title: AI Agent Integration via MCP
    details: Native Model Context Protocol server at /mcp — Claude, GPT, and Ollama-backed agents can query and manipulate platform data using structured tool calls.
    link: /mcp/overview

  - icon: 🚀
    title: GitLab CI/CD and K8s Integration
    details: Production-ready CI/CD pipelines and Kubernetes deployment manifests for automated, scalable delivery of all gateway, service, and worker components.
    link: /getting-started/setup/kubernetes-setup

  - icon: 🧠
    title: MLOps
    details: Machine Learning Operations support — integrate, deploy, and monitor ML models alongside platform services with the same observability and lifecycle tooling.
    link: /mlops/
---

Wenex is a general-purpose [platform](https://github.com/wenex-org/platform) with a microservice design, empowered through an official [SDK](https://github.com/wenex-org/platform-sdk). If you need to migrate an existing system into this ecosystem, a ready-made solution using [CDC](https://www.confluent.io/learn/change-data-capture/) and [Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html) is available in the [CDC repository](https://github.com/wenex-org/cdc).

Wenex covers a wide range of domains — Communication, E-Learning, Online Shopping, Banking, IoT, Crypto, and more — and is free to use under the [Apache 2.0 license](https://github.com/wenex-org/platform/blob/main/LICENSE).
