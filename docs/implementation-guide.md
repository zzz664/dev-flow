# 구현 착수 가이드

> 작성일: 2026-04-24
> 목적: `dev-plan.md`, `wbs.md`를 실제 구현으로 옮길 때 필요한 핵심 지식과 실무 가이드를 한 문서에 정리한다.

---

## 1. 이 문서의 범위

이 문서는 아래 기술들에 대해 "지금 프로젝트에서 바로 필요한 수준"으로 정리한다.

- Next.js 15 App Router
- Supabase SSR / Auth / RLS / Storage (`@supabase/ssr`)
- TanStack Query v5
- MDXEditor (Markdown 전용 구성)
- react-markdown + remark-gfm + rehype-highlight
- Vercel 배포

설계 결정 사항은 기존 문서(api-spec.md, schema.md 등)를 기준으로 하고,
여기서는 **구현 시 주의점, 실제 코드 패턴, 함정 회피법**을 다룬다.

---

## 2. 구현 전에 반드시 알고 시작할 점

### 2.1 Next.js App Router의 역할 분리 기준

| 역할 | 담당 | 이 프로젝트 적용 예시 |
|------|------|----------------------|
| 읽기 (정적/준정적) | Server Component | 게시글 목록, 상세, 검색, 아카이브 |
| 변경 (CUD) | Server Action | 글 작성/발행/삭제, 댓글, 좋아요 |
| 별도 HTTP 엔드포인트 | Route Handler | `GET /api/posts` (cursor), `GET /api/rss`, `/auth/callback` |
| 클라이언트 상호작용 | TanStack Query | 댓글 무한 스크롤, 좋아요 토글, 게시글 무한 스크롤 추가 로딩 |

### 2.2 `getSession()` 대신 `getUser()`를 써야 한다

Supabase 공식 문서 기준으로, **서버 측 코드(Server Component, Server Action, 미들웨어)에서는 반드시 `getUser()`를 사용해야 한다.**

- `getSession()`은 JWT를 서버에서 재검증하지 않는다 → 위조 가능
- `getUser()`는 Supabase Auth 서버에 토큰을 실제로 검증하므로 안전하다

```ts
// ❌ 위험 — JWT를 재검증하지 않음
const { data: { session } } = await supabase.auth.getSession()

// ✅ 안전 — Supabase 서버에 실제 검증 요청
const { data: { user } } = await supabase.auth.getUser()
```

### 2.3 Server Action의 기본 body size limit은 1MB다

Next.js Server Action은 기본적으로 요청 바디를 1MB로 제한한다.
이 프로젝트는 이미지 최대 5MB를 지원하므로 반드시 설정 변경이 필요하다.

```js
// next.config.js
/** @type {import('next').NextConfig} */
module.exports = {
  experimental: {
    serverActions: {
      bodySizeLimit: '6mb',  // WebP 변환 후에도 여유 있게
    },
  },
}
```

지원 포맷: `1000` (bytes), `'500kb'`, `'3mb'`

### 2.4 MDXEditor는 서버에서 렌더링할 수 없다

MDXEditor는 브라우저 API에 의존하므로 **반드시 `dynamic` + `ssr: false`로 import** 해야 한다.
직접 import하면 Next.js 빌드가 실패한다.

### 2.5 `cookies()` in Next.js 15는 async다

Next.js 15에서 `next/headers`의 `cookies()`가 비동기 함수로 변경됐다.

```ts
// Next.js 14 이하
const cookieStore = cookies()

// Next.js 15 — await 필요
const cookieStore = await cookies()
```

Supabase 서버 클라이언트를 만드는 함수도 `async`여야 한다.

### 2.6 react-markdown은 raw HTML을 기본 차단한다

`react-markdown`은 기본적으로 Markdown 내 raw HTML을 렌더링하지 않는다.
이 프로젝트는 "순수 Markdown" 정책이므로 이 기본값을 유지한다.
`rehype-raw`는 추가하지 않는다.

---

## 3. 기술별 구현 가이드

---

### 3.1 Next.js App Router

#### 권장 디렉토리 구조

