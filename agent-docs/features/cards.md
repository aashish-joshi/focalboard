# Cards

## What is a Card?

A card is the primary unit of content in Focalboard. It represents a task, note, item, or any data record. Each card:
- Belongs to exactly one **board**
- Has a **title** and optional **icon**
- Can have **property values** (defined by the board's property schema)
- Can contain **content blocks** (text, images, checkboxes, headings, etc.)
- Can receive **comments**
- Can have **file attachments**

Cards are stored as `Block` records with `type = "card"` internally, but the V3 API exposes them via the higher-level `Card` model.

---

## Card Data Model

```typescript
// Frontend type (blocks/card.ts)
interface Card {
    id: string
    boardId: string
    type: 'card'
    title: string
    fields: {
        icon?: string           // emoji icon
        isTemplate?: boolean    // is this a card template?
        properties: Record<string, string | string[]>  // property values
        contentOrder: Array<string | string[]>          // ordered content block IDs
    }
    createAt: number
    updateAt: number
    deleteAt: number
}
```

### Property Values

Property values are stored in `fields.properties` keyed by property template ID:
```json
{
  "prop_abc": "opt_123",           // select: option ID
  "prop_def": ["opt_1", "opt_2"],  // multiSelect: array of option IDs
  "prop_ghi": "2024-06-01",        // date: ISO date string
  "prop_jkl": "user_id_xyz"        // person: user ID
}
```

### Content Order

`contentOrder` is an array of content block IDs defining the display order. Elements can be:
- A `string` — a single content block
- A `string[]` — multiple content blocks displayed side-by-side (column layout)

---

## Card Lifecycle

### Creating a Card

```
User clicks "Add card" in any view
  │
mutator.insertCard(newCard)
  ├─ Creates card with generated ID
  ├─ Sets boardId, createdBy, timestamps
  ├─ If in kanban view: sets groupByProperty value to current column's option
  │
octoClient.insertBlock(boardId, block)   POST /api/v2/boards/{boardId}/blocks
  │
Redux: updateCards([card])
  │
View re-renders with new card
```

### Opening a Card

```
User clicks a card → showCard(cardId)
  │
URL updates to include cardId
  │
CardDialog mounts with cardId
  │
Redux: cards.setCurrent(cardId)
  │
CardDetail renders with full card data from Redux store
```

### Editing Card Properties

```
User changes a property value in CardDetail
  │
mutator.changeBlockProperty(boardId, card, propertyId, newValue)
  ├─ Creates BlockPatch with updatedFields.properties = { [propertyId]: newValue }
  ├─ octoClient.patchBlock(boardId, cardId, patch)
  ├─ Redux: updateCards([updatedCard]) (optimistic)
  └─ Server: app.PatchBlock() → DB update + WS broadcast
```

### Adding Content

```
User presses Enter or clicks "Add text" in CardDetail
  │
ContentBlock created (e.g., type='text')
  │
mutator.insertBlock(boardId, contentBlock)
  │
mutator.changeCardContentOrder(boardId, cardId, oldOrder, newOrder)
  │
card.fields.contentOrder updated to include new block ID
```

---

## Content Blocks

Content blocks are child blocks (`parentId = cardId`) that form the body of a card:

| Type | Description | Storage |
|------|-------------|---------|
| `text` | Rich text paragraph | `title` = markdown text |
| `h1` / `h2` / `h3` | Heading | `title` = heading text |
| `list-item` | List/bullet point | `title` = item text |
| `checkbox` | Checkbox item | `title` = label, `fields.value` = true/false |
| `image` | Image from URL or uploaded file | `fields.fileId` = file ID |
| `divider` | Horizontal rule | no data |
| `quote` | Block quote | `title` = quote text |
| `video` | Video embed | `fields.url` = video URL |
| `attachment` | File attachment | `fields.fileId` = file ID, `fields.fileName` = original name |

### Content Block Ordering

The `contentOrder` on the parent card controls display order. When blocks are reordered (drag-and-drop), a patch is sent to update `card.fields.contentOrder`.

---

## Comments

Comments are `Block` records with `type = "comment"` and `parentId = cardId`.

```typescript
interface CommentBlock extends Block {
    type: 'comment'
    title: string   // The comment text (Markdown)
}
```

### Adding a Comment

```
mutator.insertBlock(boardId, commentBlock)
  │
Redux: updateComments([comment])
  │
CommentsList re-renders
```

### Deleting a Comment

Any board editor can delete their own comments. Board admins can delete anyone's comments (`PermissionDeleteOthersComments`).

---

## Card Templates

A card with `fields.isTemplate = true` is a card template. Templates:
- Appear in the "Add from template" menu
- Can be duplicated to create new cards
- Share the same board's property schema

### Creating a Template from a Card

```
User opens CardActionsMenu → "Make template"
  │
Duplicate card + set fields.isTemplate = true
```

---

## Card Duplication

```
mutator.duplicateCard(cardId, boardId)
  │
octoClient.duplicateBlock(boardId, cardId)   POST /api/v2/boards/{boardId}/blocks/{cardId}/duplicate
  │
Server: app.DuplicateBlock()
  ├─ Copies card + all child content blocks
  ├─ Copies file attachments
  └─ Returns new block array
  │
Redux: updateCards + updateContents + updateAttachments
```

---

## Card Filters

Cards can be filtered within a view using the `FilterGroup` on `BoardView.fields.filter`.

The `cardFilter.ts` module evaluates filters on the client side:

```typescript
CardFilter.filterCards(cards, board, view): Card[]
// Applies the view's FilterGroup to the card list, returns matching cards
```

Filters are composed of `FilterClause` conditions (AND/OR) testing property values.

---

## Cloud Card Limits

In cloud deployments, a maximum number of cards per board may be enforced (`cardLimit`). Cards beyond the limit have `block.limited = true` and show a "limited" badge. The `limitTimestamp` in Redux state tracks when the limit was applied.

---

## Card Sorting

Cards can be sorted in three ways within a view:

| Sort Mode | Description |
|-----------|-------------|
| Manual | `cardOrder` on the view defines explicit order (drag-and-drop) |
| Property sort | `sortOptions` on the view defines sort rules (property + direction) |
| Default | Natural insertion order |

The selector `getCurrentViewCardsSortedFilteredAndGrouped` applies all sorting, filtering, and grouping to produce the final display list.
