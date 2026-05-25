# Setup

Quick-start guide for running the Wenex platform.

## In this section

- [Prerequisites](./prerequisites) — Node.js, Docker, and Git requirements.
- [Start Infrastructure](./start-infrastructure) — Launch the core infrastructure stack with Docker Compose.
- [Manually Setup](./manually-setup) — Clone, install, initialize, and start the platform from source.
- [Docker Setup](./docker-setup) — Run the full platform as Docker containers.
- [Kubernetes Setup](./kubernetes-setup) — Deploy using the official Helm chart.
- [Configuration](./configuration) — Full environment variable and Helm values reference.

## Prerequisites

**Git** is required to clone the repository and manage submodules. Any recent version works.

**Docker and Docker Compose** are required to run the infrastructure stack. Any recent version works.

**Node.js 22.x and pnpm 10.5.2** are required for the platform itself. The repository includes an `.nvmrc` file pinning the exact Node version. Use [nvm](https://github.com/nvm-sh/nvm) to install and activate it, then enable [corepack](https://nodejs.org/api/corepack.html) so the correct pnpm version is available automatically:

```bash
nvm install   # installs the Node version from .nvmrc
nvm use       # switches to it
corepack enable
```
