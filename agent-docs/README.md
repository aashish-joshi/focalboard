# Focalboard — Agent Documentation

Welcome to the Focalboard codebase documentation. This directory contains detailed, human-readable (and agent-readable) documentation covering every major area of the project: architecture, backend server, frontend webapp, features, and deployment.

## What is Focalboard?

Focalboard is an open-source, self-hosted project management tool that can run as a standalone server or as a plugin inside Mattermost. It provides kanban boards, tables, galleries, and calendar views for organizing cards (tasks/notes) with rich custom properties.

---

## Documentation Index

### Architecture
| File | Description |
|------|-------------|
| [architecture.md](./architecture.md) | High-level system architecture, component diagram, and data-flow overview |

### Server (Go backend)
| File | Description |
|------|-------------|
| [server/server-overview.md](./server/server-overview.md) | Server struct, startup sequence, and shutdown lifecycle |
| [server/api-reference.md](./server/api-reference.md) | All REST API routes, middleware, and request/response patterns |
| [server/app-layer.md](./server/app-layer.md) | Business logic layer: App struct and all domain methods |
| [server/data-models.md](./server/data-models.md) | Go model types: Block, Board, Card, User, Team, etc. |
| [server/database-store.md](./server/database-store.md) | Store interface and SQLite/PostgreSQL/MySQL implementation |
| [server/services.md](./server/services.md) | Backend services: auth, config, metrics, notify, webhook, telemetry, audit |
| [server/websocket.md](./server/websocket.md) | WebSocket server, message protocol, and plugin adapter |

### Webapp (React/TypeScript frontend)
| File | Description |
|------|-------------|
| [webapp/webapp-overview.md](./webapp/webapp-overview.md) | Entry points, routing, and overall frontend structure |
| [webapp/state-management.md](./webapp/state-management.md) | Redux Toolkit store: all slices, selectors, and async thunks |
| [webapp/components.md](./webapp/components.md) | All key React components and their props/behavior |
| [webapp/api-client.md](./webapp/api-client.md) | `octoClient.ts` — HTTP client wrapping all server API calls |
| [webapp/mutator.md](./webapp/mutator.md) | `mutator.ts` — Undo/redo mutation layer |
| [webapp/blocks.md](./webapp/blocks.md) | Frontend block data model and TypeScript types |
| [webapp/properties.md](./webapp/properties.md) | Card property types, registry, and UI components |
| [webapp/websocket-client.md](./webapp/websocket-client.md) | `wsclient.ts` — Real-time updates via WebSocket |

### Features
| File | Description |
|------|-------------|
| [features/boards.md](./features/boards.md) | Board creation, types, duplication, and metadata |
| [features/cards.md](./features/cards.md) | Card creation, properties, content blocks, and comments |
| [features/views.md](./features/views.md) | Board views: kanban, table, gallery, calendar |
| [features/notifications.md](./features/notifications.md) | Subscriptions, mentions, and notification delivery |
| [features/sharing.md](./features/sharing.md) | Public read-only board sharing and link tokens |
| [features/permissions.md](./features/permissions.md) | Permission model: board roles, team roles, and system permissions |
| [features/templates.md](./features/templates.md) | Board and card templates |
| [features/categories.md](./features/categories.md) | Sidebar categories for organizing boards |

### Deployment
| File | Description |
|------|-------------|
| [deployment/configuration.md](./deployment/configuration.md) | All configuration options and environment variables |

---

## Quick Reference: Code Locations

| Area | Path |
|------|------|
| Server entry point | `server/main/main.go` |
| Server struct | `server/server/server.go` |
| HTTP API handlers | `server/api/` |
| Business logic | `server/app/` |
| Data models (Go) | `server/model/` |
| Database store | `server/services/store/` |
| WebSocket server | `server/ws/` |
| Auth service | `server/auth/` |
| Config service | `server/services/config/` |
| Notifications | `server/services/notify/` |
| Frontend entry | `webapp/src/main.tsx` |
| React app root | `webapp/src/app.tsx` |
| Route definitions | `webapp/src/router.tsx` |
| Redux store | `webapp/src/store/` |
| HTTP client | `webapp/src/octoClient.ts` |
| Mutation layer | `webapp/src/mutator.ts` |
| WebSocket client | `webapp/src/wsclient.ts` |
| React components | `webapp/src/components/` |
| Block types (TS) | `webapp/src/blocks/` |
| Property types | `webapp/src/properties/` |
| i18n messages | `webapp/src/i18n/` |
