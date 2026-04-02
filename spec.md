# Spec: 선원 익명 커뮤니티 게시판

> S2의 구현 가능한 수준의 설계 명세서.
> 다른 개발자가 추가 질문 없이 이 문서만으로 구현을 시작할 수 있어야 합니다.

## 1. 데이터 모델

### 1.1 ERD 개요

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
-- Enum 타입
CREATE TYPE channel_type AS ENUM ('REAL_NAME', 'ANONYMOUS');
CREATE TYPE reaction_type AS ENUM ('LIKE', 'DISLIKE');

-- 채널
CREATE TABLE channels (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100)   NOT NULL,
    description TEXT,
    type        channel_type   NOT NULL,
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ    NOT NULL DEFAULT now()
);

-- 게시글
CREATE TABLE posts (
    id             BIGSERIAL PRIMARY KEY,
    channel_id     BIGINT         NOT NULL REFERENCES channels(id),
    user_id        VARCHAR(50)    NOT NULL,  -- X-User-Id 헤더에서 가져옴
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

-- 댓글
CREATE TABLE comments (
    id             BIGSERIAL PRIMARY KEY,
    post_id        BIGINT         NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    parent_id      BIGINT         REFERENCES comments(id) ON DELETE RESTRICT,
    user_id        VARCHAR(50)    NOT NULL,
    content        TEXT,          -- nullable: 소프트 삭제 시 NULL로 설정
    is_deleted     BOOLEAN        NOT NULL DEFAULT FALSE,
    like_count     INTEGER        NOT NULL DEFAULT 0  CHECK (like_count >= 0),
    dislike_count  INTEGER        NOT NULL DEFAULT 0  CHECK (dislike_count >= 0),
    created_at     TIMESTAMPTZ    NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ    NOT NULL DEFAULT now(),
    -- parent_id NULL = 최상위 댓글, non-null = 답글
    -- 중첩 답글 금지 규칙(답글의 답글)은 애플리케이션 레벨에서 강제
);

CREATE INDEX idx_comments_post_id_created_at ON comments(post_id, created_at);
CREATE INDEX idx_comments_parent_id ON comments(parent_id);

-- 익명 닉네임
-- 특정 게시글 내에서 사용자를 고유 닉네임에 매핑
CREATE TABLE anonymous_nicknames (
    id          BIGSERIAL PRIMARY KEY,
    post_id     BIGINT         NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id     VARCHAR(50)    NOT NULL,
    nickname    VARCHAR(50)    NOT NULL,
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT now(),

    CONSTRAINT uq_nickname_user_post UNIQUE (post_id, user_id),
    CONSTRAINT uq_nickname_post UNIQUE (post_id, nickname)
);

-- 게시글 반응
CREATE TABLE post_reactions (
    id          BIGSERIAL PRIMARY KEY,
    post_id     BIGINT         NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id     VARCHAR(50)    NOT NULL,
    type        reaction_type  NOT NULL,
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT now(),

    CONSTRAINT uq_post_reaction_user UNIQUE (post_id, user_id)
);

-- 댓글 반응
CREATE TABLE comment_reactions (
    id          BIGSERIAL PRIMARY KEY,
    comment_id  BIGINT         NOT NULL REFERENCES comments(id) ON DELETE CASCADE,
    user_id     VARCHAR(50)    NOT NULL,
    type        reaction_type  NOT NULL,
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT now(),

    CONSTRAINT uq_comment_reaction_user UNIQUE (comment_id, user_id)
);
```

### 1.3 주요 설계 노트

- **user_id는 VARCHAR(50)** — users 테이블에 대한 FK가 아님. 인증/인가는 범위 밖이므로 헤더 값을 그대로 저장
- **비정규화된 카운트** (`like_count`, `dislike_count`, `comment_count`, `view_count`)를 posts와 comments에 저장하여 빠른 읽기 성능 확보. 쓰기 작업과 같은 트랜잭션 내에서 업데이트
- **anonymous_nicknames에 두 개의 유니크 제약조건**: `(post_id, user_id)`는 게시글당 사용자당 하나의 닉네임 보장; `(post_id, nickname)`은 같은 게시글에서 두 사용자가 같은 닉네임을 받지 않도록 보장
- **소프트 삭제** — comments에 `is_deleted` 플래그 사용. 내용은 대체되지만 답글 스레드 구조를 위해 행은 보존
- **post_id FK에 ON DELETE CASCADE** — 게시글 삭제 시 모든 댓글, 반응, 닉네임이 DB에 의해 자동 정리
- **parent_id에 ON DELETE RESTRICT** — 답글이 있는 댓글의 삭제를 DB가 차단. 앱은 답글이 있으면 소프트 삭제(`is_deleted = true`, 내용 대체)해야 하며, 답글이 없을 때만 하드 삭제 가능. RESTRICT는 앱 로직의 버그가 조용히 사용자 콘텐츠를 파괴하는 대신 에러로 표면화되도록 보장
- **`updated_at`는 애플리케이션에서 설정** — DB 트리거가 아님. channels, posts, comments에 대한 모든 UPDATE 문은 명시적으로 `SET updated_at = now()`를 포함해야 합니다. DB 트리거(`BEFORE UPDATE`)가 대안이지만, 투명성과 테스트 용이성을 위해 애플리케이션 레벨 제어를 선호
- **반응 테이블에는 `updated_at` 없음** — 반응은 생성, 전환, 삭제됩니다. 반응 타입이 언제 변경되었는지 추적하는 것은 비즈니스 가치가 없음
- **중첩 답글 금지 규칙**은 앱 레벨에서 강제: `parent_id`가 있는 댓글을 삽입하기 전에 부모의 `parent_id`가 NULL인지 확인

## 2. API 설계

기본 경로: `/api`
인증: `X-User-Id` 요청 헤더 (예: `user_1`)
Content-Type: `application/json`

### 2.1 채널

#### 채널 생성

```
POST /api/channels

Request:
{
  "name": "선박 리뷰",
  "description": "경험을 공유하세요",    // 선택사항
  "type": "ANONYMOUS"                    // REAL_NAME | ANONYMOUS
}

Response: 201 Created
{
  "id": 1,
  "name": "선박 리뷰",
  "description": "경험을 공유하세요",
  "type": "ANONYMOUS",
  "createdAt": "2026-04-01T12:00:00Z"
}
```

#### 채널 목록 조회

```
GET /api/channels

Response: 200 OK
{
  "data": [
    {
      "id": 1,
      "name": "선박 리뷰",
      "description": "경험을 공유하세요",
      "type": "ANONYMOUS",
      "createdAt": "2026-04-01T12:00:00Z"
    }
  ]
}
```

### 2.2 게시글

#### 게시글 작성

```
POST /api/posts
X-User-Id: user_1

Request:
{
  "channelId": 1,
  "title": "MV Pacific 승선 경험",
  "content": "근무 환경이 좋았습니다..."
}

Response: 201 Created
{
  "id": 10,
  "channelId": 1,
  "author": {
    "userId": "user_1",           // REAL_NAME 채널에서만 표시, ANONYMOUS에서는 null
    "nickname": "용감한 고래"      // ANONYMOUS 채널에서만 표시, REAL_NAME에서는 null
  },
  "title": "MV Pacific 승선 경험",
  "content": "근무 환경이 좋았습니다...",
  "viewCount": 0,
  "likeCount": 0,
  "dislikeCount": 0,
  "commentCount": 0,
  "createdAt": "2026-04-01T12:00:00Z",
  "updatedAt": "2026-04-01T12:00:00Z"
}
```

#### 게시글 목록 조회 (커서 기반 페이지네이션)

```
GET /api/posts?channelId=1&size=20&cursor={encodedCursor}

Response: 200 OK
{
  "data": [
    {
      "id": 10,
      "channelId": 1,
      "author": {
        "nickname": "용감한 고래"       // 익명 채널
      },
      "title": "MV Pacific 승선 경험",
      "content": "근무 환경이 좋았습니다...",
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

커서는 마지막 항목의 `(createdAt, id)`를 인코딩합니다. 쿼리는 LEFT JOIN으로 익명 닉네임을 단일 쿼리로 조회합니다 (N+1 방지):

```sql
SELECT p.*, an.nickname
FROM posts p
LEFT JOIN anonymous_nicknames an ON an.post_id = p.id AND an.user_id = p.user_id
WHERE p.channel_id = :channelId
  AND (p.created_at, p.id) < (:cursorCreatedAt, :cursorId)
ORDER BY p.created_at DESC, p.id DESC
LIMIT :size
```

REAL_NAME 채널에서는 JOIN이 nickname에 대해 NULL을 반환합니다 — 앱은 대신 `author.userId`를 반환합니다.

#### 게시글 상세 조회

````
GET /api/posts/{postId}
X-User-Id: user_1

Response: 200 OK
{
  "id": 10,
  "channelId": 1,
  "author": {
    "nickname": "용감한 고래"
  },
  "title": "MV Pacific 승선 경험",
  "content": "근무 환경이 좋았습니다...",
  "viewCount": 43,            // 증가됨
  "likeCount": 5,
  "dislikeCount": 1,
  "commentCount": 3,
  "myReaction": "LIKE",       // 현재 사용자의 반응, 없으면 null
  "createdAt": "2026-04-01T12:00:00Z",
  "updatedAt": "2026-04-01T12:00:00Z"
}

닉네임과 현재 사용자의 반응을 단일 쿼리로 조회:
```sql
SELECT p.*, an.nickname, pr.type AS my_reaction
FROM posts p
LEFT JOIN anonymous_nicknames an ON an.post_id = p.id AND an.user_id = p.user_id
LEFT JOIN post_reactions pr ON pr.post_id = p.id AND pr.user_id = :currentUserId
WHERE p.id = :postId
````

```

#### 게시글 수정

```

PUT /api/posts/{postId}
X-User-Id: user_1

Request:
{
"title": "수정된 제목", // 선택사항
"content": "수정된 내용" // 선택사항
}

Response: 200 OK
{ ...수정된 게시글 객체... }

동작:

- 제공된 필드만 업데이트 (부분 업데이트)
- 매 업데이트마다 updated_at을 now()로 설정

에러:
403 Forbidden — X-User-Id가 게시글 작성자와 불일치
404 Not Found — 게시글이 존재하지 않음

```

#### 게시글 삭제

```

DELETE /api/posts/{postId}
X-User-Id: user_1

Response: 204 No Content

에러:
403 Forbidden — 작성자가 아님
404 Not Found

```

### 2.3 댓글

#### 댓글 작성

```

POST /api/comments
X-User-Id: user_1

Request:
{
"postId": 10,
"parentId": null, // null = 최상위 댓글, non-null = 답글
"content": "저도 동의합니다"
}

Response: 201 Created
{
"id": 100,
"postId": 10,
"parentId": null,
"author": {
"nickname": "용감한 고래" // 이 스레드에서의 게시글과 같은 닉네임
},
"content": "저도 동의합니다",
"likeCount": 0,
"dislikeCount": 0,
"isDeleted": false,
"createdAt": "2026-04-01T12:30:00Z"
}

에러:
400 Bad Request — parentId가 답글을 참조 (중첩 답글 불가)
404 Not Found — postId 또는 parentId가 존재하지 않음

```

#### 게시글의 댓글 목록 조회 (커서 기반, 최상위만)

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
"nickname": "용감한 고래"
},
"content": "저도 동의합니다",
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
"nickname": "조용한 돌고래"
},
"content": "저도요!",
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
"nickname": "고요한 상어"
},
"content": null, // 소프트 삭제됨
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

최상위 댓글은 커서 기반으로 페이지네이션됩니다 (`created_at ASC` 정렬). 각 최상위 댓글은 답글을 인라인으로 포함합니다 (답글은 별도로 페이지네이션되지 않음 — 최대 깊이가 1이므로 댓글당 답글 수가 관리 가능한 수준).

게시글의 모든 댓글(최상위 및 답글)을 닉네임과 현재 사용자의 반응과 함께 조회하는 쿼리:
```sql
SELECT c.*, an.nickname, cr.type AS my_reaction
FROM comments c
LEFT JOIN anonymous_nicknames an ON an.post_id = c.post_id AND an.user_id = c.user_id
LEFT JOIN comment_reactions cr ON cr.comment_id = c.id AND cr.user_id = :currentUserId
WHERE c.post_id = :postId
  AND (
    -- 커서 윈도우 내의 최상위 댓글
    (c.parent_id IS NULL AND (c.created_at, c.id) > (:cursorCreatedAt, :cursorId))
    -- 또는 해당 최상위 댓글에 속하는 답글
    OR c.parent_id IN (:topLevelCommentIds)
  )
ORDER BY c.parent_id NULLS FIRST, c.created_at ASC
````

실제로는 두 개의 쿼리로 실행됩니다: 먼저 페이지네이션된 최상위 댓글을 (JOIN과 함께) 조회하고, 그 최상위 ID들에 대한 모든 답글을 (JOIN과 함께) 조회합니다. 앱이 메모리에서 답글을 부모 아래에 그룹화합니다.

```
  "nextCursor": "eyJjcmVhdGVkQXQiOi4uLiwiaWQiOjEwMn0=",
  "hasNext": true
}
```

#### 댓글 수정

```
PUT /api/comments/{commentId}
X-User-Id: user_1