```
app/
  (public)/           ← route group: 공개 페이지 공통 레이아웃
    layout.tsx
    page.tsx           ← 홈
    posts/
      page.tsx         ← 게시글 목록 (무한스크롤 / 태그 필터)
      [slug]/
        page.tsx       ← 게시글 상세
    search/
      page.tsx
    archive/
      page.tsx
  admin/              ← 관리자 전용 (middleware로 보호)
    layout.tsx
    posts/
      new/page.tsx
      [slug]/edit/page.tsx
    comments/page.tsx
    tags/page.tsx
    images/page.tsx
  api/
    posts/route.ts     ← GET /api/posts (cursor 기반 목록)
    rss/route.ts       ← GET /api/rss
  auth/
    callback/route.ts  ← OAuth callback
  sitemap.ts
  robots.ts
  layout.tsx           ← 루트 레이아웃 (Providers 포함)

components/
  posts/               ← 게시글 관련 컴포넌트
  comments/
  ui/                  ← 재사용 UI (Button, Pagination 등)

lib/
  supabase/
    client.ts
    server.ts
    admin.ts
  auth/
    require-admin.ts
    get-user.ts
  posts/
    get-initial-posts.ts
    get-posts-by-tag.ts
    search-posts.ts

types/
  index.ts             ← 공통 타입 (Post, Tag, Comment 등)
```

#### `revalidatePath` 위치 결정 원칙

Server Action 내 mutation 성공 후에만 호출한다. 위치는 `return` 바로 위.

```ts
// app/actions/posts.ts
'use server'

export async function publishPost(postId: string) {
  // ... 발행 로직 ...

  revalidatePath('/posts')          // 목록 캐시 무효화
  revalidatePath(`/posts/${slug}`)  // 상세 캐시 무효화
  revalidatePath('/')               // 홈 캐시 무효화

  return { ok: true, slug }
}
```

#### ISR 캐싱 전략 적용 방법

```ts
// Server Component에서 캐시 정책은 fetch 옵션 또는 export로 설정
export const revalidate = 60  // 해당 라우트 전체에 ISR 60s 적용

// 특정 fetch에만 적용
const res = await fetch(url, { next: { revalidate: 60 } })

// 동적 라우트 (쿼리 파라미터 사용)는 캐시 안 함
export const dynamic = 'force-dynamic'
```

---

### 3.2 Supabase SSR 클라이언트

`@supabase/ssr` 패키지를 사용한다. `@supabase/supabase-js`만으로는 Next.js SSR 쿠키 흐름을 처리할 수 없다.

```bash
npm install @supabase/ssr @supabase/supabase-js
```

#### `lib/supabase/client.ts` — 브라우저 전용

```ts
'use client'

import { createBrowserClient } from '@supabase/ssr'

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

#### `lib/supabase/server.ts` — Server Component / Server Action / Route Handler

```ts
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createClient() {
  const cookieStore = await cookies()  // Next.js 15: await 필요

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll()
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            )
          } catch {
            // Server Component에서는 쿠키 쓰기 불가 — 미들웨어에서 처리됨
          }
        },
      },
    }
  )
}
```

#### `lib/supabase/admin.ts` — 서버 전용 (RLS 우회)

```ts
import { createClient } from '@supabase/supabase-js'

// 이 파일은 절대 'use client' 선언 금지
export const supabaseAdmin = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!  // 클라이언트 절대 노출 금지
)
```

> `supabaseAdmin`은 모듈 수준 싱글톤으로 유지해도 안전하다. Service Role Key는 환경 변수에서만 오고 서버에서만 실행된다.

---

### 3.3 Supabase 인증 + 미들웨어

#### `middleware.ts` — 세션 갱신 + `/admin` 보호

```ts
import { createServerClient, parseCookieHeader } from '@supabase/ssr'
import { NextRequest, NextResponse } from 'next/server'

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({
    request: { headers: request.headers },
  })

  // 세션 갱신용 Supabase 클라이언트
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return parseCookieHeader(request.headers.get('cookie') ?? '')
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) => {
            response.cookies.set(name, value, options)
          })
        },
      },
    }
  )

  // 세션 갱신 (getUser = 서버 재검증, getSession은 사용 금지)
  const { data: { user } } = await supabase.auth.getUser()

  // /admin 접근 시 관리자 검증
  if (request.nextUrl.pathname.startsWith('/admin')) {
    if (!user || user.email !== process.env.ADMIN_EMAIL) {
      return NextResponse.redirect(new URL('/login', request.url))
    }
  }

  return response
}

export const config = {
  matcher: [
    // 정적 파일, 이미지, favicon 제외
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
}
```

#### `lib/auth/require-admin.ts` — Server Action 내 이중 검증

미들웨어가 1차 방어선이지만, Server Action에서도 반드시 한 번 더 확인한다.

```ts
import { createClient } from '@/lib/supabase/server'

export async function requireAdmin() {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user || user.email !== process.env.ADMIN_EMAIL) {
    throw new Error('Unauthorized')
  }

  return user
}
```

```ts
// 모든 관리자 Server Action 최상단에 적용
'use server'

