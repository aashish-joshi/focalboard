# Webapp Overview

## Tech Stack

| Technology | Purpose |
|-----------|---------|
| React 17 | UI component framework |
| TypeScript | Type-safe JavaScript |
| Redux Toolkit | Global state management |
| `react-intl` | Internationalization (i18n) |
| `react-dnd` | Drag and drop (HTML5 + touch) |
| `@fullcalendar/react` | Calendar view |
| Webpack 5 | Build tool |
| Jest + Enzyme | Unit tests |
| Cypress | End-to-end tests |
| SCSS | CSS pre-processor |

---

## Directory Structure

```
webapp/src/
├── main.tsx                  # Root React entry point
├── app.tsx                   # Main app component (i18n, DnD, Router)
├── router.tsx                # Route definitions
├── octoClient.ts             # HTTP API client
├── mutator.ts                # Undo/redo mutation layer
├── wsclient.ts               # WebSocket client
├── undomanager.ts            # Undo/redo stack
├── store/                    # Redux Toolkit slices
├── blocks/                   # TypeScript block type definitions
├── components/               # React UI components
├── properties/               # Card property type system
├── pages/                    # Top-level page components
├── hooks/                    # Custom React hooks
├── i18n/                     # Translation JSON files
├── styles/                   # Global SCSS styles
├── svg/                      # SVG icon assets
├── telemetry/                # Client-side telemetry
├── utils.ts                  # Utility functions
├── boardUtils.ts             # Board-specific utilities
├── cardFilter.ts             # Card filtering logic
├── theme.ts                  # Theme definitions
├── constants.ts              # App-wide constants
└── types/                    # Shared TypeScript type declarations
```

---

## Entry Point: `main.tsx`

The application entry point. Renders into the `#focalboard-app` DOM element.

Key responsibilities:
- Wraps app with Redux `<Provider store={store}>`
- Initializes emoji-mart emoji handling via `UserSettings`
- Calls `initThemes()` to register CSS variables for themes
- Renders `<MainApp />` which is `<App />` wrapped with `<WithWebSockets />`

---

## Root Component: `app.tsx`

```tsx
function App({ history }: Props) {
    // On mount:
    // 1. dispatch(fetchMe())           - load current user
    // 2. fetch language preference
    // 3. dispatch(fetchClientConfig())  - load server config

    return (
        <IntlProvider locale={language} messages={messages}>
            <DndProvider backend={isMobile ? TouchBackend : HTML5Backend}>
                <FlashMessages milliseconds={2000} />
                <NewVersionBanner />
                <FocalboardRouter history={history} />
            </DndProvider>
        </IntlProvider>
    )
}
```

---

## Router: `router.tsx`

Uses React Router v5. Route definitions:

| Route | Component | Notes |
|-------|-----------|-------|
| `/error` | `ErrorPage` | Generic error display |
| `/login` | `LoginPage` | Login form |
| `/register` | `RegisterPage` | Registration form |
| `/change_password` | `ChangePasswordPage` | Password change form |
| `/team/:teamId/new/:channelId` | `BoardPage` | Create board linked to channel |
| `/team/:teamId/shared/:boardId?/:viewId?/:cardId?` | `BoardPage` | Shared read-only board |
| `/shared/:boardId?/:viewId?/:cardId?` | `BoardPage` | Shared board (no team) |
| `/board/:boardId?/:viewId?/:cardId?` | `BoardPage` | Personal board |
| `/team/:teamId/:boardId?/:viewId?/:cardId?` | `BoardPage` | Team board |

**Helpers:**
- `GlobalErrorRedirect` — redirects to `/error` when Redux `globalError` is set
- `WorkspaceToTeamRedirect` — handles legacy `/workspace/...` URLs

---

## Pages (`webapp/src/pages/`)

| Page | Component | Description |
|------|-----------|-------------|
| `boardPage/boardPage.tsx` | `BoardPage` | Main board page; loads board data and renders workspace |
| `loginPage.tsx` | `LoginPage` | Login form |
| `registerPage.tsx` | `RegisterPage` | Registration form |
| `changePasswordPage.tsx` | `ChangePasswordPage` | Password change form |
| `errorPage.tsx` | `ErrorPage` | Error display |

