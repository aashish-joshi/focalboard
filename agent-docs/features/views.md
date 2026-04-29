# Board Views

## Overview

A **view** is a `Block` with `type = "view"` that defines how the cards in a board are displayed and organized. A board can have multiple views (e.g., one Kanban, one Table, one Calendar).

Each view stores display settings independently:
- View type (kanban/table/gallery/calendar)
- Group-by property
- Sort options
- Filters
- Visible properties
- Column widths (table)
- Card order (manual sort)

---

## View Types

| `viewType` | Description | Component |
|-----------|-------------|-----------|
| `'board'` | Kanban columns | `<Kanban />` |
| `'table'` | Spreadsheet rows | `<Table />` |
| `'gallery'` | Card grid | `<Gallery />` |
| `'calendar'` | Calendar grid | `<CalendarFullView />` |

---

## View Data Model (`blocks/boardView.ts`)

```typescript
interface BoardView extends Block {
    type: 'view'
    fields: {
        viewType: IViewType
        groupById?: string          // Property template ID to group by
        dateDisplayPropertyId?: string  // Property for calendar date
        sortOptions: ISortOption[]
        visiblePropertyIds: string[]    // Which properties to show
        visibleOptionIds: string[]      // Visible select option groups
        hiddenOptionIds: string[]       // Hidden select option groups
        collapsedOptionIds: string[]    // Collapsed groups (kanban)
        filter: FilterGroup
        cardOrder: string[]             // Manual card ordering
        columnWidths: Record<string, number>         // Table column widths
        columnCalculations: Record<string, string>   // Table column calculations
        kanbanCalculations: Record<string, KanbanCalculationFields>
        defaultTemplateId: string
    }
}
```

---

## Creating a View

```
User clicks "Add view" in ViewHeader → view type picker
  │
mutator.insertBoardView(newView)
  ├─ Generates new view block with selected viewType
  ├─ Copies visible properties from current view (if any)
  ├─ octoClient.insertBlock(boardId, view)
  ├─ Redux: updateViews([view]), setCurrent(newView.id)
  └─ URL updates to include new viewId
```

---

## Kanban View

### Grouping

Kanban groups cards by a `select` property. Each option becomes a column. An ungrouped column holds cards with no value for the group property.

### Column Management

```
ViewHeader → "Group by" → select property
  │
mutator.changeViewGroupBy(boardId, viewId, propertyId)
  │
All columns refresh based on new groupByProperty
```

### Hiding / Showing Columns

- `visibleOptionIds` — IDs of select options shown as columns
- `hiddenOptionIds` — IDs of options hidden from view

```
KanbanColumnHeader → "Hide column"
  │
mutator.hideViewColumn(boardId, view, columnOptionId)
  │
Option ID moved from visibleOptionIds → hiddenOptionIds
```

### Collapsing Columns

`collapsedOptionIds` — columns that are visible but collapsed (showing only header).

### Card Ordering Within Columns

When `sortOptions` is empty, cards follow `cardOrder`. Drag-and-drop updates:
```
mutator.changeViewCardOrder(boardId, viewId, newCardOrder)
```

### Kanban Calculations

Each column can show an aggregate calculation (count, sum, etc.):
```typescript
kanbanCalculations: {
    "opt_id_1": { calculation: "sum", propertyId: "prop_num_1" }
}
```

---

## Table View

### Column Headers

Each `IPropertyTemplate` in `visiblePropertyIds` is shown as a column. Columns can be:
- Shown/hidden via the "Properties" menu
- Resized by dragging `<HorizontalGrip>` (stored in `columnWidths`)
- Reordered via drag-and-drop in `<ViewHeaderPropertiesMenu>`

### Grouping

Table supports grouping by a select property, collapsing/expanding groups.

### Column Calculations

Per-column aggregate calculations are shown in a row at the bottom of each column:
```typescript
columnCalculations: {
    "prop_id_1": "sum",      // "count", "sum", "average", "min", "max", etc.
}
```

---

## Gallery View

### Layout

Cards are shown in a responsive CSS grid. Card display:
- Icon + title
- First image content block (if any)
- Selected visible properties

### Sorting

Gallery supports manual sort (drag-and-drop) and property-based sort.

---

## Calendar View

### Date Property

The calendar requires a `date` property selected via "Display by" in the ViewHeader. Cards appear on the day(s) from their date property value.

### Drag-to-Reschedule

Dragging an event updates the card's date property:
```
FullCalendar event drop
  │
mutator.changeBlockProperty(boardId, card, datePropertyId, newDateValue)
```

### Display

- Month grid by default
- Supports week and day views (via FullCalendar toolbar)

---

## Filtering

Views support a `FilterGroup` (AND/OR tree of `FilterClause` conditions) that hides non-matching cards. Filters are evaluated **client-side** by `cardFilter.ts`.

### Filter UI Flow

```
User clicks "Filter" in ViewHeader
  │
<FilterComponent> opens
  │
User adds/removes/modifies clauses
  │
mutator.changeViewFilter(boardId, viewId, newFilterGroup)
  │
Redux: updateViews([patchedView])
  │
getCurrentViewCardsSortedFilteredAndGrouped re-evaluates → view re-renders
```

---

## Sorting

Views support multiple sort rules applied in priority order:

```typescript
sortOptions: [
    { propertyId: "prop_priority", reversed: false },
    { propertyId: "prop_title", reversed: false }
]
```

```
User clicks column header or "Sort" menu
  │
mutator.changeViewSortOptions(boardId, viewId, newSortOptions)
```

Manual sort (drag-and-drop) clears `sortOptions` and uses `cardOrder` instead.

---

## Property Visibility

`visiblePropertyIds` is an ordered list of property template IDs shown in the view. The "Properties" menu in ViewHeader controls this.

```
mutator.changeViewVisiblePropertiesOrder(boardId, view, template, destIndex)
// OR
mutator.hideViewColumn / unhideViewColumn for kanban
```

---

## View Default Template

`defaultTemplateId` — the card template to use when "Add card" is clicked. If not set, a blank card is created.

```
User sets default template in NewCardButton menu
  │
mutator.changeViewDefaultTemplateId(boardId, view, templateId)
```
