# VMS Holdings — Backend Engineer Task

## About This Project
This is a job application task for VMS Holdings (㈜브이엠에스홀딩스) — Product Server (Backend) Engineer (Junior) position.

**Task:** Design-only (no code). Write spec.md and decisions.md for S2 (Seafarer Anonymous Community Board).
**Tech stack reference:** Java/Kotlin + Spring Boot + PostgreSQL
**Submit to:** pli@vmsforsea.com with subject `[VMS 과제] Name - S2`
**Deadline:** Day before interview, 23:59 KST

## About the User
- Has direct experience building 2 community/social platforms
- Comfortable with denormalized counters, reactions, pagination, social features
- Not strong in Java/Kotlin — but this task is design-only, no code needed
- Prefers English for working, final submission may need Korean translation

## Key Design Decisions Made
- **S2 chosen** over S1 (medicine inventory) — domain experience with community features
- **Optimistic concurrency** pattern used for both nickname generation and reaction toggle (DB constraints catch conflicts, retry on violation)
- **Denormalized counts** — like_count, dislike_count, comment_count on rows, updated within same transaction
- **Two separate reaction tables** (post_reactions, comment_reactions) — proper FK integrity over single polymorphic table
- **Cursor-based pagination** — stable for active feeds, no duplicate/shift problems
- **Flat URL structure** — consistent /api/posts, /api/comments, not nested under channels
- **POST toggle** for reactions — single endpoint, backend determines add/switch/cancel
- **ON DELETE RESTRICT** on comment parent_id — bugs are loud, not silent
- **Soft delete** only when comment has replies, hard delete otherwise
- **Raw view count** — not unique per user, simple increment

## File Structure
- `spec.md` — the deliverable: data model, API, business logic, edge cases, tests
- `decisions.md` — the deliverable: trade-off analysis, reasoning
- `context.md` — full task email + PDF content for reference
- `task.md` — condensed task summary

## Status
- [x] Data model (6 tables with DDL)
- [x] API design (all endpoints with request/response schemas)
- [x] Business logic (nickname, reactions, soft delete, view count)
- [x] Edge cases (15 identified)
- [x] Test scenarios (27 Given-When-Then cases)
- [x] Decisions (9 key decisions + ambiguous interpretations + future improvements)
- [ ] Final review
- [ ] Korean translation (if needed)
- [ ] Submit via email