export async function createDraft() {
  try {
    const user = await requireAdmin()
    // ... 로직
  } catch (e) {
    return { ok: false, message: '권한이 없습니다.' }
  }
}
```

#### `/auth/callback/route.ts` — OAuth 리다이렉트 처리

```ts
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'
import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const { searchParams, origin } = request.nextUrl
  const code = searchParams.get('code')
  const next = searchParams.get('next') ?? '/'

  if (!code) {
    return NextResponse.redirect(`${origin}/login?error=missing_code`)
  }

  const cookieStore = await cookies()
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: (cookiesToSet) =>
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          ),
      },
    }
  )

  const { error } = await supabase.auth.exchangeCodeForSession(code)
  if (error) {
    return NextResponse.redirect(`${origin}/login?error=exchange_failed`)
  }

  // 관리자면 /admin으로, 아니면 next 파라미터 위치로
  const { data: { user } } = await supabase.auth.getUser()
  if (user?.email === process.env.ADMIN_EMAIL) {
    return NextResponse.redirect(`${origin}/admin/posts`)
  }

  return NextResponse.redirect(`${origin}${next}`)
}
```

---

### 3.4 Supabase RLS

#### 구현 원칙

- 모든 테이블에 `ALTER TABLE <table> ENABLE ROW LEVEL SECURITY;`
- 브라우저에서 접근하는 테이블은 RLS가 있어야 한다 (anon key 기준)
- 관리자 작업은 `supabaseAdmin` (service role) + `requireAdmin()` 이중 방어

#### 개발 중 RLS 검증 방법

Supabase Dashboard → Table Editor에서 역할을 `anon` / `authenticated`로 바꿔 쿼리를 직접 실행하면 정책이 의도대로 동작하는지 확인할 수 있다.

```sql
-- RLS 정책 테스트 (Supabase SQL Editor)
SET ROLE anon;
SELECT * FROM posts;  -- status = 'published'인 것만 나와야 함

SET ROLE authenticated;
-- 특정 user_id로 테스트하려면 set_config 사용
```

#### 성능상 주의할 인덱스

RLS 정책은 쿼리에 조건을 자동으로 추가하는 방식이다. 정책에서 자주 쓰는 컬럼에 인덱스가 없으면 전체 테이블 스캔이 발생한다.

이 프로젝트에서 반드시 필요한 인덱스 (schema.md에 이미 포함):

```sql
CREATE INDEX idx_posts_status     ON posts(status);
CREATE INDEX idx_posts_published  ON posts(published_at DESC) WHERE status = 'published';
CREATE INDEX idx_comments_active  ON comments(post_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_likes_post_id    ON likes(post_id);
CREATE INDEX idx_files_post_id    ON files(post_id);
```

---

### 3.5 Supabase Storage

#### 이미지 업로드 Server Action 전체 흐름

```ts
// app/actions/upload.ts
'use server'

import { requireAdmin } from '@/lib/auth/require-admin'
import { supabaseAdmin } from '@/lib/supabase/admin'

export async function uploadImage(formData: FormData) {
  try {
    await requireAdmin()

    const file = formData.get('file') as File
    const postId = formData.get('postId') as string | null
    const type = formData.get('type') as 'thumbnail' | 'content_image'

    // 서버에서 MIME 타입 재검증 (클라이언트 우회 방지)
    if (!['image/webp', 'image/jpeg', 'image/png'].includes(file.type)) {
      return { ok: false, message: '허용되지 않는 파일 형식입니다.' }
    }
    if (file.size > 6 * 1024 * 1024) {
      return { ok: false, message: '파일 크기가 너무 큽니다.' }
    }

    const uuid = crypto.randomUUID()
    const ext = file.type === 'image/webp' ? 'webp' : file.name.split('.').pop()
    const storagePath = type === 'thumbnail'
      ? `thumbnails/${uuid}.${ext}`
      : postId
        ? `posts/${postId}/${uuid}.${ext}`
        : `posts/orphan/${uuid}.${ext}`

    const { error: uploadError } = await supabaseAdmin.storage
      .from('images')
      .upload(storagePath, file, {
        contentType: file.type,
        upsert: false,
      })

    if (uploadError) return { ok: false, message: uploadError.message }

    const { data: { publicUrl } } = supabaseAdmin.storage
      .from('images')
      .getPublicUrl(storagePath)

    const { data: fileRecord, error: dbError } = await supabaseAdmin
      .from('files')
      .insert({ post_id: postId ?? null, storage_path: storagePath, public_url: publicUrl, type })
      .select('id')
      .single()

    if (dbError) {
      // DB 실패 시 Storage 파일도 롤백
      await supabaseAdmin.storage.from('images').remove([storagePath])
      return { ok: false, message: dbError.message }
    }

    return { ok: true, publicUrl, fileId: fileRecord!.id }
  } catch {
    return { ok: false, message: '서버 오류가 발생했습니다.' }
  }
}
```

#### 클라이언트 WebP 변환 유틸

```ts
// lib/image/convert-to-webp.ts
export async function convertToWebP(file: File): Promise<File> {
  const img = await createImageBitmap(file)
  const canvas = document.createElement('canvas')

  const MAX_WIDTH = 1200
  const scale = Math.min(1, MAX_WIDTH / img.width)
  canvas.width = Math.round(img.width * scale)
  canvas.height = Math.round(img.height * scale)

  canvas.getContext('2d')!.drawImage(img, 0, 0, canvas.width, canvas.height)

  const blob = await new Promise<Blob>((resolve, reject) =>
    canvas.toBlob(
      (b) => (b ? resolve(b) : reject(new Error('WebP 변환 실패'))),
      'image/webp',
      0.85
    )
  )

  return new File([blob], `${crypto.randomUUID()}.webp`, { type: 'image/webp' })
}
```

---

### 3.6 TanStack Query v5

#### `app/providers.tsx` — QueryClientProvider 설정

```tsx
'use client'

import { isServer, QueryClient, QueryClientProvider } from '@tanstack/react-query'

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 30 * 1000,       // 30초 — 기본 stale 시간
        gcTime: 5 * 60 * 1000,      // 5분 — v5에서 cacheTime이 gcTime으로 변경됨
        retry: 1,
        refetchOnWindowFocus: false,
      },
    },
  })
}

