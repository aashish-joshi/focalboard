# WebSocket Client (`wsclient.ts`)

## Overview

**File:** `webapp/src/wsclient.ts`

The WebSocket client maintains a persistent connection to the server and handles real-time updates. It uses a publish-subscribe pattern with typed change handlers.

```typescript
class WSClient {
    ws: WebSocket | null
    serverUrl: string
    state: 'init' | 'open' | 'close'
    
    // Reconnect state
    pingTimeout: ReturnType<typeof setTimeout> | null
    reopenDelay: number
    maximumReconnectAttempts: number
    reopenAttempts: number
    
    // Batch processing
    updateTimeout: ReturnType<typeof setTimeout> | null  // 100ms debounce
    batchedUpdates: {[key: string]: [...]}
}

const wsClient = new WSClient()
export default wsClient
```

---

## Connection Lifecycle

```
wsClient.open(serverUrl, sessionId)
  │
  ├─ new WebSocket(url + '/ws')
  ├─ ws.onopen → state = 'open', send AUTH message
  ├─ ws.onmessage → dispatch to handlers
  ├─ ws.onerror → log error
  └─ ws.onclose → schedule reconnect
  │
If disconnected:
  ├─ Wait reopenDelay seconds (exponential backoff)
  ├─ reopenDelay doubles each attempt (max ~30s)
  ├─ max 10 reconnect attempts
  └─ On reconnect: re-subscribe to all teams
```

### Open / Close

```typescript
open(serverUrl: string, token: string): void
// Connects, authenticates, and subscribes

close(): void
// Gracefully closes connection
```

---

## Client→Server Messages

All messages are JSON strings:

```typescript
interface WSMessage {
    action: string
    payload?: any
}
```

| Action | Payload | Description |
|--------|---------|-------------|
| `AUTH` | `{ token: "..." }` | Authenticate the WS session |
| `SUBSCRIBE_TEAM` | `{ teamId: "..." }` | Subscribe to team changes |
| `UNSUBSCRIBE_TEAM` | `{ teamId: "..." }` | Unsubscribe from team |
| `SUBSCRIBE_BLOCKS` | `{ blockIds: [...] }` | Subscribe to specific block changes |
| `UNSUBSCRIBE_BLOCKS` | `{ blockIds: [...] }` | Unsubscribe from blocks |

---

## Server→Client Messages

```typescript
interface WSMessage {
    action: string
    teamId?: string
    data?: any
}
```

| Action | Data Type | Redux Dispatch |
|--------|-----------|----------------|
| `UPDATE_BLOCK` | `Block` | `updateCards` / `updateViews` / `updateComments` / `updateContents` |
| `UPDATE_BOARD` | `Board` | `updateBoards` |
| `DELETE_BOARD` | `{ boardId: string }` | `deleteBoards([id])` |
| `UPDATE_MEMBER` | `BoardMember` | `updateMembersEnsuringBoardsAndUsers` |
| `DELETE_MEMBER` | `{ boardId, userId }` | `deleteBoardMember` |
| `UPDATE_CLIENT_CONFIG` | `ClientConfig` | `setClientConfig` |
| `UPDATE_CATEGORY` | `Category` | `updateCategories` |
| `UPDATE_BOARD_CATEGORY` | `CategoryBoards` | `updateBoardCategories` |
| `UPDATE_SUBSCRIPTION` | `Subscription` | `followBlock` / `unfollowBlock` |
| `UPDATE_CARD_LIMIT_TIMESTAMP` | `number` | `setLimitTimestamp` |
| `REORDER_CATEGORIES` | `string[]` | `updateCategoryOrder` |
| `REORDER_CATEGORY_BOARDS` | `{ categoryId, boardsOrder }` | `updateCategoryBoardsOrder` |

---

## Message Batching

Incoming messages are batched to avoid flooding React with rapid Redux updates:

```typescript
// When a message arrives:
clearTimeout(updateTimeout)
batchedUpdates[action].push(data)
updateTimeout = setTimeout(dispatchBatchedUpdates, 100)
```

After 100ms of no new messages, the batched data is dispatched in a single `batch()` call to Redux, which results in a single React re-render.

---

## Change Handlers (Subscription API)

External code (components, `withWebSockets` HOC) can register handlers for specific event types:

```typescript
type ChangeHandlerType =
    | 'block'
    | 'board'
    | 'boardMembers'
    | 'category'
    | 'blockCategories'
    | 'categoryOrder'
```

### Adding/Removing Handlers

```typescript
wsClient.addOnChange(handler: (blocks: Block[]) => void, 'block')
wsClient.removeOnChange(handler, 'block')

wsClient.addOnChange(handler: (boards: Board[]) => void, 'board')
wsClient.removeOnChange(handler, 'board')

wsClient.addOnChange(handler: (members: BoardMember[]) => void, 'boardMembers')
wsClient.removeOnChange(handler, 'boardMembers')
```

### Event Handlers

```typescript
// Connection state changes
wsClient.onStateChange = (state: 'init' | 'open' | 'close') => void

// Reconnect (server restarted)
wsClient.onReconnect = () => void

// WebSocket error
wsClient.onError = (err: Event) => void

// Client config changed
wsClient.onConfigChange = (config: ClientConfig) => void

// Cloud card limit changed
wsClient.onCardLimitTimestampChange = (timestamp: number) => void

// Block subscription events
wsClient.setOnFollowBlock(handler: (subscription: Subscription) => void)
wsClient.setOnUnfollowBlock(handler: (subscription: Subscription) => void)
```

---

## Team Subscriptions

```typescript
wsClient.subscribeToTeam(teamId: string): void
// Sends SUBSCRIBE_TEAM message

wsClient.unsubscribeToTeam(teamId: string): void
// Sends UNSUBSCRIBE_TEAM message
```

Team subscriptions are tracked in `subscribedTeams` so they can be restored after reconnect.

---

## WithWebSockets HOC (`components/withWebSockets.tsx`)

A React Higher-Order Component that wraps the entire app and:
1. Opens the WebSocket connection on mount using the current user's session token
2. Re-opens on login / session token change
3. Registers Redux dispatch handlers for all WS message types
4. Handles component unmount by closing the connection

```tsx
function WithWebSockets(WrappedComponent: React.ComponentType): React.FC {
    return function(props) {
        useEffect(() => {
            wsClient.open(serverUrl, sessionId)
            wsClient.onStateChange = ...
            wsClient.onReconnect = ...
            // register all change handlers → dispatch to Redux
            return () => wsClient.close()
        }, [sessionId])
        
        return <WrappedComponent {...props} />
    }
}
```

---

## Error Handling and Reconnection

```
ws.onclose or ws.onerror
  │
  ├─ state = 'close'
  ├─ call onStateChange('close') handlers
  ├─ reopenAttempts++
  │
  └─ If reopenAttempts < maximumReconnectAttempts:
       setTimeout(reconnect, reopenDelay)
       reopenDelay = min(reopenDelay * 2, 30000)
       
reconnect():
  ├─ open new WebSocket
  └─ re-authenticate + re-subscribe to teams on 'open'
```

Maximum reconnect attempts: **10**
Initial delay: **1 second**
Maximum delay: **30 seconds**
