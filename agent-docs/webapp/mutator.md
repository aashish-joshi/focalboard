# Mutator (`mutator.ts`)

## Overview

**File:** `webapp/src/mutator.ts`

The `mutator` is the single point of entry for all state mutations in Focalboard. It provides:
1. **Undo/redo support** — every action is reversible
2. **Optimistic updates** — Redux store is updated immediately, server is synced asynchronously
3. **Atomic groups** — multiple related mutations are wrapped into a single undo step

```typescript
class Mutator {
    private undoGroupId: string | undefined
    private undoDisplayId: string | undefined
}

const mutator = new Mutator()
export default mutator
```

---

## Undo/Redo Architecture

The mutator uses `undoManager` (an undo/redo stack) to record reversible operations.

Each mutation is submitted as a pair: `{redo: fn, undo: fn}`. When the user triggers Ctrl+Z, the undo function is called; when they trigger Ctrl+Y / Ctrl+Shift+Z, the redo function is called.

### Group Management

Multiple mutations can be grouped into a single undo step:

```typescript
// Automatic grouping:
await mutator.performAsUndoGroup(async () => {
    await mutator.insertBlock(boardId, block1, 'description')
    await mutator.insertBlock(boardId, block2, 'description')
    // Both insertions are reversed by a single Ctrl+Z
})

// Manual grouping:
mutator.beginUndoGroup()
await mutator.insertBlock(boardId, block1, 'description')
await mutator.insertBlock(boardId, block2, 'description')
mutator.endUndoGroup()
```

---

## Batch Redux Updates Helper

```typescript
function updateAllBoardsAndBlocks(boards: Board[], blocks: Block[]) {
    return batch(() => {
        store.dispatch(updateBoards(boards.filter(b => !b.deleteAt)))
        store.dispatch(deleteBoards(boards.filter(b => b.deleteAt).map(b => b.id)))
        store.dispatch(updateViews(blocks.filter(b => b.type === 'view' && !b.deleteAt)))
        store.dispatch(deleteViews(blocks.filter(b => b.type === 'view' && b.deleteAt).map(b => b.id)))
        store.dispatch(updateCards(blocks.filter(b => b.type === 'card' && !b.deleteAt)))
        store.dispatch(deleteCards(blocks.filter(b => b.type === 'card' && b.deleteAt).map(b => b.id)))
        store.dispatch(updateComments(blocks.filter(b => b.type === 'comment' && !b.deleteAt)))
        store.dispatch(updateAttachments(blocks.filter(b => b.type === 'attachment' && !b.deleteAt)))
        store.dispatch(updateContents(blocks.filter(b => !['board','view','card','comment','attachment'].includes(b.type) && !b.deleteAt)))
    })
}
```

---

## Block Mutations

### `insertBlock`

```typescript
async insertBlock(
    boardId: string,
    block: Block,
    description = 'add',
    afterRedo?: (block: Block) => Promise<void>,
    beforeUndo?: (block: Block) => Promise<void>
): Promise<Block | undefined>
```

- Redo: `octoClient.insertBlock(boardId, block)` → dispatches `updateCards`/`updateViews`/etc.
- Undo: `octoClient.deleteBlock(boardId, block.id)` → dispatches `deleteCards`/etc.

### `insertBlocks`

```typescript
async insertBlocks(
    boardId: string,
    blocks: Block[],
    description = 'add blocks',
    afterRedo?: (blocks: Block[]) => Promise<void>,
    beforeUndo?: (blocks: Block[]) => Promise<void>,
    sourceBoardID?: string
): Promise<Block[] | undefined>
```

Batch insert. Undo deletes all inserted blocks.

### `updateBlock`

```typescript
async updateBlock(
    boardId: string,
    newBlock: Block,
    oldBlock: Block,
    description: string
): Promise<void>
```

- Redo: patch with new block fields
- Undo: patch with old block fields

### `updateBlocks`

Batch update. Same as `updateBlock` but for arrays.