let browserQueryClient: QueryClient | undefined

function getQueryClient() {
  if (isServer) {
    return makeQueryClient()  // 서버에서는 매번 새로 생성
  }
  if (!browserQueryClient) browserQueryClient = makeQueryClient()
  return browserQueryClient   // 브라우저에서는 싱글톤
}

export function Providers({ children }: { children: React.ReactNode }) {
  const queryClient = getQueryClient()
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}
```

```tsx
// app/layout.tsx
import { Providers } from './providers'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  )
}
```

#### `useInfiniteQuery` 핵심 패턴 (게시글 무한 스크롤)

> v5 변경점: `getNextPageParam` 반환값이 `undefined`이면 더 이상 페이지 없음으로 처리.
> `initialPageParam` 필드가 v5에서 필수 추가됨.

```tsx
// components/posts/PostsInfiniteList.tsx
'use client'

import { useInfiniteQuery } from '@tanstack/react-query'
import { useEffect, useRef } from 'react'
import type { PostSummary, PostCursor } from '@/types'

const PAGE_SIZE = 10

async function fetchPosts(cursor?: PostCursor): Promise<PostSummary[]> {
  const params = new URLSearchParams({ limit: String(PAGE_SIZE) })
  if (cursor) {
    params.set('publishedAt', cursor.publishedAt)
    params.set('cursorId', cursor.id)
  }
  const res = await fetch(`/api/posts?${params}`)
  if (!res.ok) throw new Error('게시글 로딩 실패')
  return res.json()
}

export function PostsInfiniteList({ initialPosts }: { initialPosts: PostSummary[] }) {
  const sentinelRef = useRef<HTMLDivElement>(null)

  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam }) => fetchPosts(pageParam as PostCursor | undefined),
    initialPageParam: undefined,                         // v5 필수 필드
    getNextPageParam: (lastPage): PostCursor | undefined => {
      if (lastPage.length < PAGE_SIZE) return undefined
      const last = lastPage.at(-1)!
      return { publishedAt: last.published_at, id: last.id }
    },
    initialData: { pages: [initialPosts], pageParams: [undefined] },
    initialDataUpdatedAt: Date.now(),                    // stale 판정 기준점
  })

  useEffect(() => {
    const el = sentinelRef.current
    if (!el) return
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage()
        }
      },
      { rootMargin: '200px' }
    )
    observer.observe(el)
    return () => observer.disconnect()
  }, [hasNextPage, isFetchingNextPage, fetchNextPage])

  const posts = data.pages.flat()

  return (
    <div>
      <PostGrid posts={posts} />
      <div ref={sentinelRef} />
      {isFetchingNextPage && <PostGridSkeleton />}
      {!hasNextPage && posts.length > 0 && (
        <p className="text-center text-sm text-gray-500 py-8">모든 게시글을 불러왔습니다.</p>
      )}
    </div>
  )
}
```

#### 댓글 mutation 후 캐시 invalidate 패턴

```ts
// hooks/use-add-comment.ts
'use client'

