# Public Sharing

## Overview

Focalboard boards can be shared publicly via a read-only link. Anyone with the link can view the board without creating an account or logging in.

---

## How Sharing Works

1. A board admin enables sharing and generates a token
2. The token is appended to a URL: `<host>/shared/<boardID>?r=<token>`
3. Anyone who visits that URL can view the board in read-only mode
4. No authentication is required

---

## Sharing Data Model

```go
type Sharing struct {
    ID         string `json:"id"`         // Equal to the Board ID
    Enabled    bool   `json:"enabled"`
    Token      string `json:"token"`      // Random read-only access token
    ModifiedBy string `json:"modifiedBy"`
    UpdateAt   int64  `json:"updateAt"`
}
```

---

## Enabling/Disabling Sharing (API)

### Get sharing info
```
GET /api/v2/boards/{boardID}/sharing
Authorization: required (PermissionViewBoard)
```

Response:
```json
{
  "id": "board_123",
  "enabled": true,
  "token": "abc123xyz",
  "modifiedBy": "user_456",
  "updateAt": 1700000000000
}
```

### Update sharing
```
POST /api/v2/boards/{boardID}/sharing
Authorization: required (PermissionShareBoard = admin only)

Body:
{
  "id": "board_123",
  "enabled": true,
  "token": "abc123xyz"
}
```

The token is generated client-side and stored as-is. To regenerate the link, generate a new random token and POST it.

---

## Read Token Validation

When a read request includes `?read_token=TOKEN`:

```
API middleware: requireValidReadToken()
  │
auth.IsValidReadToken(boardID, readToken)
  ├─ Check: config.EnablePublicSharedBoards == true
  │   └─ If false: return false (sharing globally disabled)
  │
  ├─ store.GetSharing(boardID)
  │   └─ Returns Sharing record for the board
  │
  └─ Compare: sharing.Enabled == true AND sharing.Token == readToken
      ├─ True: user gets read access (no session required)
      └─ False: 401 Unauthorized
```

---

## Global Sharing Config

Sharing can be globally disabled:
```json
{
  "enablePublicSharedBoards": false
}
```

When disabled:
- The sharing UI is hidden in the frontend
- All read token validation returns false
- Existing links stop working

---

## Frontend Sharing Flow

```
Board admin opens "Share" dialog → <ShareBoard />
  │
octoClient.getSharing(boardId)   GET sharing info
  │
User toggles "Enable public link"
  │
mutator.setSharing(boardId, { enabled: true, token: newToken })
  │
octoClient.setSharing(boardId, sharing)   POST /api/v2/boards/{boardId}/sharing
  │
UI shows the shareable URL with token
```

The shareable URL format:
```
https://<host>/shared/<boardId>/<viewId>?r=<token>
```

---

## Read-Only View

When accessed via a sharing link:
- The router renders `<BoardPage readonly={true} />`
- All mutation actions (edit, delete, move) are disabled
- No authentication banner or login prompt
- The board header shows a "Read only" badge

In the router (`router.tsx`):
```tsx
<Route path="/team/:teamId/shared/:boardId?/:viewId?/:cardId?">
    <BoardPage new={false} />
</Route>
```

The `readonly` prop is determined by whether the request was authenticated or came via a valid read token.

---

## Security Notes

- Tokens are random strings (generated via `utils.NewID()`)
- Tokens are stored in plaintext in the database
- HTTPS is strongly recommended to prevent token interception
- Sharing can be disabled globally via config for maximum security
- Only board admins can create/modify sharing tokens
