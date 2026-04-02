# Spec: Seafarer Anonymous Community Board

> Implementation-ready specification for S2.
> Another developer should be able to implement this without additional questions.

## 1. Data Model

### 1.1 ERD Overview

```
channels 1──N posts 1──N comments
                │               │
                │               │
posts 1──N post_reactions       comments 1──N comment_reactions
                │
posts 1──N anonymous_nicknames
```

### 1.2 DDL

```sql
-- Enum types
CREATE TYPE channel_type AS ENUM ('REAL_NAME', 'ANONYMOUS');
CREATE TYPE reaction_type AS ENUM ('LIKE', 'DISLIKE');

-- Channels
CREATE TABLE channels (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100)   NOT NULL,
    description TEXT,
    type        channel_type   NOT NULL,
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ    NOT NULL DEFAULT now()
);

-- Posts
CREATE TABLE posts (
    id             BIGSERIAL PRIMARY KEY,
    channel_id     BIGINT         NOT NULL REFERENCES channels(id),
    user_id        VARCHAR(50)    NOT NULL,  -- from X-User-Id header
    title          VARCHAR(200)   NOT NULL,
    content        TEXT           NOT NULL,
    view_count     INTEGER        NOT NULL DEFAULT 0  CHECK (view_count >= 0),
    like_count     INTEGER        NOT NULL DEFAULT 0  CHECK (like_count >= 0),
    dislike_count  INTEGER        NOT NULL DEFAULT 0  CHECK (dislike_count >= 0),
    comment_count  INTEGER        NOT NULL DEFAULT 0  CHECK (comment_count >= 0),
    created_at     TIMESTAMPTZ    NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ    NOT NULL DEFAULT now()
);

CREATE INDEX idx_posts_channel_id_created_at ON posts(channel_id, created_at DESC);
CREATE INDEX idx_posts_user_id ON posts(user_id);

-- Comments
CREATE TABLE comments (
    id             BIGSERIAL PRIMARY KEY,
    post_id        BIGINT         NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    parent_id      BIGINT         REFERENCES comments(id) ON DELETE RESTRICT,
    user_id        VARCHAR(50)    NOT NULL,
    content        TEXT,          -- nullable: set to NULL on soft delete
    is_deleted     BOOLEAN        NOT NULL DEFAULT FALSE,
    like_count     INTEGER        NOT NULL DEFAULT 0  CHECK (like_count >= 0),
    dislike_count  INTEGER        NOT NULL DEFAULT 0  CHECK (dislike_count >= 0),
    created_at     TIMESTAMPTZ    NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ    NOT NULL DEFAULT now(),
    -- parent_id NULL = top-level comment, non-null = reply
    -- No-nested-reply rule (reply to reply) is enforced at application level
);

CREATE INDEX idx_comments_post_id_created_at ON comments(post_id, created_at);
CREATE INDEX idx_comments_parent_id ON comments(parent_id);

-- Anonymous Nicknames
-- Maps a user to a unique nickname within a specific post
CREATE TABLE anonymous_nicknames (
    id          BIGSERIAL PRIMARY KEY,
    post_id     BIGINT         NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id     VARCHAR(50)    NOT NULL,
    nickname    VARCHAR(50)    NOT NULL,
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT now(),

    CONSTRAINT uq_nickname_user_post UNIQUE (post_id, user_id),
    CONSTRAINT uq_nickname_post UNIQUE (post_id, nickname)
);

-- Post Reactions
CREATE TABLE post_reactions (
    id          BIGSERIAL PRIMARY KEY,
    post_id     BIGINT         NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id     VARCHAR(50)    NOT NULL,
    type        reaction_type  NOT NULL,
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT now(),

    CONSTRAINT uq_post_reaction_user UNIQUE (post_id, user_id)
);

-- Comment Reactions
CREATE TABLE comment_reactions (
    id          BIGSERIAL PRIMARY KEY,
    comment_id  BIGINT         NOT NULL REFERENCES comments(id) ON DELETE CASCADE,
    user_id     VARCHAR(50)    NOT NULL,
    type        reaction_type  NOT NULL,
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT now(),

    CONSTRAINT uq_comment_reaction_user UNIQUE (comment_id, user_id)
);
```

