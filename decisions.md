# Decisions Log

> Why I made each design choice — trade-offs, alternatives considered, and reasoning.

## Scenario Choice

**Chose S2 (Anonymous Community Board)** over S1 (Medicine Inventory).
- I have direct experience building community/social features — have built two platforms with similar patterns (reactions, counters, pagination)
- More confident defending these design decisions in interview
- The edge cases (anonymous identity, reactions, soft delete) align with problems I've solved before

---

## Key Decisions

### 1. Anonymous Nickname Scoping

**Problem:** User A should be "Brave Whale" in post 1 but "Quiet Dolphin" in post 2. The scope is (user, post) — same user, same post = same nickname.

**Considered:**
- **Option A: Hash-based** — deterministic hash of (user_id + post_id) mapped to a nickname. No DB table needed. But: hard to guarantee uniqueness (two different user+post combos could hash to the same nickname), and no way to retry on collision.
- **Option B: Dedicated mapping table** — `anonymous_nicknames(post_id, user_id, nickname)` with two unique constraints.

**Chose Option B.** The table makes the scoping explicit and the two unique constraints (`post_id, user_id` and `post_id, nickname`) enforce both rules at the DB level. It's a few extra rows but the correctness guarantee is worth it.

### 2. Nickname Concurrency

**Problem:** Two users comment on the same post simultaneously and randomly get assigned the same nickname.

**Considered:**
- **Pessimistic locking** — `SELECT FOR UPDATE` before inserting. Guarantees no collision but adds lock contention and complexity. Also, there's no existing row to lock for a first-time user.
- **Optimistic concurrency** — try the INSERT, let the DB unique constraint reject duplicates, retry with a new nickname on failure.

**Chose optimistic concurrency.** The collision (two users getting the same random nickname from a pool of 2,500) is extremely rare. When it does happen, the DB constraint catches it cleanly and we retry. No locking overhead for the 99.99% case. Same pattern used for reaction double-click prevention — consistent approach across the codebase.

### 3. Reaction Toggle Design

**Problem:** Like/dislike has three behaviors from one action — add, cancel, switch. How should the API work?

**Considered:**
- **Separate endpoints** — `PUT /reactions` to add/switch, `DELETE /reactions` to cancel. Semantically "correct" but forces the frontend to track current state and decide which verb to use.
- **Single toggle endpoint** — `POST /reactions {"type": "LIKE"}`. Backend checks current state and determines the action (add, cancel, or switch).

**Chose single toggle.** The frontend just says "user tapped the like button" — the backend figures out the intent. One endpoint, one request body, simpler client code. POST isn't semantically perfect (it usually means "create"), but the pragmatism wins here. This is how most social platforms handle it.

### 4. Soft Delete for Comments

**Problem:** Deleting a comment that has replies — should we remove it entirely or keep a placeholder?

**Considered:**
- **Hard delete always** — simple, but breaks the reply thread structure. Replies become orphaned with no visible parent context.
- **Soft delete with `is_deleted` flag** — keep the row, set content to NULL, show "deleted comment" placeholder. Replies remain attached.

**Chose soft delete when replies exist, hard delete otherwise.** This preserves thread structure while not keeping unnecessary rows. The `is_deleted` flag + nullable content makes it clean.

**Additional decision — `ON DELETE RESTRICT` for `parent_id`:** If a bug in app logic tries to hard-delete a comment that has replies, RESTRICT makes the DB throw an error instead of silently cascading. Bugs should be loud, not destructive. Data loss from a silent CASCADE is far worse than a caught exception.

**Comment count behavior:** Soft-deleted comments are still visible (as placeholders), so we do NOT decrement `comment_count` on soft delete. Only hard deletes decrement. This matches Reddit's behavior — users see the count matching what's on screen.

**Cleanup:** When the last reply under a soft-deleted parent is deleted, the parent is also cleaned up (hard-deleted). This prevents dead placeholders with no children lingering forever.

### 5. Counts Strategy (COUNT vs Denormalized)

**Problem:** Post listings need like_count, dislike_count, comment_count. Calculate on the fly or store?

**Considered:**
- **COUNT queries** — `SELECT COUNT(*) FROM reactions WHERE post_id = ?`. Always accurate. But: a listing of 20 posts = 20+ COUNT subqueries or complex JOINs. Slow as data grows.
- **Denormalized counters** — store counts directly on the posts/comments rows. Fast reads, but must keep in sync on every write.

**Chose denormalized counters.** I've built two platforms with this pattern. It's harder to implement correctly, but:
- Community boards are read-heavy — far more views than reactions/comments
- Sync correctness is guaranteed by wrapping counter updates in the same `@Transactional` as the write operation
- Added `CHECK (>= 0)` constraints on all count columns so a sync bug can never produce negative numbers — the DB will throw an error instead
- This is what production community platforms do (Reddit, Hacker News, etc.)

