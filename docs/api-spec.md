# API 스펙 설계 문서

## 1. 개요

Next.js App Router 기반으로 설계한다.

- **Server Actions**: 데이터 변경(CUD) 작업에 사용
- **Server Components**: 정적·준정적 데이터 조회에 사용 (posts 목록, 상세, 태그 등)
- **Route Handlers**: SEO(sitemap, robots, RSS), 조회수 증가 같은 GET 전용 엔드포인트
- **TanStack Query**: 댓글·좋아요 등 클라이언트 상호작용 데이터

---

## 2. 권한 구분

| 역할 | 조건 |
|------|------|
| Public | 인증 불필요 |
| Visitor | Supabase Auth 소셜 로그인 완료 (`auth.uid()` 존재) |
| Admin | `user.email === process.env.ADMIN_EMAIL` |

---

## 3. Server Actions

> 파일 위치: `app/actions/` 또는 각 기능 폴더 내 `actions.ts`
> 모든 Server Action은 `'use server'` 지시어를 사용한다.

---

### 3.1 게시글 (posts)

#### `createDraft()`

새 초안을 생성하고 `postId`를 반환한다. 글쓰기 버튼 클릭 시 호출.

```ts
async function createDraft(): Promise<
  | { ok: true; postId: string; slug: string }
  | { ok: false; message: string }
>
```

- **권한**: Admin
- **동작**:
  1. 관리자 검증
  2. `status = 'draft'`, 빈 제목/본문으로 posts 레코드 생성
  3. 임시 slug 생성 (`draft-{uuid 앞 8자}`)
  4. `postId`, `slug` 반환
- **에러**: 인증 실패 → `{ ok: false, message: 'Unauthorized' }`

---

#### `saveDraft(postId, data)`

초안 내용을 저장(자동 저장 포함)한다.

```ts
type SaveDraftInput = {
  title?: string;
  content?: string;
  tagIds?: string[];
  thumbnailFileId?: string | null;  // uploadImage()가 반환한 fileId
};

async function saveDraft(postId: string, data: SaveDraftInput): Promise<
  | { ok: true }
  | { ok: false; message: string }
>
```

- **권한**: Admin
- **동작**:
  1. 관리자 검증
  2. `status = 'draft'` 인 post 존재 확인
  3. 제목이 있으면 slug 재계산 (발행 전 단계에만 허용)
  4. `thumbnailFileId`가 있으면 `files` 조회 → `public_url`을 `posts.thumbnail_url`에 반영. `files.post_id`가 `null`이면 현재 `postId`로 갱신 (재연결)
  5. `posts` 업데이트
  6. `tagIds`가 있으면 `post_tags` upsert (기존 행 전체 교체)
- **자동 저장**: 클라이언트에서 20~30초 간격으로 호출
- **이미지 연결 흐름**: `createDraft()` → `uploadImage({ postId, ... })` → `saveDraft({ thumbnailFileId })`. 정상 플로우에서 모든 업로드는 `postId`를 포함하므로 orphan이 발생하지 않는다.

---

#### `publishPost(postId)`

초안을 발행 상태로 변경한다.

```ts
async function publishPost(postId: string): Promise<
  | { ok: true; slug: string }
  | { ok: false; message: string }
>
```

- **권한**: Admin
- **동작**:
  1. 관리자 검증
  2. `status = 'draft'` 확인
  3. 제목·본문 유효성 검사 (비어 있으면 실패)
  4. `status = 'published'`, `published_at = NOW()` 업데이트
  5. slug 확정 (발행 후 변경 불가)
  6. 발행된 `slug` 반환

---

#### `updatePost(postId, data)`

발행된 게시글을 수정한다.

```ts
type UpdatePostInput = {
  title?: string;
  content?: string;
  tagIds?: string[];
  thumbnailUrl?: string | null;
};

async function updatePost(postId: string, data: UpdatePostInput): Promise<
  | { ok: true }
  | { ok: false; message: string }
>
```

- **권한**: Admin
- **동작**:
  1. 관리자 검증
  2. post 존재 확인
  3. `posts` 업데이트 (slug는 변경하지 않음)
  4. `tagIds`가 있으면 `post_tags` 교체

---

#### `deletePost(postId)`

게시글(초안 포함)을 삭제하고 연결된 Storage 파일을 정리한다.