### 1.3 Key Design Notes

- **user_id is VARCHAR(50)**, not a FK to a users table — auth is out of scope, we just store the raw header value
- **Denormalized counts** (`like_count`, `dislike_count`, `comment_count`, `view_count`) on posts and comments for fast read performance. Updated within the same transaction as the write operation.
- **anonymous_nicknames has two unique constraints**: `(post_id, user_id)` ensures one nickname per user per post; `(post_id, nickname)` ensures no two users get the same nickname in the same post
- **Soft delete** on comments uses `is_deleted` flag — content is replaced but row is preserved for reply thread structure
- **ON DELETE CASCADE** on post_id FKs — when a post is deleted, all comments, reactions, and nicknames are cleaned up by the DB
- **parent_id ON DELETE RESTRICT** — the DB blocks deletion of a comment that has replies. The app must soft-delete (set `is_deleted = true`, replace content) when replies exist, and may only hard-delete when there are no replies. RESTRICT ensures a bug in app logic surfaces as an error rather than silently destroying user content.
- **`updated_at` is set by the application**, not a DB trigger — every UPDATE statement on channels, posts, and comments must explicitly include `SET updated_at = now()`. A DB trigger (`BEFORE UPDATE`) is an alternative, but explicit application-level control is preferred for transparency and testability.
- **No `updated_at` on reaction tables** — reactions are created, switched, or deleted. Tracking when a reaction type changed has no business value.
- **No-nested-reply rule** is enforced at app level: before inserting a comment with `parent_id`, check that the parent's own `parent_id` is NULL

## 2. API Design

Base path: `/api`
Auth: `X-User-Id` request header (e.g., `user_1`)
Content-Type: `application/json`

### 2.1 Channels

#### Create Channel

```
POST /api/channels

Request:
{
  "name": "Ship Reviews",
  "description": "Share your experience",    // optional
  "type": "ANONYMOUS"                         // REAL_NAME | ANONYMOUS
}

Response: 201 Created
{
  "id": 1,
  "name": "Ship Reviews",
  "description": "Share your experience",
  "type": "ANONYMOUS",
  "createdAt": "2026-04-01T12:00:00Z"
}
```

#### List Channels

```
GET /api/channels

Response: 200 OK
{
  "data": [
    {
      "id": 1,
      "name": "Ship Reviews",
      "description": "Share your experience",
      "type": "ANONYMOUS",
      "createdAt": "2026-04-01T12:00:00Z"
    }
  ]
}
```

### 2.2 Posts

#### Create Post

```
POST /api/posts
X-User-Id: user_1

Request:
{
  "channelId": 1,
  "title": "My experience on MV Pacific",
  "content": "Great working conditions..."
}

Response: 201 Created
{
  "id": 10,
  "channelId": 1,
  "author": {
    "userId": "user_1",           // REAL_NAME channel only, null in ANONYMOUS
    "nickname": "Brave Whale"     // ANONYMOUS channel only, null in REAL_NAME
  },
  "title": "My experience on MV Pacific",
  "content": "Great working conditions...",
  "viewCount": 0,
  "likeCount": 0,
  "dislikeCount": 0,
  "commentCount": 0,
  "createdAt": "2026-04-01T12:00:00Z",
  "updatedAt": "2026-04-01T12:00:00Z"
}
```

#### List Posts (cursor-based pagination)