### 6. Pagination Strategy

**Problem:** How to paginate post and comment listings.

**Considered:**
- **Offset-based** (`?page=1&size=20`) — simple, allows "jump to page N". But: if new posts are created while browsing, data shifts and users see duplicates. Also, `OFFSET 10000` is slow (DB scans and skips N rows).
- **Cursor-based** (`?cursor=xxx&size=20`) — stable pagination, no duplicates when new data arrives. Fast queries using `WHERE (created_at, id) < (cursor)`. But: no "jump to page 5".

**Chose cursor-based.** A community board with active posting is exactly the use case where offset pagination breaks — new posts shift the data between page loads. Cursor gives stable results. The lack of "jump to page N" is acceptable — users typically scroll through feeds linearly, not jump to page 47.

Cursor encodes `(created_at, id)` in base64 — using both fields ensures uniqueness even when posts have identical timestamps.

### 7. View Count — Unique vs Raw

**Problem:** Should view_count track total views or unique views per user?

**Considered:**
- **Unique views** — requires a `post_views(post_id, user_id)` table. Every read becomes a read + write. Accurate but adds write overhead to the most common operation (viewing).
- **Raw count** — simple `view_count + 1` increment. Every page load counts. Inflated number but zero overhead.

**Chose raw count.** For a seafarer community (not millions of users), the simplicity wins. Unique tracking would turn every read into a write operation — bad trade-off for a feature that doesn't need precision. The requirement says "view count increases on detail view" without specifying uniqueness.

### 8. Reaction Tables — Single vs Separate

**Problem:** Reactions can target posts or comments. One table or two?

**Considered:**
- **Single polymorphic table** — `reactions(target_type, target_id, user_id, type)`. One table, simpler code. But: can't use foreign keys (target_id could point to either table). Orphan cleanup relies on app logic.
- **Two tables** — `post_reactions` and `comment_reactions`. Proper foreign keys with ON DELETE CASCADE. DB handles cleanup. But: toggle logic is written in two places.

**Chose two separate tables.** Proper FK constraints mean the DB guarantees referential integrity and handles cascade deletes automatically. The "duplicate logic" concern is minor — in Spring Boot, a shared service method handles the toggle algorithm, and both repositories call into it. More tables is fine when the architecture is cleaner for it.

### 9. URL Structure — Nested vs Flat

**Problem:** Should posts be at `/channels/{id}/posts/{id}` or `/posts/{id}`?

**Considered:**
- **Nested** — `/channels/{channelId}/posts/{postId}` makes the relationship explicit but the channelId is redundant for detail/edit/delete (postId already implies the channel).
- **Flat** — `/posts/{postId}` for everything. channelId is a query parameter for listing.

**Chose flat.** Mixing nested (for create/list) and flat (for detail/edit/delete) is inconsistent. All-flat keeps URLs uniform. The channel relationship is expressed through the channelId field in request bodies and query params, not URL nesting. Applied the same pattern to comments.

---

## Ambiguous Requirements — My Interpretations

1. **"View count increases on detail view"** — interpreted as raw count, not unique per user. The requirement doesn't mention uniqueness, and unique tracking adds significant complexity for minimal value at this scale.

2. **"Comments can be deleted"** — interpreted as: soft delete (placeholder) when replies exist, hard delete when no replies. The requirement says "replace content with placeholder when replies exist" but doesn't specify what happens when there are no replies — hard delete is the reasonable choice.

3. **"Channel management"** — interpreted as create + list only. The requirement says "channels exist" but doesn't specify update/delete for channels. Kept it minimal.

4. **"Pagination for post listing"** — only post listings are explicitly required to be paginated. I also added pagination to comment listing as a practical measure, since comment threads can grow unbounded.

5. **Self-reactions** — the requirement doesn't mention whether users can react to their own content. Allowed it — most platforms (Reddit, YouTube) allow this, and blocking it adds complexity with no clear benefit.

---

## If I Had More Time

1. **Rate limiting** — prevent abuse of reaction toggling, comment spam, and view count inflation
2. **Nickname theming per channel** — instead of one global adjective+animal pool, each channel could have its own themed pool (sea creatures for maritime channels, etc.)
3. **Comment pagination for replies** — currently all replies under a top-level comment are loaded at once. For extremely active threads, this could be paginated too
4. **Audit log** — track all mutations (create, update, delete) for moderation purposes
5. **Database index optimization** — after load testing, add covering indexes for the most common query patterns
6. **Count reconciliation job** — a scheduled task that recalculates denormalized counts from the source tables, as a safety net against sync drift
