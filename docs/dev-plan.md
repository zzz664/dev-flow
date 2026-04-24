# 개발 계획서

## 1. 프로젝트 개요

| 항목       | 내용                                                                     |
| ---------- | ------------------------------------------------------------------------ |
| 프로젝트명 | 미정                                                                     |
| 유형       | 개인 기술 블로그                                                         |
| 개발 인원  | 1인                                                                      |
| 기술 스택  | Next.js 15 (App Router), Supabase, Tailwind CSS, TanStack Query, Zustand |

---

## 2. 기술 스택

| 영역            | 기술                                                        |
| --------------- | ----------------------------------------------------------- |
| 프레임워크      | Next.js 15 App Router                                       |
| 스타일링        | Tailwind CSS                                                |
| 서버 상태       | Supabase (PostgreSQL, Auth, Storage)                        |
| 클라이언트 상태 | TanStack Query v5 (댓글·좋아요), Zustand (에디터 임시 상태) |
| 에디터          | MDXEditor                                                   |
| Markdown 뷰어   | react-markdown + remark-gfm + rehype-highlight              |
| 배포            | Vercel                                                      |

---

## 3. 개발 원칙

- **서버 우선**: 데이터 조회는 Server Component, 변경은 Server Action. TanStack Query는 댓글·좋아요 등 클라이언트 상호작용에만 사용한다.
- **단순성 유지**: 추상화는 실제 중복이 발생할 때만 도입한다. 미래 요구사항을 위한 설계는 하지 않는다.
- **보안 우선**: `SUPABASE_SERVICE_ROLE_KEY`는 서버 전용. 모든 Server Action에서 관리자 검증 선행.
- **삭제 순서 고정**: Storage 파일 → `files` 레코드 → `posts` 레코드 순서를 절대 바꾸지 않는다.
- **Phase 완료 기준**: 해당 Phase의 핵심 기능이 로컬에서 동작하고 에러 케이스가 처리된 상태.

---

## 4. 단계별 개발 계획

### Phase 1 — 기반 구축

**목표**: 개발 환경과 Supabase 인프라를 완성한다. 이후 모든 Phase의 전제 조건.

| 작업                    | 내용                                                                    |
| ----------------------- | ----------------------------------------------------------------------- |
| Next.js 프로젝트 초기화 | TypeScript, Tailwind, ESLint, Prettier 설정                             |
| Supabase 설정           | DDL 실행, Trigger/RLS/Storage 정책 적용, OAuth 설정                     |
| 공통 기반 코드          | Supabase 클라이언트 (server/client/admin), 공통 타입, 레이아웃 컴포넌트 |

**산출물**: 빈 Next.js 앱이 Supabase에 연결되고 로그인이 동작하는 상태.

---

### Phase 2 — 관리자 인증

**목표**: 관리자만 `/admin` 접근 가능. 방문자 소셜 로그인 기반 마련.

| 작업            | 내용                                           |
| --------------- | ---------------------------------------------- |
| 로그인 페이지   | Google/GitHub OAuth 버튼                       |
| Auth Callback   | `/auth/callback` Route Handler                 |
| 라우트 보호     | `middleware.ts` — `/admin` 접근 시 이메일 검증 |
| 관리자 레이아웃 | `/admin` 공통 레이아웃                         |

**산출물**: 관리자 로그인 → `/admin` 진입 가능. 비관리자 접근 차단.

---

### Phase 3 — 게시글 작성

**목표**: 관리자가 Markdown 게시글을 작성하고 발행할 수 있다.

| 작업           | 내용                                                      |
| -------------- | --------------------------------------------------------- |
| 초안 생성      | `createDraft` Server Action                               |
| MDXEditor 연동 | 제목, 본문, 태그 선택, 썸네일                             |
| 이미지 업로드  | WebP 변환 → Storage 업로드 → URL 삽입 (붙여넣기·DnD 포함) |
| 자동 저장      | `saveDraft` — 20~30초 간격                                |
| 발행·수정·삭제 | `publishPost`, `updatePost`, `deletePost` Server Action   |

**의존성**: Phase 1, 2 완료 후 진행.

**산출물**: 에디터에서 글 작성 → 발행까지 전체 흐름 동작.

---

### Phase 4 — 공개 게시글 목록

**목표**: 방문자가 게시글 목록을 탐색할 수 있다.

