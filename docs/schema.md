# Supabase 테이블 설계 문서

## 1. 개요

- DB: Supabase PostgreSQL
- 인증: Supabase Auth (Google, GitHub OAuth)
- 관리자 식별: `ADMIN_EMAIL` 환경 변수와 `auth.users.email` 비교
- RLS(Row Level Security): 모든 테이블에 활성화

---

## 2. 테이블 스키마

### 2.1 posts

```sql
CREATE TABLE posts (
  id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  slug          TEXT        NOT NULL UNIQUE,
  title         TEXT        NOT NULL DEFAULT '',
  content       TEXT        NOT NULL DEFAULT '',
  status        TEXT        NOT NULL DEFAULT 'draft'
                            CHECK (status IN ('draft', 'published')),
  author_id     UUID        NOT NULL REFERENCES auth.users(id),
  thumbnail_url TEXT,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  published_at  TIMESTAMPTZ,
  view_count    INTEGER     NOT NULL DEFAULT 0,
  like_count    INTEGER     NOT NULL DEFAULT 0
);

CREATE INDEX idx_posts_status     ON posts(status);
CREATE INDEX idx_posts_published  ON posts(published_at DESC) WHERE status = 'published';
CREATE INDEX idx_posts_slug       ON posts(slug);
```

**설계 메모**

| 컬럼 | 설명 |
|------|------|
| `slug` | 라우팅 및 SEO 식별자. 제목 기반 생성, 발행 후 변경 불가 |
| `status` | `draft` / `published`. 초안 생성 시 즉시 레코드 삽입 |
| `view_count` | 조회수 기반 인기 글 정렬용 |
| `like_count` | `likes` 테이블과 항상 동기화 (DB trigger 사용) |
| `published_at` | 처음 발행 시 한 번 설정. 이후 변경 없음 |

---

### 2.2 tags

```sql
CREATE TABLE tags (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT        NOT NULL UNIQUE,
  slug        TEXT        NOT NULL UNIQUE,
  description TEXT,
  color       TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tags_slug ON tags(slug);
```

---

### 2.3 post_tags

```sql
CREATE TABLE post_tags (
  post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  tag_id  UUID NOT NULL REFERENCES tags(id)  ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
);

CREATE INDEX idx_post_tags_tag_id ON post_tags(tag_id);
```

---

### 2.4 comments

```sql
CREATE TABLE comments (
  id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id    UUID        NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  user_id    UUID        NOT NULL REFERENCES auth.users(id),
  nickname   TEXT        NOT NULL,
  content    TEXT        NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ,
  is_edited  BOOLEAN     NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_comments_post_id   ON comments(post_id);
CREATE INDEX idx_comments_user_id   ON comments(user_id);
CREATE INDEX idx_comments_active    ON comments(post_id) WHERE deleted_at IS NULL;
```

**설계 메모**

- 소프트 삭제(`deleted_at`)를 사용한다. 삭제된 댓글은 "삭제된 댓글입니다"로 표시 가능.
- `is_edited` + `updated_at` 조합으로 수정 여부를 UI에 표시한다.
- `nickname`은 소셜 계정의 display_name을 댓글 작성 시점에 스냅샷으로 저장한다.

---

### 2.5 likes

```sql
CREATE TABLE likes (
  id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id    UUID        NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  user_id    UUID        NOT NULL REFERENCES auth.users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (post_id, user_id)
);

CREATE INDEX idx_likes_post_id ON likes(post_id);
CREATE INDEX idx_likes_user_id ON likes(user_id);
```

**설계 메모**

- `UNIQUE (post_id, user_id)` 제약으로 DB 레벨에서 중복 좋아요를 방지한다.
- INSERT / DELETE 시 trigger가 `posts.like_count`를 자동 갱신한다.

---

### 2.6 files

