# Site AI Chat (Chat Subjects)

## Overview

Chat subjects let visitors of a site open an AI chat. Each subject is defined per realm by the realm's administrators and has its own prompt, model, tools and limits. A subject can allow anonymous visitors, or require them to be logged in.

The AI can be given tools to look up the shop:

- `catalog`: search products, get product details, list the fields usable as search filters
- `cart`: view the visitor's cart, add products, change quantities

Tools act on the visitor's own context (realm, language, currency and session cart), exactly as if the visitor had made the catalog or cart API calls themselves. Checkout stays with the visitor: the AI can fill the cart, but cannot place orders.

## Endpoints

| Method | Path | Access | Description |
|---|---|---|---|
| GET | `AI/Chat/Subject` | public | Enabled subjects of the current site |
| GET | `AI/Chat/Subject/<id>` | public | One subject |
| POST | `AI/Chat/Subject/<id>:createChat` | see below | Create a chat for the current visitor |
| POST | `AI/Chat/<id>:ask` | chat owner | Send a message, returns once the AI has answered |
| GET | `AI/Chat/<id>/History` | chat owner | Messages of the chat |
| GET | `AI/Chat/Subject?Realm__=<realm>` | realm admin | All subjects of a realm |
| POST | `AI/Chat/Subject` | realm admin | Create a subject (with `Realm__`) |
| PATCH | `AI/Chat/Subject/<id>` | realm admin | Update a subject |
| DELETE | `AI/Chat/Subject/<id>` | realm admin | Delete a subject |

Realm admin means admin (`A`) access on the subject's realm, see [Access Levels](access_levels.md).

## Subject Fields

| Field | Description |
|---|---|
| `Name` | Subject name, shown to visitors |
| `Description` | Optional description, shown to visitors |
| `System_Prompt` | Instructions for the AI, added after a base prompt naming the site and telling the AI to rely on its tools. Only returned to realm admins |
| `Model` | Model as `provider@model`, empty for the platform default |
| `Allow_Anonymous` | `Y` to let visitors chat without logging in (default `N`) |
| `Tools` | Comma separated set of `catalog`, `cart` (default `catalog`) |
| `Daily_Chats` | Chats a visitor may create per day (default 20) |
| `Daily_Messages` | Messages a visitor may send per day (default 200) |
| `Enabled` | `Y`/`N`, disabled subjects are hidden and refuse new chats and messages |

Daily limits apply per logged in user, or per IP address for anonymous visitors.

## Implementing a Chat on a Site

All visitor calls must be made on the site's own domain (the subject's realm): the AI's tools use that request's realm, language, currency and session. Calls made from another realm are refused.

### 1. List subjects

```javascript
const subjects = await rest('AI/Chat/Subject', 'GET');
// subjects.data: [{AI_Chat_Subject__, Name, Description, Allow_Anonymous, Tools, ...}]
```

If a subject has `Allow_Anonymous` set to `N`, ask the visitor to log in first (see [User Flow](userflow.md)), otherwise `createChat` fails with an authentication error.

### 2. Create a chat

```javascript
const chat = await rest(`AI/Chat/Subject/${subjectId}:createChat`, 'POST');
const chatId = chat.data.AI_Chat__;
```

The chat belongs to the visitor: their user account when logged in, otherwise their session. Store `chatId` client side (for example in session storage) to resume the conversation. An anonymous chat is only reachable from the session that created it.

### 3. Send messages

```javascript
const res = await rest(`AI/Chat/${chatId}:ask`, 'POST', { message: 'Do you have red sneakers in size 42?' });
```

The call returns once the AI has answered, which can take several seconds when tools are used. File attachments are not supported on subject chats.

If the AI changed the cart (`cart` tool set), refresh any cart display after each answer, for example with `GET Catalog/Cart/@`.

### 4. Display the conversation

```javascript
const history = await rest(`AI/Chat/${chatId}/History`, 'GET', { sort: 'Index', results_per_page: 100 });
// history.data: [{Role: 'user'|'assistant', Message, Date, Status, Index, ...}]
```

Results are paginated (20 per page by default, 100 max). Only show messages with `Status` = `final`. The system prompt and tool traffic are not listed.

## Errors

| Situation | Error |
|---|---|
| Subject disabled or deleted | Not found |
| Call made on another realm | Access denied |
| Login required (`Allow_Anonymous` = `N`) | Authentication required |
| Daily limit reached | Rate limit exceeded |
| Too many tool calls for a single message (the AI looped) | Error, the visitor can rephrase and retry |

## Managing Subjects (Admin)

```javascript
// create
await rest('AI/Chat/Subject', 'POST', {
  Realm__: realmId,
  Name: 'Shopping assistant',
  Description: 'Ask us about our products',
  System_Prompt: 'Help visitors find products matching their needs. Be concise.',
  Allow_Anonymous: 'Y',
  Tools: 'catalog,cart',
});

// update
await rest(`AI/Chat/Subject/${subjectId}`, 'PATCH', { Enabled: 'N' });
```

Changing `Model` only applies to chats created afterwards. Deleting a subject disables its existing chats.