```
GET /api/posts?channelId=1&size=20&cursor={encodedCursor}

Response: 200 OK
{
  "data": [
    {
      "id": 10,
      "channelId": 1,
      "author": {
        "nickname": "Brave Whale"       // anonymous channel
      },
      "title": "My experience on MV Pacific",
      "content": "Great working conditions...",
      "viewCount": 42,
      "likeCount": 5,
      "dislikeCount": 1,
      "commentCount": 3,
      "createdAt": "2026-04-01T12:00:00Z"
    }
  ],
  "nextCursor": "eyJjcmVhdGVkQXQiOi4uLiwiaWQiOjEwfQ==",  // base64({"createdAt":...,"id":10})
  "hasNext": true
}
```

Cursor encodes `(createdAt, id)` of the last item. Query uses a LEFT JOIN to fetch anonymous nicknames in a single query (avoids N+1):

```sql
SELECT p.*, an.nickname
FROM posts p
LEFT JOIN anonymous_nicknames an ON an.post_id = p.id AND an.user_id = p.user_id
WHERE p.channel_id = :channelId
  AND (p.created_at, p.id) < (:cursorCreatedAt, :cursorId)
ORDER BY p.created_at DESC, p.id DESC
LIMIT :size
```

In REAL_NAME channels, the JOIN produces NULL for nickname — the app returns `author.userId` instead.

#### Get Post Detail

````
GET /api/posts/{postId}
X-User-Id: user_1

Response: 200 OK
{
  "id": 10,
  "channelId": 1,
  "author": {
    "nickname": "Brave Whale"
  },
  "title": "My experience on MV Pacific",
  "content": "Great working conditions...",
  "viewCount": 43,            // incremented
  "likeCount": 5,
  "dislikeCount": 1,
  "commentCount": 3,
  "myReaction": "LIKE",       // current user's reaction, null if none
  "createdAt": "2026-04-01T12:00:00Z",
  "updatedAt": "2026-04-01T12:00:00Z"
}

Query fetches the post with nickname and current user's reaction in one query:
```sql
SELECT p.*, an.nickname, pr.type AS my_reaction
FROM posts p
LEFT JOIN anonymous_nicknames an ON an.post_id = p.id AND an.user_id = p.user_id
LEFT JOIN post_reactions pr ON pr.post_id = p.id AND pr.user_id = :currentUserId
WHERE p.id = :postId
````

```

#### Update Post

```

PUT /api/posts/{postId}
X-User-Id: user_1

Request:
{
"title": "Updated title", // optional
"content": "Updated content" // optional
}

Response: 200 OK
{ ...updated post object... }

Behavior:

- Only provided fields are updated (partial update)
- updated_at is set to now() on every update

Errors:
403 Forbidden — X-User-Id does not match post author
404 Not Found — post does not exist

```

#### Delete Post

```

DELETE /api/posts/{postId}
X-User-Id: user_1

Response: 204 No Content

Errors:
403 Forbidden — not the author
404 Not Found

```

### 2.3 Comments

#### Create Comment

```

POST /api/comments
X-User-Id: user_1

Request:
{
"postId": 10,
"parentId": null, // null = top-level comment, non-null = reply
"content": "I agree with this"
}

Response: 201 Created
{
"id": 100,
"postId": 10,
"parentId": null,
"author": {
"nickname": "Brave Whale" // same nickname as their post in this thread
},
"content": "I agree with this",
"likeCount": 0,
"dislikeCount": 0,
"isDeleted": false,
"createdAt": "2026-04-01T12:30:00Z"
}

Errors:
400 Bad Request — parentId refers to a reply (nested reply not allowed)
404 Not Found — postId or parentId does not exist

```

#### List Comments for a Post (cursor-based, top-level only)

