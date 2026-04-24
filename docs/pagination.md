# 페이지네이션 / 무한 스크롤 설계 문서

## 1. 전략 개요

| 대상 | 방식 | 이유 |
|------|------|------|
| 전체 게시글 (`/posts`) | cursor 기반 무한 스크롤 | 탐색형 UX. SSR 초기 렌더 후 클라이언트 추가 로딩 |
| 태그 필터 (`/posts?tag=nextjs`) | offset 기반 페이지네이션 | `/posts` 라우트에서 tag 파라미터 유무로 분기. 필터 결과 위치를 URL로 공유·북마크 가능 |
| 검색 결과 (`/search`) | offset 기반 페이지네이션 | 동일 |
| 댓글 | cursor 기반 무한 스크롤 | 실시간 추가에 안전, 순서 안정성 필요 |

> `/tags/[slug]` 라우트는 만들지 않는다. 태그 필터는 게시글 탐색의 한 뷰이므로 `/posts?tag=slug`로 처리하는 것이 자연스럽다. Google은 query param URL도 색인하므로 SEO 손실도 없다.

---

## 2. 게시글 목록 (`/posts`) — tag 파라미터 유무로 모드 분기

### 2.1 URL 구조

```
/posts                      → 전체 글, 무한 스크롤
/posts?tag=nextjs           → 태그 필터, 페이지네이션 1페이지
/posts?tag=nextjs&page=2    → 태그 필터, 페이지네이션 2페이지
```

---

### 2.2 Server Component 분기 구조

`app/posts/page.tsx`에서 `searchParams.tag` 유무로 렌더링 방식을 결정한다.

```tsx
// app/posts/page.tsx  (Server Component)
type Props = {
  searchParams: { tag?: string; page?: string };
};

export default async function PostsPage({ searchParams }: Props) {
  const { tag, page: pageStr } = searchParams;

  if (tag) {
    // 태그 필터 모드: 페이지네이션
    const page = Math.max(1, Number(pageStr ?? 1));
    const { posts, total } = await getPostsByTag(tag, page);
    const totalPages = Math.ceil(total / PAGE_SIZE);

    return (
      <>
        <TagFilterBadge tagSlug={tag} />
        <PostList posts={posts} />
        <Pagination
          currentPage={page}
          totalPages={totalPages}
          basePath="/posts"
          extraParams={{ tag }}   // ?tag=nextjs&page=N 형태로 생성
        />
      </>
    );
  }

  // 기본 모드: 무한 스크롤
  const initialPosts = await getInitialPosts();
  return <PostsInfiniteList initialPosts={initialPosts} />;
}
```

---

### 2.3 무한 스크롤 모드 (`tag` 없음)

전체 구조:

```
[Server Component]  초기 1페이지 SSR 렌더 (SEO용)
       ↓ initialPosts prop 전달
[Client Component]  useInfiniteQuery로 이어서 로딩
       ↓ 스크롤 하단 도달 시
[Route Handler]     GET /api/posts?publishedAt=...&cursorId=...
```

- 첫 렌더는 Server Component가 담당 → 검색엔진이 1페이지 콘텐츠를 인덱싱함
- 추가 로딩은 `useInfiniteQuery` + Route Handler 조합으로 처리
- 자동 스크롤 트리거: `IntersectionObserver`로 하단 sentinel 감지

---

### 2.4 cursor 설계 (무한 스크롤 모드)

정렬 기준: `published_at DESC, id DESC`

- `published_at`이 같은 글이 있을 때(동시 발행은 드물지만) `id`를 tie-breaker로 사용
- 다음 페이지 조건: `published_at < cursor.publishedAt` OR `(published_at = cursor.publishedAt AND id < cursor.id)`

```ts
export interface PostCursor {
  publishedAt: string;  // ISO 8601
  id: string;           // UUID
}
```

---

### 2.5 Route Handler — `GET /api/posts`

```ts
// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/server';

const POST_PAGE_SIZE = 10;

export async function GET(req: NextRequest) {
  const { searchParams } = req.nextUrl;
  const publishedAt = searchParams.get('publishedAt');
  const cursorId = searchParams.get('cursorId');
  const limit = Number(searchParams.get('limit') ?? POST_PAGE_SIZE);

  const supabase = createClient();

  let query = supabase
    .from('posts')
    .select('id, slug, title, thumbnail_url, published_at, view_count, like_count')
    .eq('status', 'published')
    .order('published_at', { ascending: false })
    .order('id', { ascending: false })
    .limit(limit);

  // cursor가 있으면 그 이후 항목만 조회
  if (publishedAt && cursorId) {
    query = query.or(
      `published_at.lt.${publishedAt},` +
      `and(published_at.eq.${publishedAt},id.lt.${cursorId})`
    );
  }

  const { data, error } = await query;

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json(data);
}
```