Request:
{
  "content": "수정된 댓글"
}

Response: 200 OK
{ ...수정된 댓글 객체... }

동작:
  - 매 업데이트마다 updated_at을 now()로 설정

에러:
  403 Forbidden — 작성자가 아님
  404 Not Found
  400 Bad Request — 댓글이 이미 삭제됨
```

#### 댓글 삭제

```
DELETE /api/comments/{commentId}
X-User-Id: user_1

Response: 204 No Content

동작:
  - 답글이 있는 댓글 → 소프트 삭제 (is_deleted = true, content = null)
  - 답글이 없는 댓글 → 하드 삭제 (행 제거)
  - 답글인 경우 → 하드 삭제 (행 제거)

에러:
  403 Forbidden — 작성자가 아님
  404 Not Found
```

### 2.4 반응

#### 게시글 반응 토글

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
  "type": "LIKE",            // 현재 반응 타입, 취소 시 null
  "likeCount": 6,            // 업데이트된 카운트
  "dislikeCount": 1
}

로직:
  - 기존 반응 없음            → INSERT, action = ADDED
  - 같은 타입 존재            → DELETE, action = CANCELLED
  - 다른 타입 존재            → UPDATE, action = SWITCHED
```

#### 댓글 반응 토글

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

### 2.5 공통 에러 응답 형식

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "이 게시글의 작성자가 아닙니다"
  }
}
```

표준 HTTP 상태 코드:

- `400` — 잘못된 요청 (유효성 검증 오류, 중첩 답글 시도)
- `403` — 작성자가 아님
- `404` — 리소스를 찾을 수 없음
- `500` — 내부 서버 오류

## 3. 핵심 비즈니스 로직

### 3.1 익명 닉네임 생성

**트리거 시점:** 사용자가 ANONYMOUS 채널에서 게시글 또는 댓글을 작성할 때마다.

**닉네임 형식:** `{형용사} {해양생물}` — 예: "용감한 고래", "조용한 돌고래", "고요한 상어"

- 형용사 풀: ~50개 단어 (용감한, 조용한, 고요한, 빠른, 대담한, 차분한, ...)
- 명사 풀: ~50개 해양 생물 (고래, 돌고래, 상어, 문어, 거북이, ...)
- 총 조합: ~2,500개 — 단일 게시글의 참여자 수로 충분

**알고리즘:**

```
function getOrCreateNickname(postId, userId):
    MAX_RETRIES = 3

    for attempt in 1..MAX_RETRIES:
        nickname = generateRandomNickname()   // 무작위 형용사 + 명사
        if attempt == MAX_RETRIES:
            nickname = nickname + " " + randomInt(1, 9999)  // 폴백: 숫자 추가

        try:
            result = INSERT INTO anonymous_nicknames (post_id, user_id, nickname)
                     VALUES (:postId, :userId, :nickname)
                     ON CONFLICT (post_id, user_id) DO NOTHING

            if result.rowsAffected == 0:
                // 이 게시글에 대해 사용자가 이미 닉네임을 보유
                return SELECT nickname FROM anonymous_nicknames
                       WHERE post_id = :postId AND user_id = :userId

            // 삽입 성공 — 새 닉네임 반환
            return nickname

        catch UNIQUE_VIOLATION on (post_id, nickname):
            // 다른 사용자가 이 게시글에서 같은 닉네임을 받음 — 재시도
            continue

    // 여기에 도달하지 않아야 함 (숫자가 추가된 폴백은 거의 확실히 유일)
