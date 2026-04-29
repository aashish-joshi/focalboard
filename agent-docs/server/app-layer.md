# App Layer — Business Logic

## Overview

**File:** `server/app/app.go`

The `App` struct is the business logic layer. It sits between the HTTP API handlers and the database store. It knows nothing about HTTP — it receives plain Go values and returns plain Go values or errors.

```go
type App struct {
    config              *config.Configuration
    store               store.Store              // Database abstraction
    auth                *auth.Auth               // Session/auth management
    wsAdapter           ws.Adapter               // WebSocket broadcaster
    filesBackend        fileBackend              // File I/O
    webhook             *webhook.Client          // Outbound webhooks
    metrics             *metrics.Metrics         // Prometheus metrics
    notifications       *notify.Service          // Notification backends
    logger              mlog.LoggerIFace
    permissions         permissions.PermissionsService
    blockChangeNotifier *utils.CallbackQueue     // Async notification queue
    servicesAPI         servicesAPI              // Mattermost plugin API

    cardLimitMux sync.RWMutex
    cardLimit    int
}
```

### Construction

```go
func New(config, wsAdapter, services) *App
```

Injects all services, calls `initialize()` to ensure root team and templates exist.

### Key lifecycle methods

| Method | Description |
|--------|-------------|
| `SetConfig(cfg)` | Hot-reload configuration |
| `GetConfig()` | Return current config |
| `CardLimit()` | Get current card limit (cloud) |
| `SetCardLimit(n)` | Set card limit (cloud) |
| `GetLicense()` | Get Mattermost license from store |
| `Shutdown()` | Stop `blockChangeNotifier` queue |

---

## Initialization (`initialize.go`)

Called from `New()`:
1. `GetRootTeam()` — ensure team ID `"0"` exists
2. `RegisterDefaultTemplates()` — load built-in templates from `templates.boardarchive`

---

## Blocks (`app/blocks.go`)

Core CRUD operations for the block entity.

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetBlocks` | `(opts QueryBlocksOptions) ([]*model.Block, error)` | Query blocks with optional filters: boardID, parentID, blockType, pagination |
| `GetBlocksWithParentAndType` | `(boardID, parentID, blockType string) ([]*model.Block, error)` | Convenience wrapper |
| `GetBlocksWithParent` | `(boardID, parentID string) ([]*model.Block, error)` | All children of a parent |
| `GetBlocksWithType` | `(boardID, blockType string) ([]*model.Block, error)` | All blocks of a given type |
| `GetSubTree2` | `(boardID, blockID string, opts QuerySubtreeOptions) ([]*model.Block, error)` | Block and immediate children |
| `GetBlockByID` | `(blockID string) (*model.Block, error)` | Single block by ID |
| `DuplicateBlock` | `(boardID, blockID, userID string, asTemplate bool) ([]*model.Block, *model.Board, error)` | Deep copy a block |
| `PatchBlock` | `(blockID, boardID string, patch *model.BlockPatch, userID string) error` | Apply a patch |
| `PatchBlockAndNotify` | same + notifications | Patch and broadcast change |
| `PatchBlocks` | `(patch *model.BlockPatchBatch, userID string) error` | Batch patch |
| `PatchBlocksAndNotify` | same + notifications | Batch patch and broadcast |
| `InsertBlock` | `(block *model.Block, userID string) error` | Insert/update a block |
| `InsertBlocksAndNotify` | `(blocks []*model.Block, userID string) error` | Insert/update and broadcast |
| `DeleteBlock` | `(blockID, boardID, userID string) error` | Soft-delete a block |
| `UndeleteBlock` | `(blockID, boardID, userID string) error` | Restore a soft-deleted block |

### Notifications on Block Changes

After every mutating block operation, `notifyBlockChanged()` is called asynchronously via `blockChangeNotifier`:

```
notifyBlockChanged(action, block, oldBlock, userID)
  ├─ wsAdapter.BroadcastBlockChange(teamID, block)
  ├─ webhook.NotifyUpdate(block)
  └─ notifications.BlockChanged(event)
       ├─ notifylogger
       ├─ notifysubscriptions (schedule per subscriber)
       └─ notifymentions (@mention detection)
