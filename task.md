# VMS Holdings — Backend Engineer Task (S2)

## Scenario: Seafarer Anonymous Community Board

**Tech Stack**: Java or Kotlin + Spring Boot + PostgreSQL
**Deliverables**: spec.md + decisions.md (no code)
**Submit to**: pli@vmsforsea.com
**Subject format**: [VMS 과제] Name - S2
**Deadline**: Day before interview, 23:59 KST

---

## Requirements Summary

### Features

1. **Channel Management** — Two types: REAL_NAME and ANONYMOUS
2. **Post CRUD** — Create, read, update, delete posts in channels. Paginated listing.
3. **Anonymous Nicknames** — Auto-generated per user per post in anonymous channels. Same user + same post = same nickname. Same user + different post = different nickname.
4. **Comments & Replies** — Comment on posts, reply to comments. Max 1 level deep (no reply-to-reply). Soft delete: if comment has replies, replace content with placeholder.
5. **Reactions (Like/Dislike)** — On posts and comments. Toggle: same reaction again = cancel, different reaction = switch. One reaction per user per target.
6. **View Count** — Increment on post detail view.

### Out of Scope
- Image uploads
- Notifications
- Admin moderation
- Auth (use X-User-Id header)
- Real-time (WebSocket/SSE)
- Search

### Key Questions to Address
1. How to model anonymous nickname scoping? (user A = "Brave Whale" in post 1, "Quiet Dolphin" in post 2)
2. Nickname concurrency — two users comment simultaneously, can nicknames collide?
3. Reaction toggle design at DB level
4. Soft-deleted comments with replies — how to model and display?
5. Like/comment counts — COUNT(*) every time vs denormalized counters?

---

## Evaluation Criteria
- Spec concreteness — could another dev implement from this alone?
- Depth of thought on the key questions above
- Trade-off analysis in decisions.md
- Everything in spec will be discussed in follow-up interview
