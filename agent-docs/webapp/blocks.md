# Block Data Model (Frontend)

## Overview

**Directory:** `webapp/src/blocks/`

This documents the TypeScript type system for Focalboard blocks on the frontend. All data exchanged with the server follows these types.

---

## Core Block Type (`blocks/block.ts`)

```typescript
interface Block {
    id: string
    boardId: string
    parentId: string
    createdBy: string
    modifiedBy: string
    schema: number          // Data version (currently 1)
    type: BlockTypes        // Discriminator field
    title: string
    fields: Record<string, any>  // Extensible payload
    createAt: number        // Unix timestamp in milliseconds
    updateAt: number
    deleteAt: number        // 0 means not deleted
    limited?: boolean       // Cloud: card is beyond the limit
}
```

### Block Types Enum

```typescript
type BlockTypes =
    | 'board'
    | 'view'
    | 'card'
    | 'comment'
    | 'attachment'
    | 'text'
    | 'image'
    | 'divider'
    | 'checkbox'
    | 'h1'
    | 'h2'
    | 'h3'
    | 'list-item'
    | 'quote'
    | 'video'
    | 'unknown'
```

### Patch Type

```typescript
interface BlockPatch {
    parentId?: string
    schema?: number
    type?: BlockTypes
    title?: string
    updatedFields?: Record<string, any>
    deletedFields?: string[]
    deleteAt?: number
}
```

### Batch Patch

```typescript
interface BlockPatchBatch {
    blockIds: string[]
    blockPatches: BlockPatch[]
}
```

### Helper Functions

```typescript
function createBlock(block?: Partial<Block>): Block
// Creates a new block with generated ID, timestamps, and type 'unknown'

function createPatchesFromBlocks(
    newBlock: Block,
    oldBlock: Block
): [BlockPatch, BlockPatch]
// Returns [forwardPatch, undoPatch]
```

---

## Board (`blocks/board.ts`)

```typescript
interface Board {
    id: string
    teamId: string
    channelId?: string      // Linked Mattermost channel
    createdBy: string
    modifiedBy: string
    type: BoardTypes        // 'O' = open, 'P' = private
    minimumRole: MemberRole // Default member role
    title: string
    description: string
    icon?: string
    showDescription: boolean
    isTemplate: boolean
    templateVersion: number
    properties: Record<string, string | string[]>
    cardProperties: IPropertyTemplate[]   // Column definitions
    createAt: number
    updateAt: number
    deleteAt: number
}

type BoardTypes = 'O' | 'P'

type MemberRole = 'viewer' | 'commenter' | 'editor' | 'admin' | 'none'
```

### Board Patch

```typescript
interface BoardPatch {
    type?: BoardTypes
    minimumRole?: MemberRole
    title?: string
    description?: string
    icon?: string
    showDescription?: boolean
    isTemplate?: boolean
    channelId?: string
    updatedProperties?: Record<string, string | string[]>
    deletedProperties?: string[]
    updatedCardProperties?: IPropertyTemplate[]
    deletedCardProperties?: string[]
}
```

### BoardMember

```typescript
interface BoardMember {
    boardId: string
    userId: string
    roles?: string
    minimumRole: string
    schemeAdmin: boolean
    schemeEditor: boolean
    schemeCommenter: boolean
    schemeViewer: boolean
    synthetic: boolean  // Derived from Mattermost channel membership
}
```

### Property Template (Column Definition)

```typescript
interface IPropertyTemplate {
    id: string
    name: string
    type: PropertyTypeEnum
    options: IPropertyOption[]
}

interface IPropertyOption {
    id: string
    value: string
    color: string
}

type PropertyTypeEnum =
    | 'text'
    | 'number'
    | 'select'
    | 'multiSelect'
    | 'date'
    | 'person'
    | 'multiPerson'
    | 'file'
    | 'checkbox'
    | 'url'
    | 'email'
    | 'phone'
    | 'createdTime'
    | 'createdBy'
    | 'updatedTime'
    | 'updatedBy'
    | 'unknown'
```

---

## Card (`blocks/card.ts`)

```typescript
interface Card extends Block {
    type: 'card'
    fields: {
        icon?: string
        isTemplate?: boolean
        properties: Record<string, string | string[]>  // Property values by template ID
        contentOrder: Array<string | string[]>          // IDs of content blocks in display order
    }
}

// CardPatch maps to BlockPatch with typed fields
interface CardPatch {
    title?: string
    icon?: string
    contentOrder?: Array<string | string[]>
    updatedProperties?: Record<string, string | string[]>
    deletedProperties?: string[]
}
```

