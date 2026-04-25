# Database Store Interface

## Overview

**File:** `server/services/store/store.go`

The `Store` interface is the complete database abstraction layer with 175+ methods. The only implementation is `sqlstore` (SQLite/PostgreSQL/MySQL via `squirrel` query builder and `database/sql`).

---

## Implementation: sqlstore

**Directory:** `server/services/store/sqlstore/`

| File | Responsibility |
|------|---------------|
| `store.go` | `Params`, `SQLStore` struct, `New()`, schema migration |
| `blocks.go` | Block CRUD |
| `boards.go` | Board CRUD + membership |
| `users.go` | User CRUD |
| `sessions.go` | Session management |
| `categories.go` | Category CRUD |
| `subscriptions.go` | Subscription management |
| `notifications.go` | Notification hints |
| `files.go` | FileInfo metadata |
| `teams.go` | Team management |
| `sharing.go` | Public sharing tokens |
| `compliance.go` | Compliance queries |
| `system.go` | System key-value settings |

### Initialization (`New(params)`)

1. Opens SQL connection with `database/sql`
2. Pings the database (retries up to `DBPingAttempts` times with exponential backoff)
3. Sets connection pool settings
4. Runs schema migrations via embedded migration files
5. Returns initialized `SQLStore`

---

## Block Operations

```go
// Query blocks with flexible filtering
GetBlocks(opts QueryBlocksOptions) ([]*Block, error)

// Convenience queries
GetBlocksWithParentAndType(boardID, parentID, blockType string) ([]*Block, error)
GetBlocksWithParent(boardID, parentID string) ([]*Block, error)
GetBlocksWithType(boardID, blockType string) ([]*Block, error)
GetBlocksByIDs(blockIDs []string) ([]*Block, error)

// Subtree traversal (block + immediate children)
GetSubTree2(boardID, blockID string, opts QuerySubtreeOptions) ([]*Block, error)

// CRUD
GetBlockByID(blockID string) (*Block, error)
InsertBlock(block *Block, userID string) error
InsertBlocks(blocks []*Block, userID string) error
DeleteBlock(blockID string, modifiedBy string) error
UndeleteBlock(blockID string, modifiedBy string) error

// Patch (partial update)
PatchBlock(blockID string, patch *BlockPatch, userID string) error
PatchBlocks(patch *BlockPatchBatch, userID string) error

// History
GetBlockHistory(blockID string, opts QueryBlockHistoryOptions) ([]*Block, error)
GetBlockHistoryDescendants(boardID string, opts QueryBlockHistoryOptions) ([]*Block, error)
GetBlockCountsByType() (map[string]int64, error)

// Deep copy
DuplicateBlock(boardID, blockID, userID string, asTemplate bool) ([]*Block, *Board, error)
```

### QueryBlocksOptions

```go
type QueryBlocksOptions struct {
    BoardID        string
    ParentID       string
    BlockType      BlockType
    Page           int
    PerPage        int
}
```

---

## Board Operations

```go
// CRUD
InsertBoard(board *Board, userID string) (*Board, error)
InsertBoardWithAdmin(board *Board, userID string) (*BoardMember, error)
PatchBoard(boardID string, patch *BoardPatch, userID string) (*Board, error)
DeleteBoard(boardID string, userID string) error
GetBoard(boardID string) (*Board, error)

// Queries
GetBoardsForUserAndTeam(userID, teamID string, includePublic bool) ([]*Board, error)
GetBoardsInTeamByIds(boardIDs []string, teamID string) ([]*Board, error)
GetBoardAndCard(block *Block) (*Board, *Block, error)
GetBoardHistory(boardID string, opts QueryBoardHistoryOptions) ([]*Board, error)
GetBoardCount() (int64, error)
GetTemplateBoards(teamID, userID string) ([]*Board, error)

// Search
SearchBoardsForUserInTeam(teamID, query, userID string) ([]*Board, error)
SearchBoardsForUser(query, userID string, includePublic bool) ([]*Board, error)

// Deep copy
DuplicateBoard(boardID, userID, toTeamID string, asTemplate bool) (*BoardBlocks, []*BoardMember, error)
```

---

## Board Membership

```go
SaveMember(member *BoardMember) (*BoardMember, error)
DeleteMember(boardID, userID string) error
GetMemberForBoard(boardID, userID string) (*BoardMember, error)
GetMembersForBoard(boardID string) ([]*BoardMember, error)
GetMembersForUser(userID string) ([]*BoardMember, error)
GetBoardMemberHistory(boardID, userID string, limit uint64) ([]*BoardMemberHistoryEntry, error)
```

---

## User Operations