import { useMutation, useQueryClient } from '@tanstack/react-query'
import { addComment } from '@/app/actions/comments'

export function useAddComment(postId: string) {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (content: string) => addComment(postId, content),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['comments', postId] })
    },
  })
}
```

#### v4 → v5 주요 변경점 요약

| v4 | v5 |
|----|----|
| `cacheTime` | `gcTime` |
| `getNextPageParam` 반환 `null` | `undefined` 으로 변경 |
| `initialPageParam` 없음 | `useInfiniteQuery`에서 필수 |
| `isLoading` (항상) | `isPending` (쿼리 없을 때) / `isLoading` (refetch 중) |

---

### 3.7 MDXEditor

#### 설치

```bash
npm install @mdxeditor/editor
```

#### 동적 import 래퍼 (SSR 비활성화 필수)

```tsx
// components/editor/PostEditor.tsx (서버에서 import하는 쪽)
import dynamic from 'next/dynamic'

export const PostEditor = dynamic(
  () => import('./PostEditorInner').then((m) => m.PostEditorInner),
  {
    ssr: false,
    loading: () => <div className="h-96 animate-pulse bg-gray-100 rounded" />,
  }
)
```

#### 에디터 본체 구현

```tsx
// components/editor/PostEditorInner.tsx
'use client'

import { useRef } from 'react'
import {
  MDXEditor,
  MDXEditorMethods,
  headingsPlugin,
  listsPlugin,
  quotePlugin,
  thematicBreakPlugin,
  linkPlugin,
  linkDialogPlugin,
  imagePlugin,
  codeBlockPlugin,
  codeMirrorPlugin,
  toolbarPlugin,
  BlockTypeSelect,
  BoldItalicUnderlineToggles,
  CodeToggle,
  CreateLink,
  InsertImage,
  InsertCodeBlock,
  ListsToggle,
  Separator,
  UndoRedo,
} from '@mdxeditor/editor'
import '@mdxeditor/editor/style.css'
import { convertToWebP } from '@/lib/image/convert-to-webp'
import { uploadImage } from '@/app/actions/upload'

interface Props {
  postId: string
  value: string
  onChange: (markdown: string) => void
}

export function PostEditorInner({ postId, value, onChange }: Props) {
  const editorRef = useRef<MDXEditorMethods>(null)

  async function handleImageUpload(file: File): Promise<string> {
    const webp = await convertToWebP(file)
    const formData = new FormData()
    formData.append('file', webp)
    formData.append('postId', postId)
    formData.append('type', 'content_image')
    const result = await uploadImage(formData)
    if (!result.ok) throw new Error(result.message)
    return result.publicUrl
  }

  return (
    <MDXEditor
      ref={editorRef}
      markdown={value}
      onChange={onChange}
      contentEditableClassName="prose prose-neutral dark:prose-invert max-w-none min-h-[400px] px-4 py-2 focus:outline-none"
      plugins={[
        headingsPlugin(),
        listsPlugin(),
        quotePlugin(),
        thematicBreakPlugin(),
        linkPlugin(),
        linkDialogPlugin(),
        imagePlugin({ imageUploadHandler: handleImageUpload }),
        codeBlockPlugin({ defaultCodeBlockLanguage: 'ts' }),
        codeMirrorPlugin({
          codeBlockLanguages: {
            ts: 'TypeScript',
            tsx: 'TSX',
            js: 'JavaScript',
            jsx: 'JSX',
            css: 'CSS',
            sql: 'SQL',
            bash: 'Bash',
            json: 'JSON',
            md: 'Markdown',
          },
        }),
        toolbarPlugin({
          toolbarContents: () => (
            <>
              <UndoRedo />
              <Separator />
              <BlockTypeSelect />
              <Separator />
              <BoldItalicUnderlineToggles />
              <CodeToggle />
              <Separator />
              <ListsToggle />
              <Separator />
              <CreateLink />
              <InsertImage />
              <InsertCodeBlock />
            </>
          ),
        }),
      ]}
    />
  )
}
```

> **주의**: `codeMirrorPlugin`은 `codeBlockPlugin`보다 반드시 뒤에 선언해야 한다.
> CSS가 없으면 에디터가 스타일 없이 깨진 상태로 나타나므로 `@mdxeditor/editor/style.css` import는 필수다.

---

### 3.8 Markdown 렌더링 (react-markdown)

#### 설치

```bash
npm install react-markdown remark-gfm rehype-highlight highlight.js
```

#### Markdown 뷰어 컴포넌트

```tsx
// components/posts/MarkdownViewer.tsx
import ReactMarkdown from 'react-markdown'
import remarkGfm from 'remark-gfm'
import rehypeHighlight from 'rehype-highlight'
import 'highlight.js/styles/github-dark.css'  // 다크 테마 (라이트: github.css)