**요청 예시**

```
# 첫 페이지 (cursor 없음)
GET /api/posts?limit=10

# 다음 페이지 (마지막 항목의 published_at, id 전달)
GET /api/posts?publishedAt=2024-08-10T09:00:00Z&cursorId=abc123&limit=10
```

**응답**: `PostSummary[]` (JSON 배열)

---

### 2.6 Server Component — 초기 데이터 fetch (무한 스크롤 모드)

```ts
// lib/posts/get-initial-posts.ts
export async function getInitialPosts(): Promise<PostSummary[]> {
  const supabase = createClient();

  const { data } = await supabase
    .from('posts')
    .select('id, slug, title, thumbnail_url, published_at, view_count, like_count')
    .eq('status', 'published')
    .order('published_at', { ascending: false })
    .order('id', { ascending: false })
    .limit(10);

  return data ?? [];
}
```

```tsx
// app/posts/page.tsx  (Server Component)
export default async function PostsPage() {
  const initialPosts = await getInitialPosts();

  return <PostsInfiniteList initialPosts={initialPosts} />;
}
```

---

### 2.7 Client Component — `useInfiniteQuery`

```tsx
// components/posts/PostsInfiniteList.tsx
'use client';

import { useInfiniteQuery } from '@tanstack/react-query';
import { useEffect, useRef } from 'react';

const POST_PAGE_SIZE = 10;

async function fetchMorePosts(cursor?: PostCursor): Promise<PostSummary[]> {
  const params = new URLSearchParams({ limit: String(POST_PAGE_SIZE) });
  if (cursor) {
    params.set('publishedAt', cursor.publishedAt);
    params.set('cursorId', cursor.id);
  }
  const res = await fetch(`/api/posts?${params}`);
  if (!res.ok) throw new Error('게시글 로딩 실패');
  return res.json();
}

interface Props {
  initialPosts: PostSummary[];
}

export function PostsInfiniteList({ initialPosts }: Props) {
  const sentinelRef = useRef<HTMLDivElement>(null);

  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    useInfiniteQuery({
      queryKey: ['posts'],
      queryFn: ({ pageParam }) => fetchMorePosts(pageParam as PostCursor | undefined),
      initialPageParam: undefined,
      getNextPageParam: (lastPage): PostCursor | undefined => {
        if (lastPage.length < POST_PAGE_SIZE) return undefined;
        const last = lastPage[lastPage.length - 1];
        return { publishedAt: last.published_at, id: last.id };
      },
      // Server Component에서 받은 초기 데이터로 hydration
      initialData: {
        pages: [initialPosts],
        pageParams: [undefined],
      },
    });

  // IntersectionObserver: sentinel이 뷰포트에 들어오면 다음 페이지 로드
  useEffect(() => {
    const el = sentinelRef.current;
    if (!el) return;

    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      },
      { rootMargin: '200px' }  // 하단 200px 전부터 미리 요청
    );

    observer.observe(el);
    return () => observer.disconnect();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  const posts = data.pages.flat();

  return (
    <div>
      <PostGrid posts={posts} />

      {/* 스크롤 감지용 sentinel */}
      <div ref={sentinelRef} />

      {isFetchingNextPage && <PostGridSkeleton />}
      {!hasNextPage && posts.length > 0 && (
        <p>모든 게시글을 불러왔습니다.</p>
      )}
    </div>
  );
}
```

---

### 2.8 `initialData` vs `HydrationBoundary` 선택 이유

TanStack Query의 공식 SSR 패턴은 `HydrationBoundary + dehydrate`이지만, `useInfiniteQuery`의 hydration은 설정이 복잡하다. 이 프로젝트에서는:

- Server Component가 첫 페이지 데이터를 prop으로 내려주고
- Client Component에서 `initialData`로 주입하는 방식을 사용한다

이 방식의 트레이드오프:

| | `initialData` prop 방식 | `HydrationBoundary` 방식 |
|-|------------------------|--------------------------|
| 설정 복잡도 | 낮음 | 높음 (`dehydrate` + `HydrationBoundary` 구성 필요) |
| 캐시 재사용 | 동일 queryKey면 캐시 히트 | 동일 |
| staleTime | `initialData`는 즉시 stale로 처리됨 | dehydrated 시각 기준 |
| 권장 상황 | 단순 초기 로딩 | 여러 쿼리를 한꺼번에 dehydrate할 때 |

`initialData`가 즉시 stale로 처리되는 문제는 `initialDataUpdatedAt: Date.now()`를 함께 전달해 해결할 수 있다.

```ts
initialData: { pages: [initialPosts], pageParams: [undefined] },
initialDataUpdatedAt: Date.now(),  // stale 판정 기준점 설정
```

---

## 3. 태그 필터 모드 (`/posts?tag=slug`) — offset 기반 페이지네이션

별도 라우트 없이 `/posts` 페이지에서 처리한다 (섹션 2.2 분기 구조 참고).

### 3.1 Supabase 쿼리

```ts
const PAGE_SIZE = 10;

async function getPostsByTag(tagSlug: string, page: number) {
  const from = (page - 1) * PAGE_SIZE;
  const to = from + PAGE_SIZE - 1;

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

  return {
    posts: data?.map((row) => row.posts) ?? [],
    total: count ?? 0,
  };
}
```

---

### 3.2 Pagination 컴포넌트 동작 원칙

- `<Link href="/posts?tag=nextjs&page=N">` 기반으로 구현 (SEO 유지)
- 노출 범위: `현재 페이지 ±2` + 처음/끝 페이지
- `page < 1`이면 1, `page > totalPages`이면 마지막 페이지로 clamp
- `extraParams` prop으로 `tag`, `q` 등 추가 query param을 URL에 유지

---

### 3.3 응답 타입

```ts
export interface PaginatedPosts {
  posts: PostSummary[];
  total: number;
  page: number;
  pageSize: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
}
```

---

## 4. 검색 결과 (`/search`) — offset 기반 페이지네이션

```
/search?q=nextjs&page=1
```

```ts
async function searchPosts(rawQuery: string, page: number) {
  // 검색어 전처리: ILIKE 특수문자(%, _, \) 이스케이프, PostgREST or() 구문 문자(,, (, )) 제거
  const q = rawQuery.trim().replace(/[%_\\]/g, '\\$&').replace(/[,()]/g, '');
  if (!q) return { posts: [], total: 0 };

  const from = (page - 1) * PAGE_SIZE;
  const to = from + PAGE_SIZE - 1;

  // Step 1: 태그명으로 매칭되는 post_id 수집
  const { data: tagMatches } = await supabase
    .from('post_tags')
    .select('post_id, tags!inner( name )')
    .ilike('tags.name', `%${q}%`);

  const tagPostIds = (tagMatches ?? []).map((r) => r.post_id);

  // Step 2: 제목·본문 OR 태그 매칭 통합 검색
  let filter = `title.ilike.%${q}%,content.ilike.%${q}%`;
  if (tagPostIds.length > 0) {
    filter += `,id.in.(${tagPostIds.join(',')})`;
  }

  const { data, count } = await supabase
    .from('posts')
    .select('id, slug, title, thumbnail_url, published_at', { count: 'exact' })
    .eq('status', 'published')
    .or(filter)
    .order('published_at', { ascending: false })
    .range(from, to);

  return { posts: data ?? [], total: count ?? 0 };
}
```

> 검색 결과는 탐색형 UX보다 "원하는 결과를 찾는" 목적이 강하므로 페이지네이션이 적합하다.

---

## 5. 댓글 — cursor 기반 무한 스크롤

### 5.1 cursor 설계

정렬 기준: `created_at ASC, id ASC`

```ts
export interface CommentCursor {
  createdAt: string;  // ISO 8601
  id: string;         // UUID (tie-breaker)
}
```

---

### 5.2 Supabase 쿼리

```ts
const COMMENT_PAGE_SIZE = 20;

async function getComments(postId: string, cursor?: CommentCursor) {
  let query = supabase
    .from('comments')
    .select('id, nickname, content, created_at, updated_at, is_edited')
    .eq('post_id', postId)
    .is('deleted_at', null)
    .order('created_at', { ascending: true })
    .order('id', { ascending: true })
    .limit(COMMENT_PAGE_SIZE);

  if (cursor) {
    query = query.or(
      `created_at.gt.${cursor.createdAt},` +
      `and(created_at.eq.${cursor.createdAt},id.gt.${cursor.id})`
    );
  }

  const { data } = await query;
  return data ?? [];
}
```

