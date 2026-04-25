# Focalboard System Architecture

## Overview

Focalboard is split into two primary components:

1. **Server** — A Go HTTP server that exposes a REST API, a WebSocket push channel, and manages all persistent state.
2. **Webapp** — A single-page React/TypeScript application that communicates with the server via REST and WebSocket.

The system supports two deployment modes:

| Mode | Description |
|------|-------------|
| **Personal Server** | Standalone binary (`server/main/`). Uses its own auth (username + password sessions). SQLite, PostgreSQL, or MySQL. |
| **Mattermost Plugin** | Embedded inside a Mattermost instance. Uses Mattermost's auth, permissions, channels, and user system. PostgreSQL or MySQL. |

---

## Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Browser (Webapp)                          │
│                                                                  │
│  React SPA (TypeScript)                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  Redux Store │  │  octoClient  │  │  WebSocket Client      │ │
│  │  (state mgmt)│  │  (REST HTTP) │  │  (wsclient.ts)         │ │
│  └──────┬───────┘  └──────┬───────┘  └────────────┬───────────┘ │
│         │                 │                        │             │
└─────────┼─────────────────┼────────────────────────┼────────────┘
          │                 │                        │
          │         REST /api/v2/*          WS /ws
          │                 │                        │
┌─────────┼─────────────────┼────────────────────────┼────────────┐
│                        Server (Go)                               │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────────────────────────┐ │
│  │   web.Server     │  │         api.API                      │ │
│  │  (gorilla/mux)   │  │  HTTP handlers (26 route files)      │ │
│  └────────┬─────────┘  └────────────────┬─────────────────────┘ │
│           │                             │                        │
│  ┌────────┴─────────────────────────────┴──────────────────────┐ │
│  │                       app.App                               │ │
│  │  Business Logic: blocks, boards, cards, auth, files,        │ │
│  │  sharing, subscriptions, categories, permissions, teams     │ │
│  └──────┬──────────────────────────────────────────────────────┘ │
│         │                                                        │
│  ┌──────┴──────────────────────────────────────────────────────┐ │
│  │  Services Layer                                             │ │
│  │  ┌───────────┐ ┌───────────┐ ┌────────────┐ ┌───────────┐  │ │
│  │  │   store   │ │   auth   │ │  notify    │ │  metrics  │  │ │
│  │  │(sqlstore) │ │(sessions)│ │(backends)  │ │(Prometheus│  │ │
│  │  └─────┬─────┘ └──────────┘ └────────────┘ └───────────┘  │ │
│  │        │       ┌───────────┐ ┌────────────┐ ┌───────────┐  │ │
│  │        │       │ permissions│ │  webhook  │ │ telemetry │  │ │
│  │        │       │(local/MM) │ │(outbound) │ │ (analytics│  │ │
│  │        │       └───────────┘ └────────────┘ └───────────┘  │ │
│  └────────┼────────────────────────────────────────────────────┘ │
│           │                                                       │
│  ┌────────┴───────────────────────────────────────────────────┐  │
│  │                   Database (SQL)                           │  │
│  │   SQLite (default) | PostgreSQL | MySQL                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   File Storage                              │ │
│  │   Local filesystem | Amazon S3 (via Mattermost filestore)   │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

---

## Layer Responsibilities

### `web.Server` (HTTP routing)
- Wraps `gorilla/mux` router
- Serves static files for the webapp
- Registers route groups for the REST API and WebSocket endpoint
- Handles TLS (optional)

### `api.API` (HTTP handler layer)
- One file per resource group (boards.go, cards.go, blocks.go, etc.)
- Reads request, calls app layer, writes JSON response
- Enforces CSRF protection and extracts session user
- Delegates permission checks to the permissions service

### `app.App` (business logic layer)
- No HTTP or database knowledge — only domain logic
- Calls `store` for persistence
- Calls `wsAdapter` to push real-time updates
- Calls `webhook.Client` for outbound notifications
- Calls `notify.Service` for subscriptions/mentions
- Calls `filesBackend` for file I/O
- Enqueues async post-write callbacks via `blockChangeNotifier`

### `store.Store` (database abstraction)
- Interface with 175+ methods covering every entity
- Implementation in `services/store/sqlstore` using `squirrel` query builder
- Supports SQLite, PostgreSQL, MySQL via `database/sql`
- Automatic schema migrations on startup

### WebSocket (`ws.Adapter`)
- Bidirectional JSON-framed WebSocket at `/ws`
- Clients subscribe to teams and individual blocks
- Server broadcasts changes (block, board, member, category, config)
- Plugin mode uses Mattermost's cluster-aware WebSocket API

---

## Data Flow Examples

### Create a Card

```
Browser
  │
  ├─ POST /api/v2/boards/{boardID}/cards  (octoClient.insertBlock)
  │
Server API (api/cards.go → handleCreateCard)
  │
  ├─ Validate session token
  ├─ Check permission: ManageBoardCards
  ├─ Parse JSON body → model.Card
  │
App Layer (app/cards.go → CreateCard)
  │
  ├─ Convert Card → Block
  ├─ Assign ID, timestamps, boardID
  ├─ store.InsertBlock() — SQL INSERT
  ├─ Enqueue blockChangeNotifier callback
  │
Async Callback
  ├─ wsAdapter.BroadcastBlockChange(teamID, block)
  │   └─ Push UPDATE_BLOCK to all team WebSocket subscribers
  ├─ webhook.NotifyUpdate(block)
  │   └─ HTTP POST block JSON to configured webhook URLs
  └─ notifications.BlockChanged(event)
      ├─ notifylogger: log
      ├─ notifysubscriptions: schedule email/DM to subscribers
      └─ notifymentions: parse @mentions, send DMs
  │
HTTP Response
  └─ 200 JSON: { block data }
```

### Load a Board (Frontend)

```
Browser navigates to /team/{teamId}/{boardId}
  │
router.tsx matches route → BoardPage component mounts
  │
  ├─ octoClient.getBoard(boardId)
  ├─ octoClient.getBlocksWithType(boardId, 'view')
  ├─ octoClient.getBlocksWithType(boardId, 'card')
  ├─ octoClient.getBoardMembers(teamId, boardId)
  │
Redux store updated:
  ├─ boards slice: setCurrent(boardId), updateBoards([board])
  ├─ views slice: setCurrent(viewId), updateViews([...views])
  ├─ cards slice: updateCards([...cards])
  │
WebSocket subscribes:
  └─ wsclient.subscribeToTeam(teamId)
       └─ sends SUBSCRIBE_TEAM over WS
  │
Render:
  └─ workspace.tsx → centerPanel.tsx
       └─ [based on activeView.fields.viewType]
            ├─ 'board'    → <Kanban />
            ├─ 'table'    → <Table />
            ├─ 'gallery'  → <Gallery />
            └─ 'calendar' → <CalendarFullView />
```

### Real-Time Update (Another User Changes a Card)

```
Other user's browser POSTes PATCH /api/v2/boards/{id}/blocks/{id}
  │
Server app layer patches block in DB
  │
wsAdapter.BroadcastBlockChange(teamID, block)
  │
WebSocket server pushes to all subscribers of that team:
  { action: "UPDATE_BLOCK", teamId: "...", data: { ...block } }
  │
wsclient.ts receives message
  │
  ├─ Debounce 100ms batch
  ├─ dispatch(updateCards([card]))  ← if it's a card
  ├─ dispatch(updateViews([view]))  ← if it's a view
  └─ onChange handlers notified
  │
React re-renders affected components automatically
```

---

## Authentication Modes

| Mode | Description | Session Storage |
|------|-------------|----------------|
| `native` | Username + password managed by Focalboard | Sessions in SQL `sessions` table |
| `mattermost` | Delegates to Mattermost's auth system | Mattermost sessions |

In native mode, login returns a session token stored in `localStorage` as `focalboardSessionId`. Every API request includes it in the `X-Auth-Token` header.

In Mattermost plugin mode, the auth token comes from Mattermost and the `auth.Auth` struct delegates permission checks to the Mattermost plugin API.

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Backend language | Go 1.19+ |
| HTTP routing | `gorilla/mux` |
| WebSocket | `gorilla/websocket` |
| Database ORM | `squirrel` (query builder) + `database/sql` |
| Database drivers | `mattn/go-sqlite3`, `lib/pq` (PostgreSQL), `go-sql-driver/mysql` |
| File storage | Mattermost `filestore` (local + S3) |
| Metrics | Prometheus (`prometheus/client_golang`) |
| Logging | Mattermost `mlog` |
| Frontend language | TypeScript |
| Frontend framework | React 17 |
| State management | Redux Toolkit |
| HTTP client | Fetch API (wrapped in `octoClient.ts`) |
| Drag and drop | `react-dnd` |
| Internationalization | `react-intl` |
| Calendar view | `@fullcalendar/react` |
| Build tool | Webpack 5 |
| Testing (Go) | `testify` |
| Testing (TS) | Jest + Enzyme |
| E2E tests | Cypress |
