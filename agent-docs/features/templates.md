# Board and Card Templates

## Overview

Focalboard supports **board templates** and **card templates** to quickly scaffold new boards and cards with predefined structure and content.

---

## Board Templates

A board with `isTemplate = true` is a board template. It appears in the `BoardTemplateSelector` when users add a new board.

### Types of Templates

| Type | Description |
|------|-------------|
| **Global templates** | Built-in templates bundled with Focalboard (stored in `app/templates.boardarchive`) |
| **Team templates** | Templates created by team members (stored in the database like normal boards) |

### Built-in Templates

Built-in templates are stored as a binary `.boardarchive` file at `server/app/templates.boardarchive`. They are imported into the database on server startup if they are not already present.

Default templates include:
- Project Tasks
- Personal Goals
- Meeting Notes
- Personal To-do list
- Company Goals
- Research Project
- Sales Pipeline
- Competitive Analysis
- Team Retrospective
- Bug Tracking
- User Interview
- Content Calendar
- Sprint Planner
- Roadmap

### Template Versioning

Each template board has a `templateVersion` integer. When the built-in templates are updated (new version), the server detects outdated templates in the database and re-imports them.

### Creating a Board from a Template

```
User selects template in BoardTemplateSelector
  │
mutator.duplicateBoard(templateId, teamId, asTemplate=false)
  │
octoClient.duplicateBoard(templateId, asTemplate=false, toTeamID=teamId)
  POST /api/v2/boards/{templateId}/duplicate
  │
Server: app.DuplicateBoard(templateId, userID, teamID, asTemplate=false)
  ├─ Copies board with isTemplate=false
  ├─ Copies all views, cards, content blocks
  └─ Sets title = template title (user can rename)
  │
New board opens in the UI
```

### Creating a Template from a Board

```
User opens BoardHeader → "Convert to template"
  │
mutator.updateBoard({ ...board, isTemplate: true })
  │
Board now appears in template gallery
```

### Deleting a Template

Only team templates can be deleted. Global templates cannot be deleted by users.

---

## Card Templates

A card with `fields.isTemplate = true` is a card template. It appears in the "Add from template" button in any view.

### Creating a Card Template

```
User opens an existing card → CardActionsMenu → "Make template"
  │
mutator.duplicateCard(cardId, boardId, asTemplate=true)
  │
New card created with isTemplate=true, same properties + content
```

Or directly:
```
User clicks "+" in "Add from template" menu → "New template"
  │
mutator.insertCard({ ..., fields: { isTemplate: true, ... } })
```

### Creating a Card from a Template

```
User clicks "Add from template" → selects a template → creates card
  │
mutator.duplicateCard(templateId, boardId, asTemplate=false)
  │
New card with same properties and content as template
If in kanban: card placed in the appropriate column (based on group property)
```

### Default Card Template for a View

Each view can specify a `defaultTemplateId`. When the user clicks "Add card" (not "Add from template"), the default template is used:

```
User clicks "Add card"
  │
If view.fields.defaultTemplateId is set:
  └─ mutator.duplicateCard(defaultTemplateId, ...)
Else:
  └─ mutator.insertCard({ blank card with group property pre-filled })
```

Setting the default template:
```
NewCardButton → "Set as default" → mutator.changeViewDefaultTemplateId(...)
```

---

## Template Archive Format (`.boardarchive`)

Board archives are used for import/export and for the built-in templates.

```json
{
  "version": 2,
  "date": 1700000000000,
  "boards": [
    {
      "board": { ...board object... },
      "blocks": [ ...block objects... ]
    }
  ]
}
```

### Import

```
POST /api/v2/boards/archive/import
Body: multipart form with archive file
  │
app.ImportArchive(reader, teamID, userID, imports)
  │
Parses archive, creates all boards and blocks
```

### Export

```
POST /api/v2/boards/{boardID}/archive/export
  │
app.ExportArchive(writer, boardIDs)
  │
Returns archive as downloadable file
```