```ts
async function deletePost(postId: string): Promise<
  | { ok: true }
  | { ok: false; message: string }
>
```

- **권한**: Admin
- **동작**:
  1. 관리자 검증
  2. `files` 테이블에서 해당 `post_id`의 파일 목록 조회
  3. Supabase Storage에서 파일 일괄 삭제
  4. `files` 레코드 삭제 (`files.post_id`는 `ON DELETE SET NULL`이므로 명시적으로 삭제)
  5. `posts` 레코드 삭제 (CASCADE로 `post_tags`, `comments`, `likes` 연쇄 삭제)
- **주의**: Storage 삭제 실패 시 이후 단계를 진행하지 않는다 (orphan 파일 방지)

---

### 3.2 태그 (tags)

#### `createTag(data)`

```ts
type CreateTagInput = {
  name: string;
  slug: string;
  description?: string;
  color?: string;
};

async function createTag(data: CreateTagInput): Promise<
  | { ok: true; tagId: string }
  | { ok: false; message: string }
>
```

- **권한**: Admin
- **동작**: `name`, `slug` 중복 검사 후 삽입

---

#### `updateTag(tagId, data)`

```ts
async function updateTag(tagId: string, data: Partial<CreateTagInput>): Promise<
  | { ok: true }
  | { ok: false; message: string }
>
```

- **권한**: Admin

---

#### `deleteTag(tagId)`

```ts
async function deleteTag(tagId: string): Promise<
  | { ok: true }
  | { ok: false; message: string }
>
```

- **권한**: Admin
- **동작**: `post_tags` CASCADE로 연결 관계 자동 삭제

---

### 3.3 댓글 (comments)

#### `addComment(postId, content)`

```ts
async function addComment(postId: string, content: string): Promise<
  | { ok: true; commentId: string }
  | { ok: false; message: string }
>
```

- **권한**: Visitor (소셜 로그인)
- **동작**:
  1. `auth.getUser()`로 사용자 확인
  2. 소셜 계정 `user.user_metadata.full_name` 또는 `user.email`을 `nickname`으로 사용
  3. `content` 길이 제한 검사 (최대 1000자 권장)
  4. `comments` 삽입

---

#### `updateComment(commentId, content)`

```ts
async function updateComment(commentId: string, content: string): Promise<
  | { ok: true }
  | { ok: false; message: string }
>
```

- **권한**: Visitor (본인 댓글만)
- **동작**:
  1. 사용자 확인
  2. `comments.user_id = auth.uid()` 검증
  3. `content`, `updated_at`, `is_edited = true` 업데이트

---

#### `deleteComment(commentId)`

```ts
async function deleteComment(commentId: string): Promise<
  | { ok: true }
  | { ok: false; message: string }
>
```

- **권한**: Visitor (본인) 또는 Admin
- **동작**: `deleted_at = NOW()` 소프트 삭제

---

### 3.4 좋아요 (likes)

#### `toggleLike(postId)`

좋아요가 없으면 추가, 있으면 취소한다.

```ts
async function toggleLike(postId: string): Promise<
  | { ok: true; liked: boolean; likeCount: number }
  | { ok: false; message: string }
>
```

- **권한**: Visitor (소셜 로그인)
- **동작**:
  1. 사용자 확인
  2. `likes` 테이블에서 `(post_id, user_id)` 존재 확인
  3. 없으면 INSERT → trigger가 `like_count` 증가
  4. 있으면 DELETE → trigger가 `like_count` 감소
  5. 갱신된 `like_count`와 현재 `liked` 상태 반환

---

### 3.5 이미지 업로드 (files)

#### `uploadImage(formData)`

Supabase Storage에 이미지를 업로드하고 `public_url`을 반환한다.

```ts
async function uploadImage(formData: FormData): Promise<
  | { ok: true; publicUrl: string; fileId: string }
  | { ok: false; message: string }
>
```

- **권한**: Admin
- **FormData 필드**:
  - `file`: 이미지 파일 (jpg/jpeg/png/webp, 최대 5MB)
  - `postId`: 연결할 게시글 ID. 정상 글쓰기 플로우에서는 `createDraft()`로 확보한 값을 항상 전달한다. `null`은 에지 케이스 안전장치용이며 `posts/orphan/` 경로에 저장된다.
  - `type`: `'thumbnail'` | `'content_image'`