```

GET /api/comments?postId=10&size=20&cursor={encodedCursor}

Response: 200 OK
{
"data": [
{
"id": 100,
"postId": 10,
"parentId": null,
"author": {
"nickname": "Brave Whale"
},
"content": "I agree with this",
"likeCount": 2,
"dislikeCount": 0,
"isDeleted": false,
"myReaction": null,
"createdAt": "2026-04-01T12:30:00Z",
"replies": [
{
"id": 101,
"postId": 10,
"parentId": 100,
"author": {
"nickname": "Quiet Dolphin"
},
"content": "Me too!",
"likeCount": 0,
"dislikeCount": 0,
"isDeleted": false,
"myReaction": "LIKE",
"createdAt": "2026-04-01T12:35:00Z"
}
]
},
{
"id": 102,
"postId": 10,
"parentId": null,
"author": {
"nickname": "Silent Shark"
},
"content": null, // soft-deleted
"isDeleted": true,
"likeCount": 0,
"dislikeCount": 0,
"myReaction": null,
"createdAt": "2026-04-01T13:00:00Z",
"replies": [
{
"id": 103,
...
}
]
}
]
}

````

Top-level comments are cursor-paginated (sorted by `created_at ASC`). Each top-level comment includes all its replies inline (replies are not separately paginated — max depth is 1, so reply counts per comment stay manageable).

Query fetches all comments for the post (both top-level and replies) in one query, with nicknames and the current user's reaction:
```sql
SELECT c.*, an.nickname, cr.type AS my_reaction
FROM comments c
LEFT JOIN anonymous_nicknames an ON an.post_id = c.post_id AND an.user_id = c.user_id
LEFT JOIN comment_reactions cr ON cr.comment_id = c.id AND cr.user_id = :currentUserId
WHERE c.post_id = :postId
  AND (
    -- top-level comments within cursor window
    (c.parent_id IS NULL AND (c.created_at, c.id) > (:cursorCreatedAt, :cursorId))
    -- or replies belonging to those top-level comments
    OR c.parent_id IN (:topLevelCommentIds)
  )
ORDER BY c.parent_id NULLS FIRST, c.created_at ASC
````

This is executed as two queries in practice: first fetch the paginated top-level comments (with JOINs), then fetch all replies for those top-level IDs (with JOINs). The app groups replies under their parent in memory.

```
  "nextCursor": "eyJjcmVhdGVkQXQiOi4uLiwiaWQiOjEwMn0=",
  "hasNext": true
}
```

#### Update Comment

```
PUT /api/comments/{commentId}
X-User-Id: user_1

Request:
{
  "content": "Updated comment"
}

Response: 200 OK
{ ...updated comment object... }

Behavior:
  - updated_at is set to now() on every update

Errors:
  403 Forbidden — not the author
  404 Not Found
  400 Bad Request — comment is already deleted
```

#### Delete Comment

```
DELETE /api/comments/{commentId}
X-User-Id: user_1

Response: 204 No Content

Behavior:
  - If comment has replies → soft delete (is_deleted = true, content = null)
  - If comment has no replies → hard delete (remove row)
  - If comment is a reply → hard delete (remove row)

Errors:
  403 Forbidden — not the author
  404 Not Found
```

### 2.4 Reactions

#### Toggle Post Reaction

```
POST /api/posts/{postId}/reactions
X-User-Id: user_1

Request:
{
  "type": "LIKE"       // LIKE | DISLIKE
}

Response: 200 OK
{
  "action": "ADDED",         // ADDED | SWITCHED | CANCELLED
  "type": "LIKE",            // current reaction type, null if cancelled
  "likeCount": 6,            // updated counts
  "dislikeCount": 1
}

Logic:
  - No existing reaction      → INSERT, action = ADDED
  - Same type exists           → DELETE, action = CANCELLED
  - Different type exists      → UPDATE, action = SWITCHED
```

#### Toggle Comment Reaction

```
POST /api/comments/{commentId}/reactions
X-User-Id: user_1

Request:
{
  "type": "DISLIKE"
}

Response: 200 OK
{
  "action": "SWITCHED",
  "type": "DISLIKE",
  "likeCount": 1,
  "dislikeCount": 1
}
```