```sql
CREATE TABLE files (
  id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id      UUID        REFERENCES posts(id) ON DELETE SET NULL,
  storage_path TEXT        NOT NULL UNIQUE,
  public_url   TEXT        NOT NULL,
  type         TEXT        NOT NULL CHECK (type IN ('thumbnail', 'content_image')),
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_files_post_id ON files(post_id);
```

**설계 메모**

- 초안 단계의 이미지는 `post_id = NULL`로 저장될 수 있다.
- 초안 생성 시 먼저 posts 레코드를 만들고 그 id를 연결하는 방식을 권장한다.
- 게시글 삭제 순서: Storage 파일 삭제 → `files` 레코드 삭제 → `posts` 레코드 삭제 (애플리케이션 레벨에서 직접 처리).
- `post_id ON DELETE SET NULL`은 안전장치다. 애플리케이션 외부에서 `posts`가 삭제되는 경우 `files` 레코드가 `post_id = NULL`로 남아 고아 파일 정리 대상으로 식별될 수 있다.

---

## 3. DB Trigger / Function

### 3.1 like_count 동기화

`likes` INSERT 시 `posts.like_count`를 1 증가시킨다.

```sql
CREATE OR REPLACE FUNCTION increment_like_count()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE posts
  SET like_count = like_count + 1
  WHERE id = NEW.post_id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_increment_like_count
AFTER INSERT ON likes
FOR EACH ROW EXECUTE FUNCTION increment_like_count();
```

`likes` DELETE 시 `posts.like_count`를 1 감소시킨다.

```sql
CREATE OR REPLACE FUNCTION decrement_like_count()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE posts
  SET like_count = GREATEST(like_count - 1, 0)
  WHERE id = OLD.post_id;
  RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_decrement_like_count
AFTER DELETE ON likes
FOR EACH ROW EXECUTE FUNCTION decrement_like_count();
```

---

### 3.2 updated_at 자동 갱신

`posts`, `comments` 테이블의 `updated_at`을 자동으로 갱신한다.

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_posts_updated_at
BEFORE UPDATE ON posts
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER trg_comments_updated_at
BEFORE UPDATE ON comments
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

---

### 3.3 view_count 증가 (RPC)

클라이언트에서 직접 UPDATE하지 않고 RPC를 통해 원자적으로 증가시킨다.

```sql
CREATE OR REPLACE FUNCTION increment_view_count(target_slug TEXT)
RETURNS void AS $$
BEGIN
  UPDATE posts
  SET view_count = view_count + 1
  WHERE slug = target_slug AND status = 'published';
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

호출 예시 (클라이언트/서버):

```ts
await supabase.rpc('increment_view_count', { target_slug: slug });
```

---

## 4. RLS 정책

> 모든 테이블에 `ALTER TABLE <table> ENABLE ROW LEVEL SECURITY;` 선행 필요

### 4.1 posts

```sql
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- 누구나 발행된 글 조회
CREATE POLICY "posts: public read published"
ON posts FOR SELECT
USING (status = 'published');

-- 관리자만 초안 포함 전체 조회
CREATE POLICY "posts: admin read all"
ON posts FOR SELECT
USING (auth.email() = current_setting('app.admin_email', true));

-- 관리자만 INSERT
CREATE POLICY "posts: admin insert"
ON posts FOR INSERT
WITH CHECK (auth.email() = current_setting('app.admin_email', true));

-- 관리자만 UPDATE
CREATE POLICY "posts: admin update"
ON posts FOR UPDATE
USING (auth.email() = current_setting('app.admin_email', true));

-- 관리자만 DELETE
CREATE POLICY "posts: admin delete"
ON posts FOR DELETE
USING (auth.email() = current_setting('app.admin_email', true));
```

> `app.admin_email`은 Supabase Dashboard → Settings → Database → Configuration에서
> `ALTER DATABASE postgres SET app.admin_email = 'your@email.com';` 으로 설정하거나,
> 서버 액션에서 Service Role Key를 사용해 우회한다.
> 실용적인 대안: 관리자 전용 작업은 서버 액션에서 `SUPABASE_SERVICE_ROLE_KEY`를 사용한다.

---

### 4.2 tags

```sql
ALTER TABLE tags ENABLE ROW LEVEL SECURITY;