```

**동시성 안전성:**

- `UNIQUE(post_id, user_id)` — 한 사용자가 같은 게시글에서 두 개의 닉네임을 받는 것을 방지
- `UNIQUE(post_id, nickname)` — 게시글당 두 사용자가 같은 닉네임을 받는 것을 방지
- 두 제약조건 모두 앱 로직이 아닌 DB에 의해 강제
- 패턴: **낙관적 동시성** — 쓰기를 시도하고, DB가 충돌을 거부하면, 적절히 처리

**REAL_NAME 채널에서:** 닉네임 로직을 완전히 건너뜁니다. user_id를 작성자 식별자로 반환합니다.

### 3.2 반응 토글

**트리거 시점:** `POST /api/posts/{postId}/reactions` 또는 `POST /api/comments/{commentId}/reactions`

**알고리즘 (게시글과 댓글에 동일한 로직):**

```
@Transactional
function toggleReaction(targetId, userId, newType):

    // Step 1: 기존 반응 확인
    existing = SELECT * FROM post_reactions
               WHERE post_id = :targetId AND user_id = :userId

    // Step 2: 기존 반응 없음 → 추가
    if existing == null:
        INSERT INTO post_reactions (post_id, user_id, type) VALUES (...)
        UPDATE posts SET like_count = like_count + 1   // 또는 dislike_count
            WHERE id = :targetId
        return { action: "ADDED", type: newType }

    // Step 3: 같은 타입 → 취소 (반응 제거)
    if existing.type == newType:
        DELETE FROM post_reactions WHERE id = existing.id
        UPDATE posts SET like_count = like_count - 1   // 또는 dislike_count
            WHERE id = :targetId
        return { action: "CANCELLED", type: null }

    // Step 4: 다른 타입 → 전환
    UPDATE post_reactions SET type = :newType WHERE id = existing.id
    UPDATE posts SET like_count = like_count + 1,      // 새 타입 증가
                     dislike_count = dislike_count - 1  // 이전 타입 감소
        WHERE id = :targetId
    return { action: "SWITCHED", type: newType }