### 2.5 Common Error Response Format

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "You are not the author of this post"
  }
}
```

Standard HTTP status codes:

- `400` — Bad request (validation error, nested reply attempt)
- `403` — Not the author
- `404` — Resource not found
- `500` — Internal server error

## 3. Core Business Logic

### 3.1 Anonymous Nickname Generation

**When triggered:** Any time a user creates a post or comment in an ANONYMOUS channel.

**Nickname format:** `{Adjective} {SeaCreature}` — e.g., "Brave Whale", "Quiet Dolphin", "Silent Shark"

- Adjective pool: ~50 words (brave, quiet, silent, swift, bold, calm, ...)
- Noun pool: ~50 sea creatures (whale, dolphin, shark, octopus, turtle, ...)
- Total combinations: ~2,500 — sufficient for any single post's participants

**Algorithm:**

```
function getOrCreateNickname(postId, userId):
    MAX_RETRIES = 3

    for attempt in 1..MAX_RETRIES:
        nickname = generateRandomNickname()   // random adjective + noun
        if attempt == MAX_RETRIES:
            nickname = nickname + " " + randomInt(1, 9999)  // fallback: append number

        try:
            result = INSERT INTO anonymous_nicknames (post_id, user_id, nickname)
                     VALUES (:postId, :userId, :nickname)
                     ON CONFLICT (post_id, user_id) DO NOTHING

            if result.rowsAffected == 0:
                // User already has a nickname for this post
                return SELECT nickname FROM anonymous_nicknames
                       WHERE post_id = :postId AND user_id = :userId

            // Insert succeeded — return the new nickname
            return nickname

        catch UNIQUE_VIOLATION on (post_id, nickname):
            // Another user got the same nickname for this post — retry
            continue

    // Should not reach here (fallback with number is near-guaranteed unique)
```

**Concurrency safety:**

- `UNIQUE(post_id, user_id)` — prevents a user from getting two nicknames per post
- `UNIQUE(post_id, nickname)` — prevents two users from getting the same nickname per post
- Both constraints are enforced by the DB, not app logic
- Pattern: **optimistic concurrency** — try the write, let the DB reject conflicts, handle gracefully

**In REAL_NAME channels:** Skip nickname logic entirely. Return the user_id as the author identity.

### 3.2 Reaction Toggle

**When triggered:** `POST /api/posts/{postId}/reactions` or `POST /api/comments/{commentId}/reactions`

**Algorithm (same logic for both posts and comments):**

```
@Transactional
function toggleReaction(targetId, userId, newType):

    // Step 1: Check for existing reaction
    existing = SELECT * FROM post_reactions
               WHERE post_id = :targetId AND user_id = :userId

    // Step 2: No existing reaction → ADD
    if existing == null:
        INSERT INTO post_reactions (post_id, user_id, type) VALUES (...)
        UPDATE posts SET like_count = like_count + 1   // or dislike_count
            WHERE id = :targetId
        return { action: "ADDED", type: newType }

    // Step 3: Same type → CANCEL (remove reaction)
    if existing.type == newType:
        DELETE FROM post_reactions WHERE id = existing.id
        UPDATE posts SET like_count = like_count - 1   // or dislike_count
            WHERE id = :targetId
        return { action: "CANCELLED", type: null }

    // Step 4: Different type → SWITCH
    UPDATE post_reactions SET type = :newType WHERE id = existing.id
    UPDATE posts SET like_count = like_count + 1,      // increment new
                     dislike_count = dislike_count - 1  // decrement old
        WHERE id = :targetId
    return { action: "SWITCHED", type: newType }
