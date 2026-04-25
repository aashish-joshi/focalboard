# REST API Reference

## Overview

**File:** `server/api/api.go`

The `API` struct is the HTTP handler layer. It registers routes on a `gorilla/mux` router and delegates all business logic to `app.App`.

```go
type API struct {
    app             *app.App
    authService     string                        // "native" or "mattermost"
    permissions     PermissionsService
    singleUserToken string
    MattermostAuth  bool
    logger          mlog.LoggerIFace
    audit           *audit.Audit
}
```

---

## Route Structure

All routes live under the prefix `/api/v2`. Routes are registered in `RegisterRoutes()`.

### Middleware Applied to All Routes
| Middleware | Purpose |
|-----------|---------|
| `panicHandler` | Catches any panics and returns a 500 error |
| `requireCSRFToken` | Validates CSRF token header for mutating requests |

### Admin Routes (Unix socket only)
Registered via `RegisterAdminRoutes()` on the local router (not exposed over the public network).

---

## Endpoint Groups

### Authentication (`auth.go`)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v2/login` | Login with username/email + password |
| `POST` | `/api/v2/logout` | Invalidate current session |
| `POST` | `/api/v2/register` | Register new user account |
| `POST` | `/api/v2/users/{userID}/changepassword` | Change password |

**Login request body:**
```json
{
  "type": "normal",
  "username": "admin",
  "email": "admin@example.com",
  "password": "password123",
  "mfa_token": ""
}
```
**Login response:**
```json
{
  "token": "<session-token>"
}
```

---

### Users (`users.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/users/me` | Get current logged-in user |
| `GET` | `/api/v2/users/{userID}` | Get user by ID |
| `POST` | `/api/v2/users` | Get list of users by IDs |
| `GET` | `/api/v2/users/me/config` | Get current user preferences |
| `PUT` | `/api/v2/users/{userID}/config` | Update user preferences |
| `GET` | `/api/v2/teams/{teamID}/users` | List team members |
| `GET` | `/api/v2/teams/{teamID}/users/search` | Search team members |

---

### Teams (`teams.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/teams` | List teams for current user |
| `GET` | `/api/v2/teams/{teamID}` | Get team by ID |
| `POST` | `/api/v2/teams/{teamID}/regenerate_signup_token` | Regenerate team signup token |

---

### Boards (`boards.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/teams/{teamID}/boards` | List all boards in a team |
| `POST` | `/api/v2/boards` | Create a new board |
| `GET` | `/api/v2/boards/{boardID}` | Get board by ID |
| `PATCH` | `/api/v2/boards/{boardID}` | Update board properties |
| `DELETE` | `/api/v2/boards/{boardID}` | Delete (soft) a board |
| `POST` | `/api/v2/boards/{boardID}/duplicate` | Duplicate a board |
| `POST` | `/api/v2/boards/{boardID}/undelete` | Restore a deleted board |
| `GET` | `/api/v2/boards/{boardID}/metadata` | Get board metadata |

**Board create/patch body (partial):**
```json
{
  "title": "My Board",
  "type": "O",
  "icon": "🏠",
  "description": "Project board",
  "isTemplate": false,
  "cardProperties": [...]
}
```

---

### Board Members (`members.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/boards/{boardID}/members` | List board members |
| `POST` | `/api/v2/boards/{boardID}/members` | Add a member to a board |
| `PUT` | `/api/v2/boards/{boardID}/members/{userID}` | Update member role |
| `DELETE` | `/api/v2/boards/{boardID}/members/{userID}` | Remove member from board |

---

### Blocks (`blocks.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/boards/{boardID}/blocks` | Get all blocks in a board |
| `POST` | `/api/v2/boards/{boardID}/blocks` | Create/update blocks (upsert) |
| `PATCH` | `/api/v2/boards/{boardID}/blocks` | Batch patch blocks |
| `GET` | `/api/v2/boards/{boardID}/blocks/{blockID}` | Get single block |
| `PATCH` | `/api/v2/boards/{boardID}/blocks/{blockID}` | Patch single block |
| `DELETE` | `/api/v2/boards/{boardID}/blocks/{blockID}` | Delete a block |
| `POST` | `/api/v2/boards/{boardID}/blocks/{blockID}/duplicate` | Duplicate a block |
| `POST` | `/api/v2/boards/{boardID}/blocks/{blockID}/undelete` | Restore a deleted block |

**Block query parameters:**
- `parent_id` — filter by parent block
- `type` — filter by block type
- `all` — include all block types
- `block_id` — filter by specific block ID

---

