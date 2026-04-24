# WBS (Work Breakdown Structure)

> **난이도**: S = 반나절 이하 / M = 1일 / L = 2~3일  
> **선행**: 해당 태스크 착수 전에 완료되어야 하는 태스크 ID

---

## Phase 1 — 기반 구축

### 1.1 개발 환경 설정

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 1.1.1 | Next.js 프로젝트 초기화 | `create-next-app` (TypeScript, Tailwind, App Router) | S | — |
| 1.1.2 | 패키지 설치 | supabase-js, TanStack Query, Zustand, MDXEditor, react-markdown, remark-gfm, rehype-highlight | S | 1.1.1 |
| 1.1.3 | 코드 품질 도구 | ESLint, Prettier, `.editorconfig` 설정 | S | 1.1.1 |
| 1.1.4 | 디렉토리 구조 설정 | `app/`, `components/`, `lib/`, `types/`, `app/actions/` 골격 생성 | S | 1.1.1 |

### 1.2 Supabase 인프라

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 1.2.1 | Supabase 프로젝트 생성 | 프로젝트 생성, `.env.local` 환경 변수 설정 | S | — |
| 1.2.2 | DDL 실행 | posts, tags, post_tags, comments, likes, files 테이블 생성 + 인덱스 | M | 1.2.1 |
| 1.2.3 | DB Trigger / RPC | like_count 동기화 trigger, updated_at trigger, increment_view_count RPC | M | 1.2.2 |
| 1.2.4 | RLS 정책 적용 | 6개 테이블 RLS 정책 (schema.md 기준) | M | 1.2.2 |
| 1.2.5 | Storage 설정 | `images` 버킷 생성, public read / admin write 정책 | S | 1.2.1 |
| 1.2.6 | OAuth 설정 | Google/GitHub Provider 등록, Redirect URL 설정 | S | 1.2.1 |

### 1.3 공통 기반 코드

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 1.3.1 | Supabase 클라이언트 | `lib/supabase/server.ts`, `client.ts`, `admin.ts` | S | 1.2.1 |
| 1.3.2 | 인증 유틸 | `lib/auth/require-admin.ts`, `lib/auth/get-user.ts` | S | 1.3.1 |
| 1.3.3 | 공통 타입 정의 | `types/` — Post, Tag, Comment, Like, StoredFile, ActionResult | S | 1.1.4 |
| 1.3.4 | TanStack Query Provider | `app/providers.tsx` — QueryClientProvider 설정 | S | 1.1.2 |
| 1.3.5 | 공통 레이아웃 | `app/layout.tsx`, Header, Footer 컴포넌트 (빈 껍데기) | M | 1.1.1 |

---

## Phase 2 — 관리자 인증

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 2.1 | 로그인 페이지 | `/login` — Google/GitHub OAuth 버튼, `signInWithOAuth` 호출 | S | 1.3.1, 1.2.6 |
| 2.2 | Auth Callback | `app/auth/callback/route.ts` — code 교환, 세션 설정, 리다이렉트 | S | 2.1 |
| 2.3 | 관리자 미들웨어 | `middleware.ts` — `/admin` 접근 시 `ADMIN_EMAIL` 검증, 비관리자 `/` 리다이렉트 | M | 1.3.2, 2.2 |
| 2.4 | 관리자 레이아웃 | `app/admin/layout.tsx` — 사이드바, 네비게이션 | M | 2.3 |
| 2.5 | 로그아웃 | `signOut` 호출, 세션 삭제 | S | 2.2 |
| 2.6 | 인증 동작 검증 | 관리자 계정 로그인, 비관리자 차단, 세션 유지 확인 | S | 2.1~2.5 |

---

## Phase 3 — 게시글 작성

### 3.1 Server Actions

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 3.1.1 | `createDraft` | `status='draft'`, 임시 slug 생성, postId 반환 | S | 1.3.2 |
| 3.1.2 | `saveDraft` | title/content/tagIds/thumbnailFileId 업데이트, thumbnail 재연결 처리 | M | 3.1.1 |
| 3.1.3 | `publishPost` | status='published', published_at 설정, slug 확정 | S | 3.1.2 |
| 3.1.4 | `updatePost` | 발행 후 수정 (slug 변경 불가) | S | 3.1.3 |
| 3.1.5 | `deletePost` | Storage → files 레코드 → posts 순서 삭제 | M | 3.1.1 |

### 3.2 이미지 업로드

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 3.2.1 | `uploadImage` Server Action | 파일 검증, UUID 경로 생성, Storage 업로드, files 레코드 삽입 | M | 1.3.1 |
| 3.2.2 | 클라이언트 WebP 변환 | Canvas API — 최대 1200px 리사이즈, WebP 변환 유틸 | M | — |
| 3.2.3 | MDXEditor 이미지 핸들러 | 이미지 삽입 시 uploadImage 호출, 반환 URL 본문 삽입 | M | 3.2.1, 3.2.2 |
| 3.2.4 | 클립보드 붙여넣기 | paste 이벤트 감지 → 자동 업로드 | M | 3.2.3 |
| 3.2.5 | Drag & Drop | dragover/drop 이벤트 → 자동 업로드 | M | 3.2.3 |

