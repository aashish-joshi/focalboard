# Notifications

## Overview

Focalboard supports a notification system that alerts users when boards or cards they are watching are changed. Notifications are primarily used in Mattermost plugin mode, but the architecture is extensible.

---

## Notification Architecture

```
Block change in DB
  │
app.blockChangeNotifier.Enqueue(callback)
  │
(async) callback executes:
  │
  └─ notifications.BlockChanged(event)
       │
       ├─ notifylogger backend
       │   └─ Logs the change (always enabled)
       │
       ├─ notifysubscriptions backend (plugin mode)
       │   ├─ Find all subscribers for the changed block
       │   ├─ Store NotificationHint in DB (scheduled for later)
       │   └─ A goroutine dequeues hints and sends notifications
       │
       ├─ notifymentions backend (plugin mode)
       │   ├─ Parse @username mentions in block title/content
       │   └─ Send DM via Mattermost plugin API
       │
       └─ plugindelivery backend (plugin mode)
           └─ Format and deliver notification messages
```

---

## Subscriptions

A **Subscription** links a user (or channel) to a board or card. When that entity changes, the subscriber receives a notification.

### Subscription Model

```go
type Subscription struct {
    BlockType      BlockType      // "board" or "card"
    BlockID        string         // ID of watched board/card
    SubscriberType SubscriberType // "user" or "channel"
    SubscriberID   string         // User ID or channel ID
    NotifiedAt     int64          // Last notification time (ms)
    CreateAt       int64
    DeleteAt       int64
}
```

### Subscribe to a Block

**Frontend:**
```typescript
// Follow a card
wsClient.setOnFollowBlock((sub) => dispatch(followBlock(sub)))

// Through octoClient:
octoClient.createSubscription({
    blockType: 'card',
    blockId: cardId,
    subscriberType: 'user',
    subscriberId: userId
})
```

**API:** `POST /api/v2/subscriptions`

**Server:** `app.CreateSubscription()` → `store.CreateSubscription()`

### Unsubscribe

**API:** `DELETE /api/v2/subscriptions/{blockID}/{subscriberID}`

---

## Notification Hints

When a block changes, the `notifysubscriptions` backend creates a **NotificationHint** in the database. This serves as a work queue for deferred notification delivery.

```go
type NotificationHint struct {
    BlockType    BlockType   // Type of changed block
    BlockID      string      // ID of changed block
    ModifiedByID string      // User who made the change
    CreateAt     int64       // When hint was created
    NotifyAt     int64       // Scheduled delivery time
}
```

### Hint Scheduling

- **Card changes:** scheduled after `config.NotifyFreqCardSeconds` (default: 120 seconds)
- **Board changes:** scheduled after `config.NotifyFreqBoardSeconds` (default: 86400 seconds / 1 day)

The delay groups multiple rapid changes into a single notification, preventing notification spam.

### Hint Processing

A background goroutine (`notifysubscriptions` backend) polls `store.GetNextNotificationHint()` and:
1. Gets all subscribers for the changed block
2. Gets the block's change since `subscriber.NotifiedAt`
3. Formats a notification message
4. Delivers via `plugindelivery` backend
5. Updates `subscriber.NotifiedAt` to current time

---

## Mention Notifications (@mentions)

The `notifymentions` backend processes `@username` mentions in block content.

### Detection

When a block changes, the `notifymentions` backend:
1. Extracts `@username` patterns from the block's `Title` and content fields
2. Resolves usernames to user IDs via `store.GetUserByUsername()`
3. Sends a direct message to each mentioned user via `store.SendMessage()`

Mentions in the previous block state (`BlockOld`) are not re-notified.

---

## Notification Delivery (Plugin Mode)

The `plugindelivery` backend formats and sends notifications through Mattermost:

### Direct Messages

Subscription notifications are sent as direct messages from a system bot user to the subscriber.

### Message Format

Example notification:
```
@alice updated the card "Fix login bug" in the "Bug Tracker" board.
> Status changed from "In Progress" to "Done"
```

### Channel Posts

If a subscription has `SubscriberType = "channel"`, notifications are posted to that Mattermost channel.

---

## WebSocket Subscription Updates

Subscription changes (create/delete) are also broadcast over WebSocket:

```
Server creates/deletes subscription
  │
wsAdapter.BroadcastSubscriptionChange(teamID, subscription)
  │
WebSocket message: ACTION_UPDATE_SUBSCRIPTION
  │
wsclient.ts dispatches:
  └─ subscription.DeleteAt = 0 → dispatch(followBlock(sub))
  └─ subscription.DeleteAt > 0 → dispatch(unfollowBlock(sub))
```

---

## Frontend Subscription Management

The `users` Redux slice tracks the current user's block subscriptions in `blockSubscriptions`.

```typescript
// Check if user is following a block
const isFollowing = useAppSelector(state =>
    getBlockSubscription(state, blockId, 'user') !== undefined
)

// Follow
octoClient.createSubscription({ blockId, blockType: 'card', subscriberType: 'user', subscriberId: userId })
dispatch(followBlock(subscription))

// Unfollow
octoClient.deleteSubscription(blockId, 'user', userId)
dispatch(unfollowBlock({ blockId }))
```

---

## Notification Service Interface

```go
type Backend interface {
    Start() error
    ShutDown() error
    BlockChanged(evt BlockChangeEvent) error
    Name() string
}

type BlockChangeEvent struct {
    Action       Action          // "add", "update", "delete"
    TeamID       string
    Board        *model.Board
    Card         *model.Block    // Parent card
    BlockChanged *model.Block    // Changed block
    BlockOld     *model.Block    // Previous state
    ModifiedBy   *model.BoardMember
}
```
