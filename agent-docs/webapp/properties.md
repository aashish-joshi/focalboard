# Properties System

## Overview

**Directory:** `webapp/src/properties/`

Focalboard supports a rich set of card property types. Each property type has:
- A TypeScript class defining its behavior
- A display component for read-only rendering
- An edit component for value editing
- Serialization/deserialization logic

---

## Property Registry (`properties/index.ts`)

The registry maps `PropertyTypeEnum` strings to property handler classes:

```typescript
const registry: Record<PropertyTypeEnum, PropertyType> = {
    text:        new TextProperty(),
    number:      new NumberProperty(),
    select:      new SelectProperty(),
    multiSelect: new MultiSelectProperty(),
    date:        new DateProperty(),
    person:      new PersonProperty(),
    multiPerson: new MultiPersonProperty(),
    file:        new FileProperty(),
    checkbox:    new CheckboxProperty(),
    url:         new UrlProperty(),
    email:       new EmailProperty(),
    phone:       new PhoneProperty(),
    createdTime: new CreatedTimeProperty(),
    createdBy:   new CreatedByProperty(),
    updatedTime: new UpdatedTimeProperty(),
    updatedBy:   new UpdatedByProperty(),
    unknown:     new UnknownProperty(),
}
```

---

## PropertyType Base Class

All property types extend a common interface:

```typescript
abstract class PropertyType {
    type: PropertyTypeEnum
    displayName: string
    canGroup: boolean         // Can cards be grouped by this property?
    canFilter: boolean        // Can cards be filtered by this property?
    canSort: boolean          // Can cards be sorted by this property?
    canEdit: boolean          // Is the value user-editable?
    filterValueType: 'options' | 'text' | 'none'
    
    // Display component (read-only)
    displayValue(value, card, template, board): string
    
    // Edit component
    editComponent(value, card, template, board, readOnly): JSX.Element | null
    
    // Filter/sort helpers
    filterClause(template): FilterClause
    sortValue(card, template): string | number
    
    // Validation
    validateValue(value): boolean
}
```

---

## Property Types Reference

### `text` — Plain Text

- **Storage:** string
- **Display:** plain text
- **Edit:** text input
- **Group support:** no
- **Filter:** contains, not contains, is empty, is not empty, starts with, ends with

### `number` — Numeric Value

- **Storage:** string (number serialized as string)
- **Display:** formatted number
- **Edit:** number input
- **Group support:** no
- **Filter:** =, ≠, <, >, ≤, ≥, is empty, is not empty

### `select` — Single Select

- **Storage:** option ID string
- **Display:** colored badge with option value
- **Edit:** dropdown with options
- **Group support:** yes — cards can be grouped by select option
- **Filter:** includes, not includes, is empty, is not empty
- **Options:** defined in `IPropertyTemplate.options[]`

### `multiSelect` — Multi Select

- **Storage:** `string[]` (array of option IDs)
- **Display:** multiple colored badges
- **Edit:** multi-select dropdown
- **Group support:** yes
- **Filter:** includes any, includes all, not includes, is empty, is not empty

### `date` — Date or Date Range

- **Storage:** JSON string: `{ from: number, to?: number }` (ms timestamps)
- **Display:** formatted date (or date range)
- **Edit:** calendar picker with optional end date
- **Group support:** no
- **Filter:** is, is before, is after, is on or before, is on or after, is between, is empty, is not empty

### `person` — Single User

- **Storage:** user ID string
- **Display:** user avatar + name
- **Edit:** user search picker
- **Group support:** yes
- **Filter:** includes, not includes, is empty, is not empty

### `multiPerson` — Multiple Users

- **Storage:** `string[]` (array of user IDs)
- **Display:** multiple user avatars
- **Edit:** multi-user search picker
- **Group support:** yes
- **Filter:** includes, not includes, is empty, is not empty

### `file` — File/Attachment (display only)

- **Storage:** file ID string
- **Display:** file link/icon
- **Edit:** not editable as property (use attachment content block instead)
- **Group support:** no

### `checkbox` — Boolean Toggle

- **Storage:** `"true"` or `"false"` (string)
- **Display:** checkmark icon or empty
- **Edit:** toggle
- **Group support:** yes (checked / unchecked groups)
- **Filter:** is checked, is not checked

### `url` — URL Link

- **Storage:** URL string
- **Display:** clickable link
- **Edit:** text input with URL validation
- **Group support:** no
- **Filter:** contains, not contains, is empty, is not empty

### `email` — Email Address

- **Storage:** email string
- **Display:** `mailto:` link
- **Edit:** text input with email validation
- **Group support:** no
- **Filter:** contains, not contains, is empty, is not empty

### `phone` — Phone Number

- **Storage:** phone string
- **Display:** `tel:` link
- **Edit:** text input
- **Group support:** no
- **Filter:** contains, not contains, is empty, is not empty

### `createdTime` — Created Timestamp (read-only)

- **Storage:** auto-populated from `block.createAt`
- **Display:** formatted date/time
- **Edit:** read-only
- **Group support:** no

### `createdBy` — Created By User (read-only)

- **Storage:** auto-populated from `block.createdBy`
- **Display:** user avatar + name
- **Edit:** read-only
- **Group support:** no

### `updatedTime` — Last Updated Timestamp (read-only)

- **Storage:** auto-populated from `block.updateAt`
- **Display:** formatted date/time
- **Edit:** read-only
- **Group support:** no

### `updatedBy` — Last Updated By User (read-only)

- **Storage:** auto-populated from `block.modifiedBy`
- **Display:** user avatar + name
- **Edit:** read-only
- **Group support:** no

### `unknown` — Fallback Type

- Used when a property type is not recognized
- Displays raw value as text

---

## Property Value Storage Format

All property values are stored in `Card.fields.properties` (a `Record<string, any>`) keyed by the property template ID:

```typescript
card.fields.properties = {
    "prop_id_1": "option_id_abc",    // select: single option ID
    "prop_id_2": ["opt_id_1", "opt_id_2"],  // multiSelect: array of option IDs
    "prop_id_3": "2024-01-15",       // date: ISO string or JSON
    "prop_id_4": "user_id_xyz",      // person: user ID
    "prop_id_5": "42",               // number: numeric string
    "prop_id_6": "true",             // checkbox: "true"/"false"
    "prop_id_7": "https://...",      // url: string
}
```

---

## Property Display Component (`components/propertyValueElement.tsx`)

The main component that renders any property value given the template:

```tsx
<PropertyValueElement
    board={board}
    card={card}
    propertyTemplate={template}
    readOnly={readOnly}
    showEmptyPlaceholder={true}
    onCardPropertyChange={handleChange}
/>
```

- Looks up the property type from the registry
- Renders the appropriate display or edit component
- Handles value changes and dispatches mutations

---

## Adding a New Property Type

1. Create a new folder `webapp/src/properties/{type}/`
2. Implement `property.tsx` extending `PropertyType`
3. Implement the edit component
4. Add the type to `PropertyTypeEnum` in `blocks/board.ts`
5. Register the handler in `properties/index.ts`
6. Add i18n translations for the type name