### 3.3 에디터 페이지

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 3.3.1 | 에디터 페이지 진입 | `/admin/posts/new` — createDraft 호출 후 postId 전달 | S | 3.1.1 |
| 3.3.2 | MDXEditor 기본 연동 | 제목 입력, 본문 편집, 기본 툴바 (h1~h6, 목록, 인용, 코드, 링크, 이미지) | L | 3.2.3 |
| 3.3.3 | 태그 선택 UI | 기존 태그 목록 조회, 다중 선택, `tagIds` 상태 관리 | M | 3.1.2 |
| 3.3.4 | 썸네일 업로드 UI | 이미지 선택 → WebP 변환 → uploadImage → fileId 상태 저장 | M | 3.2.1, 3.2.2 |
| 3.3.5 | 자동 저장 | 20~30초 interval로 saveDraft 호출, 저장 상태 표시 | M | 3.1.2 |
| 3.3.6 | 발행 버튼 | publishPost 호출, 성공 시 `/posts/[slug]`로 이동 | S | 3.1.3 |
| 3.3.7 | 글 수정 페이지 | `/admin/posts/[slug]/edit` — 기존 데이터 로딩, updatePost | M | 3.1.4, 3.3.2 |
| 3.3.8 | 글 삭제 | 삭제 확인 다이얼로그, deletePost 호출 | S | 3.1.5 |

---

## Phase 4 — 공개 게시글 목록

### 4.1 Route Handler

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 4.1.1 | `GET /api/posts` | cursor 기반 페이지 조회 (publishedAt + cursorId), limit 파라미터 | M | 1.3.1 |

### 4.2 게시글 목록 페이지

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 4.2.1 | PostCard 컴포넌트 | 썸네일, 제목, 태그, 날짜, 조회수/좋아요 | M | — |
| 4.2.2 | `/posts` Server Component | 초기 1페이지 fetch (getInitialPosts), PostsInfiniteList에 전달 | S | 4.1.1 |
| 4.2.3 | PostsInfiniteList | `useInfiniteQuery`, IntersectionObserver sentinel, initialData 주입 | L | 4.2.2 |
| 4.2.4 | PostListSkeleton | 로딩 fallback UI | S | 4.2.1 |
| 4.2.5 | 태그 필터 분기 | `searchParams.tag` 유무로 무한스크롤 / 페이지네이션 분기 | M | 4.2.3 |
| 4.2.6 | `getPostsByTag` | post_tags!inner + tags!inner 조인 쿼리, offset 페이지네이션 | M | 1.3.1 |
| 4.2.7 | Pagination 컴포넌트 | 현재 페이지 ±2 + 처음/끝, Link 기반, extraParams 지원 | M | — |
| 4.2.8 | TagFilterBadge | 현재 필터 태그 표시, 제거 버튼 | S | — |

### 4.3 홈 페이지

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 4.3.1 | 최신 글 목록 | 최근 4개 (Server Component, ISR 60s) | S | 4.2.1 |
| 4.3.2 | 인기 글 목록 | 좋아요 TOP / 조회수 TOP (Server Component) | S | 4.2.1 |
| 4.3.3 | 초안 목록 (관리자) | 관리자 로그인 상태일 때만 노출 (draft 목록) | M | 2.3 |

---

## Phase 5 — 게시글 상세

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 5.1 | 상세 페이지 Server Component | post + post_tags + tags 조회, 없으면 not-found | M | 1.3.1 |
| 5.2 | Markdown 렌더링 | react-markdown + remark-gfm + rehype-highlight, 스타일링 | L | 5.1 |
| 5.3 | 목차 자동 생성 | 헤딩 파싱 → 목차 컴포넌트, anchor 링크, 활성 항목 하이라이트 | L | 5.2 |
| 5.4 | 조회수 증가 | `increment_view_count` RPC 비동기 호출 (응답 대기 불필요) | S | 5.1 |
| 5.5 | 이전/다음 글 | published_at 기준 인접 게시글 조회 | M | 5.1 |
| 5.6 | metadata | `generateMetadata` — OG title/description/image, canonical | M | 5.1 |
| 5.7 | `toggleLike` Server Action | likes INSERT/DELETE, like_count 반환 | S | 1.3.2 |
| 5.8 | LikeButton | `useQuery` + `useMutation`, 낙관적 업데이트 | M | 5.7 |
| 5.9 | `addComment` Server Action | user_id, nickname(스냅샷), content 검증 후 삽입 | S | 1.3.2 |
| 5.10 | `updateComment` / `deleteComment` | 본인 검증 후 수정 / 소프트 삭제 | S | 5.9 |
| 5.11 | CommentList | `useInfiniteQuery` (cursor 기반), 버튼 트리거 추가 로딩 | L | 5.9 |
| 5.12 | CommentForm | 소셜 로그인 요구 처리 (비로그인 시 로그인 유도) | M | 5.9 |
| 5.13 | 코드 복사 버튼 | rehype-highlight 코드블록에 복사 버튼 삽입 | M | 5.2 |