### Cards (V3 API, `cards.go`)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v2/boards/{boardID}/cards` | Create a new card |
| `GET` | `/api/v2/boards/{boardID}/cards` | List cards in a board (paginated) |
| `GET` | `/api/v2/cards/{cardID}` | Get a single card |
| `PATCH` | `/api/v2/cards/{cardID}` | Patch (update) a card |

Cards use a higher-level `Card` model than raw blocks, hiding internal field structure.

---

### Boards and Blocks (`boards_and_blocks.go`)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v2/boards-and-blocks` | Atomically create boards + blocks |
| `PATCH` | `/api/v2/boards-and-blocks` | Atomically patch boards + blocks |
| `DELETE` | `/api/v2/boards-and-blocks` | Atomically delete boards + blocks |

Used for template instantiation and other multi-entity operations.

---

### Sharing (`sharing.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/boards/{boardID}/sharing` | Get sharing info for a board |
| `POST` | `/api/v2/boards/{boardID}/sharing` | Create/update public sharing |

---

### Templates (`templates.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/teams/{teamID}/templates` | List team + global templates |

---

### Files (`files.go`)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v2/teams/{teamID}/boards/{boardID}/files` | Upload a file |
| `GET` | `/api/v2/files/teams/{teamID}/{boardID}/{fileName}` | Download a file |

---

### Categories (`categories.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/teams/{teamID}/categories` | List user's sidebar categories |
| `POST` | `/api/v2/teams/{teamID}/categories` | Create a category |
| `PUT` | `/api/v2/teams/{teamID}/categories` | Reorder all categories |
| `PUT` | `/api/v2/teams/{teamID}/categories/{categoryID}` | Update a category |
| `DELETE` | `/api/v2/teams/{teamID}/categories/{categoryID}` | Delete a category |
| `POST` | `/api/v2/teams/{teamID}/categories/{categoryID}/boards` | Add boards to a category |
| `PUT` | `/api/v2/teams/{teamID}/categories/{categoryID}/boards` | Reorder boards in a category |
| `DELETE` | `/api/v2/teams/{teamID}/categories/{categoryID}/boards/{boardID}` | Remove board from category |

---

### Subscriptions (`subscriptions.go`)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v2/subscriptions` | Subscribe to a block |
| `DELETE` | `/api/v2/subscriptions/{blockID}/{subscriberID}` | Unsubscribe |
| `GET` | `/api/v2/subscriptions/{subscriberID}` | List subscriptions for a user |

---

### Search (`search.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/teams/{teamID}/boards/search` | Search boards by title |
| `GET` | `/api/v2/boards/{boardID}/blocks/search` | Search blocks in a board |
| `GET` | `/api/v2/search` | Global search across boards and blocks |

---

### Channels (`channels.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/teams/{teamID}/channels` | Search Mattermost channels (plugin mode only) |

---

### System (`system.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/clientConfig` | Get client configuration |
| `GET` | `/api/v2/ping` | Health check — returns 200 |

---

### Statistics (`statistics.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/statistics` | Get server statistics (blocks, boards, users) |

---

### Onboarding (`onboarding.go`)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v2/teams/{teamID}/onboard` | Create onboarding board for a team |

---

### Archive (`archive.go`)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v2/boards/{boardID}/archive/export` | Export board archive |
| `POST` | `/api/v2/boards/archive/import` | Import board archive |
| `POST` | `/api/v2/archive/export` | Export workspace archive |

---

### Compliance (`compliance.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/compliance/boards` | Get boards for compliance export |
| `GET` | `/api/v2/compliance/boards/{boardID}/blocks` | Get blocks compliance history |

---

### Audit (`audit.go`)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/auditlogs` | Get audit log entries |

---

### Admin (`admin.go`) — Unix socket only

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v2/admin/users/{username}/password` | Reset a user's password |

---

## Error Responses

All errors return JSON:
```json
{
  "errorCode": 1,
  "message": "error description"
}
```

Common HTTP status codes:
| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad request / invalid input |
| 401 | Not authenticated |
| 403 | Permission denied |
| 404 | Resource not found |
| 500 | Internal server error |

---

## Authentication Header

All authenticated requests must include:
```
X-Auth-Token: <session-token>
```

In Mattermost plugin mode, the Mattermost session cookie is used instead.

---

## CSRF Protection

Non-`GET` requests (POST/PATCH/PUT/DELETE) require one of:
- `X-Requested-With: XMLHttpRequest` header, OR
- `MMCSRF` cookie matching `X-CSRF-Token` header (Mattermost plugin mode)

---

## Context Values

Middleware sets context values accessible in handlers:

| Key | Type | Description |
|-----|------|-------------|
| `sessionKey` | `*model.Session` | Current user session |
| `userIDKey` | `string` | Current user ID |
| `connKey` | `net.Conn` | Raw connection (for local mode) |
