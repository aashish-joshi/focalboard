# Sidebar Categories

## Overview

The sidebar organizes boards into **categories** (folders). Each user has their own category configuration per team — categories are personal and not shared across users.

Categories can be:
- **System categories** — created automatically (e.g., "Boards" default category)
- **Custom categories** — created by the user

---

## Data Model

### Category

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

### CategoryBoards

Sent to the client, includes category metadata + ordered board list:

```go
type CategoryBoards struct {
    Category
    BoardMetadata []CategoryBoardMetadata `json:"boardMetadata"`
}

type CategoryBoardMetadata struct {
    BoardID string `json:"boardID"`
    Hidden  bool   `json:"hidden"`   // true = board is hidden from sidebar
}
```

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v2/teams/{teamID}/categories` | Get user's categories with board list |
| `POST` | `/api/v2/teams/{teamID}/categories` | Create a new category |
| `PUT` | `/api/v2/teams/{teamID}/categories` | Reorder categories |
| `PUT` | `/api/v2/teams/{teamID}/categories/{categoryID}` | Rename/modify a category |
| `DELETE` | `/api/v2/teams/{teamID}/categories/{categoryID}` | Delete a category |
| `POST` | `/api/v2/teams/{teamID}/categories/{categoryID}/boards` | Add boards to category |
| `PUT` | `/api/v2/teams/{teamID}/categories/{categoryID}/boards` | Reorder boards in category |
| `DELETE` | `/api/v2/teams/{teamID}/categories/{categoryID}/boards/{boardID}` | Remove board from category |

---

## Category Operations Flow

### Loading Categories

```
BoardPage mounts
  │
dispatch(fetchSidebarCategories(teamId))
  │
octoClient.getTeamCategories(teamId)   GET /api/v2/teams/{teamId}/categories
  │
Redux: sidebar.updateCategories(categoryBoards)
  │
Sidebar renders category list
```

### Creating a Category

```
User clicks "Add category" in sidebar
  │
mutator.createCategory({ name, teamId, userId })
  │
octoClient.createCategory(category)   POST /api/v2/teams/{teamId}/categories
  │
Redux: sidebar.updateCategories([...existing, newCategory])
```

### Moving a Board to a Category

```
User drags board to a different category
  OR
Right-click board → "Move to" → select category
  │
mutator.moveBoardToCategory(teamID, boardID, toCategoryID, fromCategoryID)
  │
octoClient.updateCategoryBoard(teamId, categoryId, [boardId])
  POST /api/v2/teams/{teamId}/categories/{categoryId}/boards
  │
Redux: sidebar.addBoardToCategory({ boardId, categoryId })
```

### Reordering Categories

```
User drags a category to a new position
  │
mutator.reorderCategories(teamID, newOrder)
  │
octoClient.updateCategoriesOrder(teamId, categoryOrder)
  PUT /api/v2/teams/{teamId}/categories  (body: string[] of IDs in new order)
  │
Redux: sidebar.updateCategoryOrder(newOrder)
```

### Hiding a Board

```
User right-clicks a board → "Hide board"
  │
Redux: sidebar.hiddenBoardsInCategory([boardId])
  (local only — hidden state is stored per session)
```

---

## System Categories

System categories are created automatically by the server:
- `"Boards"` — default category for all boards not explicitly placed
- Named after the Mattermost channel (in plugin mode)

Users **cannot delete** system categories. They can rename them. If a system category would be empty after moving all boards out of it, it is hidden from the sidebar.

---

## Custom Categories

Custom categories can be:
- Created with any name
- Renamed
- Deleted (boards in the deleted category move to the "Boards" system category)
- Reordered via drag-and-drop

---

## WebSocket Real-Time Updates

Category changes are broadcast to the user's WebSocket session:

| WS Action | When Triggered |
|-----------|---------------|
| `UPDATE_CATEGORY` | Category created, renamed, or deleted |
| `UPDATE_BOARD_CATEGORY` | Board moved to a different category |
| `REORDER_CATEGORIES` | Category order changed |
| `REORDER_CATEGORY_BOARDS` | Board order within category changed |

These ensure that if the user has multiple tabs open, all tabs stay in sync.

---

## Redux Sidebar State

```typescript
interface SidebarState {
    categoryAttributes: CategoryBoards[]    // Ordered category list with boards
    hiddenBoardIDs: string[]                // Locally hidden boards
}
```

### Key Selectors

```typescript
getSidebarCategories(state): CategoryBoards[]
getHiddenBoardIDs(state): string[]
getCategoryOfBoard(state, boardId): CategoryBoards | undefined
```
