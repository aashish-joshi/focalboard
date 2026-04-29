# WebSocket Server

## Overview

**Directory:** `server/ws/`

The WebSocket layer provides real-time push notifications from server to browser. When a block, board, member, category, or config changes, the server broadcasts a JSON message to all connected clients that have subscribed to the relevant team or blocks.

---

## ws.Adapter Interface

**File:** `server/ws/adapter.go`

All WebSocket functionality is behind this interface, enabling two different implementations:

```go
type Adapter interface {
    BroadcastBlockChange(teamID string, block *model.Block)
    BroadcastBlockDelete(teamID, blockID, boardID string)
    BroadcastBoardChange(teamID string, board *model.Board)
    BroadcastBoardDelete(teamID, boardID string)
    BroadcastMemberChange(teamID, boardID string, member *model.BoardMember)
    BroadcastMemberDelete(teamID, boardID, userID string)
    BroadcastConfigChange(clientConfig model.ClientConfig)
    BroadcastCategoryChange(category model.Category)
    BroadcastCategoryBoardChange(teamID, userID string, categoryData model.CategoryBoards)
    BroadcastCardLimitTimestampChange(timestamp int64)
    BroadcastSubscriptionChange(teamID string, subscription *model.Subscription)
    BroadcastCategoryReorder(teamID, userID string, categoryOrder []string)
    BroadcastCategoryBoardsReorder(teamID, userID, categoryID string, boardsOrder []string)
}
```

Two implementations:
1. **`ws.Server`** — Standalone server WebSocket handler
2. **`ws.PluginAdapter`** — Mattermost plugin WebSocket adapter

---

## Standalone WebSocket Server

**File:** `server/ws/server.go`

### Server struct

```go
type Server struct {
    upgrader         websocket.Upgrader
    listeners        map[*websocketSession]bool
    listenersByTeam  map[string][]*websocketSession
    listenersByBlock map[string][]*websocketSession
    mu               sync.RWMutex
    auth             *auth.Auth
    singleUserToken  string
    isMattermostAuth bool
    logger           mlog.LoggerIFace
    store            Store
}
```

### websocketSession

```go
type websocketSession struct {
    conn   *websocket.Conn
    userID string
    mu     sync.Mutex
    teams  []string   // subscribed team IDs
    blocks []string   // subscribed block IDs
}
```

### HTTP Upgrade

The WebSocket endpoint is registered at `/ws` (or `/board` in some modes). When a client connects:

1. HTTP connection is upgraded to WebSocket via `websocket.Upgrader`
2. A new `websocketSession` is created and added to `listeners`
3. A goroutine loops reading messages from the client

### Client→Server Commands

| Command | Payload | Description |
|---------|---------|-------------|
| `AUTH` | `{ "token": "..." }` | Authenticate the WS session with a session token or single-user token |
| `SUBSCRIBE_TEAM` | `{ "teamId": "..." }` | Subscribe to all changes in a team |
| `UNSUBSCRIBE_TEAM` | `{ "teamId": "..." }` | Unsubscribe from team changes |
| `SUBSCRIBE_BLOCKS` | `{ "blockIds": [...] }` | Subscribe to changes of specific blocks |
| `UNSUBSCRIBE_BLOCKS` | `{ "blockIds": [...] }` | Unsubscribe from specific blocks |

### Server→Client Message Format

```json
{
  "action": "ACTION_NAME",
  "teamId": "team-id",
  "data": { ... }
}
```

### Server→Client Action Types

| Action | Data | Description |
|--------|------|-------------|
| `UPDATE_BLOCK` | `Block` | Block was created, updated, or soft-deleted |
| `UPDATE_BOARD` | `Board` | Board was created or updated |
| `DELETE_BOARD` | `{ boardId }` | Board was deleted |
| `UPDATE_MEMBER` | `BoardMember` | Board member was added or updated |
| `DELETE_MEMBER` | `{ boardId, userId }` | Board member was removed |
| `UPDATE_CLIENT_CONFIG` | `ClientConfig` | Server config changed |
| `UPDATE_CATEGORY` | `Category` | Sidebar category changed |
| `UPDATE_BOARD_CATEGORY` | `CategoryBoards` | Board moved to different category |
| `UPDATE_SUBSCRIPTION` | `Subscription` | Block subscription changed |
| `UPDATE_CARD_LIMIT_TIMESTAMP` | `int64` | Cloud card limit timestamp changed |
| `REORDER_CATEGORIES` | `[]string` | Category order changed |
| `REORDER_CATEGORY_BOARDS` | `{ categoryId, boardsOrder }` | Board order in category changed |

### Broadcast Functions

Each broadcast function:
1. Acquires a read lock on `mu`
2. Iterates over `listenersByTeam[teamID]`
3. Sends JSON message to each session's WebSocket connection
4. Logs errors but does not disconnect on send failure

---

## Plugin Adapter

**File:** `server/ws/plugin_adapter.go`

Used when Focalboard runs as a Mattermost plugin. Instead of managing its own WebSocket server, it uses Mattermost's cluster-aware WebSocket API.

```go
type PluginAdapter struct {
    api        PluginAPI
    staleThreshold int64
    mu             sync.RWMutex
    listeners      map[string]*PluginAdapterClient
    listenersByTeam map[string][]*PluginAdapterClient
    listenersByBlock map[string][]*PluginAdapterClient
    logger     mlog.LoggerIFace
}
```

### Plugin API Interface

```go
type PluginAPI interface {
    PublishWebSocketEvent(event string, payload map[string]interface{}, broadcast *mm_model.WebsocketBroadcast)
    GetDirectChannel(userID1, userID2 string) (*mm_model.Channel, *mm_model.AppError)
}
```

### Cluster Support

**File:** `server/ws/plugin_adapter_cluster.go`

In multi-instance Mattermost deployments, WebSocket events are published to the entire cluster using Mattermost's `PublishWebSocketEvent`. This ensures all instances forward the event to their connected clients.

---

## Connection Lifecycle

```
Browser connects to ws://<host>/ws
  │
Server upgrades to WebSocket
  │
Session created, added to listeners map
  │
Client sends AUTH command
  ├─ Token validated via auth.GetSession()
  └─ userID set on session
  │
Client sends SUBSCRIBE_TEAM { teamId }
  └─ Session added to listenersByTeam[teamId]
  │
Server broadcasts changes to listenersByTeam[teamId]
  │
Client sends UNSUBSCRIBE_TEAM or disconnects
  └─ Session removed from listenersByTeam and listeners
```

---

## Thread Safety

All map operations on `listeners`, `listenersByTeam`, `listenersByBlock` are protected by `sync.RWMutex`:
- Reads (broadcast) use `RLock()`
- Writes (subscribe/unsubscribe/connect/disconnect) use `Lock()`

Each `websocketSession.conn` write is protected by its own `sync.Mutex` to prevent concurrent writes to the same WebSocket connection.
