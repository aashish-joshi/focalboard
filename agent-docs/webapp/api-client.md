# API Client (`octoClient.ts`)

## Overview

**File:** `webapp/src/octoClient.ts`

`OctoClient` is a singleton HTTP client that wraps all server API calls. It handles authentication tokens, error handling, and provides strongly-typed methods for every API endpoint.

```typescript
class OctoClient {
    serverUrl: string
    sessionId: string   // from localStorage 'focalboardSessionId'
    token: string       // current auth token
}
```

The default export is a singleton instance:
```typescript
const octoClient = new OctoClient()
export default octoClient
```

---

## Authentication

### Session Token

The token is stored in `localStorage` under key `focalboardSessionId` and sent with every request in the `X-Auth-Token` header:

```typescript
headers: {
    'X-Auth-Token': this.token,
    'X-Requested-With': 'XMLHttpRequest',
    'Content-Type': 'application/json',
}
```

### Login/Logout Methods

```typescript
login(username: string, password: string, mfaToken?: string): Promise<string | undefined>
// Returns session token string, or undefined on failure

logout(): Promise<void>

register(email: string, username: string, password: string, token?: string): Promise<{code: number, json: any}>

changePassword(userId: string, oldPassword: string, newPassword: string): Promise<{code: number, json: any}>
```

---

## User Methods

```typescript
getMe(): Promise<IUser | undefined>
// GET /api/v2/users/me

getUser(userId: string): Promise<IUser | undefined>
// GET /api/v2/users/{userId}

getUsersList(userIds: string[]): Promise<IUser[]>
// POST /api/v2/users  (body: string[])

getMyConfig(): Promise<Record<string, UserPreference> | undefined>
// GET /api/v2/users/me/config

patchUserConfig(userID: string, patch: UserConfigPatch): Promise<UserPreference[] | undefined>
// PUT /api/v2/users/{userID}/config

getTeamUsers(teamId: string, search?: string): Promise<IUser[]>
// GET /api/v2/teams/{teamId}/users[?search=...]
```

---

## Board Methods

```typescript
getBoard(boardId: string): Promise<Board | undefined>
// GET /api/v2/boards/{boardId}

getBoards(teamId?: string): Promise<Board[]>
// GET /api/v2/teams/{teamId}/boards

createBoard(board: Board): Promise<Board | undefined>
// POST /api/v2/boards

patchBoard(boardId: string, patch: BoardPatch): Promise<Board | undefined>
// PATCH /api/v2/boards/{boardId}

deleteBoard(boardId: string): Promise<void>
// DELETE /api/v2/boards/{boardId}

duplicateBoard(boardId: string, asTemplate?: boolean, toTeamID?: string): Promise<BoardsAndBlocks | undefined>
// POST /api/v2/boards/{boardId}/duplicate

getTemplates(teamId: string): Promise<Board[]>
// GET /api/v2/teams/{teamId}/templates

getBoardMetadata(boardId: string): Promise<BoardMetadata | undefined>
// GET /api/v2/boards/{boardId}/metadata
```

---

## Block Methods

```typescript
getBlocksWithBlockID(blockId: string, boardId: string): Promise<Block[]>
// GET /api/v2/boards/{boardId}/blocks?block_id={blockId}

getBlocksWithParent(boardId: string, parentId: string): Promise<Block[]>
// GET /api/v2/boards/{boardId}/blocks?parent_id={parentId}

getBlocksWithType(boardId: string, type: string): Promise<Block[]>
// GET /api/v2/boards/{boardId}/blocks?type={type}

getAllBlocks(boardId: string): Promise<Block[]>
// GET /api/v2/boards/{boardId}/blocks?all=true

insertBlock(boardId: string, block: Block, sourceBoardID?: string): Promise<Block[] | undefined>
// POST /api/v2/boards/{boardId}/blocks

insertBlocks(boardId: string, blocks: Block[], sourceBoardID?: string): Promise<Block[] | undefined>
// POST /api/v2/boards/{boardId}/blocks  (body: Block[])

patchBlock(boardId: string, blockId: string, patch: BlockPatch): Promise<void>
// PATCH /api/v2/boards/{boardId}/blocks/{blockId}

patchBlocks(boardId: string, blockIds: string[], patches: BlockPatch[]): Promise<void>
// PATCH /api/v2/boards/{boardId}/blocks  (body: { blockIds, blockPatches })

deleteBlock(boardId: string, blockId: string): Promise<void>
// DELETE /api/v2/boards/{boardId}/blocks/{blockId}

undeleteBlock(boardId: string, blockId: string): Promise<Block | undefined>
// POST /api/v2/boards/{boardId}/blocks/{blockId}/undelete

duplicateBlock(boardId: string, blockId: string, asTemplate?: boolean): Promise<Block[] | undefined>
// POST /api/v2/boards/{boardId}/blocks/{blockId}/duplicate
```

---

## Card Methods (V3 API)

```typescript
createCard(boardId: string, card: Card, sourceBoardID?: string): Promise<Card | undefined>
// POST /api/v2/boards/{boardId}/cards

getCard(cardId: string): Promise<Card | undefined>
// GET /api/v2/cards/{cardId}

patchCard(cardId: string, cardPatch: CardPatch): Promise<Card | undefined>
// PATCH /api/v2/cards/{cardId}

getCardsByBoard(boardId: string, page?: number, perPage?: number): Promise<Card[]>
// GET /api/v2/boards/{boardId}/cards[?page=n&per_page=m]
```

---

