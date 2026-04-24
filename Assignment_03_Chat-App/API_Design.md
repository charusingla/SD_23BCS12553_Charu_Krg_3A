# API Design — Real-Time Chat System

All REST endpoints are prefixed with `/api/v1`.  
WebSocket endpoint: `wss://chat.example.com/ws`.  
Authentication: Bearer JWT in `Authorization` header (REST) or initial WebSocket handshake.

---

## 1. Authentication APIs

### POST /auth/register
Register a new user.

**Request Body**
```json
{
  "phone": "+919876543210",
  "display_name": "Riya Sharma",
  "password": "hashed_client_side",
  "device_id": "uuid-v4",
  "public_key": "<base64-encoded-identity-key>"
}
```

**Response** `201 Created`
```json
{
  "user_id": "usr_01J5XK...",
  "access_token": "<jwt>",
  "refresh_token": "<opaque-token>",
  "expires_in": 3600
}
```

---

### POST /auth/login
```json
// Request
{ "phone": "+919876543210", "password": "...", "device_id": "uuid-v4" }

// Response 200
{ "access_token": "...", "refresh_token": "...", "expires_in": 3600 }
```

### POST /auth/refresh
```json
// Request
{ "refresh_token": "..." }
// Response 200
{ "access_token": "...", "expires_in": 3600 }
```

### POST /auth/logout
Revokes the refresh token for the current device.

---

## 2. User APIs

### GET /users/me
Returns the authenticated user's profile.

**Response** `200 OK`
```json
{
  "user_id": "usr_01J5XK...",
  "phone": "+919876543210",
  "display_name": "Riya Sharma",
  "avatar_url": "https://cdn.example.com/avatars/...",
  "status": "Hey there!",
  "last_seen": "2025-09-01T10:22:00Z",
  "online": true
}
```

### PUT /users/me
Update profile (display_name, avatar, status, last_seen_visibility).

### GET /users/search?q={query}&limit=20
Search users by name or phone number.

### GET /users/{user_id}
Fetch another user's public profile.

### POST /users/{user_id}/block
### DELETE /users/{user_id}/block

---

## 3. Conversation APIs

### GET /conversations?cursor={cursor}&limit=20
List conversations (1:1 and groups), sorted by last message timestamp (descending), paginated with a cursor.

**Response** `200 OK`
```json
{
  "conversations": [
    {
      "conversation_id": "conv_...",
      "type": "direct",
      "participants": ["usr_A", "usr_B"],
      "last_message": {
        "message_id": "msg_...",
        "content_type": "text",
        "preview": "See you tomorrow!",
        "sender_id": "usr_A",
        "sent_at": "2025-09-01T10:00:00Z"
      },
      "unread_count": 3
    }
  ],
  "next_cursor": "eyJsYXN0..."
}
```

### POST /conversations
Create a new 1:1 conversation.
```json
// Request
{ "participant_id": "usr_B" }
// Response 201 — returns conversation object
```

### GET /conversations/{conversation_id}
Get conversation metadata.

### DELETE /conversations/{conversation_id}
Delete conversation for the caller only (soft delete).

---

## 4. Group APIs

### POST /groups
Create a group.
```json
// Request
{
  "name": "Team Backend",
  "member_ids": ["usr_A", "usr_B", "usr_C"],
  "avatar_url": null
}
// Response 201 — returns conversation object with type="group"
```

### PUT /groups/{group_id}
Update group name / avatar (admin only).

### POST /groups/{group_id}/members
Add members (admin only).
```json
{ "user_ids": ["usr_D", "usr_E"] }
```

### DELETE /groups/{group_id}/members/{user_id}
Remove a member (admin only) or self (leave group).

### PUT /groups/{group_id}/members/{user_id}/role
Promote / demote to admin.
```json
{ "role": "admin" | "member" }
```

---

## 5. Message APIs

### GET /conversations/{conversation_id}/messages?before={msg_id}&limit=50
Paginated message history (reverse-chronological). Use `before` cursor for infinite scroll.

