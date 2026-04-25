# Redux State Management

## Overview

**Directory:** `webapp/src/store/`

The Redux store uses Redux Toolkit (`@reduxjs/toolkit`) with `configureStore`. All state slices use `createSlice` and selectors use `createSelector` (from `reselect`).

---

## Store Configuration (`store/index.ts`)

```typescript
const store = configureStore({
  reducer: {
    users,
    teams,
    channels,
    language,
    globalTemplates,
    boards,
    views,
    cards,
    contents,
    comments,
    searchText,
    globalError,
    clientConfig,
    sidebar,
    limits,
    attachments,
  }
})

export type RootState = ReturnType<typeof store.getState>
export type AppDispatch = typeof store.dispatch
```

---

## Typed Hooks (`store/hooks.ts`)

```typescript
export const useAppDispatch = () => useDispatch<AppDispatch>()
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector
```

Always use these typed hooks instead of the raw Redux hooks.

---

## Slice: `users` (`store/users.ts`)

### State

```typescript
interface UsersState {
  me: IUser | null
  boardUsers: Record<string, IUser>
  loggedIn: boolean | null
  blockSubscriptions: Subscription[]
  myConfig: Record<string, UserPreference>
}
```

### Async Thunks

| Thunk | Description |
|-------|-------------|
| `fetchMe()` | GET `/api/v2/users/me` + GET `/api/v2/users/me/config`. Sets `me`, `loggedIn`, `myConfig`. |
| `fetchUserBlockSubscriptions(userID)` | GET `/api/v2/subscriptions/{userId}`. Sets `blockSubscriptions`. |

### Actions

| Action | Description |
|--------|-------------|
| `setMe(user)` | Set current user |
| `setBoardUsers(users)` | Replace board users map |
| `addBoardUsers(users)` | Merge users into board users map |
| `removeBoardUsersById(userIDs)` | Remove users from board users map |
| `followBlock(subscription)` | Add to `blockSubscriptions` |
| `unfollowBlock({ blockId })` | Remove from `blockSubscriptions` |
| `patchProps(prefs)` | Merge preferences into `myConfig` |

### Selectors

```typescript
getMe(state): IUser | null
getLoggedIn(state): boolean | null
getMyId(state): string | undefined
getBoardUsers(state): Record<string, IUser>
getBoardUsersList(state): IUser[]
getUser(state, userId): IUser | undefined
getBlockSubscription(state, blockId, subscriberType): Subscription | undefined
getMyConfig(state): Record<string, UserPreference>
```

---

## Slice: `boards` (`store/boards.ts`)

### State

```typescript
interface BoardsState {
  current: string
  loadingBoard: boolean
  boards: Record<string, Board>
  templates: Record<string, Board>
  membersInBoards: Record<string, Record<string, BoardMember>>
  myBoardMemberships: Record<string, BoardMember>
}
```

### Async Thunks

| Thunk | Description |
|-------|-------------|
| `fetchBoardMembers(teamId, boardId)` | Fetches members, then user details for each member. Dispatches `addBoardUsers` and `setBoardMembersForBoard`. |
| `updateMembersEnsuringBoardsAndUsers(members)` | After a WS member update, ensures related boards and users are loaded in state. |

### Actions

| Action | Description |
|--------|-------------|
| `setCurrent(boardId)` | Set active board ID |
| `setLoadingBoard(loading)` | Set loading state |
| `updateBoards(boards)` | Upsert boards into map |
| `deleteBoards(boardIds)` | Remove boards from map |
| `updateMembersEnsuringBoardsAndUsers` | (thunk action) |
| `setBoardMembersForBoard({ boardId, members })` | Set members for a board |
| `updateBoardMembers(members)` | Upsert individual members |
| `deleteBoardMember({ boardId, userId })` | Remove a member |
| `updateMyBoardMemberships(memberships)` | Update current user's memberships |
| `addMyBoardMemberships(memberships)` | Add to current user's memberships |
| `deleteMyBoardMemberships(memberships)` | Remove from current user's memberships |