```

**동시성 안전성:**

- `UNIQUE(post_id, user_id)`가 DB 레벨에서 중복 반응을 방지
- 더블클릭으로 두 개의 동시 요청이 발생하면:
  - 요청 A: SELECT → null, INSERT → 성공
  - 요청 B: SELECT → null, INSERT → 유니크 제약조건 위반
  - 위반을 캐치하고, 다시 SELECT, 토글 로직 재실행 (이제 A의 반응을 보고 취소 — 올바른 동작)
- 비정규화된 카운트는 같은 `@Transactional` 내에서 업데이트 — 반응 변경과 원자적

**카운트 안전성:** 카운트는 절대 0 미만이 될 수 없음 — 카운트 컬럼의 `CHECK (>= 0)` 제약조건으로 강제 (DDL, 섹션 1.2에서 정의). 과도한 감소 버그는 음수를 생성하는 대신 제약조건 위반을 발생시킵니다.

### 3.3 댓글 소프트 삭제

**트리거 시점:** `DELETE /api/comments/{commentId}`

**알고리즘:**

```
@Transactional
function deleteComment(commentId, userId):

    // Step 1: 댓글 로드
    comment = SELECT * FROM comments WHERE id = :commentId
    if comment == null: return 404
    if comment.user_id != userId: return 403
    if comment.is_deleted: return 404   // 이미 삭제됨

    // Step 2: 답글인가? (parent_id가 있는지)
    if comment.parent_id != null:
        // 답글은 항상 하드 삭제
        DELETE FROM comments WHERE id = :commentId
        UPDATE posts SET comment_count = comment_count - 1
            WHERE id = comment.post_id
        cleanupParentIfNeeded(comment.parent_id)   // 소프트 삭제된 부모에 남은 답글이 없으면 부모도 제거
        return 204

    // Step 3: 최상위 댓글 — 답글 확인
    replyCount = SELECT COUNT(*) FROM comments
                 WHERE parent_id = :commentId AND is_deleted = FALSE

    if replyCount == 0:
        // 활성 답글 없음 — 하드 삭제
        // (소프트 삭제된 답글도 먼저 하드 삭제)
        DELETE FROM comments WHERE parent_id = :commentId
        DELETE FROM comments WHERE id = :commentId
        UPDATE posts SET comment_count = comment_count - 1
            WHERE id = comment.post_id
        return 204
    else:
        // 활성 답글 있음 — 소프트 삭제
        UPDATE comments SET is_deleted = TRUE, content = NULL, updated_at = now()
            WHERE id = :commentId
        // comment_count를 감소시키지 않음 (플레이스홀더가 여전히 표시됨)
        return 204
