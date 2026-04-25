# Data Models

**Directory:** `server/model/`

This file documents all Go data model types used throughout the Focalboard server.

---

## Block (`model/block.go`)

The fundamental data unit in Focalboard. Boards, views, cards, and all content are represented as blocks.

```go
type Block struct {
    ID         string                 `json:"id"`
    ParentID   string                 `json:"parentId"`
    CreatedBy  string                 `json:"createdBy"`
    ModifiedBy string                 `json:"modifiedBy"`
    Schema     int64                  `json:"schema"`
    Type       BlockType              `json:"type"`
    Title      string                 `json:"title"`
    Fields     map[string]interface{} `json:"fields"`
    CreateAt   int64                  `json:"createAt"`  // ms since epoch
    UpdateAt   int64                  `json:"updateAt"`
    DeleteAt   int64                  `json:"deleteAt"`  // 0 = not deleted
    BoardID    string                 `json:"boardId"`
    Limited    bool                   `json:"limited,omitempty"` // cloud limit
}
```

**Constraints:**
- `Title` max: 65,535 bytes
- `Fields` max: 800,000 runes

### BlockPatch

```go
type BlockPatch struct {
    ParentID       *string
    Schema         *int64
    Type           *BlockType
    Title          *string
    UpdatedFields  map[string]interface{}
    DeletedFields  []string
    UpdateAt       *int64
    DeleteAt       *int64
}
```

### BlockPatchBatch

```go
type BlockPatchBatch struct {
    BlockIDs    []string     // Parallel array with BlockPatches
    BlockPatches []BlockPatch
}
```

### BlockType (`model/blocktype.go`)

```go
type BlockType string

const (
    TypeBoard      BlockType = "board"
    TypeView       BlockType = "view"
    TypeCard       BlockType = "card"
    TypeText       BlockType = "text"
    TypeImage      BlockType = "image"
    TypeComment    BlockType = "comment"
    TypeCheckbox   BlockType = "checkbox"
    TypeDivider    BlockType = "divider"
    TypeAttachment BlockType = "attachment"
    // ... h1, h2, h3, list-item, quote, video
)
```

---

## Board (`model/board.go`)

The top-level workspace object. Contains metadata and defines the card properties schema.

```go
type Board struct {
    ID              string                   `json:"id"`
    TeamID          string                   `json:"teamId"`
    ChannelID       string                   `json:"channelId,omitempty"` // linked Mattermost channel
    CreatedBy       string                   `json:"createdBy"`
    ModifiedBy      string                   `json:"modifiedBy"`
    Type            BoardType                `json:"type"`         // "O" or "P"
    MinimumRole     BoardRole                `json:"minimumRole"`  // viewer, commenter, editor, admin
    Title           string                   `json:"title"`
    Description     string                   `json:"description"`
    Icon            string                   `json:"icon"`
    ShowDescription bool                     `json:"showDescription"`
    IsTemplate      bool                     `json:"isTemplate"`
    TemplateVersion int                      `json:"templateVersion"`
    Properties      map[string]interface{}   `json:"properties"`
    CardProperties  []map[string]interface{} `json:"cardProperties"` // property schema
    CreateAt        int64                    `json:"createAt"`
    UpdateAt        int64                    `json:"updateAt"`
    DeleteAt        int64                    `json:"deleteAt"`
}
```

### BoardPatch

```go
type BoardPatch struct {
    Type              *BoardType
    MinimumRole       *BoardRole
    Title             *string
    Description       *string
    Icon              *string
    ShowDescription   *bool
    IsTemplate        *bool
    ChannelID         *string
    UpdatedProperties  map[string]interface{}
    DeletedProperties  []string
    UpdatedCardProperties  []map[string]interface{}
    DeletedCardProperties  []string
}
```

### BoardType

```go
type BoardType string

const (
    BoardTypeOpen    BoardType = "O"  // Any team member can join
    BoardTypePrivate BoardType = "P"  // Invite-only
)
```

### BoardRole

```go
type BoardRole string

const (
    BoardRoleViewer   BoardRole = "viewer"
    BoardRoleCommenter BoardRole = "commenter"
    BoardRoleEditor   BoardRole = "editor"
    BoardRoleAdmin    BoardRole = "admin"
    BoardRoleNone     BoardRole = "none"
)
```

### BoardMember