- **동작**:
  1. 파일 형식·크기 검사
  2. UUID 기반 storage_path 생성
    - 썸네일: `thumbnails/{uuid}.webp`
    - 본문 이미지: `posts/{postId}/{uuid}.webp`
    - `postId` 없는 경우: `posts/orphan/{uuid}.webp`
  3. Supabase Storage에 업로드
  4. `files` 테이블에 메타데이터 삽입
  5. `publicUrl`, `fileId` 반환

---

#### `deleteImage(fileId)`

Storage 파일과 `files` 레코드를 함께 삭제한다.

```ts
async function deleteImage(fileId: string): Promise<
  | { ok: true }
  | { ok: false; message: string }
>
```

- **권한**: Admin

---

### 3.6 관리자 인증

Supabase Auth 내장 기능을 사용하므로 별도 Server Action 없이 클라이언트 SDK로 처리한다.

```ts
// 소셜 로그인 (클라이언트)
await supabase.auth.signInWithOAuth({ provider: 'google' });
await supabase.auth.signInWithOAuth({ provider: 'github' });

// 로그아웃 (클라이언트)
await supabase.auth.signOut();
```

관리자 여부는 로그인 후 `user.email === process.env.ADMIN_EMAIL`로 서버에서 검증한다.

---

## 4. Route Handlers

> `app/api/` 하위에 위치

### 4.1 `GET /api/rss`

RSS 2.0 피드를 반환한다.

```
GET /api/rss
Content-Type: application/xml
```

- 최근 발행 게시글 20개 포함
- `<item>`: title, link, description(excerpt), pubDate, guid(slug)
- 캐싱: `revalidate = 3600` (1시간)

---

## 5. Next.js 내장 파일 기반 라우트

> `app/` 하위에 파일로 정의 (Route Handler 불필요)

### 5.1 `app/sitemap.ts`

```ts
export default async function sitemap(): Promise<MetadataRoute.Sitemap>
```

- 발행된 전체 게시글의 `slug` 기반 URL 생성 (`/posts/[slug]`)
- `/posts?tag=slug` 형태의 태그 필터 URL은 포함하지 않는다. query param URL은 검색엔진이 수집 여부를 자체 판단하며, sitemap에 넣어도 실익이 없다.
- `changeFrequency`: posts → `weekly`

### 5.2 `app/robots.ts`

```ts
export default function robots(): MetadataRoute.Robots
```

- `User-agent: *` → Allow: `/`
- `Disallow: /admin`
- `Sitemap: {baseUrl}/sitemap.xml`

---

## 6. 데이터 조회 패턴 (Server Components)

Server Action이 아닌 Server Component 내 직접 fetch 패턴.

### 6.1 게시글 목록

```ts
// app/posts/page.tsx (Server Component) — 초기 1페이지 fetch (cursor 기반 무한 스크롤)
const { data } = await supabase
  .from('posts')
  .select('id, slug, title, thumbnail_url, published_at, view_count, like_count')
  .eq('status', 'published')
  .order('published_at', { ascending: false })
  .order('id', { ascending: false })
  .limit(10);

// 추가 페이지는 GET /api/posts?publishedAt=...&cursorId=... 로 로딩 (pagination.md 참고)
```

### 6.2 게시글 상세

```ts
// app/posts/[slug]/page.tsx (Server Component)
const { data: post } = await supabase
  .from('posts')
  .select(`
    *,
    post_tags ( tag_id, tags ( id, name, slug, color ) )
  `)
  .eq('slug', slug)
  .eq('status', 'published')
  .single();

// 조회수 증가 (비동기, 응답 대기 불필요)
supabase.rpc('increment_view_count', { target_slug: slug });
```

### 6.3 태그 필터 게시글 목록 (`/posts?tag=slug`)

`/tags/[slug]` 라우트 없이 `/posts` 페이지의 `searchParams.tag`로 처리한다.

```ts
// tagSlug = searchParams.tag
const { data, count } = await supabase
  .from('post_tags')
  .select(
    `posts!inner( id, slug, title, thumbnail_url, published_at, like_count ),
     tags!inner( slug )`,
    { count: 'exact' }
  )
  .eq('tags.slug', tagSlug)
  .eq('posts.status', 'published')
  .order('posts.published_at', { ascending: false })
  .range(from, to);
```