### `deleteBlock`

```typescript
async deleteBlock(
    block: Block,
    description?: string,
    beforeRedo?: (block: Block) => Promise<void>,
    afterUndo?: (block: Block) => Promise<void>
): Promise<void>
```

- Redo: `octoClient.deleteBlock(boardId, blockId)`
- Undo: `octoClient.undeleteBlock(boardId, blockId)` + re-dispatch the block

### `duplicateBlock`

```typescript
async duplicateBlock(
    boardId: string,
    blockId: string,
    description = 'duplicate block',
    asTemplate = false
): Promise<void>
```

---

## Card Mutations

### `insertCard`

```typescript
async insertCard(
    newCard: Card,
    description = 'add card',
    showAfterRedo = false,
    afterRedo?: (card: Card) => Promise<void>
): Promise<Card | undefined>
```

### `duplicateCard`

```typescript
async duplicateCard(
    cardId: string,
    boardId: string,
    asTemplate = false,
    description = 'duplicate card',
    isFromTemplate = false,
    templateId?: string,
    afterRedo?: (cardId: string) => Promise<void>,
    beforeUndo?: () => Promise<void>
): Promise<void>
```

### `changeCardTitle`

```typescript
async changeCardTitle(
    boardId: string,
    cardId: string,
    oldTitle: string,
    newTitle: string,
    description = 'rename card'
): Promise<void>
```

### `changeCardIcon`

```typescript
async changeCardIcon(
    boardId: string,
    cardId: string,
    oldIcon: string,
    icon: string,
    description = 'change card icon'
): Promise<void>
```

### `changeCardContentOrder`

```typescript
async changeCardContentOrder(
    boardId: string,
    cardId: string,
    oldContentOrder: string[],
    newContentOrder: string[],
    description = 'reorder card content'
): Promise<void>
```

---

## Board Mutations

### `createBoard`

```typescript
async createBoard(newBoard: Board, description = 'add board'): Promise<void>
```

### `updateBoard`

```typescript
async updateBoard(newBoard: Board, oldBoard: Board, description: string): Promise<void>
```

### `deleteBoard`

```typescript
async deleteBoard(
    board: Board,
    description = 'delete board',
    afterRedo?: () => Promise<void>,
    beforeUndo?: () => Promise<void>
): Promise<void>
```

### `duplicateBoard`

```typescript
async duplicateBoard(
    boardId: string,
    description = 'duplicate board',
    asTemplate = false,
    afterRedo?: (boardId: string) => Promise<void>
): Promise<void>
```

### `changeBoardTitle` / `changeBoardDescription` / `changeBoardIcon`

Each follows the same pattern: `updateBoard(newBoard, oldBoard, description)`.

### `changeBoardCardProperties`

```typescript
async changeBoardCardProperties(
    boardId: string,
    oldCardProperties: IPropertyTemplate[],
    newCardProperties: IPropertyTemplate[],
    description = 'change card properties'
): Promise<void>
```

---

## Property Mutations

### Add/Delete Options

```typescript
async insertPropertyOption(
    boardId: string,
    cardProperties: IPropertyTemplate[],
    propertyTemplate: IPropertyTemplate,
    option: IPropertyOption,
    description = 'add option'
): Promise<void>

async deletePropertyOption(
    boardId: string,
    cardProperties: IPropertyTemplate[],
    propertyTemplate: IPropertyTemplate,
    option: IPropertyOption,
    description = 'delete option'
): Promise<void>
```

### Rename Option

```typescript
async changePropertyOptionValue(
    boardId: string,
    cardProperties: IPropertyTemplate[],
    propertyTemplate: IPropertyTemplate,
    option: IPropertyOption,
    value: string,
    description = 'rename option'
): Promise<void>
```

### Change Option Color

```typescript
async changePropertyOptionColor(
    boardId: string,
    cardProperties: IPropertyTemplate[],
    propertyTemplate: IPropertyTemplate,
    option: IPropertyOption,
    color: string
): Promise<void>
```