```go
type BoardMember struct {
    BoardID         string `json:"boardId"`
    UserID          string `json:"userId"`
    Roles           string `json:"roles"`
    MinimumRole     string `json:"minimumRole"`
    SchemeAdmin     bool   `json:"schemeAdmin"`
    SchemeEditor    bool   `json:"schemeEditor"`
    SchemeCommenter bool   `json:"schemeCommenter"`
    SchemeViewer    bool   `json:"schemeViewer"`
    Synthetic       bool   `json:"synthetic"` // derived from channel membership
}
```

---

## Card (`model/card.go`)

The Card model is a higher-level V3 API representation of a card block. It hides the internal `Fields` map and exposes typed properties.

```go
type Card struct {
    ID           string         `json:"id"`
    BoardID      string         `json:"boardId"`
    CreatedBy    string         `json:"createdBy"`
    ModifiedBy   string         `json:"modifiedBy"`
    Title        string         `json:"title"`
    ContentOrder []string       `json:"contentOrder"` // IDs of content blocks in order
    Icon         string         `json:"icon"`
    IsTemplate   bool           `json:"isTemplate"`
    Properties   map[string]any `json:"properties"` // property values keyed by property template ID
    CreateAt     int64          `json:"createAt"`
    UpdateAt     int64          `json:"updateAt"`
    DeleteAt     int64          `json:"deleteAt"`
}
```

### CardPatch

```go
type CardPatch struct {
    Title            *string
    ContentOrder     *[]string
    Icon             *string
    UpdatedProperties map[string]any
    DeletedProperties []string
}
```

### Conversion Functions

```go
func Card2Block(card *Card) *Block    // Card → Block (serializes properties into Fields)
func Block2Card(block *Block) *Card  // Block → Card (deserializes Fields into typed Card)
```

---

## User (`model/user.go`)

```go
type User struct {
    ID          string   `json:"id"`
    Username    string   `json:"username"`
    Email       string   `json:"email,omitempty"`      // hidden in JSON responses
    Nickname    string   `json:"nickname"`
    FirstName   string   `json:"firstName"`
    LastName    string   `json:"lastName"`
    Password    string   `json:"password,omitempty"`   // hashed, never returned
    MfaSecret   string   `json:"mfa_secret,omitempty"`
    AuthService string   `json:"auth_service"`         // "mattermost", "ldap", etc.
    AuthData    string   `json:"auth_data,omitempty"`
    CreateAt    int64    `json:"create_at"`
    UpdateAt    int64    `json:"update_at"`
    DeleteAt    int64    `json:"delete_at"`
    IsBot       bool     `json:"is_bot"`
    IsGuest     bool     `json:"is_guest"`
    Permissions []string `json:"permissions,omitempty"` // "manage_system"
    Roles       string   `json:"roles"`
}
```

### UserPreference

```go
type UserPreference struct {
    UserID   string `json:"userId"`
    Category string `json:"category"` // "focalboard"
    Name     string `json:"name"`
    Value    string `json:"value"`
}
```

---

## Team (`model/team.go`)

```go
type Team struct {
    ID          string                 `json:"id"`
    Title       string                 `json:"title"`
    SignupToken string                 `json:"signupToken"`
    Settings    map[string]interface{} `json:"settings"`
    ModifiedBy  string                 `json:"modifiedBy"`
    UpdateAt    int64                  `json:"updateAt"`
}
```

The root/global team always has ID `"0"`.

---

## Session (`model/auth.go`)

```go
type Session struct {
    ID          string `json:"id"`
    Token       string `json:"token"`
    UserID      string `json:"userId"`
    AuthService string `json:"authService"`
    Props       StringMap
    CreateAt    int64  `json:"createAt"`
    UpdateAt    int64  `json:"updateAt"`
}
```

---

## Sharing (`model/sharing.go`)

```go
type Sharing struct {
    ID         string `json:"id"`      // Board ID
    Enabled    bool   `json:"enabled"`
    Token      string `json:"token"`   // Random read-only token
    ModifiedBy string `json:"modifiedBy"`
    UpdateAt   int64  `json:"updateAt"`
}
```

---

## Subscription (`model/subscription.go`)

```go
type Subscription struct {
    BlockType      BlockType      `json:"blockType"`
    BlockID        string         `json:"blockId"`
    SubscriberType SubscriberType `json:"subscriberType"` // "user" or "channel"
    SubscriberID   string         `json:"subscriberId"`
    NotifiedAt     int64          `json:"notifiedAt"` // last notification timestamp
    CreateAt       int64          `json:"createAt"`
    DeleteAt       int64          `json:"deleteAt"`
}
```