**Response** `200 OK`
```json
{
  "messages": [
    {
      "message_id": "msg_01J5...",
      "conversation_id": "conv_...",
      "sender_id": "usr_A",
      "content_type": "text",
      "body": "<encrypted-ciphertext>",
      "replied_to_id": null,
      "reactions": { "❤️": ["usr_B"], "👍": ["usr_C"] },
      "status": "read",
      "sent_at": "2025-09-01T10:00:00Z",
      "delivered_at": "2025-09-01T10:00:01Z",
      "read_at": "2025-09-01T10:00:05Z"
    }
  ],
  "has_more": true
}
```

### POST /conversations/{conversation_id}/messages
Send a message (fallback REST for offline queuing; primary path is WebSocket).
```json
{
  "idempotency_key": "uuid-v4",
  "content_type": "text | image | video | audio | document | location",
  "body": "<encrypted-ciphertext>",
  "replied_to_id": "msg_..." // optional
}
```
**Response** `201 Created` — returns message object with `status: "sent"`.

### DELETE /messages/{message_id}?scope=self|everyone
Delete a message.

### POST /messages/{message_id}/reactions
```json
{ "emoji": "❤️" }
```

### DELETE /messages/{message_id}/reactions/{emoji}

---

## 6. Media Upload API

### POST /media/upload-url
Request a pre-signed URL for direct-to-S3 upload.
```json
// Request
{
  "file_name": "photo.jpg",
  "mime_type": "image/jpeg",
  "file_size_bytes": 4096000,
  "conversation_id": "conv_..."
}

// Response 200
{
  "upload_url": "https://s3.amazonaws.com/...",
  "media_id": "med_01J5...",
  "expires_in": 300
}
```

Client uploads directly to S3, then sends `POST /conversations/{id}/messages` with `content_type: "image"` and `body` containing `media_id` + encryption metadata.

---

## 7. WebSocket Protocol

### Connection
```
wss://chat.example.com/ws?token=<access_token>
```

### Envelope Format (all messages)
```json
{
  "type": "message.new | message.delivered | message.read | typing.start | typing.stop | presence.update | ack",
  "payload": { ... },
  "id": "client-generated-uuid"   // for ack correlation
}
```

### Client → Server Events

**Send a message**
```json
{
  "type": "message.send",
  "id": "ck_xyz",
  "payload": {
    "idempotency_key": "uuid-v4",
    "conversation_id": "conv_...",
    "content_type": "text",
    "body": "<ciphertext>",
    "replied_to_id": null
  }
}
```

**Acknowledge delivery**
```json
{
  "type": "message.delivered",
  "payload": { "message_id": "msg_..." }
}
```

**Mark as read**
```json
{
  "type": "message.read",
  "payload": { "conversation_id": "conv_...", "up_to_message_id": "msg_..." }
}
```

**Typing indicator**
```json
{
  "type": "typing.start",
  "payload": { "conversation_id": "conv_..." }
}
```

### Server → Client Events

**New message**
```json
{
  "type": "message.new",
  "payload": { /* full message object */ }
}
```

**Delivery / read receipts**
```json
{
  "type": "message.status",
  "payload": {
    "message_id": "msg_...",
    "status": "delivered | read",
    "by_user_id": "usr_B",
    "at": "2025-09-01T10:00:05Z"
  }
}
```

**Presence update**
```json
{
  "type": "presence.update",
  "payload": { "user_id": "usr_B", "online": false, "last_seen": "..." }
}
```

**Server ACK (message accepted)**
```json
{
  "type": "ack",
  "id": "ck_xyz",
  "payload": { "message_id": "msg_...", "sent_at": "..." }
}
```

---

## 8. Rate Limits

| Endpoint / Action | Limit |
|---|---|
| Send message (WebSocket) | 60 msg / min per user |
| REST message send | 30 req / min per user |
| Media upload URL | 20 req / min per user |
| Auth endpoints | 10 req / min per IP |
| Search | 30 req / min per user |

Exceeding limits returns `HTTP 429 Too Many Requests` with a `Retry-After` header.

---

## 9. Error Format

```json
{
  "error": {
    "code": "MESSAGE_TOO_LARGE",
    "message": "Message body exceeds the 64 KB limit.",
    "request_id": "req_01J5..."
  }
}
```

Common error codes: `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `RATE_LIMITED`, `VALIDATION_ERROR`, `INTERNAL_ERROR`.