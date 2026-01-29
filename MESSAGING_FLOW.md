# Messaging Flow – Comprehensive Documentation

This document describes the end-to-end messaging flow: backend APIs, client-side logic, data shapes, and third-party libraries.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Third-Party Libraries](#2-third-party-libraries)
3. [Backend APIs](#3-backend-apis)
4. [Frontend API Layer](#4-frontend-api-layer)
5. [Matrix Client (Real-Time)](#5-matrix-client-real-time)
6. [Client-Side Hooks](#6-client-side-hooks)
7. [Utils & Transformations](#7-utils--transformations)
8. [UI Components](#8-ui-components)
9. [Page Flows](#9-page-flows)
10. [Data Flow Summary](#10-data-flow-summary)

---

## 1. Overview

- **Protocol**: [Matrix](https://matrix.org/) with Synapse as the homeserver.
- **Backend**: NestJS app (`backend/`) proxies Matrix and stores room metadata; frontend calls backend, not Synapse directly for REST.
- **Real-time**: Frontend also runs **matrix-js-sdk** in the browser (token from backend) for live messages, presence, and read receipts.
- **Entry points**: Recruiter messages at `/messages`, company/client messages at `/company/messages`.

---

## 2. Third-Party Libraries

| Library | Version | Purpose |
|--------|---------|--------|
| **matrix-js-sdk** | ^40.0.0-rc.0 | Matrix client in the browser: sync, `Room.timeline`, presence, read receipts, room tags. |
| **axios** | ^1.11.0 | HTTP client for all backend API calls (base URL from `NEXT_PUBLIC_API_URL`). |
| **antd** | ^5.27.1 | UI: `Avatar`, `Badge`, `Button`, `Dropdown`, `Input`, `Tabs`, `Tooltip`, `message`. |
| **emoji-picker-react** | ^4.16.1 | Emoji picker in chat input. |
| **lucide-react** | ^0.555.0 | Icons (e.g. `File`, `Image`, `Download`) in `MessageItem`. |
| **iconoir-react** | ^7.11.0 | Icons in ChatView (e.g. `Star`, `Pin`, `Send`, `Megaphone`). |

Backend uses **NestJS**, **@nestjs/axios** (HttpService), and **TypeORM** (User, MatrixRoomMetadata).

---

## 3. Backend APIs

Base path: `/matrix` (mounted under the app’s API prefix, e.g. `/api/matrix`). All listed routes require **JWT** (`JwtAuthGuard`).

### 3.1 Authentication & Token

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/matrix/token` | Returns Matrix credentials for the current user. |

**Response:**  
`{ access_token, user_id, base_url }`  
Tokens are stored encrypted on `User`; backend decrypts and returns. Used by the frontend to init **matrix-js-sdk**.

---

### 3.2 Rooms

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/matrix/rooms` | List rooms the user is in (sync + metadata). |

**Backend flow:**

1. Decrypt user’s `matrix_access_token`, call Synapse `GET /_matrix/client/v3/sync` with filter (state: `m.room.name`, `m.room.topic`, `m.room.avatar`; timeline limit 1).
2. For each joined room: read name/topic from state events, last message from timeline (last `m.room.message`).
3. Load `MatrixRoomMetadata` by `room_id` (job_title, company_name, recruiter_name, client_contact_name, room_type).
4. Return enriched rooms.

**Response:**  
`{ success: true, data: { rooms: Array<{ room_id, name, topic, avatar_url, last_message, unread_count, metadata }> } }`

---

### 3.3 Messages

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/matrix/rooms/:roomId/messages` | Paginated messages for a room. |
| `POST` | `/matrix/rooms/:roomId/messages` | Send text or file message. |

**GET messages**

- Backend calls Synapse `GET /_matrix/client/v3/rooms/:roomId/messages` (dir: `b`, limit default 50).
- **Filter:** only events with `type === 'm.room.message'` (state events like `m.room.member` are dropped).
- File messages get normalized `content.url`, `content.info`.

**Response:**  
`{ success: true, data: { messages: Array<MatrixMessage> } }`  
Each message: `event_id`, `sender`, `content: { body, msgtype, url?, info? }`, `origin_server_ts`, `type`.

**POST messages (text)**

- Body: `{ message: string }`.
- Backend calls Synapse `PUT .../send/m.room.message/{txnId}` with `msgtype: 'm.text'`, `body`.

**POST messages (file)**

- Body: `{ content_uri, file_name, file_size, mimetype, msgtype?: 'm.image' | 'm.file', message? }`.
- Backend uses `sendFileMessage` (PUT with content including `url`, `info`).

**Response:**  
`{ success: true, data: { event_id } }`

---

### 3.4 File Upload & Download

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/matrix/rooms/:roomId/upload` | Upload file to Matrix media repository. |
| `GET` | `/matrix/media/convert` | Convert `mxc_url` to HTTP URL (optional). |
| `GET` | `/matrix/media/download/:serverName/:mediaId` | Proxy download from Synapse (stream). |

**Upload:** `multipart/form-data` with `file`; max size from `MATRIX_MAX_FILE_SIZE` (default 10MB). Backend calls Synapse `POST /_matrix/media/v3/upload`, returns `content_uri` (and file_name, file_size, mimetype).

**Download:** Backend uses user’s access token and calls Synapse `GET /_matrix/media/r0/download/:serverName/:mediaId`, streams response with appropriate `Content-Type` and `Content-Disposition`.

---

### 3.5 User Lookup

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/matrix/users/lookup` | Resolve Matrix user IDs to app users (name, avatar). |

**Body:** `{ matrixUserIds: string[] }`  
**Response:** `{ users: Record<matrix_user_id, { id, name, avatar }> }`  
Backend looks up by `matrix_user_id` on User and returns display name and profile photo.

---

## 4. Frontend API Layer

**File:** `app/lib/matrixApi.ts`  
Uses shared **axios** instance (base URL `NEXT_PUBLIC_API_URL`, credentials). Response is often `response.data` (interceptor).

| Function | HTTP | Purpose |
|----------|------|---------|
| `getMatrixToken()` | GET `/matrix/token` | Get access_token, user_id, base_url for SDK init. |
| `getConversations()` | GET `/matrix/rooms` | Get rooms list. |
| `getRoomMessages(roomId)` | GET `/matrix/rooms/:roomId/messages` | Get messages (array). |
| `sendMessage(roomId, message, fileAttachment?)` | POST `/matrix/rooms/:roomId/messages` | Send text or file. |
| `uploadFile(roomId, file)` | POST `/matrix/rooms/:roomId/upload` | Upload file; returns content_uri etc. |
| `lookupUsers(matrixUserIds)` | POST `/matrix/users/lookup` | Resolve sender names/avatars. |
| `downloadFile(mxcUrl)` | GET `/matrix/media/download/:serverName/:mediaId` | Download blob (extracts serverName/mediaId from mxc). |

**Types:** `MatrixRoom`, `MatrixMessage`, `UserInfo`, `FileUploadResponse` (see `matrixApi.ts`).

---

## 5. Matrix Client (Real-Time)

**File:** `app/lib/matrix-client.ts`  
Dynamically imports **matrix-js-sdk**, creates a single client, starts sync.

- **initMatrixClient(baseUrl, accessToken, userId)**  
  Creates client, `startClient({ initialSyncLimit: 10 })`, stores in module-level `matrixClient`.

- **getMatrixClient()**  
  Returns current client or null.

- **stopMatrixClient()**  
  Stops client and clears reference.

- **setupMatrixEventListeners(client, onNewMessage, onRoomUpdate)**  
  - `Room.timeline` → if event type `m.room.message`, call `onNewMessage(event, room)`.
  - `Room` → call `onRoomUpdate(room)`.

- **isDirectRoom(client, roomId)**  
  Uses account data `m.direct` or “joined members count === 2”.

- **getDirectOtherUserId(client, roomId)**  
  From `m.direct` or first joined member that is not self.

- **getRoomMemberProfile(client, roomId, userId, size)**  
  Room member display name and avatar URL (mxc); name cleaned of Matrix ID patterns.

- **mxcToHttp(client, mxcUrl, width, height)**  
  Uses client’s `mxcUrlToHttp` for avatars/thumbnails.

---

## 6. Client-Side Hooks

### 6.1 useMatrixClient

**File:** `app/hooks/useMatrixClient.ts`

- Calls `getMatrixToken()`, then `initMatrixClient(baseUrl, accessToken, userId)`.
- Calls `setupMatrixEventListeners(client, onMessage, onRoomUpdate)`.
- **Returns:** `{ initializing, matrixUserId, isReady }`.
- **Options:** `onMessage(event, room)`, `onRoomUpdate()` (e.g. refetch room list).

Used by both `/messages` and `/company/messages` to bootstrap Matrix and handle new messages / room updates.

---

### 6.2 useMatrixRooms

**File:** `app/hooks/useMatrixRooms.ts`

- `loadMatrixRooms()`: calls `getConversations()`, then for each room optionally merges unread count from `client.getRoom(room_id).getUnreadNotificationCount()`.
- **Returns:** `{ matrixRooms, loadMatrixRooms, setMatrixRooms }`.

Rooms are then transformed to `Conversation[]` in the page (see Utils).

---

### 6.3 useMatrixMessages

**File:** `app/hooks/useMatrixMessages.ts`

- **loadMessages(roomId):**
  1. `getRoomMessages(roomId)`.
  2. Filter to `type === 'm.room.message'`.
  3. `lookupUsers(senderIds)` for names/avatars.
  4. `transformMatrixMessage(...)` per message, then reverse (chronological).
  5. `setMessages(transformed)`.
  6. Optionally set read marker via client `setRoomReadMarkers`.
- **Returns:** `{ messages, setMessages, loading, loadMessages }`.

---

### 6.4 useSendMessage

**File:** `app/hooks/useSendMessage.ts`

- **send(content, file?):**
  1. If file: `uploadFile(roomId, file)` → get content_uri etc.
  2. Build optimistic `Message` (id: `local-${Date.now()}`, deliveryStatus: `'sending'`).
  3. `sendMessage(roomId, content, fileAttachment)`.
  4. Call `onMessageSent` (e.g. refresh room list); no full message refetch (avoids flicker/duplicate).
- **Returns:** `{ send, sendingMessage }`; `send` returns the optimistic message so the page can append it.

---

### 6.5 useMatrixPresence

**File:** `app/hooks/useMatrixPresence.ts`

- Subscribes to `User.presence` and `Room.receipt` for the given `roomId`.
- **Members:** from `room.getJoinedMembers()`; **admin** user (localpart `admin`) is filtered out so they never appear in the UI.
- Builds `chatMembers` (with `hideAdminInDisplayName` on names), `readReceiptsByUserId`, `readUpToEventId`.
- For DMs, sets `presenceText` from the other user’s presence (again excluding admin).
- **Returns:** `{ presenceText, chatMembers, readReceiptsByUserId, readUpToEventId }`.

---

### 6.6 useRoomTags

**File:** `app/hooks/useRoomTags.ts`

- Reads `getRoomTags(roomId)` from matrix-client account data (`m.tag`); derives `isStarred`, `isPinned`.
- **toggleRoomTag(tag, onUpdate):** `client.setRoomTag` / `deleteRoomTag` for `m.favourite` or `m.pinned`, then `onUpdate()` (e.g. refresh room list).
- **Returns:** `{ isStarred, isPinned, toggleRoomTag }`.

---

## 7. Utils & Transformations

### 7.1 conversationUtils.ts

- **filterConversationsBySearch**, **filterConversationsByTab**, **filterOutForums**, **sortConversations**  
  Filter/sort conversation lists.
- **formatLastMessagePreview(content, sender, currentUserId)**  
  Truncates and adds "You:" for own messages.
- **formatDisplayTitle(title)**  
  Replaces `↔` with ` - `, strips word "admin", trims.
- **formatRoleForumTitle(title)**  
  Normalizes "Role Forum: …" for role rooms.
- **hideAdminInDisplayName(name)**  
  Returns `""` when name is exactly `"admin"` (case-insensitive); used so admin is not shown in titles/names.

---

### 7.2 transformUtils.ts

- **transformRoomToConversation(room, currentUserId?, matrixClient?)**  
  - Uses `room.metadata` (direct_chat: job_title, recruiter_name, client_contact_name, company_name) for title/subtitle when present.
  - Else DM: other member’s name via `getRoomMemberProfile` (with hideAdmin).
  - Applies `formatDisplayTitle`, `formatRoleForumTitle`; fallback title "Untitled Room".
  - Reads tags/unread/lastMessage from matrixUtils and room.
  - **Returns:** `Conversation` (id, title, subtitle, avatar, date, unread, isStarred, isPinned, lastMessage, …).

- **transformMatrixMessage(msg, currentUserId, roomId, client?, userInfoMap?)**  
  - Maps Matrix event to UI `Message`: id, sender (name, avatar, id), time, type (text/file/link/image), content, fileUrl, fileSize, isOwn.
  - Uses **messageUtils**: `getSenderName`, `getSenderAvatar`, `getMessageType`, `convertMxcToHttp`, `formatFileSize`.

---

### 7.3 messageUtils.ts

- **cleanSenderName**, **extractSenderNameFromId**  
  Strip Matrix ID patterns from names.
- **getSenderName(senderId, roomId, client?, userInfoMap?)**  
  Member profile or user lookup; then `hideAdminInDisplayName(cleanSenderName(...))`.
- **getSenderAvatar(...)**  
  Avatar URL from room member or user info.
- **getMessageType(msg)**  
  `m.image` → image, `m.file` → file, body URL → link, else text.
- **convertMxcToHttp**, **formatFileSize**  
  Used for file/image messages.

---

### 7.4 matrixUtils.ts

- **isAdminUserId(userId)**  
  True if localpart is `"admin"` (e.g. `@admin:localhost`). Used to hide admin from members, counts, read receipts.
- **getRoomTags(roomId)**  
  From client room account data `m.tag` (e.g. m.favourite, m.pinned).
- **getRoomUnreadCount(roomId)**, **getRoomLastMessageTimestamp(roomId)**  
  From client room state.
- **isRoomStarred**, **isRoomPinned**  
  Derived from tags.

---

## 8. UI Components

### 8.1 ChatView

**File:** `app/components/messages/ChatView.tsx`

- **Props:** header (title, participantCount, …), messages, onSendMessage, loading, sendingMessage, isDirect, currentUserId, presenceText, members (ChatMember[]), readReceiptsByUserId, isStarred, isPinned, onToggleStar, onTogglePin.
- Renders header (title, member count dropdown, star/pin, mute), search, optional tabs (Announcements / Role forum / Hiring Team), message list, input (textarea + file + emoji).
- **Message list:** messages grouped by date; each group has date pill and list of **MessageItem** (with showAvatar, showSenderName, isGrouped, read state).
- **Members dropdown:** uses `members` (admin already filtered in useMatrixPresence).
- **Read/Not read dropdown:** when a message is selected, uses `selectedMessageReadState` derived from `members` and `readReceiptsByUserId` (admin excluded because members exclude admin).
- **Seen count:** `seenCountForEvent` uses same members + read receipts; status text "Sent" / "Seen" / "Seen by N".

### 8.2 MessageItem

**File:** `app/components/messages/MessageItem.tsx`

- Renders one message: avatar (left/right for own), sender name (for others), bubble (text / file / image / link), time, optional status (Sending / Sent / Seen).
- File/image: download button; uses `getMatrixClient()` and/or backend download for mxc URLs.

### 8.3 Others

- **MessagesList**, **ConversationList**, **ConversationItem**, **ConversationSearchBar**, **ConversationTabs**  
  Sidebar/list of conversations.
- **RecruiterMessagesSidebar** (`/messages`), **ActiveRolesSidebar** + list (`/company/messages`)  
  Role/conversation selection and then thread list.

---

## 9. Page Flows

### 9.1 Recruiter Messages (`/messages`)

1. **useMatrixClient**  
   Token → init SDK → setup listeners.  
   **onMessage:** if event is for selected room, transform event to Message, append/replace optimistic by server event (dedupe by event_id / content).  
   **onRoomUpdate:** loadMatrixRooms().

2. **useMatrixRooms**  
   loadMatrixRooms() on ready; rooms from GET `/matrix/rooms` + local unread.

3. **Conversations**  
   `matrixRooms` → `transformRoomToConversation(room, currentUserId, client)` → list; admin hidden in titles/names.

4. **Select conversation**  
   setSelectedMessageThread → clear messages, loadMessages(roomId).

5. **useMatrixMessages**  
   loadMessages fetches GET messages, filters m.room.message, transforms, sets messages; no refetch after send.

6. **useSendMessage**  
   onMessageSent only refreshes room list (loadMatrixRooms), not messages; optimistic message is replaced by real one when onMessage fires.

7. **useMatrixPresence**  
   chatMembers (admin filtered), readReceiptsByUserId (admin filtered), presenceText for DM other user (admin excluded).

8. **useRoomTags**  
   Star/pin for selected room.

9. **getChatHeader**  
   Participants and participantCount from joined members **with admin filtered**; title from conversation or other member name (admin hidden).

10. **ChatView**  
    Receives messages, members, readReceiptsByUserId, header (with participantCount excluding admin).

### 9.2 Company Messages (`/company/messages`)

Same hooks and flow; differences:

- Uses **ActiveRolesSidebar** and role/conversation selection.
- Conversations built from same `matrixRooms` + transformRoomToConversation.
- Same admin filtering in getChatHeader and in useMatrixPresence; same no-refetch-after-send and real-time onMessage.

---

## 10. Data Flow Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           FRONTEND (Next.js)                              │
├─────────────────────────────────────────────────────────────────────────┤
│  Pages (/messages, /company/messages)                                     │
│    ├── useMatrixClient → token, init matrix-js-sdk, onMessage/onRoomUpdate│
│    ├── useMatrixRooms → getConversations() → matrixRooms                  │
│    ├── useMatrixMessages → getRoomMessages() + transform → messages      │
│    ├── useSendMessage → uploadFile (if file) + sendMessage → optimistic   │
│    ├── useMatrixPresence → chatMembers (admin filtered), read receipts    │
│    └── useRoomTags → star/pin via client                                  │
│                                                                           │
│  matrixApi (axios)          matrix-client (matrix-js-sdk)                 │
│    GET/POST /matrix/*          Room.timeline, User.presence, Room.receipt │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    BACKEND (NestJS) – /matrix/*                           │
│  Controller: JWT → decrypt token → MatrixService → Synapse HTTP           │
│  Rooms: sync + MatrixRoomMetadata → enriched rooms                        │
│  Messages: GET (filter m.room.message), POST text/file                     │
│  Upload/Download, User lookup                                            │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    MATRIX HOMESERVER (Synapse)                           │
│  Client API: /_matrix/client/v3/... (messages, send, sync, upload)        │
│  Media: /_matrix/media/...                                                │
│  Admin API: /_synapse/admin/... (user creation, join room)                 │
└─────────────────────────────────────────────────────────────────────────┘
```

- **REST:** All message send/fetch and room list go through the backend; backend talks to Synapse with the user’s token.
- **Real-time:** Frontend Matrix client receives timeline and receipt events and updates messages/members/read state without refetching the full list after send.
- **Admin:** Admin user is hidden everywhere: titles, names, members list, participant count, read/not-read dropdown, presence (see conversationUtils, matrixUtils, useMatrixPresence, getChatHeader).

For setup (Synapse, env, testing), see **MESSAGES_SETUP.md** in the project root.