### 6.4 인기 글 (좋아요 기준)

```ts
const { data } = await supabase
  .from('posts')
  .select('slug, title, like_count, thumbnail_url')
  .eq('status', 'published')
  .order('like_count', { ascending: false })
  .limit(10);
```

### 6.5 인기 글 (조회수 기준)

```ts
const { data } = await supabase
  .from('posts')
  .select('slug, title, view_count, thumbnail_url')
  .eq('status', 'published')
  .order('view_count', { ascending: false })
  .limit(10);
```

### 6.6 댓글 목록 (TanStack Query)

```ts
// TanStack Query key: ['comments', postId]
const { data } = await supabase
  .from('comments')
  .select('id, nickname, content, created_at, updated_at, is_edited')
  .eq('post_id', postId)
  .is('deleted_at', null)
  .order('created_at', { ascending: true });
```

### 6.7 검색

```ts
// 검색어 전처리: ILIKE 특수문자(%, _, \) 이스케이프, PostgREST or() 구문 문자(,, (, )) 제거
const q = rawQ.trim().replace(/[%_\\]/g, '\\$&').replace(/[,()]/g, '');
if (!q) return [];

// 제목 + 본문 + 태그명 통합 검색

// Step 1: 태그명으로 매칭되는 post_id 수집
const { data: tagMatches } = await supabase
  .from('post_tags')
  .select('post_id, tags!inner( name )')
  .ilike('tags.name', `%${q}%`);

const tagPostIds = (tagMatches ?? []).map((r) => r.post_id);

// Step 2: 제목·본문 OR 태그 매칭 posts 검색
let filter = `title.ilike.%${q}%,content.ilike.%${q}%`;
if (tagPostIds.length > 0) {
  filter += `,id.in.(${tagPostIds.join(',')})`;
}

const { data } = await supabase
  .from('posts')
  .select('slug, title, published_at')
  .eq('status', 'published')
  .or(filter)
  .order('published_at', { ascending: false });
```

> 성능 개선이 필요하면 `to_tsvector` + `tsquery` 기반 full-text search index 추가를 권장한다.

### 6.8 아카이브

```ts
// 연도별/월별 게시글 수
const { data } = await supabase
  .from('posts')
  .select('published_at')
  .eq('status', 'published')
  .order('published_at', { ascending: false });

// 클라이언트에서 연도·월 기준으로 그루핑
```

---

## 7. 에러 처리 원칙

| 상황 | 응답 |
|------|------|
| 인증 실패 (비로그인) | `{ ok: false, message: '로그인이 필요합니다.' }` |
| 권한 없음 (비관리자) | `{ ok: false, message: '권한이 없습니다.' }` |
| 존재하지 않는 리소스 | `{ ok: false, message: '찾을 수 없습니다.' }` |
| 유효성 검사 실패 | `{ ok: false, message: '입력 값을 확인해 주세요.' }` |
| DB/Storage 오류 | `{ ok: false, message: '서버 오류가 발생했습니다.' }` + 서버 콘솔 로그 |

Server Action 결과 타입 표준:

```ts
// 모든 Server Action의 기본 응답 형태
type ActionResult = { ok: true } | { ok: false; message: string };

// ok: true 분기에 필요한 데이터가 있으면 필드를 추가한다
// 예: { ok: true; postId: string; slug: string } | { ok: false; message: string }
```

---

## 8. 페이지별 데이터 의존성 요약

| 페이지 | 데이터 소스 | 캐싱 전략 |
|--------|------------|-----------|
| `/` | Server Component (posts, tags) | ISR revalidate 60s |
| `/posts` (tag 없음) | Server Component 초기 1페이지 + Route Handler | ISR revalidate 60s (초기) / no-store (Route Handler) |
| `/posts?tag=slug` | Server Component (태그 필터 + 페이지네이션) | no-store (동적 query param) |
| `/posts/[slug]` | Server Component (post detail) | ISR revalidate 60s |
| `/posts/[slug]` 댓글 | TanStack Query | staleTime 30s |
| `/posts/[slug]` 좋아요 | TanStack Query | staleTime 0 |
| `/search` | Server Component (동적) | no-store |
| `/archive` | Server Component | ISR revalidate 3600s |
| `/admin/**` | Server Component + Server Actions | no-store |