### Selectors

```typescript
getCurrentBoard(state): Board | undefined
getCurrentBoardId(state): string
getBoard(state, boardId): Board | undefined
getBoards(state): Record<string, Board>
getMySortedBoards(state): Board[]     // sorted alphabetically
getTemplates(state): Record<string, Board>
getMyBoardMembership(state, boardId): BoardMember | undefined
getBoardMembersList(state, boardId): BoardMember[]
```

---

## Slice: `views` (`store/views.ts`)

### State

```typescript
interface ViewsState {
  current: string
  views: Record<string, BoardView>
}
```

### Smart Update

`smartViewUpdate()` is a helper that only updates view fields that have actually changed, preventing unnecessary re-renders.

### Actions

| Action | Description |
|--------|-------------|
| `setCurrent(viewId)` | Set active view ID |
| `updateViews(views)` | Upsert views (with smart change detection) |
| `updateView(view)` | Upsert single view |
| `deleteViews(viewIds)` | Remove views from map |
| `setCollapsedCategories(ids)` | Set collapsed group IDs |

### Selectors

```typescript
getCurrentView(state): BoardView | undefined
getCurrentViewId(state): string
getView(state, viewId): BoardView | undefined
getCurrentBoardViews(state): BoardView[]
getCurrentViewGroupBy(state): IPropertyTemplate | undefined
getCurrentViewDisplayBy(state): IPropertyTemplate | undefined
getCurrentViewProperties(state): IPropertyTemplate[]
```

---

## Slice: `cards` (`store/cards.ts`)

### State

```typescript
interface CardsState {
  current: string
  limitTimestamp: number
  cards: Record<string, Card>
  templates: Record<string, Card>
  cardHiddenWarning: boolean
}
```

### Async Thunks

| Thunk | Description |
|-------|-------------|
| `refreshCards(timestamp)` | Reload cards that were hidden due to cloud card limit |

### Actions

| Action | Description |
|--------|-------------|
| `setCurrent(cardId)` | Set active card ID |
| `updateCards(cards)` | Upsert cards |
| `deleteCards(cardIds)` | Remove cards |
| `addCard(card)` | Add single card |
| `addTemplate(card)` | Add card template |
| `updateTemplates(cards)` | Upsert card templates |
| `setLimitTimestamp(ts)` | Set cloud limit timestamp |
| `showCardHiddenWarning(show)` | Show/hide hidden cards warning |

### Selectors

```typescript
getCard(state, cardId): Card | undefined
getCurrentCard(state): Card | undefined
getSortedCards(state): Card[]
getCurrentBoardCards(state): Card[]                           // all cards in current board
getCurrentBoardTemplates(state): Card[]                       // card templates in current board
getCurrentViewCardsSortedFilteredAndGrouped(state): Card[][]  // cards after filtering, sorting, grouping
getCardLimitTimestamp(state): number
getCurrentBoardHiddenCardsCount(state): number
```

The `getCurrentViewCardsSortedFilteredAndGrouped` selector is the most important — it computes the final display list used by Kanban/Table/Gallery/Calendar views.

---

## Slice: `contents` (`store/contents.ts`)

### State

```typescript
interface ContentsState {
  contents: Record<string, ContentBlock>
}
```

### Actions

| Action | Description |
|--------|-------------|
| `updateContents(contents)` | Upsert content blocks |
| `deleteContents(contentIds)` | Remove content blocks |

### Selectors

```typescript
getCardContents(state, cardId): Array<ContentBlock | ContentBlock[]>
// Returns ordered content blocks for a card, respecting contentOrder from the card
```

---

## Slice: `comments` (`store/comments.ts`)

### State

```typescript
interface CommentsState {
  comments: Record<string, CommentBlock>
}
```

### Actions