| 작업                           | 내용                                                                             |
| ------------------------------ | -------------------------------------------------------------------------------- |
| `/posts` 무한 스크롤           | Server Component 초기 렌더 + `useInfiniteQuery` + `GET /api/posts` (cursor 기반) |
| `/posts?tag=slug` 페이지네이션 | tag 파라미터 분기, `getPostsByTag`                                               |
| 홈 페이지                      | 최신 글, 인기 글(좋아요/조회수)                                                  |

**의존성**: Phase 3 (게시글 데이터 필요).

---

### Phase 5 — 게시글 상세

**목표**: 방문자가 게시글을 읽을 수 있다. 조회수 집계 포함.

| 작업        | 내용                                                                          |
| ----------- | ----------------------------------------------------------------------------- |
| 상세 페이지 | Markdown 렌더링, 목차 생성, 이전/다음 글                                      |
| 조회수 증가 | `increment_view_count` RPC 호출                                               |
| 좋아요      | `toggleLike` Server Action + `LikeButton` Client Component                    |
| 댓글        | `addComment/updateComment/deleteComment` + `CommentList` (cursor 무한 스크롤) |
| metadata    | OG 태그, canonical URL                                                        |

**의존성**: Phase 2 (소셜 로그인으로 댓글·좋아요), Phase 4 (게시글 존재).

---

### Phase 6 — 태그 관리 및 검색

**목표**: 태그를 관리할 수 있고, 방문자가 검색할 수 있다.

| 작업      | 내용                                                 |
| --------- | ---------------------------------------------------- |
| 태그 관리 | `/admin/tags` CRUD (Server Action)                   |
| 검색      | `/search` — 제목·본문·태그명 통합 검색, 페이지네이션 |
| 아카이브  | `/archive` — 연도/월별 그루핑                        |

---

### Phase 7 — SEO 및 피드

**목표**: 검색 엔진 인덱싱과 구독 기능을 완성한다.

| 작업       | 내용                                     |
| ---------- | ---------------------------------------- |
| sitemap    | `app/sitemap.ts` — 발행 게시글 slug 기반 |
| robots.txt | `app/robots.ts` — `/admin` disallow      |
| RSS        | `GET /api/rss` — 최근 20개               |

---

### Phase 8 — 관리자 대시보드

**목표**: 운영에 필요한 관리 기능을 갖춘다.

| 작업        | 내용                                    |
| ----------- | --------------------------------------- |
| 댓글 관리   | `/admin/comments` — 전체 목록·삭제      |
| 이미지 관리 | `/admin/images` — 파일 목록·orphan 정리 |
| 게시글 통계 | 조회수·좋아요 요약                      |

---

### Phase 9 — 마무리

**목표**: 품질 점검과 배포.

| 작업        | 내용                                        |
| ----------- | ------------------------------------------- |
| 읽기 편의   | 다크 모드, 코드 복사 버튼                   |
| 에러 처리   | `not-found.tsx`, `error.tsx`, 로딩 Skeleton |
| 반응형 점검 | 모바일·태블릿 레이아웃                      |
| 성능 최적화 | Next.js `Image`, 번들 사이즈 점검           |
| 배포        | Vercel 배포, 도메인 연결, Supabase URL 설정 |

---

## 5. Phase 의존성 요약

```
Phase 1 (기반)
  └─ Phase 2 (인증)
       └─ Phase 3 (글 작성)
            ├─ Phase 4 (목록)
            │    └─ Phase 5 (상세) ─── Phase 6 (태그·검색)
            │                               └─ Phase 7 (SEO)
            └─ Phase 8 (관리자 대시보드)  ─────┘
                    └─ Phase 9 (마무리)
```

---

## 6. 리스크 관리

| 리스크                             | 영향         | 대응                                                             |
| ---------------------------------- | ------------ | ---------------------------------------------------------------- |
| MDXEditor 커스터마이징 복잡도      | Phase 3 지연 | 기본 플러그인 구성 우선 적용, 세부 커스터마이징은 Phase 9로 이동 |
| Supabase RLS 정책 오동작           | 인증 버그    | Phase 1에서 RLS 정책을 테스트 계정으로 반드시 검증               |
| `useInfiniteQuery` + SSR hydration | Phase 4 지연 | `initialData` prop 방식 사용 (HydrationBoundary 회피)            |
| 이미지 WebP 변환 브라우저 호환성   | Phase 3 품질 | Canvas API 우선, 실패 시 원본 업로드로 fallback                  |