interface Props {
  content: string
}

export function MarkdownViewer({ content }: Props) {
  return (
    <ReactMarkdown
      remarkPlugins={[remarkGfm]}
      rehypePlugins={[rehypeHighlight]}
      components={{
        // 테이블: 모바일에서 가로 스크롤
        table: ({ ...props }) => (
          <div className="overflow-x-auto my-6">
            <table className="w-full border-collapse" {...props} />
          </div>
        ),
        th: ({ ...props }) => (
          <th className="border border-gray-300 dark:border-gray-700 bg-gray-100 dark:bg-gray-800 px-3 py-2 text-left font-semibold" {...props} />
        ),
        td: ({ ...props }) => (
          <td className="border border-gray-300 dark:border-gray-700 px-3 py-2" {...props} />
        ),
        // 이미지: 반응형
        img: ({ src, alt }) => (
          <img src={src} alt={alt ?? ''} className="max-w-full h-auto rounded-lg my-4" loading="lazy" />
        ),
        // 인라인 코드 vs 코드 블록 구분
        code: ({ className, children, ...props }) => {
          const isBlock = Boolean(className)
          if (isBlock) {
            return <code className={className} {...props}>{children}</code>
          }
          return (
            <code className="bg-gray-100 dark:bg-gray-800 text-pink-600 dark:text-pink-400 px-1.5 py-0.5 rounded text-sm font-mono" {...props}>
              {children}
            </code>
          )
        },
        // 링크: 외부 링크는 새 탭으로
        a: ({ href, children, ...props }) => {
          const isExternal = href?.startsWith('http')
          return (
            <a
              href={href}
              target={isExternal ? '_blank' : undefined}
              rel={isExternal ? 'noopener noreferrer' : undefined}
              className="text-blue-600 dark:text-blue-400 hover:underline"
              {...props}
            >
              {children}
            </a>
          )
        },
      }}
    >
      {content}
    </ReactMarkdown>
  )
}
```

> `MarkdownViewer`는 Server Component로 쓸 수 있다. `'use client'` 불필요.

#### highlight.js CSS 테마 선택지

| 테마 | import 경로 |
|------|-------------|
| GitHub Dark | `highlight.js/styles/github-dark.css` |
| GitHub Light | `highlight.js/styles/github.css` |
| Atom One Dark | `highlight.js/styles/atom-one-dark.css` |
| Nord | `highlight.js/styles/nord.css` |

다크 모드를 지원할 경우, CSS를 두 개 import하거나 CSS 변수 기반 테마(`highlight.js/styles/base16/one-light.css` 계열)를 사용하는 방법이 있다.

---

## 4. 바로 구현에 반영할 체크리스트

### 4.1 Phase 1 시작 체크리스트

- [ ] `create-next-app` — App Router + TypeScript + Tailwind + ESLint
- [ ] `npm install @supabase/ssr @supabase/supabase-js @tanstack/react-query zustand`
- [ ] `npm install @mdxeditor/editor react-markdown remark-gfm rehype-highlight highlight.js`
- [ ] `next.config.js` — `serverActions.bodySizeLimit: '6mb'` 설정
- [ ] `.env.local` 작성 (env.md 참고)
- [ ] `lib/supabase/{client,server,admin}.ts` 생성
- [ ] `lib/auth/require-admin.ts` 생성
- [ ] `types/index.ts` 공통 타입 작성
- [ ] `app/providers.tsx` — QueryClientProvider
- [ ] `app/layout.tsx` — Providers 적용
- [ ] Supabase: DDL 실행 (schema.md 기준)
- [ ] Supabase: Trigger / RPC 실행
- [ ] Supabase: RLS 정책 적용 후 anon/authenticated 역할로 동작 검증
- [ ] Supabase: `images` 버킷 생성 + Storage 정책 적용
- [ ] Supabase: Google/GitHub OAuth Provider 등록 + Redirect URL 설정

### 4.2 인증 체크리스트

- [ ] `app/login/page.tsx` — `signInWithOAuth` 버튼
- [ ] `app/auth/callback/route.ts` — code 교환, 관리자 분기 리다이렉트
- [ ] `middleware.ts` — 세션 갱신 + `/admin` 보호
- [ ] `lib/auth/require-admin.ts`
- [ ] 로그아웃 버튼 (`supabase.auth.signOut()`)
- [ ] 비로그인 상태에서 `/admin` 접근 시 `/login`으로 리다이렉트 동작 확인
- [ ] 비관리자 계정으로 로그인 후 `/admin` 접근 차단 확인

### 4.3 글 작성 체크리스트

- [ ] `createDraft` Server Action
- [ ] `saveDraft` Server Action (thumbnailFileId 연결 포함)
- [ ] `publishPost` Server Action + `revalidatePath`
- [ ] `updatePost` Server Action
- [ ] `deletePost` Server Action (Storage → files → posts 순서)
- [ ] `uploadImage` Server Action (서버 MIME 재검증 포함)
- [ ] `convertToWebP` 클라이언트 유틸
- [ ] `PostEditorInner` (dynamic import, 이미지 업로드 핸들러 연결)
- [ ] 자동 저장 인터벌 (20~30초, 저장 상태 표시)
- [ ] 클립보드 붙여넣기 이미지 업로드
- [ ] Drag & Drop 이미지 업로드

### 4.4 공개 화면 체크리스트

- [ ] `GET /api/posts` Route Handler (cursor 기반)
- [ ] `app/(public)/posts/page.tsx` — tag 분기 (무한스크롤 / 페이지네이션)
- [ ] `PostsInfiniteList` Client Component
- [ ] `getPostsByTag` 서버 함수 (tags!inner(slug) 포함)
- [ ] `Pagination` 컴포넌트
- [ ] `app/(public)/posts/[slug]/page.tsx` — `generateMetadata` 포함
- [ ] `MarkdownViewer` 컴포넌트
- [ ] `LikeButton` Client Component (TanStack Query)
- [ ] `CommentList` Client Component (`useInfiniteQuery`)
- [ ] `app/sitemap.ts`
- [ ] `app/robots.ts`
- [ ] `GET /api/rss` Route Handler

---

## 5. 추천 구현 순서

```
1. Phase 1 인프라 완성 (Supabase 연결 + 인증 동작 확인)
2. Phase 2 관리자 인증 (/admin 보호 완성)
3. Phase 3 글 작성 최소 버전 (제목 + 본문 + 발행 + 삭제)
   → 이 시점에 게시글 데이터가 생겨서 이후 화면을 실데이터로 개발 가능