---

### 5.3 TanStack Query — `useInfiniteQuery`

```ts
// hooks/use-comments.ts
export function useComments(postId: string) {
  return useInfiniteQuery({
    queryKey: ['comments', postId],
    queryFn: ({ pageParam }) =>
      getComments(postId, pageParam as CommentCursor | undefined),
    initialPageParam: undefined,
    getNextPageParam: (lastPage): CommentCursor | undefined => {
      if (lastPage.length < COMMENT_PAGE_SIZE) return undefined;
      const last = lastPage[lastPage.length - 1];
      return { createdAt: last.created_at, id: last.id };
    },
    staleTime: 30_000,
  });
}
```

---

### 5.4 댓글 목록 컴포넌트

```tsx
'use client';

export function CommentList({ postId }: { postId: string }) {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    useComments(postId);

  const comments = data?.pages.flat() ?? [];

  return (
    <div>
      {comments.map((c) => (
        <CommentItem key={c.id} comment={c} />
      ))}

      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? '불러오는 중...' : '댓글 더 보기'}
        </button>
      )}
    </div>
  );
}
```

> 댓글은 글 읽기 중 등장하므로 자동 스크롤 대신 **버튼 트리거** 방식을 사용한다.
> 사용자가 글을 읽는 도중 댓글이 갑자기 로드되어 레이아웃이 밀리는 현상을 방지한다.

---

### 5.5 댓글 추가 후 캐시 갱신

```ts
const queryClient = useQueryClient();

// addComment Server Action 호출 성공 후
queryClient.invalidateQueries({ queryKey: ['comments', postId] });
```

---

## 6. 관리자 — 댓글 관리 (`/admin/comments`)

Server Component + offset 기반 페이지네이션.

```ts
async function getAdminComments(page: number) {
  const from = (page - 1) * PAGE_SIZE;
  const to = from + PAGE_SIZE - 1;

  const { data, count } = await supabaseAdmin
    .from('comments')
    .select(
      `id, nickname, content, created_at, deleted_at, is_edited,
       posts(slug, title)`,
      { count: 'exact' }
    )
    .order('created_at', { ascending: false })
    .range(from, to);

  return { comments: data ?? [], total: count ?? 0 };
}
```

---

## 7. 타입 정의 요약

```ts
// 게시글 cursor (무한 스크롤)
export interface PostCursor {
  publishedAt: string;
  id: string;
}

// 댓글 cursor (무한 스크롤)
export interface CommentCursor {
  createdAt: string;
  id: string;
}

// 태그 필터 / 검색 결과 (페이지네이션)
export interface PaginatedPosts {
  posts: PostSummary[];
  total: number;
  page: number;
  pageSize: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
}
```

---

## 8. 설계 결정 요약

| 항목 | 결정 | 이유 |
|------|------|------|
| `/posts` 방식 | cursor 무한 스크롤 | 탐색형 UX, 연속적인 콘텐츠 소비에 적합 |
| `/posts` 초기 렌더 | Server Component SSR | 첫 1페이지를 검색엔진이 인덱싱할 수 있음 |
| `/posts` 추가 로딩 | Route Handler + `useInfiniteQuery` | Server Action은 변경 작업 전용, 데이터 조회는 Route Handler |
| `/posts` 스크롤 트리거 | `IntersectionObserver` 자동 | 탐색 페이지라 자동 로딩이 자연스러운 UX |
| `/posts?tag=slug` 방식 | offset 페이지네이션 | `/posts` 단일 라우트 내 분기. 필터 결과를 URL로 공유·북마크 가능 |
| 검색 결과 방식 | offset 페이지네이션 | 원하는 결과 위치를 다시 찾을 수 있어야 함 |
| 게시글 pageSize | 10 | 썸네일 포함 목록 기준 적정량 |
| 댓글 방식 | cursor 무한 스크롤 | 실시간 추가에 안전, 순서 안정성 보장 |
| 댓글 스크롤 트리거 | 버튼 클릭 | 글 읽는 중 자동 로딩 시 레이아웃 밀림 방지 |
| 댓글 pageSize | 20 | 텍스트 위주라 한 번에 더 많이 보여도 무방 |