```

**Concurrency safety:**

- `UNIQUE(post_id, user_id)` prevents duplicate reactions at the DB level
- If a double-click causes two concurrent requests:
  - Request A: SELECT → null, INSERT → success
  - Request B: SELECT → null, INSERT → unique constraint violation
  - Catch the violation, re-SELECT, re-run toggle logic (now it sees the reaction from A and cancels it, which is correct behavior)
- Denormalized counts are updated within the same `@Transactional` — atomic with the reaction change

**Count safety:** Counts can never go below 0 — enforced by `CHECK (>= 0)` constraints on the count columns (defined in DDL, section 1.2). A bug that over-decrements will throw a constraint violation instead of producing negative counts.

### 3.3 Comment Soft Delete

**When triggered:** `DELETE /api/comments/{commentId}`

**Algorithm:**

```
@Transactional
function deleteComment(commentId, userId):

    // Step 1: Load the comment
    comment = SELECT * FROM comments WHERE id = :commentId
    if comment == null: return 404
    if comment.user_id != userId: return 403
    if comment.is_deleted: return 404   // already deleted

    // Step 2: Is this a reply? (has parent_id)
    if comment.parent_id != null:
        // Replies are always hard-deleted
        DELETE FROM comments WHERE id = :commentId
        UPDATE posts SET comment_count = comment_count - 1
            WHERE id = comment.post_id
        cleanupParentIfNeeded(comment.parent_id)   // remove parent too if soft-deleted with no remaining replies
        return 204

    // Step 3: Top-level comment — check for replies
    replyCount = SELECT COUNT(*) FROM comments
                 WHERE parent_id = :commentId AND is_deleted = FALSE

    if replyCount == 0:
        // No active replies — hard delete
        // (also hard-delete any soft-deleted replies first)
        DELETE FROM comments WHERE parent_id = :commentId
        DELETE FROM comments WHERE id = :commentId
        UPDATE posts SET comment_count = comment_count - 1
            WHERE id = comment.post_id
        return 204
    else:
        // Has active replies — soft delete
        UPDATE comments SET is_deleted = TRUE, content = NULL, updated_at = now()
            WHERE id = :commentId
        // Do NOT decrement comment_count (placeholder is still visible)
        return 204
```

**Cleanup opportunity:** When the last reply under a soft-deleted parent is deleted, the parent can be fully removed:

```
// After hard-deleting a reply, check if parent should be cleaned up
function cleanupParentIfNeeded(parentId):
    parent = SELECT * FROM comments WHERE id = :parentId
    if parent.is_deleted:
        remainingReplies = SELECT COUNT(*) FROM comments WHERE parent_id = :parentId
        if remainingReplies == 0:
            DELETE FROM comments WHERE id = :parentId
            UPDATE posts SET comment_count = comment_count - 1
                WHERE id = parent.post_id