---

## NotificationHint (`model/notification.go`)

Represents a scheduled notification for block changes.

```go
type NotificationHint struct {
    BlockType    BlockType `json:"blockType"`
    BlockID      string    `json:"blockId"`
    ModifiedByID string    `json:"modifiedById"` // user who made the change
    CreateAt     int64     `json:"createAt"`
    NotifyAt     int64     `json:"notifyAt"`     // scheduled send time
}
```

---

## Category (`model/category.go`)

```go
type Category struct {
    ID        string       `json:"id"`
    UserID    string       `json:"userId"`
    TeamID    string       `json:"teamId"`
    Name      string       `json:"name"`
    Type      CategoryType `json:"type"`    // "system" or "custom"
    Collapsed bool         `json:"collapsed"`
    SortOrder int          `json:"sortOrder"`
    CreateAt  int64        `json:"createAt"`
    UpdateAt  int64        `json:"updateAt"`
    DeleteAt  int64        `json:"deleteAt"`
}
```

### CategoryType

```go
type CategoryType string

const (
    CategoryTypeSystem CategoryType = "system"
    CategoryTypeCustom CategoryType = "custom"
)
```

---

## CategoryBoards (`model/category_boards.go`)

Represents a category with its associated board list (used for sidebar).

```go
type CategoryBoards struct {
    Category
    BoardMetadata []CategoryBoardMetadata `json:"boardMetadata"`
}

type CategoryBoardMetadata struct {
    BoardID string `json:"boardID"`
    Hidden  bool   `json:"hidden"`
}
```

---

## File (`model/file.go`)

Wraps Mattermost's `FileInfo` type:

```go
type FileInfo struct {
    Id              string `json:"id"`
    Name            string `json:"name"`
    Extension       string `json:"extension"`
    Size            int64  `json:"size"`
    MimeType        string `json:"mime_type"`
    Path            string `json:"path"`
    ThumbnailPath   string `json:"thumbnail_path"`
    PreviewPath     string `json:"preview_path"`
    HasPreviewImage bool   `json:"has_preview_image"`
}
```

---

## Permission (`model/permission.go`)

### System Permissions (from Mattermost)
- `PermissionViewTeam`
- `PermissionManageTeam`
- `PermissionManageSystem`
- `PermissionReadChannel`
- `PermissionCreatePost`
- `PermissionViewMembers`

### Focalboard-specific Permissions
| Constant | Description |
|----------|-------------|
| `PermissionViewBoard` | Can view a board |
| `PermissionManageBoardRoles` | Can add/remove members and change roles |
| `PermissionShareBoard` | Can create public sharing links |
| `PermissionManageBoardCards` | Can create, edit, delete cards |
| `PermissionManageBoardProperties` | Can add/edit/delete card properties |
| `PermissionCommentBoardCards` | Can add comments to cards |
| `PermissionDeleteOthersComments` | Can delete other users' comments |
| `PermissionManageBoardType` | Can change board type (open/private) |
| `PermissionDeleteBoard` | Can delete the board |

---

## ClientConfig (`model/clientConfig.go`)

Sent to the browser on startup.

```go
type ClientConfig struct {
    Telemetry                  bool   `json:"telemetry"`
    TelemetryID                string `json:"telemetryId"`
    EnablePublicSharedBoards   bool   `json:"enablePublicSharedBoards"`
    FeatureFlags               map[string]string `json:"featureFlags"`
    MaxFileSize                int64  `json:"maxFileSize"`
    AllowEmailInvitations      bool   `json:"allowEmailInvitations"`
    // ... more settings
}
```

---

## Cloud (`model/cloud.go`)

Cloud plan limits.

```go
type BoardsCloudLimits struct {
    Cards       int `json:"cards"`       // Max cards per board
    Views       int `json:"views"`       // Max views per board
    UsedCards   int `json:"usedCards"`   // Cards used
}
```

---

## Import/Export (`model/import_export.go`)

```go
type Archive struct {
    Version int           `json:"version"`
    Date    int64         `json:"date"`
    Boards  []BoardPack   `json:"boards"`
}

type BoardPack struct {
    Board  Board   `json:"board"`
    Blocks []Block `json:"blocks"`
}
```

---

## Error Models (`model/error.go`, `model/errorResponse.go`)

```go
type ErrNotFound struct {
    Resource string
}

type ErrorResponse struct {
    Error     string `json:"error"`
    ErrorCode int    `json:"errorCode"`
}
```