### Add/Delete Property

```typescript
async insertCardProperty(
    boardId: string,
    cardProperties: IPropertyTemplate[],
    property: IPropertyTemplate,
    description = 'add property'
): Promise<void>

async deleteCardProperty(
    boardId: string,
    cardProperties: IPropertyTemplate[],
    propertyId: string,
    description = 'delete property'
): Promise<void>
```

### Change Property Value on Card

```typescript
async changeBlockProperty(
    boardId: string,
    card: Card,
    propertyId: string,
    value: string | string[],
    description = 'change property'
): Promise<void>
```

---

## View Mutations

### Create/Delete View

```typescript
async insertBoardView(
    view: BoardView,
    description = 'add view',
    afterRedo?: (newView: BoardView) => Promise<void>,
    beforeUndo?: () => Promise<void>
): Promise<BoardView | undefined>

async deleteView(view: BoardView): Promise<void>
```

### View Settings

```typescript
async changeViewCardOrder(boardId: string, viewId: string, cardOrder: string[], description?): Promise<void>
async changeViewGroupBy(boardId: string, viewId: string, groupById: string, description?): Promise<void>
async changeViewDisplayBy(boardId: string, viewId: string, displayById: string, description?): Promise<void>
async changeViewSortOptions(boardId: string, viewId: string, sortOptions: ISortOption[], description?): Promise<void>
async changeViewFilter(boardId: string, viewId: string, filterGroup: FilterGroup, description?): Promise<void>
async changeViewVisiblePropertiesOrder(boardId: string, view: BoardView, template: IPropertyTemplate, destIndex: number, description?): Promise<void>
async changeViewKanbanCalculations(boardId: string, view: BoardView, newCalculations: Record<string, KanbanCalculationFields>, description?): Promise<void>
async changeViewColumnCalculations(boardId: string, view: BoardView, newCalculations: Record<string, string>, description?): Promise<void>
async changeViewColumnWidth(boardId: string, viewId: string, sourceColumnId: string, newWidth: number, description?): Promise<void>
async hideViewColumn(boardId: string, view: BoardView, columnOptionId: string): Promise<void>
async unhideViewColumn(boardId: string, view: BoardView, columnOptionId: string): Promise<void>
```

---

## Member Mutations

```typescript
async updateBoardMember(boardId: string, newMember: BoardMember, oldMember: BoardMember): Promise<void>
async addBoardMember(member: BoardMember, board: Board): Promise<BoardMember | undefined>
async removeBoardMember(member: BoardMember, board: Board): Promise<void>
```

---

## Category Mutations

```typescript
async createCategory(category: Category): Promise<void>
async updateCategory(category: Category): Promise<void>
async deleteCategory(teamID: string, categoryID: string): Promise<void>
async moveBoardToCategory(teamID: string, boardID: string, toCategoryID: string, fromCategoryID: string): Promise<void>
async reorderCategories(teamID: string, newOrder: string[]): Promise<void>
async reorderCategoryBoards(teamID: string, categoryID: string, newOrder: string[]): Promise<void>
```

---

## Sharing Mutations

```typescript
async setSharing(boardId: string, sharing: Sharing): Promise<void>
```

---

## File Mutations

```typescript
async uploadFile(boardId: string, file: File): Promise<string | undefined>
// Returns the fileId (used as block field value)
```

---

## Undo Manager (`undomanager.ts`)

The undo manager maintains two stacks: `undoStack` and `redoStack`.

```typescript
class UndoManager {
    undoStack: UndoCommand[]
    redoStack: UndoCommand[]

    perform(redo: () => Promise<void>, undo: () => Promise<void>, description: string, groupId?: string): Promise<void>
    undo(): Promise<string>
    redo(): Promise<string>
    beginGroupId(id: string): void
    endGroupId(): void
    clear(): void
}

interface UndoCommand {
    undo: () => Promise<void>
    redo: () => Promise<void>
    description: string
    groupId?: string
}
```