---

## Phase 6 — 태그 관리 및 검색

### 6.1 태그 관리

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 6.1.1 | `createTag` Server Action | name, slug 중복 검사 후 삽입 | S | 1.3.2 |
| 6.1.2 | `updateTag` / `deleteTag` | 수정 / CASCADE 삭제 | S | 6.1.1 |
| 6.1.3 | `/admin/tags` 페이지 | 태그 목록, 추가/수정/삭제 UI | M | 6.1.2 |

### 6.2 검색

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 6.2.1 | `searchPosts` 서버 함수 | 검색어 전처리(escape), 제목·본문·태그명 2단계 쿼리 | M | 1.3.1 |
| 6.2.2 | `/search` 페이지 | SearchForm, 결과 목록(PostListView), 페이지네이션 | M | 6.2.1, 4.2.7 |

### 6.3 아카이브

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 6.3.1 | `/archive` 페이지 | published_at 전체 조회 → 클라이언트 연도/월 그루핑 | M | 1.3.1 |

---

## Phase 7 — SEO 및 피드

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 7.1 | `app/sitemap.ts` | 발행 게시글 slug → URL 목록 생성 | S | 5.1 |
| 7.2 | `app/robots.ts` | `/admin` Disallow, Sitemap URL 포함 | S | — |
| 7.3 | `GET /api/rss` | RSS 2.0 XML, 최근 20개, revalidate 1시간 | M | 5.1 |
| 7.4 | 공통 metadata | `app/layout.tsx` — 기본 title, description, OG, favicon | S | — |

---

## Phase 8 — 관리자 대시보드

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 8.1 | `/admin/comments` | 전체 댓글 목록(페이지네이션), 소프트 삭제 | M | 5.10 |
| 8.2 | `deleteImage` Server Action | Storage 파일 + files 레코드 삭제 | S | 1.3.2 |
| 8.3 | `/admin/images` | 파일 목록(게시글 연결 여부), orphan 정리 실행 버튼 | M | 8.2 |
| 8.4 | 게시글 통계 | `/admin/posts` 목록에 조회수·좋아요 수 표시 | S | 5.4 |
| 8.5 | 초안 목록 | `/admin/posts` — 초안/발행 탭 분리 | M | 3.1.1 |

---

## Phase 9 — 마무리

| ID | 태스크 | 상세 | 난이도 | 선행 |
|----|--------|------|--------|------|
| 9.1 | 다크 모드 | Tailwind `dark:` 클래스, 시스템 설정 연동 | M | — |
| 9.2 | 에러 페이지 | `not-found.tsx`, `error.tsx` | S | — |
| 9.3 | 로딩 Skeleton | 게시글 목록, 상세, 댓글 Skeleton UI | M | — |
| 9.4 | 반응형 점검 | 모바일(375px), 태블릿(768px) 레이아웃 최종 확인 | M | — |
| 9.5 | 성능 점검 | `next/image` 적용, Lighthouse 측정, 불필요 번들 제거 | M | 모든 Phase |
| 9.6 | Vercel 배포 | 배포, 환경 변수 설정 | S | 9.5 |
| 9.7 | 도메인 및 Supabase URL | 도메인 연결, Supabase Redirect URL·Site URL 갱신 | S | 9.6 |

---

## 전체 태스크 수 요약

| Phase | 태스크 수 | 합계 난이도 |
|-------|-----------|-------------|
| Phase 1 — 기반 구축 | 12 | M×6, S×6 |
| Phase 2 — 관리자 인증 | 6 | M×3, S×3 |
| Phase 3 — 게시글 작성 | 13 | L×1, M×8, S×4 |
| Phase 4 — 게시글 목록 | 11 | L×1, M×7, S×3 |
| Phase 5 — 게시글 상세 | 13 | L×3, M×6, S×4 |
| Phase 6 — 태그·검색·아카이브 | 7 | M×5, S×2 |
| Phase 7 — SEO·피드 | 4 | M×1, S×3 |
| Phase 8 — 관리자 대시보드 | 5 | M×3, S×2 |
| Phase 9 — 마무리 | 7 | M×4, S×3 |
| **합계** | **78** | L×5, M×43, S×30 |