-- 누구나 조회
CREATE POLICY "tags: public read"
ON tags FOR SELECT USING (true);

-- 관리자만 쓰기
CREATE POLICY "tags: admin write"
ON tags FOR ALL
USING (auth.email() = current_setting('app.admin_email', true))
WITH CHECK (auth.email() = current_setting('app.admin_email', true));
```

---

### 4.3 post_tags

```sql
ALTER TABLE post_tags ENABLE ROW LEVEL SECURITY;

CREATE POLICY "post_tags: public read"
ON post_tags FOR SELECT USING (true);

CREATE POLICY "post_tags: admin write"
ON post_tags FOR ALL
USING (auth.email() = current_setting('app.admin_email', true))
WITH CHECK (auth.email() = current_setting('app.admin_email', true));
```

---

### 4.4 comments

```sql
ALTER TABLE comments ENABLE ROW LEVEL SECURITY;

-- 삭제되지 않은 댓글은 누구나 조회
CREATE POLICY "comments: public read active"
ON comments FOR SELECT
USING (deleted_at IS NULL);

-- 로그인한 사용자는 댓글 작성
CREATE POLICY "comments: authenticated insert"
ON comments FOR INSERT
WITH CHECK (auth.uid() = user_id);

-- 본인 댓글만 수정 (소프트 삭제 포함)
CREATE POLICY "comments: own update"
ON comments FOR UPDATE
USING (auth.uid() = user_id);

-- 관리자는 모든 댓글 수정/삭제 가능
CREATE POLICY "comments: admin all"
ON comments FOR ALL
USING (auth.email() = current_setting('app.admin_email', true));
```

---

### 4.5 likes

```sql
ALTER TABLE likes ENABLE ROW LEVEL SECURITY;

-- 누구나 좋아요 수 조회 (집계 용도)
CREATE POLICY "likes: public read"
ON likes FOR SELECT USING (true);

-- 로그인한 사용자만 좋아요 추가 (본인 user_id만)
CREATE POLICY "likes: authenticated insert"
ON likes FOR INSERT
WITH CHECK (auth.uid() = user_id);

-- 본인 좋아요만 취소
CREATE POLICY "likes: own delete"
ON likes FOR DELETE
USING (auth.uid() = user_id);
```

---

### 4.6 files

```sql
ALTER TABLE files ENABLE ROW LEVEL SECURITY;

-- 관리자만 전체 접근
CREATE POLICY "files: admin all"
ON files FOR ALL
USING (auth.email() = current_setting('app.admin_email', true))
WITH CHECK (auth.email() = current_setting('app.admin_email', true));
```

---

## 5. 관리자 RLS 운영 패턴

RLS에서 `current_setting('app.admin_email')`을 사용하는 대신, Next.js 서버 액션에서는 `SUPABASE_SERVICE_ROLE_KEY`로 생성한 `supabaseAdmin` 클라이언트를 사용한다. 이 클라이언트는 RLS를 우회하므로 서버에서만 사용해야 한다.

```ts
// lib/supabase/admin.ts (서버 전용)
import { createClient } from '@supabase/supabase-js';

export const supabaseAdmin = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!, // 절대 클라이언트에 노출 금지
);
```

서버 액션에서 관리자 여부는 Supabase Auth 세션의 `user.email`과 `ADMIN_EMAIL` 환경 변수를 비교해 검증한다.

```ts
// lib/auth/is-admin.ts
export async function requireAdmin(supabase: SupabaseClient) {
  const { data: { user } } = await supabase.auth.getUser();
  if (user?.email !== process.env.ADMIN_EMAIL) {
    throw new Error('Unauthorized');
  }
  return user;
}
```