## Board Member Methods

```typescript
getBoardMembers(teamId: string, boardId: string): Promise<BoardMember[]>
// GET /api/v2/boards/{boardId}/members

addBoardMember(member: BoardMember): Promise<BoardMember | undefined>
// POST /api/v2/boards/{boardId}/members

updateBoardMember(member: BoardMember): Promise<BoardMember | undefined>
// PUT /api/v2/boards/{boardId}/members/{userId}

deleteBoardMember(member: BoardMember): Promise<void>
// DELETE /api/v2/boards/{boardId}/members/{userId}
```

---

## Boards and Blocks (Atomic)

```typescript
insertBoardsAndBlocks(boardsAndBlocks: BoardsAndBlocks): Promise<BoardsAndBlocks | undefined>
// POST /api/v2/boards-and-blocks

patchBoardsAndBlocks(patch: PatchBoardsAndBlocks): Promise<BoardsAndBlocks | undefined>
// PATCH /api/v2/boards-and-blocks

deleteBoardsAndBlocks(boardIds: string[], blockIds: string[]): Promise<void>
// DELETE /api/v2/boards-and-blocks
```

---

## Sharing Methods

```typescript
getSharing(boardId: string): Promise<Sharing | undefined>
// GET /api/v2/boards/{boardId}/sharing

setSharing(boardId: string, sharing: Sharing): Promise<void>
// POST /api/v2/boards/{boardId}/sharing
```

---

## Category Methods

```typescript
getTeamCategories(teamId: string): Promise<CategoryBoards[] | undefined>
// GET /api/v2/teams/{teamId}/categories

createCategory(category: Category): Promise<Category | undefined>
// POST /api/v2/teams/{teamId}/categories

updateCategory(category: Category): Promise<Category | undefined>
// PUT /api/v2/teams/{teamId}/categories/{categoryId}

deleteCategory(teamId: string, categoryId: string): Promise<void>
// DELETE /api/v2/teams/{teamId}/categories/{categoryId}

updateCategoryBoard(teamId: string, categoryId: string, boardIds: string[]): Promise<void>
// POST /api/v2/teams/{teamId}/categories/{categoryId}/boards

updateCategoryBoardsOrder(teamId: string, categoryId: string, boardsOrder: string[]): Promise<void>
// PUT /api/v2/teams/{teamId}/categories/{categoryId}/boards

updateCategoriesOrder(teamId: string, categoriesOrder: string[]): Promise<void>
// PUT /api/v2/teams/{teamId}/categories
```

---

## File Methods

```typescript
uploadFile(boardId: string, file: File): Promise<string | undefined>
// POST /api/v2/teams/{teamId}/boards/{boardId}/files
// Returns: { fileId: "..." }

getFileAsUrl(filename: string, boardId: string): string
// Returns the URL to download a file: /api/v2/files/teams/{teamId}/{boardId}/{filename}
```

---

## Subscription Methods

```typescript
createSubscription(subscription: Subscription): Promise<Subscription | undefined>
// POST /api/v2/subscriptions

deleteSubscription(blockId: string, subscriberType: string, subscriberId: string): Promise<void>
// DELETE /api/v2/subscriptions/{blockId}/{subscriberId}

getUserBlockSubscriptions(userId: string): Promise<Subscription[]>
// GET /api/v2/subscriptions/{userId}
```

---

## Search Methods

```typescript
searchBoards(teamId: string, query: string): Promise<Board[]>
// GET /api/v2/teams/{teamId}/boards/search?q={query}

searchBlocks(teamId: string, boardId: string, query: string): Promise<Block[]>
// GET /api/v2/boards/{boardId}/blocks/search?q={query}

searchBoardsAndBlocks(teamId: string, query: string): Promise<{ boards: Board[], cards: Card[] }>
// GET /api/v2/search?q={query}&team_id={teamId}
```

---

## System / Config Methods

```typescript
getClientConfig(): Promise<ClientConfig | undefined>
// GET /api/v2/clientConfig

getTeam(teamId: string): Promise<Team | undefined>
// GET /api/v2/teams/{teamId}

getAllTeams(): Promise<Team[]>
// GET /api/v2/teams

getChannel(teamId: string, channelId: string): Promise<Channel | undefined>
// GET /api/v2/teams/{teamId}/channels/{channelId}

searchTeamChannels(teamId: string, query: string): Promise<Channel[]>
// GET /api/v2/teams/{teamId}/channels?search={query}
```

---

## Onboarding

```typescript
prepareOnboarding(teamId: string): Promise<{ teamID: string, boardID: string } | undefined>
// POST /api/v2/teams/{teamId}/onboard
```

---

## Archive Methods

```typescript
exportBoardArchive(boardID: string): Promise<Response>
// POST /api/v2/boards/{boardId}/archive/export

importArchive(teamID: string, file: File): Promise<Response>
// POST /api/v2/boards/archive/import
```

---

## Error Handling

All methods follow the pattern:
```typescript
const response = await fetch(url, options)
if (response.ok) {
    return await response.json()
}
// Log error
return undefined
```

Non-2xx responses are logged and `undefined` (or empty array) is returned. The calling component must handle `undefined` results.

---

## Token Management

```typescript
setToken(token: string): void  // stores to localStorage
getToken(): string             // reads from localStorage

// The token is automatically included in all request headers
```

---

## Base URL

The client supports dynamic server URLs for testing and multi-instance setups:
```typescript
constructor(serverUrl?: string)
// Default: window.location.origin (same origin)
```