```

**정리 기회:** 소프트 삭제된 부모 아래의 마지막 답글이 삭제되면, 부모도 완전히 제거할 수 있습니다:

```
// 답글을 하드 삭제한 후, 부모를 정리해야 하는지 확인
function cleanupParentIfNeeded(parentId):
    parent = SELECT * FROM comments WHERE id = :parentId
    if parent.is_deleted:
        remainingReplies = SELECT COUNT(*) FROM comments WHERE parent_id = :parentId
        if remainingReplies == 0:
            DELETE FROM comments WHERE id = :parentId
            UPDATE posts SET comment_count = comment_count - 1
                WHERE id = parent.post_id
```

### 3.4 조회수

**트리거 시점:** `GET /api/posts/{postId}` (게시글 상세 조회)

**알고리즘:**

```sql
UPDATE posts SET view_count = view_count + 1 WHERE id = :postId
```

**설계 결정 — 누적 카운트, 고유 조회수 아님:**

- 어떤 사용자가 어떤 게시글을 봤는지 추적하지 않음
- 모든 상세 조회 요청마다 카운터 증가
- 고유 조회 추적은 `post_views` 테이블 (post_id, user_id, viewed_at)이 필요하며, 모든 읽기 작업에 쓰기 오버헤드를 추가
- 선원 커뮤니티에서는 누적 카운트로 충분하며 훨씬 간단
- 이것은 모호한 요구사항 — decisions.md에 문서화

## 4. 엣지 케이스

| #   | 시나리오                                       | 처리 방법                                                                                                                                                                                   |
| --- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **X-User-Id 헤더 누락 또는 비어있음**          | 400 Bad Request 반환. 모든 엔드포인트는 사용자 식별이 필요.                                                                                                                                 |
| 2   | **존재하지 않는 채널 또는 게시글에 작성**      | 404 Not Found 반환. INSERT 전에 FK 존재 여부 검증.                                                                                                                                          |
| 3   | **소프트 삭제된 댓글에 반응**                  | 400 Bad Request 반환. 삭제된 댓글은 새 반응을 받을 수 없음.                                                                                                                                 |
| 4   | **소프트 삭제된 댓글 수정**                    | 400 Bad Request 반환. 소프트 삭제 후 내용을 복원할 수 없음.                                                                                                                                 |
| 5   | **소프트 삭제된 댓글에 답글**                  | 400 Bad Request 반환. 삭제된 댓글 아래에서 대화를 시작할 수 없음.                                                                                                                           |
| 6   | **댓글, 반응, 닉네임이 있는 게시글 삭제**      | CASCADE가 정리를 처리. 모든 하위 행(댓글, 반응, 닉네임)이 DB에 의해 자동 제거.                                                                                                              |
| 7   | **비어있거나 공백만 있는 제목/내용**           | 400 Bad Request 반환. 공백만 있는 문자열 거부. DB INSERT 전에 앱 레벨에서 검증.                                                                                                             |
| 8   | **닉네임 풀 소진**                             | 마지막 재시도(3번째 시도)에서 닉네임에 무작위 숫자를 추가 (예: "용감한 고래 42"). 2,500개 기본 조합을 넘어서는 유일성을 보장. 사용자의 액션은 항상 성공 — 닉네임 생성이 절대 실패하지 않음. |
| 9   | **자신의 게시글/댓글에 반응**                  | 허용. 제한 없음 — 다른 사람의 콘텐츠에 반응하는 것과 같은 동작.                                                                                                                             |
| 10  | **유효하지 않거나 변조된 커서 파라미터**       | 400 Bad Request 반환, "invalid cursor" 메시지. try-catch에서 디코딩, 잘못된 값 거부.                                                                                                        |
| 11  | **익명 채널에서 게시글 수정**                  | 닉네임은 동일하게 유지. (post_id, user_id)에 연결되어 있으며, 내용이 아님. 내용 수정이 닉네임을 재생성하거나 변경하지 않음.                                                                 |
| 12  | **익명 채널에서 동시 게시글 작성**             | 두 사용자가 동시에 게시글을 작성. 각각 INSERT ON CONFLICT 패턴으로 자체 닉네임을 받음. 다른 user_id이므로 충돌 불가.                                                                        |
| 13  | **소프트 삭제된 부모 아래의 유일한 답글 삭제** | 답글을 하드 삭제한 후 부모를 정리: 부모가 소프트 삭제되었고 답글이 0개이므로 부모도 하드 삭제. 둘 다 comment_count 감소.                                                                    |
| 14  | **존재하지 않는 게시글/댓글에 반응**           | 404 Not Found 반환.                                                                                                                                                                         |
| 15  | **중첩 답글 시도 (답글의 답글)**               | 400 Bad Request 반환. 앱이 삽입 전에 `parent.parent_id IS NULL`인지 확인.                                                                                                                   |

## 5. 테스트 시나리오 (Given-When-Then)

### 5.1 익명 닉네임

**T1: 익명 채널에서 첫 사용자가 닉네임을 받음**

- Given: ANONYMOUS 채널에 게시글(id=10)이 있고, user_1은 이 게시글에 대한 닉네임이 없음
- When: user_1이 게시글 10에 댓글 작성
- Then: 닉네임이 생성되어 anonymous_nicknames에 저장되고 응답에 반환

**T2: 같은 사용자가 같은 게시글에서 같은 닉네임을 받음**

- Given: user_1이 게시글 10에 대해 "용감한 고래" 닉네임을 보유
- When: user_1이 게시글 10에 또 다른 댓글 작성
- Then: author.nickname은 "용감한 고래" (재사용, 재생성 아님)

**T3: 같은 사용자가 다른 게시글에서 다른 닉네임을 받음**

- Given: user_1이 게시글 10에 대해 "용감한 고래" 닉네임을 보유
- When: user_1이 게시글 20에 댓글 작성
- Then: 새로운 닉네임이 독립적으로 생성 ("용감한 고래"가 아님)

**T4: 동시성 하에서 닉네임 충돌**

- Given: 게시글 10에 아직 닉네임이 없음
- When: 두 사용자가 동시에 "용감한 고래"를 배정받음
- Then: 하나는 성공하고 다른 하나는 다른 닉네임으로 재시도. 둘 다 고유한 닉네임을 받음.

**T5: 실명 채널에서는 닉네임 건너뜀**

- Given: REAL_NAME 채널에 게시글이 있음
- When: user_1이 댓글 작성
- Then: author.userId는 "user_1", author.nickname은 null

### 5.2 반응

**T6: 반응 추가**

- Given: 게시글 10에 좋아요 0개, user_1에 반응 없음
- When: user_1이 POST /api/posts/10/reactions {"type": "LIKE"} 전송
- Then: action = "ADDED", likeCount = 1, dislikeCount = 0

**T7: 반응 취소 (같은 타입 재클릭)**

- Given: user_1이 게시글 10에 LIKE 보유, likeCount = 1
- When: user_1이 POST /api/posts/10/reactions {"type": "LIKE"} 전송
- Then: action = "CANCELLED", type = null, likeCount = 0

**T8: 반응 전환**

- Given: user_1이 게시글 10에 LIKE 보유, likeCount = 1, dislikeCount = 0
- When: user_1이 POST /api/posts/10/reactions {"type": "DISLIKE"} 전송
- Then: action = "SWITCHED", type = "DISLIKE", likeCount = 0, dislikeCount = 1

**T9: 더블클릭 반응 (동시성)**

- Given: user_1이 게시글 10에 반응 없음
- When: 동일한 LIKE 요청 두 개가 동시에 도착
- Then: 하나가 추가하고, 다른 하나가 제약조건 위반을 캐치하여 취소. 최종 상태: 반응 없음 (순 토글).

**T10: 소프트 삭제된 댓글에 반응**

- Given: 댓글 100이 소프트 삭제됨 (is_deleted = true)
- When: user_1이 POST /api/comments/100/reactions {"type": "LIKE"} 전송
- Then: 400 Bad Request

### 5.3 댓글 & 소프트 삭제

**T11: 최상위 댓글 작성**

- Given: 게시글 10 존재, comment_count = 0
- When: user_1이 POST /api/comments {"postId": 10, "parentId": null, "content": "안녕하세요"} 전송
- Then: 댓글 생성, 게시글 comment_count = 1

**T12: 답글 작성**

- Given: 게시글 10에 최상위 댓글 100이 존재
- When: user_2가 POST /api/comments {"postId": 10, "parentId": 100, "content": "답글"} 전송
- Then: parentId = 100인 답글 생성, 게시글 comment_count 증가

**T13: 중첩 답글 거부 (답글의 답글)**

- Given: 댓글 101이 답글 (parent_id = 100)
- When: user_3가 POST /api/comments {"postId": 10, "parentId": 101, "content": "중첩"} 전송
- Then: 400 Bad Request

**T14: 답글이 있는 댓글 소프트 삭제**

- Given: 댓글 100에 답글 2개, 게시글 comment_count = 3
- When: 작성자가 댓글 100 삭제
- Then: 댓글 100의 is_deleted = true, content = null. 답글은 유지. comment_count는 3으로 유지.

**T15: 답글이 없는 댓글 하드 삭제**

- Given: 댓글 100에 답글 없음, 게시글 comment_count = 1
- When: 작성자가 댓글 100 삭제
- Then: 댓글 100 행이 DB에서 제거. comment_count = 0.

**T16: 답글 하드 삭제**

- Given: 댓글 101이 댓글 100의 답글, 게시글 comment_count = 2
- When: 작성자가 댓글 101 삭제
- Then: 댓글 101 행 제거. comment_count = 1.

**T17: 소프트 삭제된 부모 아래의 마지막 답글 삭제**

- Given: 댓글 100이 소프트 삭제, 남은 답글 1개(101), comment_count = 2
- When: 작성자가 답글 101 삭제
- Then: 답글 101 제거. 부모 100도 제거(정리). comment_count = 0.

**T18: 소프트 삭제된 댓글에 답글**

- Given: 댓글 100이 소프트 삭제됨
- When: 사용자가 POST /api/comments {"postId": 10, "parentId": 100, "content": "답글"} 전송
- Then: 400 Bad Request

### 5.4 게시글

**T19: 상세 조회 시 조회수 증가**

- Given: 게시글 10의 view_count = 5
- When: GET /api/posts/10
- Then: 응답에 viewCount = 6

**T20: 작성자만 게시글 수정 가능**

- Given: 게시글 10의 작성자가 user_1
- When: user_2가 PUT /api/posts/10 전송
- Then: 403 Forbidden

**T21: 작성자만 게시글 삭제 가능**

- Given: 게시글 10의 작성자가 user_1
- When: user_2가 DELETE /api/posts/10 전송
- Then: 403 Forbidden

**T22: 게시글 삭제 시 모든 하위 항목 캐스케이드**

- Given: 게시글 10에 댓글 3개, 반응 5개, 닉네임 2개
- When: 작성자가 게시글 10 삭제
- Then: 게시글 행 제거. 게시글 10의 모든 댓글, 반응, 닉네임이 CASCADE로 제거.

### 5.5 페이지네이션

**T23: 첫 페이지 (커서 없이)**

- Given: 채널 1에 게시글 25개
- When: GET /api/posts?channelId=1&size=20
- Then: 20개 게시글 반환 (최신순), hasNext = true, nextCursor 설정

**T24: 두 번째 페이지 (커서 사용)**

- Given: 채널 1에 게시글 25개, T23의 커서
- When: GET /api/posts?channelId=1&size=20&cursor={nextCursor}
- Then: 5개 게시글 반환, hasNext = false, nextCursor는 null

**T25: 유효하지 않은 커서**

- Given: 임의의 상태
- When: GET /api/posts?channelId=1&size=20&cursor=invalidgarbage
- Then: 400 Bad Request

### 5.6 유효성 검증

**T26: X-User-Id 헤더 누락**

- Given: 인증이 필요한 임의의 엔드포인트
- When: X-User-Id 헤더 없이 요청 전송
- Then: 400 Bad Request

**T27: 비어있는 게시글 제목 또는 내용**

- Given: 유효한 채널 존재
- When: POST /api/posts {"channelId": 1, "title": "", "content": "텍스트"}
- Then: 400 Bad Request