```

### 3.4 View Count

**When triggered:** `GET /api/posts/{postId}` (post detail view)

**Algorithm:**

```sql
UPDATE posts SET view_count = view_count + 1 WHERE id = :postId
```

**Design decision — raw count, not unique views:**

- We don't track which users viewed which posts
- Every detail view request increments the counter
- Unique view tracking would require a `post_views` table (post_id, user_id, viewed_at), adding write overhead to every read operation
- For a seafarer community, raw count is sufficient and much simpler
- This is an ambiguous requirement — documented in decisions.md

## 4. Edge Cases

| #   | Scenario                                              | Handling                                                                                                                                                                                                                                  |
| --- | ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Missing or empty X-User-Id header**                 | Return 400 Bad Request. All endpoints require user identification.                                                                                                                                                                        |
| 2   | **Post/comment in non-existent channel or post**      | Return 404 Not Found. Validate FK existence before insert.                                                                                                                                                                                |
| 3   | **React to a soft-deleted comment**                   | Return 400 Bad Request. Deleted comments cannot receive new reactions.                                                                                                                                                                    |
| 4   | **Edit a soft-deleted comment**                       | Return 400 Bad Request. Once soft-deleted, content cannot be restored.                                                                                                                                                                    |
| 5   | **Reply to a soft-deleted comment**                   | Return 400 Bad Request. Cannot start a conversation under a deleted comment.                                                                                                                                                              |
| 6   | **Delete a post with comments, reactions, nicknames** | CASCADE handles cleanup. All child rows (comments, reactions, nicknames) are automatically removed by the DB.                                                                                                                             |
| 7   | **Empty or blank title/content**                      | Return 400 Bad Request. Reject whitespace-only strings. Validate at app level before DB insert.                                                                                                                                           |
| 8   | **Nickname pool exhaustion**                          | On the final retry (attempt 3), a random number is appended to the nickname (e.g., "Brave Whale 42"). This guarantees uniqueness beyond the 2,500 base combinations. The user's action always succeeds — nickname generation never fails. |
| 9   | **User reacts to their own post/comment**             | Allowed. No restriction — same behavior as reacting to anyone else's content.                                                                                                                                                             |
| 10  | **Invalid or tampered cursor parameter**              | Return 400 Bad Request with "invalid cursor" message. Decode in try-catch, reject malformed values.                                                                                                                                       |
| 11  | **Update post in anonymous channel**                  | Nickname stays the same. It's tied to (post_id, user_id), not the content. Editing content does not regenerate or change the nickname.                                                                                                    |
| 12  | **Concurrent post creation in anonymous channel**     | Two users create posts simultaneously. Each gets their own nickname via the INSERT ON CONFLICT pattern. No conflict possible — different user_ids.                                                                                        |
| 13  | **Delete the only reply under a soft-deleted parent** | Hard-delete the reply, then clean up the parent: since parent is soft-deleted and now has zero replies, hard-delete the parent too. Decrement comment_count for both.                                                                     |
| 14  | **React to a non-existent post/comment**              | Return 404 Not Found.                                                                                                                                                                                                                     |
| 15  | **Nested reply attempt (reply to a reply)**           | Return 400 Bad Request. App checks that `parent.parent_id IS NULL` before inserting.                                                                                                                                                      |

## 5. Test Scenarios (Given-When-Then)

### 5.1 Anonymous Nickname

**T1: First-time user gets a nickname in anonymous channel**

- Given: an ANONYMOUS channel with a post (id=10), user_1 has no nickname for this post
- When: user_1 creates a comment on post 10
- Then: a nickname is generated, stored in anonymous_nicknames, and returned in the response

**T2: Same user gets same nickname within same post**

- Given: user_1 already has nickname "Brave Whale" for post 10
- When: user_1 creates another comment on post 10
- Then: author.nickname is "Brave Whale" (reused, not regenerated)

**T3: Same user gets different nickname on different post**

- Given: user_1 has nickname "Brave Whale" for post 10
- When: user_1 creates a comment on post 20
- Then: a new nickname is independently generated (not "Brave Whale")

**T4: Nickname collision under concurrency**

- Given: post 10 has no nicknames yet
- When: two users simultaneously get assigned "Brave Whale"
- Then: one succeeds, the other retries with a different nickname. Both end up with unique nicknames.

**T5: Real-name channel skips nickname**

- Given: a REAL_NAME channel with a post
- When: user_1 creates a comment
- Then: author.userId is "user_1", author.nickname is null

### 5.2 Reactions

**T6: Add a reaction**

- Given: post 10 has 0 likes, user_1 has no reaction
- When: user_1 sends POST /api/posts/10/reactions {"type": "LIKE"}
- Then: action = "ADDED", likeCount = 1, dislikeCount = 0

**T7: Cancel a reaction (same type again)**

- Given: user_1 has LIKE on post 10, likeCount = 1
- When: user_1 sends POST /api/posts/10/reactions {"type": "LIKE"}
- Then: action = "CANCELLED", type = null, likeCount = 0

**T8: Switch a reaction**

- Given: user_1 has LIKE on post 10, likeCount = 1, dislikeCount = 0
- When: user_1 sends POST /api/posts/10/reactions {"type": "DISLIKE"}
- Then: action = "SWITCHED", type = "DISLIKE", likeCount = 0, dislikeCount = 1

**T9: Double-click reaction (concurrent)**

- Given: user_1 has no reaction on post 10
- When: two identical LIKE requests arrive simultaneously
- Then: one adds, the other catches the constraint violation and cancels. Final state: no reaction (net toggle).

**T10: React to soft-deleted comment**

- Given: comment 100 is soft-deleted (is_deleted = true)
- When: user_1 sends POST /api/comments/100/reactions {"type": "LIKE"}
- Then: 400 Bad Request

### 5.3 Comments & Soft Delete

**T11: Create top-level comment**

- Given: post 10 exists, comment_count = 0
- When: user_1 sends POST /api/comments {"postId": 10, "parentId": null, "content": "Hello"}
- Then: comment created, post comment_count = 1

**T12: Create reply**

- Given: top-level comment 100 exists on post 10
- When: user_2 sends POST /api/comments {"postId": 10, "parentId": 100, "content": "Reply"}
- Then: reply created with parentId = 100, post comment_count incremented

**T13: Reject nested reply (reply to a reply)**

- Given: comment 101 is a reply (parent_id = 100)
- When: user_3 sends POST /api/comments {"postId": 10, "parentId": 101, "content": "Nested"}
- Then: 400 Bad Request

**T14: Soft delete comment with replies**

- Given: comment 100 has 2 replies, post comment_count = 3
- When: author deletes comment 100
- Then: comment 100 has is_deleted = true, content = null. Replies remain. comment_count stays 3.

**T15: Hard delete comment with no replies**

- Given: comment 100 has no replies, post comment_count = 1
- When: author deletes comment 100
- Then: comment 100 row is removed from DB. comment_count = 0.

**T16: Hard delete a reply**

- Given: comment 101 is a reply to comment 100, post comment_count = 2
- When: author deletes comment 101
- Then: comment 101 row removed. comment_count = 1.

**T17: Delete last reply under soft-deleted parent**

- Given: comment 100 is soft-deleted, has one remaining reply (101), comment_count = 2
- When: author deletes reply 101
- Then: reply 101 removed. Parent 100 is also removed (cleanup). comment_count = 0.

**T18: Reply to soft-deleted comment**

- Given: comment 100 is soft-deleted
- When: user sends POST /api/comments {"postId": 10, "parentId": 100, "content": "Reply"}
- Then: 400 Bad Request

### 5.4 Posts

**T19: View count increments on detail view**

- Given: post 10 has view_count = 5
- When: GET /api/posts/10
- Then: response shows viewCount = 6

**T20: Only author can edit post**

- Given: post 10 is authored by user_1
- When: user_2 sends PUT /api/posts/10
- Then: 403 Forbidden

**T21: Only author can delete post**

- Given: post 10 is authored by user_1
- When: user_2 sends DELETE /api/posts/10
- Then: 403 Forbidden

**T22: Delete post cascades all children**

- Given: post 10 has 3 comments, 5 reactions, 2 nicknames
- When: author deletes post 10
- Then: post row removed. All comments, reactions, and nicknames for post 10 are removed by CASCADE.

### 5.5 Pagination

**T23: First page (no cursor)**

- Given: channel 1 has 25 posts
- When: GET /api/posts?channelId=1&size=20
- Then: 20 posts returned (newest first), hasNext = true, nextCursor is set

**T24: Second page (with cursor)**

- Given: channel 1 has 25 posts, cursor from T23
- When: GET /api/posts?channelId=1&size=20&cursor={nextCursor}
- Then: 5 posts returned, hasNext = false, nextCursor is null

**T25: Invalid cursor**

- Given: any state
- When: GET /api/posts?channelId=1&size=20&cursor=invalidgarbage
- Then: 400 Bad Request

### 5.6 Validation

**T26: Missing X-User-Id header**

- Given: any endpoint requiring auth
- When: request sent without X-User-Id header
- Then: 400 Bad Request

**T27: Empty post title or content**

- Given: valid channel exists
- When: POST /api/posts {"channelId": 1, "title": "", "content": "text"}
- Then: 400 Bad Request