```go
GetUserByID(userID string) (*User, error)
GetUserByEmail(email string) (*User, error)
GetUserByUsername(username string) (*User, error)
GetUsersList(userIDs []string, showEmail, showFullName bool) ([]*User, error)
GetUsersByTeam(teamID string, asGuestID string, showEmail, showFullName bool) ([]*User, error)
SearchUsersByTeam(teamID, query string, asGuestID string, showEmail, showFullName, excludeBots bool) ([]*User, error)

CreateUser(user *User) error
UpdateUser(user *User) error
UpdateUserPassword(username, password string) error
UpdateUserPasswordByID(userID, password string) error

GetRegisteredUserCount() (int, error)
GetActiveUserCount(updatedAfter int64) (int, error)
GetUserTimezone(userID string) (string, error)

GetUserConfig(userID string) ([]*UserPreference, error)
PatchUserConfig(userID string, patch StringMap) ([]*UserPreference, error)
```

---

## Session Operations

```go
GetSession(token string, expireTime int64) (*Session, error)
CreateSession(session *Session) error
RefreshSession(sessionID string) error
UpdateSession(session *Session) error
DeleteSession(sessionID string) error
CleanUpSessions(expireTime int64) error
```

---

## Team Operations

```go
GetTeam(teamID string) (*Team, error)
GetTeamsForUser(userID string) ([]*Team, error)
GetAllTeams() ([]*Team, error)
GetTeamCount() (int64, error)
UpsertTeamSignupToken(team Team) error
UpsertTeamSettings(team Team) error
```

---

## Category Operations

```go
// Category CRUD
GetCategory(categoryID string) (*Category, error)
GetUserCategories(userID, teamID string) ([]Category, error)
CreateCategory(category Category) error
UpdateCategory(category Category) error
DeleteCategory(categoryID, userID, teamID string) error
ReorderCategories(userID, teamID string, newCategoryOrder []string) ([]string, error)

// Category-Board associations
GetUserCategoryBoards(userID, teamID string) ([]CategoryBoards, error)
AddUpdateCategoryBoard(teamID, userID, categoryID string, boardIDs []string) error
ReorderCategoryBoards(userID, teamID, categoryID string, newBoardsOrder []string) ([]string, error)
DeleteBoardsFromCategoryForUser(userID, teamID string, boardIDs []string) error
```

---

## Sharing Operations

```go
GetSharing(boardID string) (*Sharing, error)
UpsertSharing(sharing Sharing) error
```

---

## Subscription Operations

```go
CreateSubscription(sub *Subscription) (*Subscription, error)
DeleteSubscription(blockID, subscriberID string) error
GetSubscription(blockID, subscriberID string) (*Subscription, error)
GetSubscriptions(subscriberID string) ([]*Subscription, error)
GetSubscribersForBlock(blockID string) ([]*Subscriber, error)
GetSubscribersCountForBlock(blockID string) (int, error)
UpdateSubscribersNotifiedAt(blockID string, notifiedAt int64) error
```

---

## Notification Hint Operations

```go
UpsertNotificationHint(hint *NotificationHint, freq time.Duration) (*NotificationHint, error)
DeleteNotificationHint(blockID string) error
GetNotificationHint(blockID string) (*NotificationHint, error)
GetNextNotificationHint(remove bool) (*NotificationHint, error) // dequeue oldest pending
```

---

## File Operations

```go
GetFileInfo(id string) (*FileInfo, error)
SaveFileInfo(fileInfo *FileInfo) error
```

---

## System Settings

```go
GetSystemSetting(key string) (string, error)
GetSystemSettings() (map[string]string, error)
SetSystemSetting(key, value string) error
```

---

## License

```go
GetLicense() *mm_model.License
```

---

## Compliance / Audit Trail

```go
GetBoardsForCompliance(opts QueryBoardsForComplianceOptions) (BoardsComplianceResponse, error)
GetBoardsComplianceHistory(opts QueryBoardHistoryOptions) ([]*Board, error)
GetBlocksComplianceHistory(opts QueryBlocksHistoryOptions) ([]*Block, error)
RunDataRetention(globalRetentionDate, boardsRetentionDate int64) (BoardsDataRetentionResults, error)
```

---

## Mattermost Integration Methods

These methods are available only in plugin mode and delegate to the Mattermost plugin API:

```go
SearchUserChannels(teamID, userID, query string) ([]*mm_model.Channel, error)
GetChannel(teamID, channelID string) (*mm_model.Channel, error)
PostMessage(message, postType, channelID string) error
SendMessage(message, postType string, receipients []string) error
GetUserTimezone(userID string) (string, error)
```

---

## Utility / Introspection

```go
DBType() string          // "sqlite3", "postgres", "mysql"
DBVersion() string       // Database server version string
Shutdown() error         // Close all connections gracefully
```

---

## Schema Migrations

Migrations are stored as embedded SQL files in `server/services/store/sqlstore/migrations/`. Migrations are run automatically on `New()` using a migration runner that tracks the current version in the `schema_migrations` table.

Migration files follow a numbered naming convention:
```
000001_init.up.sql
000001_init.down.sql
000002_add_boards.up.sql
000002_add_boards.down.sql
...
```

Database-specific SQL variations are handled by conditional logic in the migration runner for `sqlite3`, `postgres`, and `mysql`.