### Helper Functions

```typescript
function createCard(block?: Partial<Block>): Card
// Creates a new card with type='card', empty properties and contentOrder
```

---

## BoardView (`blocks/boardView.ts`)

```typescript
interface BoardView extends Block {
    type: 'view'
    fields: {
        viewType: IViewType
        groupById?: string          // Property ID to group by
        dateDisplayPropertyId?: string  // Calendar grouping property
        sortOptions: ISortOption[]
        visiblePropertyIds: string[]
        visibleOptionIds: string[]   // Visible select option IDs
        hiddenOptionIds: string[]
        collapsedOptionIds: string[] // Collapsed group IDs
        filter: FilterGroup
        cardOrder: string[]          // Manual card sort order
        columnWidths: Record<string, number>
        columnCalculations: Record<string, string>       // Table calculations per column
        kanbanCalculations: Record<string, KanbanCalculationFields>
        defaultTemplateId: string    // Default card template ID
    }
}

type IViewType = 'board' | 'table' | 'gallery' | 'calendar'

interface ISortOption {
    propertyId: string
    reversed: boolean
}

interface KanbanCalculationFields {
    calculation: string   // 'count', 'sum', 'average', etc.
    propertyId: string
}
```

### Helper Functions

```typescript
function createBoardView(block?: Partial<Block>): BoardView
// Creates a new view with type='view', empty sortOptions, filter, etc.

function createPatchesFromBoardViews(
    newView: BoardView,
    oldView: BoardView
): [BlockPatch, BlockPatch]
```

---

## Content Blocks (`blocks/contentBlock.ts`)

```typescript
// Base for all content blocks
interface ContentBlock extends Block {
    type: ContentBlockTypes
}

type ContentBlockTypes =
    | 'text'
    | 'image'
    | 'divider'
    | 'checkbox'
    | 'h1'
    | 'h2'
    | 'h3'
    | 'list-item'
    | 'quote'
    | 'video'
```

---

## Comment Block (`blocks/commentBlock.ts`)

```typescript
interface CommentBlock extends Block {
    type: 'comment'
}
```

Comments are child blocks of cards, identified by `parentId = cardId` and `type = 'comment'`.

---

## Attachment Block (`blocks/attachmentBlock.ts`)

```typescript
interface AttachmentBlock extends Block {
    type: 'attachment'
    fields: {
        fileId: string
        fileName: string
        attachmentDeleted?: boolean
    }
}
```

---

## Filtering (`blocks/filterGroup.ts`, `blocks/filterClause.ts`)

```typescript
// A group of filter conditions (AND or OR)
interface FilterGroup {
    operation: 'and' | 'or'
    filters: Array<FilterClause | FilterGroup>  // Can be nested
}

// A single filter condition
interface FilterClause {
    propertyId: string
    condition: FilterCondition
    values: string[]
}

type FilterCondition =
    | 'includes'
    | 'notIncludes'
    | 'isEmpty'
    | 'isNotEmpty'
    | 'isSet'
    | 'isNotSet'
    | 'includes'
    | 'notContains'
    | 'startsWith'
    | 'notStartsWith'
    | 'endsWith'
    | 'notEndsWith'
```

### Helper Functions

```typescript
function createFilterGroup(o?: FilterGroup): FilterGroup
// Deep clones a filter group

function createFilterClause(o?: FilterClause): FilterClause
// Creates a new filter clause with empty values

function isAFilterGroupFilter(filter: FilterClause | FilterGroup): filter is FilterGroup
// Type guard to distinguish clauses from groups
```

---

## Data Types Summary

| Type | Location | Parent | key `type` field |
|------|----------|--------|-----------------|
| `Block` | `blocks/block.ts` | — | varies |
| `Board` | `blocks/board.ts` | — | (separate entity) |
| `BoardView` | `blocks/boardView.ts` | Board | `'view'` |
| `Card` | `blocks/card.ts` | Board | `'card'` |
| `CommentBlock` | `blocks/commentBlock.ts` | Card | `'comment'` |
| `AttachmentBlock` | `blocks/attachmentBlock.ts` | Card | `'attachment'` |
| `ContentBlock` | `blocks/contentBlock.ts` | Card | `'text'`,`'image'`, etc. |

---

## ID Generation

All IDs are generated client-side using UUID v4 via `createGuid()` from `utils.ts`:
```typescript
function createGuid(): string {
    // Returns a UUID v4 string
}
```

IDs are assigned before calling `octoClient.insertBlock()`, so the server uses the provided ID.