```

---

## Boards (`app/boards.go`)

| Method | Description |
|--------|-------------|
| `GetBoard(boardID)` | Get board by ID |
| `GetBoardCount()` | Total board count |
| `GetBoardMetadata(boardID)` | Get metadata (creation/last updated) |
| `CreateBoard(board, userID, addMember)` | Create board; optionally add creator as admin |
| `PatchBoard(boardID, patch, userID)` | Apply a board patch |
| `DeleteBoard(boardID, userID)` | Soft-delete board |
| `UndeleteBoard(boardID, userID)` | Restore deleted board |
| `GetBoardsForUserAndTeam(userID, teamID, includePublic)` | List boards accessible to user |
| `GetBoardsInTeamByIds(boardIDs, teamID)` | Batch fetch boards |
| `GetTemplateBoards(teamID, userID)` | List template boards |
| `GetMembersForBoard(boardID)` | List board members |
| `GetMemberForBoard(boardID, userID)` | Get single member |
| `AddMemberToBoard(member)` | Add/update board membership |
| `DeleteBoardMember(boardID, userID)` | Remove member |
| `DuplicateBoard(boardID, userID, toTeam, asTemplate)` | Deep copy board + all blocks |
| `GetBoardHistory(boardID, opts)` | Audit trail for board changes |
| `SearchBoardsForUserInTeam(teamID, query, userID)` | Full-text search |
| `GetBoardsForCompliance(opts)` | Compliance export |

### Board Types
- `"O"` — **Open** board: any team member can view
- `"P"` — **Private** board: only added members can view

### Board Duplication
`DuplicateBoard` performs a deep copy:
1. Copies the board record itself
2. Copies all blocks (views, cards, content blocks)
3. Copies file attachments referenced in blocks
4. Returns the new board + all new blocks

---

## Cards (`app/cards.go`)

Cards use the higher-level V3 Card model built on top of Blocks.

| Method | Description |
|--------|-------------|
| `CreateCard(card, boardID, userID)` | Convert `Card` → `Block`, insert |
| `GetCardsForBoard(boardID, page, perPage)` | Paginated card list |
| `GetCardByID(cardID)` | Fetch single card |
| `PatchCard(cardPatch, cardID, userID)` | Apply card patch |

### Card ↔ Block Conversion

`Card2Block(card) *Block` — serializes Card fields into Block.Fields map.
`Block2Card(block) *Card` — deserializes from Block.Fields map.

---

## Authentication (`app/auth.go`)

| Method | Description |
|--------|-------------|
| `GetSession(token)` | Validate session token; auto-refresh if near expiry |
| `IsValidReadToken(boardID, token)` | Check public share token |
| `GetRegisteredUserCount()` | Count all registered (non-deleted) users |
| `Login(username, email, password, mfaToken)` | Authenticate; return session |
| `GetDailyActiveUsers()` | Users active in last 24 hours |
| `GetWeeklyActiveUsers()` | Users active in last 7 days |
| `GetMonthlyActiveUsers()` | Users active in last 30 days |

---

## Users (`app/user.go`)

| Method | Description |
|--------|-------------|
| `GetUser(userID)` | Get user by ID |
| `GetUsersList(userIDs)` | Batch fetch users |
| `GetTeamUsers(teamID, asGuestID)` | List users in a team |
| `SearchTeamUsers(teamID, query, asGuestID)` | Search users by name/email |
| `UpdateUserConfig(userID, patch)` | Save user preferences |
| `UserIsGuest(userID)` | Check if user is guest |
| `CanSeeUser(seerID, seenID)` | Permission: can user A see user B? |
| `SearchUserChannels(teamID, userID, query)` | Search Mattermost channels user belongs to |
| `GetChannel(teamID, channelID)` | Get a Mattermost channel |

---

## Teams (`app/teams.go`)

| Method | Description |
|--------|-------------|
| `GetRootTeam()` | Get/create the global root team (ID `"0"`) |
| `GetTeam(teamID)` | Get team by ID |
| `GetTeamsForUser(userID)` | List teams for a user |
| `DoesUserHaveTeamAccess(userID, teamID)` | Check team access |
| `GetTeamCount()` | Total team count |
| `UpsertTeamSettings(team)` | Create/update team settings |
| `UpsertTeamSignupToken(team)` | Regenerate signup token |

---

## Permissions (`app/permissions.go`)

| Method | Description |
|--------|-------------|
| `HasPermissionToBoard(userID, boardID, permission)` | Delegate to `permissions.PermissionsService` |

All other permission checks are performed directly via the permissions service injected into the app.

---

## Sharing (`app/sharing.go`)

| Method | Description |
|--------|-------------|
| `GetSharing(boardID)` | Get sharing configuration for a board |
| `UpsertSharing(sharing)` | Create/update public link token |

---

## Files (`app/files.go`)

| Method | Description |
|--------|-------------|
| `SaveFile(reader, teamID, boardID, filename, limitFileSize)` | Store file, create FileInfo record |
| `GetFileInfo(filename)` | Retrieve FileInfo metadata |
| `GetFile(teamID, boardID, filename)` | Stream file content |
| `GetFilePath(teamID, boardID, filename)` | Return storage path |
| `CopyAndUpdateCardFiles(srcBoardID, dstBoardID, blocks, userID, toTeamID)` | Copy file references during board duplication |

File paths follow the pattern: `teams/{teamID}/boards/{boardID}/{filename}`

---

## Subscriptions (`app/subscriptions.go`)

| Method | Description |
|--------|-------------|
| `CreateSubscription(sub)` | Subscribe user/channel to a block |
| `DeleteSubscription(blockID, subscriberID)` | Unsubscribe |
| `GetSubscriptions(subscriberID)` | List subscriptions for a user |

---

## Categories (`app/category.go`, `app/category_boards.go`)

| Method | Description |
|--------|-------------|
| `GetUserCategories(userID, teamID)` | List all categories for user in team |
| `GetCategory(categoryID)` | Get category by ID |
| `CreateCategory(category)` | Create new user category |
| `UpdateCategory(category)` | Rename/modify category (cannot modify system categories) |
| `DeleteCategory(categoryID, userID, teamID)` | Soft-delete category |
| `ReorderCategories(userID, teamID, newOrder)` | Change sort order |
| `AddUpdateUserCategoryBoard(teamID, userID, categoryID, boardIDs)` | Move boards into category |
| `ReorderCategoryBoards(userID, teamID, categoryID, newBoardsOrder)` | Reorder boards within category |
| `GetUserCategoryBoards(userID, teamID)` | Get categories with their boards |

---

## Templates (`app/templates.go`)

| Method | Description |
|--------|-------------|
| `GetTemplateBoards(teamID, userID)` | List both global and team templates |
| `RegisterDefaultTemplates()` | Load built-in templates from `templates.boardarchive` |
| `initializeTemplates()` | Called on startup; imports default templates if not already present |

Built-in templates are stored in `app/templates.boardarchive` (binary board archive format).

---

## Onboarding (`app/onboarding.go`)

| Method | Description |
|--------|-------------|
| `PrepareOnboardingTour(userID, teamID)` | Create onboarding board with welcome content |
| `getOnboardingBoardID(teamID)` | Get existing onboarding board ID |

---

## Content Blocks (`app/content_blocks.go`)

| Method | Description |
|--------|-------------|
| `GetContentBlock(boardID, contentID)` | Get a content block by ID |
| `CreateContentBlock(block, boardID, userID)` | Create a content block |
| `DeleteContentBlock(blockID, boardID, userID)` | Delete a content block |
| `MoveContentBlock(blockID, dstBoardID, where, userID)` | Move content block within card |

---

## Import / Export (`app/import.go`, `app/export.go`)

| Method | Description |
|--------|-------------|
| `ImportArchive(r, teamID, userID, imports)` | Import board archive (`.boardarchive` format) |
| `ExportArchive(w, boardIDs)` | Export boards to archive file |

---

## Statistics (`app/statistics.go`)

| Method | Description |
|--------|-------------|
| `GetBlockCountsByType()` | Count blocks grouped by type |
| `GetBoardCount()` | Total board count |
| `GetTeamCount()` | Total team count |

---

## Server Metadata (`app/server_metadata.go`)

| Method | Description |
|--------|-------------|
| `GetServerMetadata()` | Return version, build number, edition, operating system |

---

## Cloud / Card Limits (`app/clientConfig.go`)

| Method | Description |
|--------|-------------|
| `GetClientConfig()` | Return `model.ClientConfig` to the client |
| `SetCardLimit(n)` | Update the cloud card limit (thread-safe) |
| `CardLimit()` | Get the current card limit |

---

## Compliance (`app/compliance.go`)

| Method | Description |
|--------|-------------|
| `GetBoardsForCompliance(opts)` | Paginated board list for compliance export |
| `GetBoardsComplianceHistory(opts)` | Board change history |
| `GetBlocksComplianceHistory(opts)` | Block change history |

---

## blockChangeNotifier (Async Notification Queue)

**File:** `server/utils/callback_queue.go`

- Queue size: 1000
- Worker pool size: 10
- Shutdown timeout: 10 seconds

When a block is mutated, a callback is enqueued. Workers process callbacks concurrently, calling:
1. WebSocket broadcast
2. Webhook notification
3. Notification service (subscriptions, mentions, logging)

This ensures mutation API calls return quickly without waiting for notification delivery.