### BoardPage

The most important page. On mount:
1. Reads `teamId`, `boardId`, `viewId`, `cardId` from URL params
2. Dispatches `fetchSidebarCategories(teamId)` for sidebar
3. Dispatches board loading (via `octoClient`)
4. Subscribes to WebSocket team events
5. Renders `<Workspace />` once data is loaded

---

## Workspace Component (`components/workspace.tsx`)

Renders the main two-panel layout:
- Left: `<Sidebar />`
- Right: `<CenterPanel />` (when board is loaded)

Also renders `<BoardTemplateSelector />` when no board is selected.

---

## Build Configuration

### Webpack

| File | Config |
|------|--------|
| `webpack.common.js` | Shared config: TypeScript loader, SCSS, asset imports |
| `webpack.dev.js` | Dev server with hot reload, source maps |
| `webpack.prod.js` | Production build with minification |
| `webpack.editor.js` | Separate editor bundle (for plugin embedding) |

### TypeScript

- `tsconfig.json` — strict mode enabled, `jsx: react`
- Target: `ES2017`

### Testing

- Jest configuration in `package.json`
- `__mocks__/` — manual mocks for file imports
- `testUtils.tsx` — test utilities (fake store, fake intl)
- Enzyme for component testing

---

## Internationalization (`webapp/src/i18n/`)

Supported languages (16):

| Code | Language |
|------|---------|
| `en` | English (default) |
| `de` | German |
| `es` | Spanish |
| `fr` | French |
| `it` | Italian |
| `ja` | Japanese |
| `nl` | Dutch |
| `pt-br` | Portuguese (Brazil) |
| `ru` | Russian |
| `sv` | Swedish |
| `tr` | Turkish |
| `zh-cn` | Chinese Simplified |
| `zh-tw` | Chinese Traditional |
| `ca` | Catalan |
| `el` | Greek |
| `id` | Indonesian |
| `oc` | Occitan |

Messages are loaded per language from `webapp/src/i18n/{lang}.json` files containing key-value pairs.

---

## Theme System (`theme.ts`)

Focalboard supports multiple color themes applied via CSS variables.

```typescript
type Theme = {
    mainBg: string
    sidebarBg: string
    centerBg: string
    sidebarText: string
    mainText: string
    buttonBg: string
    buttonText: string
    // ... more variables
}
```

Themes available: Default, Dark, Light, System.
The selected theme is stored in `localStorage` under `theme`.

`initThemes()` registers all themes and applies the current one on startup.

---

## Constants (`constants.ts`)

Frequently used across the app:

```typescript
const EMPTY_GUID = '00000000-0000-0000-0000-000000000000'
const KeyCodes = {
    ENTER: ['Enter', 13],
    ESCAPE: ['Escape', 27],
    DELETE: ['Delete', 46],
    BACKSPACE: ['Backspace', 8],
    // ...
}
```

---

## Utilities

### `utils.ts`
- `createGuid()` — UUID v4
- `getDefaultLanguage()` — detect browser language
- `formatTimestamp(ts)` — format millisecond timestamps
- `isSmallScreen()` — detect mobile viewport

### `boardUtils.ts`
- `sortBoardViewsAlphabetically(views)` — sort view list
- `isBoardType(block)` — type guard
- `showBoard(boardId, viewId, cardId, history)` — navigate to board

### `cardFilter.ts` / `cardFilter.test.ts`
- Evaluates `FilterGroup` against a `Card`
- `FilterGroup` contains `FilterClause` items connected by AND/OR
- Each clause tests a property value against a condition (is, contains, starts with, etc.)
- Used in all four view types to filter visible cards

### `octoUtils.tsx`
- `OctoUtils.hydrateBlock(block)` — cast raw JSON to typed block
- `OctoUtils.getBlockOrder(...)` — compute display order for cards in a view
- Property display utilities

### `mutator.test.ts`, `octoClient.test.ts`, `utils.test.ts`
Unit tests for the corresponding modules.
