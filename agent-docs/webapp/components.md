# Key React Components

## Overview

**Directory:** `webapp/src/components/`

This document covers the major React components, their purpose, props, and how they fit together.

---

## Component Tree (Simplified)

```
App
└── FocalboardRouter
    └── BoardPage
        ├── Sidebar
        │   ├── SidebarCategory
        │   │   └── SidebarBoardItem
        │   ├── SidebarUserMenu
        │   └── SidebarSettingsMenu
        └── Workspace
            ├── BoardTemplateSelector (if no board selected)
            └── CenterPanel
                ├── ViewHeader
                │   ├── ViewHeaderGroupByMenu
                │   ├── ViewHeaderSortMenu
                │   ├── ViewHeaderFilterMenu
                │   ├── ViewHeaderActionsMenu
                │   └── NewCardButton
                ├── [Kanban | Table | Gallery | CalendarFullView]
                └── CardDialog (modal when card is open)
                    └── CardDetail
                        ├── CardDetailProperties
                        ├── CardDetailContents
                        ├── CommentsList
                        └── AttachmentList
```

---

## CenterPanel (`components/centerPanel.tsx`)

The main content area that renders the active board view.

### Props

```typescript
type Props = {
    clientConfig?: ClientConfig
    board: Board
    cards: Card[]
    activeView: BoardView
    views: BoardView[]
    groupByProperty?: IPropertyTemplate
    dateDisplayProperty?: IPropertyTemplate
    readonly: boolean
    shownCardId?: string
    showCard: (cardId?: string) => void
    hiddenCardsCount: number
}
```

### Behavior

- Renders view header + the appropriate view component
- Tracks selected card IDs for multi-select operations
- Keyboard shortcuts:
  - `ESC` — deselect all cards
  - `Ctrl+D` — duplicate selected cards
  - `Del` / `Backspace` — delete selected cards
- Delegates rendering to view-specific component based on `activeView.fields.viewType`

### View Dispatch

| `viewType` | Rendered Component |
|-----------|-------------------|
| `'board'` | `<Kanban />` |
| `'table'` | `<Table />` |
| `'gallery'` | `<Gallery />` |
| `'calendar'` | `<CalendarFullView />` |

---

## Workspace (`components/workspace.tsx`)

Two-panel layout wrapper:

- Renders `<Sidebar />` on the left
- Renders `<CenterPanel />` (via `BoardPage`) on the right when a board is loaded
- Renders `<BoardTemplateSelector />` when no board is selected
- Applies the active theme CSS class

---

## Sidebar (`components/sidebar/sidebar.tsx`)

Left navigation panel showing all boards organized by category.

### Features

- **Board list** per category (system + custom)
- **Active board indicator** (highlighted item)
- **Drag-and-drop** board reordering within and across categories
- **Board search** via `<BoardsSwitcher />`
- **Settings** via `<SidebarSettingsMenu />`
- **User profile** via `<SidebarUserMenu />`
- **New board** button (opens `<BoardTemplateSelector />`)

### Child Components

- `<SidebarCategory>` — renders a single category section
- `<SidebarBoardItem>` — renders an individual board link
- `<SidebarUserMenu>` — user profile/logout
- `<SidebarSettingsMenu>` — app settings

---

## Kanban (`components/kanban/kanban.tsx`)

Kanban board view rendering cards in columns grouped by a select property.

### Props

```typescript
type Props = {
    board: Board
    activeView: BoardView
    cards: Card[]
    groupByProperty?: IPropertyTemplate
    visibleGroups: BoardGroup[]
    hiddenGroups: BoardGroup[]
    selectedCardIds: string[]
    readonly: boolean
    onCardClicked: (e: MouseEvent, card: Card) => void
    addCard: (groupByOptionId?: string, show?: boolean) => Promise<void>
    addCardFromTemplate: (cardTemplateId: string, groupByOptionId?: string) => void
    showCard: (cardId?: string) => void
    hiddenCardsCount: number
    showHiddenCardCountNotification: (show: boolean) => void
}
```

### Features

- Drag-and-drop cards between columns (react-dnd with auto-scroll)
- Column management (show/hide/reorder)
- Collapsible columns
- Per-column calculations (count, sum, average, etc.)
- "Add a card" button per column
- Default and custom card templates per column
- Hidden columns side panel

### Child Components

- `<KanbanColumn>` — Single column container
- `<KanbanColumnHeader>` — Column header with dropdown menu
- `<KanbanCard>` — Individual card tile
- `<KanbanHiddenColumnItem>` — Hidden column indicator in collapsed area

### BoardGroup type

```typescript
interface BoardGroup {
    option: IPropertyOption
    cards: Card[]
}
```

---

## Table (`components/table/table.tsx`)

Spreadsheet-style table view with rows and columns.

### Features

- Column headers with sort indicators and resize handles
- Inline property editing per cell
- Row grouping by a property (collapsible groups)
- Per-column or per-group calculation row
- Drag-and-drop row reordering
- Horizontal scroll for many columns
- Hidden cards count indicator

### Child Components

- `<TableHeaders>` — Column header row
- `<TableRows>` — All data rows (non-grouped)
- `<TableGroup>` — Grouped rows section
- `<TableGroupHeaderRow>` — Group heading row
- `<CalculationRow>` — Per-column calculation values
- `<HorizontalGrip>` — Column resize handle

---

## Gallery (`components/gallery/gallery.tsx`)

Grid layout showing card thumbnails.

### Features