| Action | Description |
|--------|-------------|
| `updateComments(comments)` | Upsert comment blocks |
| `deleteComments(commentIds)` | Remove comment blocks |

### Selectors

```typescript
getCardComments(state, cardId): CommentBlock[]
// Returns comments for a card, sorted by createAt ascending
```

---

## Slice: `attachments` (`store/attachments.ts`)

### State

```typescript
interface AttachmentsState {
  attachments: Record<string, AttachmentBlock>
  uploadPercent: Record<string, number>
}
```

### Actions

| Action | Description |
|--------|-------------|
| `updateAttachments(attachments)` | Upsert attachment blocks |
| `deleteAttachments(ids)` | Remove attachment blocks |
| `updateUploadPercent({ blockId, percent })` | Track upload progress (0–100) |

### Selectors

```typescript
getCardAttachments(state, cardId): AttachmentBlock[]
```

---

## Slice: `sidebar` (`store/sidebar.ts`)

### State

```typescript
interface SidebarState {
  categoryAttributes: CategoryBoards[]
  hiddenBoardIDs: string[]
}
```

### Async Thunks

| Thunk | Description |
|-------|-------------|
| `fetchSidebarCategories(teamID)` | GET `/api/v2/teams/{teamId}/categories`. Updates `categoryAttributes`. |

### Actions

| Action | Description |
|--------|-------------|
| `updateCategories(categories)` | Replace category list |
| `updateBoardCategories(categoryBoards)` | Update board-category associations |
| `addBoardToCategory({ boardId, categoryId })` | Move board to category |
| `updateCategoryBoardsOrder({ categoryId, boardsOrder })` | Reorder boards in a category |
| `updateCategoryOrder(categoryOrder)` | Reorder categories |
| `hiddenBoardsInCategory(boardIds)` | Mark boards as hidden |
| `unhideBoardsFromCategory(boardIds)` | Un-hide boards |

### Selectors

```typescript
getSidebarCategories(state): CategoryBoards[]
getHiddenBoardIDs(state): string[]
getCategoryOfBoard(state, boardId): CategoryBoards | undefined
```

---

## Slice: `teams` (`store/teams.ts`)

### State

```typescript
interface TeamsState {
  current: string | null
  allTeams: Team[]
}
```

### Actions

```typescript
setTeam(team: Team)
setAllTeams(teams: Team[])
```

### Selectors

```typescript
getCurrentTeam(state): Team | undefined
getAllTeams(state): Team[]
```

---

## Slice: `limits` (`store/limits.ts`)

Tracks cloud plan limits for boards.

### State

```typescript
interface LimitsState {
  limits: BoardsCloudLimits
}
```

### Actions / Selectors

```typescript
setLimits(limits: BoardsCloudLimits)
getLimits(state): BoardsCloudLimits
```

---

## Slice: `clientConfig` (`store/clientConfig.ts`)

Holds the `ClientConfig` received from `/api/v2/clientConfig`.

### Selectors

```typescript
getClientConfig(state): ClientConfig
getTelemetryEnabled(state): boolean
getFeatureFlags(state): Record<string, string>
```

---

## Slice: `searchText` (`store/searchText.ts`)

Simple string state for the board search input.

```typescript
setSearchText(text: string)
getSearchText(state): string
```

---

## Slice: `globalError` (`store/globalError.ts`)

```typescript
setGlobalError(message: string)
getGlobalError(state): string
```

When set, `GlobalErrorRedirect` navigates to `/error`.

---

## Slice: `language` (`store/language.ts`)

```typescript
setLanguage(lang: string)
getCurrentLanguage(state): string
```

---

## Slice: `globalTemplates` (`store/globalTemplates.ts`)

```typescript
setGlobalTemplates(templates: Board[])
getGlobalTemplates(state): Board[]
```

---

## Slice: `channels` (`store/channels.ts`)

Mattermost channel data (available in plugin mode).

```typescript
setChannels(channels: Channel[])
getChannel(state, channelId): Channel | undefined
```
