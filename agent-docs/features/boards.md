# Boards

## What is a Board?

A board is the top-level workspace object in Focalboard. It contains:
- A schema defining what **card properties** (columns) exist
- One or more **views** (kanban, table, gallery, calendar)
- A collection of **cards** (rows/items)

---

## Board Types

| Type | Value | Description |
|------|-------|-------------|
| Open | `"O"` | Any team member can view the board without being explicitly added as a member |
| Private | `"P"` | Only users added as members can access the board |

---

## Board Roles (Member Permissions)

Each board member has a role that controls what they can do:

| Role | View | Comment | Edit Cards | Edit Properties | Manage Members | Delete Board |
|------|------|---------|-----------|----------------|---------------|-------------|
| `viewer` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `commenter` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `editor` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| `admin` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

The board `minimumRole` field sets the role granted to any team member who views an open board.

---

## Board Data Flow

### Creating a Board

```
User clicks "Add a board" → BoardTemplateSelector opens
  │
User selects template or blank board
  │
mutator.createBoard(newBoard)
  ├─ octoClient.createBoard(board)     POST /api/v2/boards
  ├─ Server: app.CreateBoard()         inserts board + adds creator as admin
  ├─ Server: ws.BroadcastBoardChange() pushes to team subscribers
  ├─ Redux: updateBoards([board])
  └─ Redux: setCurrent(board.id)
  │
Board loads → default view renders
```

### Loading a Board

```
Browser navigates to /team/{teamId}/{boardId}
  │
BoardPage mounts
  ├─ octoClient.getBoard(boardId)             GET /api/v2/boards/{id}
  ├─ octoClient.getBlocksWithType(id,'view')  GET /api/v2/boards/{id}/blocks?type=view
  ├─ octoClient.getAllBlocks(boardId)         GET /api/v2/boards/{id}/blocks?all=true
  ├─ octoClient.getBoardMembers(teamId, id)   GET /api/v2/boards/{id}/members
  │
Redux updates:
  ├─ boards.setCurrent(boardId)
  ├─ boards.updateBoards([board])
  ├─ views.updateViews([...views])
  ├─ cards.updateCards([...cards])
  └─ views.setCurrent(firstViewId or urlViewId)
```

---

## Board Properties (Metadata)

Stored in `Board.properties` as a flexible `map[string]interface{}` (server) or `Record<string, string | string[]>` (client). Used for custom board-level settings.

---

## Card Properties Schema (`cardProperties`)

The `cardProperties` field on a Board is an array of `IPropertyTemplate` objects. Each template defines one column/property that all cards in the board can have:

```json
[
    {
        "id": "prop_abc123",
        "name": "Status",
        "type": "select",
        "options": [
            { "id": "opt_1", "value": "Todo", "color": "propColorGray" },
            { "id": "opt_2", "value": "In Progress", "color": "propColorBlue" },
            { "id": "opt_3", "value": "Done", "color": "propColorGreen" }
        ]
    },
    {
        "id": "prop_def456",
        "name": "Assignee",
        "type": "person",
        "options": []
    }
]
```

### Adding a Property

```
User clicks "Add a property" → property type picker
  │
mutator.insertCardProperty(boardId, cardProperties, newTemplate)
  ├─ Creates new template with generated ID
  ├─ Patches board: updatedCardProperties = [...existing, newTemplate]
  └─ All cards now have this property (with empty value)
```

---

## Board Duplication

`app.DuplicateBoard()` creates a complete copy:

1. New board record with a new ID
2. All blocks (views, cards, content blocks, comments) copied with new IDs
3. Parent/child ID references updated to new IDs
4. File attachments copied to new paths

```
mutator.duplicateBoard(boardId)
  │
octoClient.duplicateBoard(boardId)    POST /api/v2/boards/{id}/duplicate
  │
Server: app.DuplicateBoard()
  ├─ store.DuplicateBoard()           Deep copy in DB
  ├─ app.CopyAndUpdateCardFiles()     Copy file attachments
  └─ BroadcastBoardChange()
  │
Redux: updateBoards + updateViews + updateCards (all new IDs)
```

---

## Board Deletion (Soft Delete)

Boards are soft-deleted: the `DeleteAt` field is set to the current timestamp. They can be undeleted.

---

## Mattermost Channel Linking (Plugin Mode)

In Mattermost plugin mode, a board can be linked to a Mattermost channel via `Board.channelId`. This:
1. Shows the board in the channel sidebar
2. May affect member permissions (channel members get board access)

---

## Global Board (Root Team)

The root team (ID `"0"`) is created on first startup. In standalone mode, all boards belong to this single team. In plugin mode, each Mattermost team maps to a Focalboard team.

---

## Board Templates

A board with `isTemplate = true` is a template board. Template boards:
- Appear in the `BoardTemplateSelector` component
- Can be duplicated to create real boards
- Are stored in the same database table as regular boards
- Built-in templates are imported from `app/templates.boardarchive` on startup