- CSS Grid card layout
- Cards show icon, title, and visible properties
- Drag-and-drop between cards for manual sort
- Property value display
- Badge display (priority, assignees, etc.)

### Child Component

- `<GalleryCard>` — Individual card thumbnail

---

## Calendar (`components/calendar/fullCalendar.tsx`)

Date-based calendar view powered by FullCalendar.

### Features

- Cards appear as events on dates determined by a selected date property
- Drag-and-drop to reschedule cards
- Click an event to open card detail
- Month, week, day views

### Dependencies

- `@fullcalendar/react`
- `@fullcalendar/daygrid`
- `@fullcalendar/interaction`

---

## CardDialog (`components/cardDialog.tsx`)

Modal overlay that wraps `CardDetail`.

### Props

```typescript
type Props = {
    board: Board
    activeView: BoardView
    views: BoardView[]
    cards: Card[]
    cardId: string
    onClose: () => void
    showCard: (cardId?: string) => void
    readonly: boolean
}
```

### Features

- Opens as modal on top of current view
- Delete confirmation dialog
- Create template from card option
- `<CardActionsMenu>` — actions dropdown

### Child Components

- `<CardDetail>` — actual card content
- `<CardActionsMenu>` — card action buttons

---

## CardDetail (`components/cardDetail/cardDetail.tsx`)

Full card editing interface.

### Props

```typescript
type Props = {
    board: Board
    activeView: BoardView
    views: BoardView[]
    cards: Card[]
    card: Card
    comments: CommentBlock[]
    attachments: AttachmentBlock[]
    contents: Array<ContentBlock | ContentBlock[]>
    readonly: boolean
    onClose: () => void
    onDelete: (block: Block) => void
    addAttachment: () => void
}
```

### Features

- Editable card title (with emoji icon selector)
- Content blocks editor (drag-and-drop reordering)
- Property editor panel (`<CardDetailProperties>`)
- Comments section with add/edit/delete (`<CommentsList>`)
- Attachment list (`<AttachmentList>`)
- Breadcrumb navigation
- Keyboard: Enter to submit title, ESC to close
- Image paste support

### Child Components

| Component | Purpose |
|-----------|---------|
| `<CardDetailProperties>` | Render + edit all card properties |
| `<CardDetailContents>` | Rich content editor |
| `<CommentsList>` | Comment thread |
| `<AttachmentList>` | Attachment files |

---

## ViewHeader (`components/viewHeader/viewHeader.tsx`)

Toolbar above the board view with controls for filtering, sorting, grouping, etc.

### Features

- View name (editable for non-default views)
- **Properties** — toggle visible card properties
- **Group by** — select which property groups cards
- **Display by** — select date property for calendar
- **Sort** — add/remove/reorder sort rules
- **Filter** — add/remove filter conditions
- **Search** — filter cards by text search
- **New card** — add a new card with optional template selection
- **Actions** — view actions menu (export, duplicate, etc.)

### Child Components

| Component | Purpose |
|-----------|---------|
| `<ViewHeaderPropertiesMenu>` | Toggle property visibility |
| `<ViewHeaderGroupByMenu>` | Change group-by property |
| `<ViewHeaderDisplayByMenu>` | Change date display property |
| `<ViewHeaderSortMenu>` | Manage sort rules |
| `<ViewHeaderActionsMenu>` | Export, import, settings |
| `<FilterComponent>` | Active filters display + edit |
| `<ViewHeaderSearch>` | Text search input |
| `<NewCardButton>` | Add card / from template |

---

## BoardTemplateSelector (`components/boardTemplateSelector/boardTemplateSelector.tsx`)

Template gallery displayed when no board is selected or user clicks "Add a board".

### Features

- Lists global templates (from server) and team templates
- Preview of template contents
- Create board from template
- Create empty board
- Delete team templates
- Onboarding tour integration

### Child Components

- `<BoardTemplateSelectorItem>` — Template card in the list
- `<BoardTemplateSelectorPreview>` — Template preview panel

---

## MarkdownEditor (`components/markdownEditor.tsx`)

Rich text editor used for card title, comments, and text content blocks.

### Features

- Supports basic Markdown syntax (bold, italic, lists, code, links, headings)
- Live preview mode
- Customizable placeholder text
- `onChange` callback on every keystroke
- `onFocus` / `onBlur` callbacks

### Underlying editor

Uses a custom Markdown editor (`components/markdownEditorInput/`) with `react-contenteditable`.

---

## Dialog (`components/dialog.tsx`)

Base modal dialog wrapper used by several modals.

### Props

```typescript
type Props = {
    onClose: () => void
    toolsMenu?: JSX.Element
    hideCloseButton?: boolean
    className?: string
    title?: JSX.Element | string
    children: React.ReactNode
}
```

---

## FlashMessages (`components/flashMessages.tsx`)

Toast notification system.

### Usage

```typescript
FlashMessages.show('Message text', 'success' | 'error' | 'info', durationMs)
```

Managed through a static event emitter rather than Redux state.

---

## ConfirmationDialogBox (`components/confirmationDialogBox.tsx`)

Reusable confirmation dialog with heading, subtext, confirm + cancel buttons.

```typescript
type Props = {
    heading: string
    subText?: string
    confirmButtonText?: string
    destructive?: boolean
    onConfirm: () => void
    onClose: () => void
}
```

---

## PersonSelector (`components/personSelector.tsx`)

User search and select component used by `person` and `multiPerson` properties.

### Features

- Real-time search via `octoClient.getTeamUsers(teamId, query)`
- Avatar + name display
- Keyboard navigation
- Single or multiple selection mode
