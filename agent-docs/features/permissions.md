# Permissions System

## Overview

Focalboard uses a layered permission model:
1. **System permissions** — admin-level operations (manage users, system settings)
2. **Team permissions** — access to a specific Mattermost team
3. **Board permissions** — role-based access to individual boards

---

## Permission Service Interface

**File:** `server/services/permissions/permissions.go`

```go
type PermissionsService interface {
    HasPermissionTo(userID string, permission *mm_model.Permission) bool
    HasPermissionToTeam(userID, teamID string, permission *mm_model.Permission) bool
    HasPermissionToChannel(userID, channelID string, permission *mm_model.Permission) bool
    HasPermissionToBoard(userID, boardID string, permission *mm_model.Permission) bool
}
```

---

## Permission Constants

### System Permissions (from Mattermost)

| Constant | Description |
|----------|-------------|
| `PermissionManageSystem` | Can perform system-level operations |
| `PermissionViewTeam` | Can view the team |
| `PermissionManageTeam` | Can manage team settings |
| `PermissionReadChannel` | Can read a channel |
| `PermissionCreatePost` | Can post in a channel |
| `PermissionViewMembers` | Can see team members |

### Focalboard Board Permissions

| Constant | Description | Minimum Role |
|----------|-------------|-------------|
| `PermissionViewBoard` | View a board and its cards | `viewer` |
| `PermissionCommentBoardCards` | Add comments to cards | `commenter` |
| `PermissionManageBoardCards` | Create, edit, delete cards | `editor` |
| `PermissionManageBoardProperties` | Add/edit/delete card properties | `editor` |
| `PermissionManageBoardType` | Change board type (open/private) | `admin` |
| `PermissionManageBoardRoles` | Add/remove members, change roles | `admin` |
| `PermissionShareBoard` | Create/manage public sharing links | `admin` |
| `PermissionDeleteBoard` | Delete the board | `admin` |
| `PermissionDeleteOthersComments` | Delete other users' comments | `admin` |

---

## Role Hierarchy

Roles are cumulative — each role includes all permissions of lower roles:

```
admin
  └─ includes editor +
     └─ includes commenter +
        └─ includes viewer
```

| Permission | viewer | commenter | editor | admin |
|-----------|:------:|:---------:|:------:|:-----:|
| View board | ✅ | ✅ | ✅ | ✅ |
| Comment on cards | ❌ | ✅ | ✅ | ✅ |
| Create/edit/delete cards | ❌ | ❌ | ✅ | ✅ |
| Manage card properties | ❌ | ❌ | ✅ | ✅ |
| Change board type | ❌ | ❌ | ❌ | ✅ |
| Manage board members | ❌ | ❌ | ❌ | ✅ |
| Share board publicly | ❌ | ❌ | ❌ | ✅ |
| Delete board | ❌ | ❌ | ❌ | ✅ |

---

## Board Access Rules

A user can access a board if **any** of the following are true:

1. **Explicit member**: The user is in `board_members` table with a non-empty role
2. **Open board + team member**: The board `Type = "O"` and the user is a team member
3. **System admin**: The user has `PermissionManageSystem`
4. **Mattermost channel member (plugin mode)**: The user is a member of the board's linked channel

---

## Permission Implementations

### Local Permissions (`localpermissions/`)

Used in standalone server mode.

```
HasPermissionToBoard(userID, boardID, permission)
  │
  ├─ store.GetMemberForBoard(boardID, userID)
  │   ├─ If member found: check role against required permission
  │   └─ If not found + board is Open: grant viewer-level permissions
  │
  └─ HasPermissionTo(userID, PermissionManageSystem)
      └─ Check if user.Permissions contains "manage_system"
```

### Mattermost Permissions (`mmpermissions/`)

Used in plugin mode. Delegates system/team/channel permissions to Mattermost's permission API. Board-level permissions still use Focalboard's membership table, but also check:
- Channel membership (via `PluginAPI.GetChannelMember()`)
- Team membership (via `PluginAPI.GetTeamMember()`)
- System admin status

---

## Permission Checks in API Handlers

Every API handler that accesses a board calls `app.HasPermissionToBoard()`:

```go
// Example in api/boards.go
if !a.permissions.HasPermissionToBoard(userID, boardID, model.PermissionManageBoardCards) {
    a.errorResponse(w, r.URL.Path, http.StatusForbidden, "", PermissionError{"access denied"})
    return
}
```

---

## Board Member Management

### Adding a Member

```
POST /api/v2/boards/{boardID}/members
  │
app.AddMemberToBoard(member)
  │
store.SaveMember(member)
  │
wsAdapter.BroadcastMemberChange(teamID, boardID, member)
```

Requires `PermissionManageBoardRoles`.

### Roles

`BoardMember` has scheme-based role flags:
- `SchemeAdmin` — admin role
- `SchemeEditor` — editor role
- `SchemeCommenter` — commenter role
- `SchemeViewer` — viewer role

Only the highest applicable role is active.

### Synthetic Members

In Mattermost plugin mode, users who are members of a board's linked channel but not explicitly added to the board may appear as "synthetic" members with the board's `minimumRole`.

---

## Public Sharing

When a board has sharing enabled (`Sharing.Enabled = true`), anyone with the sharing token can view the board **without authentication**:

```
GET /api/v2/boards/{boardId}/blocks?read_token=TOKEN
  │
api.requireValidReadToken() middleware
  │
auth.IsValidReadToken(boardID, readToken)
  ├─ store.GetSharing(boardID)
  ├─ Compare token
  └─ Grant read-only access if valid
```

Only board admins can enable sharing (`PermissionShareBoard`).

Public sharing can be globally disabled via `config.EnablePublicSharedBoards = false`.

---

## Permission Checks in Frontend

The frontend does not enforce permissions independently — it relies on the server to reject unauthorized requests. However, it conditionally hides UI elements based on the user's role:

```typescript
// From Redux state
const myMembership = useAppSelector(state => getMyBoardMembership(state, boardId))
const isAdmin = myMembership?.schemeAdmin

// Conditionally show admin-only UI
{isAdmin && <ManageMembersButton />}
```

The `<PermissionsComponent>` wrapper component also handles conditional rendering based on permissions.
