# VitePress v2 Alpha — Docs Site Design

**Date:** 2026-05-17
**Status:** Approved

---

## Goal

Add VitePress 2 alpha to serve the existing `docs/` directory as a static documentation site for the Wenex Platform. The setup must be root-integrated (scripts in the root `package.json`), use auto-generated sidebar (zero maintenance), and produce a static build suitable for self-hosted deployment.

---

## Decisions

| Question | Decision |
|----------|----------|
| Integration | Root-integrated (scripts in root `package.json`) |
| Sidebar | Auto-generated via `vitepress-sidebar` plugin |
| Deployment target | Static build, self-hosted (no special `base` path) |

---

## Submodule Note

`docs/` is a **git submodule** (`platform-docs.git`, branch `claude`). Changes to `docs/` must be committed inside the submodule; the main repo then updates its submodule pointer. This means:

- `docs/.vitepress/config.mts` and `docs/index.md` → committed to the `docs` submodule
- Root `package.json` scripts + devDeps → committed to the main repo
- `.gitignore` entries for VitePress output → added to both the docs submodule's own `.gitignore` **and** the root `.gitignore`

---

## File Structure

```
platform/                          (main repo)
├── docs/                          (submodule: platform-docs.git)
│   ├── .vitepress/
│   │   └── config.mts             # NEW — VitePress config
│   ├── .gitignore                 # MODIFIED — dist & cache entries
│   ├── index.md                   # NEW — VitePress home page (hero layout)
│   ├── README.md                  # UNCHANGED — excluded from sidebar
│   └── ... (all existing .md files unchanged)
├── package.json                   # MODIFIED — 3 scripts + 2 devDeps
└── .gitignore                     # MODIFIED — dist & cache entries
```

No existing file is renamed or deleted.

---

## Package Changes

### `package.json` — scripts

```json
"docs:dev":     "vitepress dev docs",
"docs:build":   "vitepress build docs",
"docs:preview": "vitepress preview docs"
```

### `package.json` — devDependencies

```json
"vitepress": "2.0.0-alpha.x",
"vitepress-sidebar": "^2.0.0"
```

Installed via:

```bash
pnpm add -D vitepress@alpha vitepress-sidebar
```

### `.gitignore` additions

```
docs/.vitepress/dist
docs/.vitepress/cache
```

---

## VitePress Config (`docs/.vitepress/config.mts`)

```ts
import { defineConfig } from 'vitepress'
import { generateSidebar } from 'vitepress-sidebar'

export default defineConfig({
  title: 'Wenex Platform',
  description: 'Developer documentation for Wenex Platform v1.6.0',
  srcDir: '.',
  outDir: '.vitepress/dist',

  themeConfig: {
    nav: [
      { text: 'Getting Started', link: '/getting-started' },
      { text: 'API', link: '/api/rest-reference' },
      { text: 'SDK', link: '/sdk/' },
      { text: 'Services', link: '/services/' },
      { text: 'MCP', link: '/mcp/overview' },
    ],

    sidebar: generateSidebar({
      documentRootPath: 'docs',
      collapsed: false,
      capitalizeFirst: true,
      excludePattern: ['README.md', 'LICENSE'],
    }),

    socialLinks: [
      { icon: 'github', link: 'https://github.com/wenex-org/platform' },
    ],

    search: { provider: 'local' },
  },
})
```

---

## Home Page (`docs/index.md`)

```markdown
---
layout: home

hero:
  name: Wenex Platform
  text: Developer Documentation
  tagline: Distributed microservices — REST, GraphQL, gRPC, MCP
  actions:
    - theme: brand
      text: Get Started
      link: /getting-started
    - theme: alt
      text: API Reference
      link: /api/rest-reference

features:
  - title: 15 Microservices
    details: Domain-driven services across auth, identity, financial, content, and more.
  - title: Multi-Protocol Gateway
    details: Unified REST, GraphQL, and MCP entry point on port 3010.
  - title: AI-Ready via MCP
    details: Native Model Context Protocol server for Claude and other AI agents.
---
```

---

## Out of Scope

- Custom theme / CSS overrides
- Algolia search (local search provider used instead)
- GitHub Pages / CI deployment pipeline
- Internationalization