4. Phase 3 이미지 업로드 + 자동 저장 추가
5. Phase 4 공개 목록 (무한스크롤 + 태그 필터)
6. Phase 5 게시글 상세 (Markdown 렌더링 + 목차)
7. Phase 5 좋아요 + 댓글 추가
8. Phase 6 검색 + 태그 관리
9. Phase 7 SEO (sitemap, robots, RSS, metadata)
10. Phase 8 관리자 대시보드 (댓글·이미지 관리)
11. Phase 9 마무리 (다크 모드, 에러 페이지, 반응형, 배포)
```

> **이 순서의 이유**: "관리자 작성 경험"이 먼저 완성되어야 이후 모든 공개 화면을 실제 데이터로 검증할 수 있다.

---

## 6. 자주 막히는 포인트

### 6.1 이미지 업로드가 실패하는 경우

**증상**: 업로드 Server Action이 오류 반환하거나 파일이 저장되지 않음

우선 확인 순서:

1. `next.config.js` — `bodySizeLimit` 설정 여부
2. Storage RLS — admin insert 정책이 service role Key를 사용할 때는 우회됨, 확인 불필요
3. `contentType` — Storage upload 시 `contentType: file.type` 지정 여부
4. `SUPABASE_SERVICE_ROLE_KEY` — 환경 변수가 Vercel/로컬에 제대로 설정됐는지
5. 버킷명 오타 — `'images'`가 정확한지

### 6.2 로그인은 됐는데 Server Component에서 유저가 비는 경우

**증상**: 로그인 후에도 `supabase.auth.getUser()`가 `null` 반환

우선 확인 순서:

1. `middleware.ts` 존재 여부 + matcher 설정이 `/admin/**` 경로를 포함하는지
2. `lib/supabase/server.ts`의 `createClient()`가 `async function`으로 선언됐는지
3. `const cookieStore = await cookies()`로 호출했는지 (`await` 누락 시 쿠키 없음)
4. `/auth/callback`에서 `exchangeCodeForSession` 성공했는지 (URL의 `?error=` 확인)

### 6.3 RLS 때문에 조회가 비거나 쓰기가 실패하는 경우

**증상**: Supabase 응답 `data`가 `[]`이거나 insert/update 에러

우선 확인 순서:

1. Supabase Dashboard → Table Editor → 해당 테이블의 RLS enabled 여부
2. Authentication → Policies → SELECT/INSERT/UPDATE/DELETE 정책 각각 존재 여부
3. SQL Editor에서 `SET ROLE anon;` 후 직접 쿼리 실행해 결과 확인
4. `supabaseAdmin` 써야 할 곳에서 일반 `supabase`를 쓰고 있지 않은지

### 6.4 `useInfiniteQuery` initialData가 즉시 다시 fetch되는 경우

**증상**: 페이지 첫 렌더 직후 `/api/posts`가 즉시 호출됨

원인: `initialData`는 `staleTime` 기준으로 즉시 stale 판정됨.

해결:

```ts
useInfiniteQuery({
  // ...
  initialData: { pages: [initialPosts], pageParams: [undefined] },
  initialDataUpdatedAt: Date.now(),  // 이 줄이 없으면 즉시 재fetch됨
})
```

### 6.5 MDXEditor CSS가 적용 안 되는 경우

**증상**: 에디터가 텍스트만 나오고 스타일이 없음

원인: CSS import 누락.

```ts
// PostEditorInner.tsx 최상단에 반드시 포함
import '@mdxeditor/editor/style.css'
```

### 6.6 rehype-highlight에서 코드가 하이라이팅되지 않는 경우

**증상**: 코드 블록이 plain text로 렌더됨

우선 확인 순서:

1. `highlight.js/styles/*.css` import 여부
2. Markdown 코드 펜스에 언어 명시 여부: ` ```ts ` vs ` ``` `
3. `rehypeHighlight` 옵션에 `detect: true` 추가 (언어 없을 때 자동 감지)

### 6.7 Markdown 테이블이 모바일에서 넘치는 경우

`MarkdownViewer`의 `table` 커스텀 컴포넌트에서 `overflow-x-auto` 감싸기 패턴을 적용한다 (3.8 섹션 참고).

---

## 7. Vercel 배포 시 주의사항

```
Vercel Project Settings → Environment Variables에 아래 값을 모두 설정:
- NEXT_PUBLIC_SUPABASE_URL
- NEXT_PUBLIC_SUPABASE_ANON_KEY
- SUPABASE_SERVICE_ROLE_KEY
- ADMIN_EMAIL
- NEXT_PUBLIC_SITE_URL   ← 프로덕션 도메인으로 변경 필요
```

배포 후 Supabase Dashboard에서도 수정 필요:

```
Authentication → URL Configuration:
- Site URL: 프로덕션 도메인
- Redirect URLs: https://yourdomain.com/auth/callback
```

이 설정이 없으면 OAuth 로그인 후 callback이 실패한다.

---

## 8. 참고 자료

- [Next.js App Router](https://nextjs.org/docs/app)
- [Next.js Server Actions Config (bodySizeLimit)](https://nextjs.org/docs/app/api-reference/config/next-config-js/serverActions)
- [Supabase Next.js SSR 가이드](https://supabase.com/docs/guides/auth/server-side/nextjs)
- [Supabase SSR 클라이언트 생성 상세](https://supabase.com/docs/guides/auth/server-side/creating-a-client)
- [TanStack Query v5 Advanced SSR](https://tanstack.com/query/v5/docs/framework/react/guides/advanced-ssr)
- [TanStack Query v5 마이그레이션 가이드](https://tanstack.com/query/v5/docs/framework/react/guides/migrating-to-v5)
- [MDXEditor 시작 가이드](https://mdxeditor.dev/editor/docs/getting-started)
- [MDXEditor 플러그인 목록](https://mdxeditor.dev/editor/docs/plugins)
- [react-markdown](https://github.com/remarkjs/react-markdown)
- [remark-gfm](https://github.com/remarkjs/remark-gfm)
- [rehype-highlight](https://github.com/rehypejs/rehype-highlight)
- [Vercel 환경 변수](https://vercel.com/docs/environment-variables)
