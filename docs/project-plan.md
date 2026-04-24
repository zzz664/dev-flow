# 개인 블로그 프로젝트 기획서

## 1. 프로젝트 개요

이 프로젝트는 `Next.js + Tailwind CSS + Zustand + TanStack Query + Supabase` 조합으로 만드는 개인 기술 블로그다.

핵심 방향은 다음과 같다.

- 블로그 주인 1명만 관리자 권한을 가진다.
- 방문자는 글을 읽을 수 있다.
- 댓글과 좋아요는 소셜로그인을 통해 사용한다.
- 회원가입은 제공하지 않는다.
- 게시판은 1개만 두고 태그 중심으로 탐색한다.
- 글쓰기는 Markdown 기반으로 진행한다.

## 2. 서비스 목표

이 블로그는 커뮤니티 서비스가 아니라 개인 콘텐츠 발행 플랫폼이다.

목표는 다음과 같다.

- 글을 빠르고 안정적으로 작성할 수 있어야 한다.
- 방문자는 글을 쉽게 찾고 읽을 수 있어야 한다.
- 댓글과 좋아요는 최소한의 진입 장벽으로 운영할 수 있어야 한다.
- 관리자 입장에서는 유지보수가 쉬워야 한다.

## 3. 사용자 유형

### 3.1 관리자

- 블로그 주인
- 게시글 작성, 수정, 삭제 가능
- 초안 저장 가능
- 태그 관리 가능
- 댓글 관리 가능
- 좋아요 통계 확인 가능
- 이미지 업로드 및 정리 가능

### 3.2 방문자

- 글 읽기 가능
- 태그별 글 목록 확인 가능
- 댓글 작성 가능
- 좋아요 가능
- 댓글/좋아요는 소셜로그인을 통해 식별한다

## 4. 핵심 정책

### 4.1 회원가입 없음

별도의 회원가입 기능은 두지 않는다.

방문자가 댓글이나 좋아요를 사용하려면 소셜로그인을 사용한다.

권장 소셜 계정:

- Google
- GitHub

### 4.2 댓글과 좋아요의 인증 방식

방문자가 댓글을 달거나 좋아요를 누르려면 소셜로그인을 사용한다.

이 방식을 선택하는 이유는 다음과 같다.

- 익명 식별보다 안정적이다.
- 댓글 수정/삭제, 좋아요 중복 방지가 쉽다.
- 스팸 관리가 쉬워진다.
- 운영 규칙이 단순해진다.

읽기는 로그인 없이 가능하지만, 반응 기능은 로그인 후 사용하도록 한다.

### 4.3 게시판은 하나, 태그는 여러 개

게시판은 하나만 두고 태그를 중심으로 탐색 구조를 만든다.

예시:

- 전체 글
- 태그별 글 목록
- 최신 글
- 인기 글(조회수 TOP)
- 인기 글(좋아요 TOP)
- 아카이브

인기 글은 두 가지 기준으로 분리해서 노출한다.

- 조회수 기반 인기 글
- 좋아요 기반 인기 글

## 5. 필수 기능

### 5.1 관리자 인증

- 관리자 로그인
- 세션 유지
- 로그아웃
- 관리자 전용 라우트 보호

### 5.2 게시글 관리

- 글 작성
- 글 수정
- 글 삭제
- 초안 저장
- 발행
- 비발행 상태 관리
- 작성 중 자동 저장

### 5.3 Markdown 글쓰기

- Markdown 입력
- 제목, 본문, 태그, 썸네일 입력
- 코드 블록 지원
- 이미지 삽입 지원

### 5.4 이미지 업로드

- 파일 업로드
- 클립보드 붙여넣기
- Drag & Drop 업로드
- Supabase Storage 저장
- public URL 반환
- 본문에 자동 삽입

### 5.5 태그 탐색

- 태그별 글 목록
- 태그 필터링
- 태그 다중 선택 가능 여부는 추후 확장 가능

### 5.6 댓글

- 댓글 작성
- 댓글 목록 조회
- 댓글 삭제
- 댓글 수정 지원
- 댓글 작성자 식별

### 5.7 좋아요

- 좋아요 누르기/취소
- 게시글별 좋아요 수 집계
- 로그인한 사용자 기준 중복 방지

좋아요 수는 `likes` 테이블과 `posts.like_count`가 항상 일치해야 한다.
이를 위해 좋아요 추가/삭제 시점에 Supabase DB trigger 또는 function으로 `posts.like_count`를 자동 갱신한다.

### 5.8 SEO

기술 블로그에서는 노출이 곧 기능이다.

따라서 아래 항목은 필수로 본다.

- `metadata`
- Open Graph
- sitemap
- robots.txt
- canonical URL

### 5.9 검색

- 제목 검색
- 본문 검색
- 태그 검색

검색은 기술 블로그의 핵심 기능이므로 추가 기능이 아니라 필수 기능으로 본다.

## 6. 추가로 있으면 좋은 기능

### 6.1 아카이브

- 연도별 글 목록
- 월별 글 목록

오래된 글 탐색이 쉬워진다.

### 6.2 RSS

- 기술 블로그와 잘 맞는다.
- 구독자 유입에 유리하다.

### 6.3 읽기 편의 기능

- 목차 자동 생성
- 생성된 목차 항목 클릭 시 해당 헤딩 위치로 스크롤 이동
- 코드 복사 버튼
- 다크 모드
- 글 하단 이전/다음 글 이동

### 6.4 운영 편의 기능

- 게시글 통계
- 댓글 관리 페이지
- 이미지 정리 페이지
- 태그 관리 페이지

## 7. 추천 기술 구조

### 7.1 프론트엔드

- `Next.js App Router`
- `Tailwind CSS`
- `Zustand`
  - 작성 중인 글 상태
  - 관리자 세션 보조 상태
- `TanStack Query`
  - 댓글 조회
  - 좋아요 상태
  - 태그 목록
  - 사용자 상호작용이 필요한 서버 상태

#### 역할 분리 원칙

App Router의 Server Component는 이미 서버 데이터 페칭에 강하다.
따라서 게시글 목록, 상세글 같은 정적 성격 데이터는 Server Component 중심으로 가져가고, TanStack Query는 댓글, 좋아요, 실시간 갱신처럼 클라이언트 상호작용이 강한 영역에만 쓰는 것을 권장한다.

### 7.2 백엔드

- `Supabase Auth`
  - 관리자 로그인
  - 방문자 소셜로그인
- `Supabase Database`
  - posts
  - tags
  - post_tags
  - comments
  - likes
  - like_count 동기화용 trigger/function
- `Supabase Storage`
  - 썸네일
  - 본문 삽입 이미지

## 8. 관리자 식별 방식

블로그 주인 1명만 관리자라는 조건은 명시적으로 코드와 DB 정책에 반영해야 한다.

선택한 방식은 **8.1 환경 변수 기반 허용 이메일**이다.

### 8.1 채택: 환경 변수 기반 허용 이메일

- `ADMIN_EMAIL` 환경 변수를 사용한다 (단일 이메일 값).
- 로그인 후 Supabase Auth의 `user.email`이 `ADMIN_EMAIL`과 일치하면 관리자 권한으로 판단한다.

장점:

- 구현이 단순하다.
- 운영자가 1명일 때 가장 직관적이다.

### 8.2 보류 대안: `user_metadata.role = admin`

- Supabase Auth 사용자 메타데이터에 `role: admin`을 둔다.

장점:

- 계정 내부에 권한이 명확히 남는다.

### 8.3 보류 대안: 별도 `admins` 테이블

- 관리자 목록을 DB 테이블로 관리한다.

장점:

- 확장성이 좋다.

현재 개인 블로그라면 환경 변수 기반 허용 이메일이 가장 실용적이다.

## 9. 권장 데이터 모델

## 9.1 posts

- `id: string` // UUID
- `slug: string`
- `title: string`
- `content: string`
- `status: 'draft' | 'published'`
- `author_id: string`
- `thumbnail_url: string | null`
- `created_at: string`
- `updated_at: string`
- `published_at: string | null`
- `view_count: number`
- `like_count: number`

### 설계 메모

- `id`는 내부 식별자이므로 UUID를 사용한다.
- `slug`는 라우팅과 SEO를 위해 필수로 둔다.
- `slug`는 게시글 제목을 기반으로 생성한다.
- `slug` 생성 시 한글은 유지한다.
- 동일한 `slug`가 이미 존재하면 `-2`, `-3` 같은 숫자 suffix를 붙여 유니크하게 만든다.
- 초안 단계에서는 제목 수정에 따라 `slug`를 다시 계산할 수 있다.
- 이미 발행된 글의 `slug`는 제목이 수정되어도 변경하지 않는다.
- `view_count`는 인기 글 기준을 위해 둔다.
- `like_count`는 별도 집계 캐시로 두되, `likes` 테이블과 DB trigger/function으로 항상 동기화한다.

## 9.2 tags

- `id: string`
- `name: string`
- `slug: string`
- `description?: string`
- `color?: string`

### 설계 메모

- `slug`는 `/posts?tag=slug` 형태의 태그 필터 URL에 사용한다 (별도 `/tags/[slug]` 라우트 없음).
- `description`, `color`는 선택 사항이지만 UI 표현을 풍부하게 만든다.

## 9.3 post_tags

- `post_id: string`
- `tag_id: string`

## 9.4 comments

- `id: string`
- `post_id: string`
- `user_id: string`
- `nickname: string`
- `content: string`
- `created_at: string`
- `updated_at: string`
- `deleted_at: string | null`
- `is_edited: boolean`

### 설계 메모

- 댓글 수정은 지원한다.
- `updated_at`과 `is_edited`를 함께 관리한다.

## 9.5 likes

- `id: string`
- `post_id: string`
- `user_id: string`
- `created_at: string`
- `unique(post_id, user_id)`

## 9.6 files

- `id: string`
- `post_id: string | null`
- `storage_path: string`
- `public_url: string`
- `type: 'thumbnail' | 'content_image'`
- `created_at: string`

### 설계 메모

- 초안 작성 중 업로드된 이미지는 `post_id`가 아직 없을 수 있으므로 `null` 허용이 필요하다.
- 또는 초안 생성 시 먼저 글 레코드를 만들고 그 `id`를 기준으로 파일을 저장하는 정책도 가능하다.
- 개인 블로그에서는 `post_id: string | null`이 가장 유연하다.

## 10. Markdown 에디터와 뷰어 설계

### 10.1 작성용 에디터

게시글 작성 화면은 직접 마크다운 엔진을 구현하지 않고 `MDXEditor`를 사용한다.

선택 이유:

- Markdown 문자열을 기준으로 저장할 수 있다.
- 제목, 목록, 인용, 코드블록, 이미지, 테이블 같은 기능을 플러그인 형태로 붙일 수 있다.
- 툴바 구성이 쉽고 작성 경험이 안정적이다.
- 이미지 업로드 핸들러를 연결하기 쉬워 Supabase Storage와 연동하기 좋다.

작성 경험 목표:

- 기본 입력은 텍스트 중심으로 유지한다.
- 툴바에서 `H1~H6`, 목록, 링크, 인용, 코드블록, 테이블, 이미지 삽입 기능을 제공한다.
- 클립보드에 복사된 이미지는 붙여넣기 시 자동 업로드 후 본문에 삽입한다.
- 최종 저장 값은 Markdown 문자열로 통일한다.
- `MDX` 고유 기능인 JSX 삽입 같은 기능은 비활성화하고, 순수 Markdown만 허용한다.

### 10.2 읽기용 뷰어

게시글 상세 화면과 미리보기 화면은 `react-markdown + remark-gfm + rehype-highlight` 조합으로 렌더링한다.

이 조합을 선택하는 이유:

- `react-markdown`은 Markdown을 React 컴포넌트로 안전하게 렌더링할 수 있다.
- `remark-gfm`으로 GitHub Flavored Markdown 문법을 지원할 수 있다.
- `rehype-highlight`로 코드 블록 하이라이트를 적용할 수 있다.
- 저장 형식과 렌더링 형식을 분리할 수 있어서 유지보수가 쉽다.

### 10.3 미리보기 전략

글쓰기 화면에서는 작성 중인 Markdown을 즉시 미리 볼 수 있게 한다.

추천 구조:

- 편집 영역: `MDXEditor`
- 미리보기 영역: `react-markdown`
- GFM 처리: `remark-gfm`
- 코드 하이라이트: `rehype-highlight`

### 10.4 스타일링 전략

Markdown 문법 문자 자체를 스타일링하기보다, 렌더된 요소를 기준으로 스타일을 정의한다.

예:

- `h1`, `h2`, `h3`는 명확한 타이포그래피 계층을 준다.
- `blockquote`는 좌측 보더와 배경색으로 강조한다.
- `code`는 인라인 코드와 블록 코드를 구분해서 스타일링한다.
- `pre`는 가로 스크롤과 충분한 패딩을 제공한다.
- `table`은 작은 화면에서 깨지지 않도록 가로 스크롤을 허용한다.
- `img`는 최대 너비를 제한하고 반응형으로 표시한다.

### 10.5 헤딩 기반 목차 이동

게시글 상세 화면에서는 헤딩(`h1~h6`)을 기준으로 목차를 자동 생성한다.

목차 동작 정책:

- 목차 항목을 클릭하면 해당 헤딩 위치로 스크롤 이동한다.
- 각 헤딩은 고유한 anchor id를 가진다.
- 목차 링크는 해당 anchor를 참조한다.
- 긴 글에서는 현재 읽는 위치에 따라 활성 목차 항목을 표시하는 기능도 추후 확장 가능하다.

## 11. Markdown 이미지 처리 방식

글쓰기 기능에서 이미지는 다음 방식으로 처리한다.

### 11.1 업로드 흐름

1. 사용자가 이미지 파일을 선택하거나 붙여넣는다.
2. 프론트엔드에서 파일을 감지한다.
3. 파일을 Supabase Storage에 업로드한다.
4. 업로드가 끝나면 public URL을 받는다.
5. 커서가 있던 위치에 Markdown 문법으로 이미지 링크를 삽입한다.

예시:

```md
![alt text](https://public-url...)
```

### 11.2 저장 전략

- 원본 파일명은 저장하지 않고 UUID 기반 경로를 사용한다.
- 가능하면 WebP로 변환한다.
- 썸네일과 본문 이미지는 구분된 폴더에 저장한다.

예:

- `thumbnails/`
- `posts/{postId}/`

### 11.3 업로드 제한 정책

명시적으로 정해둘 항목:

- 허용 파일 형식: `jpg`, `jpeg`, `png`, `webp`
- 최대 파일 크기: 예를 들어 `5MB`
- 이미지 리사이즈: 업로드 전 클라이언트에서 축소
- WebP 변환: 업로드 전 클라이언트에서 변환 또는 서버에서 변환

WebP 변환은 Supabase Storage가 자동으로 해주지 않으므로, 클라이언트 사이드 변환이나 서버 사이드 변환을 반드시 설계에 포함해야 한다.

## 12. 초안 삭제 시 이미지 정리 정책

초안 상태의 글을 삭제하면 해당 글에 사용된 이미지도 서버 스토리지에서 함께 삭제한다.

권장 처리 방식:

1. 초안 글의 본문에서 이미지 URL 또는 Storage path를 수집한다.
2. 본문에서 사용한 파일 목록을 DB 또는 메타데이터에서 조회한다.
3. Supabase Storage에서 해당 파일을 삭제한다.
4. posts 레코드를 삭제한다.
5. 첨부 파일 메타데이터도 함께 삭제한다.

주의할 점:

- 본문 문자열만 파싱해서 이미지를 찾는 방식은 누락 가능성이 있다.
- 그래서 저장 시점에 파일 경로를 별도 테이블에 기록하는 편이 안전하다.
- 초안 삭제와 이미지 삭제는 가능하면 한 번에 처리하는 구조가 좋다.

권장 설계:

- 게시글 본문에 사용된 이미지 경로를 `files` 테이블에 저장
- 게시글 삭제 시 `files` 기준으로 Storage 정리

## 13. 초안 생애주기와 자동 저장

글쓰기 버튼을 누르는 순간 초안을 생성하고, 그 초안의 `id`를 기반으로 편집을 시작한다.

권장 흐름:

1. `새 글 작성` 버튼 클릭
2. 서버에서 `draft` 상태의 `posts` 레코드 생성
3. 생성된 `post.id`를 반환
4. 글쓰기 페이지로 진입
5. 일정 주기마다 자동으로 초안 업데이트
6. 사용자가 `발행`을 누르면 동일 레코드를 `published`로 변경

자동 저장 정책:

- 일정 주기 예: 20초~30초
- 제목, 본문, 태그, 썸네일 변경 사항을 반영
- 저장 성공 시 `updated_at` 갱신
- 저장 실패 시 사용자에게 재시도 안내

빈 초안 정리 정책:

- 제목과 본문이 모두 비어 있는 초안은 빈 초안으로 본다
- 일정 시간 이상 수정이 없는 빈 초안은 자동 정리 대상으로 본다
- 초안 목록에서는 빈 초안을 숨기거나 별도 구역으로 분리할 수 있다
- 초안 삭제 시 해당 초안과 연결된 파일도 함께 정리한다

운영 권장안:

- 최근 생성된 빈 초안은 일정 기간 보관
- 오래된 빈 초안은 배치 작업 또는 관리자 정리 기능으로 제거
- 초안 목록에서는 `빈 초안`, `작성 중`, `최근 수정` 같은 상태 태그를 표시한다

## 14. 라우팅 구조 예시

- `/` 홈
- `/posts` 전체 글 목록
- `/posts/[slug]` 글 상세
- `/posts?tag=slug` 태그 필터 (별도 라우트 없이 `/posts` 내에서 분기)
- `/archive` 아카이브
- `/search` 검색 결과
- `/login` 관리자 또는 소셜 로그인 진입
- `/admin/posts/new` 새 글 작성
- `/admin/posts/[slug]/edit` 글 수정
- `/admin/comments` 댓글 관리
- `/admin/tags` 태그 관리
- `/admin/images` 이미지 관리

### `admin/images` 경로를 둔 이유

- 본문 이미지와 썸네일을 별도로 확인하기 위해서다
- 업로드된 파일이 실제로 어떤 게시글과 연결돼 있는지 추적하기 위해서다
- 사용되지 않는 이미지, 깨진 링크, 중복 업로드 파일을 정리하기 위해서다
- 초안 삭제나 게시글 삭제 후 Storage 정리 상태를 눈으로 확인하기 위해서다

즉 `admin/images`는 이미지 업로드 화면이 아니라, 운영 중인 이미지 자산을 관리하는 페이지다.

## 15. TanStack Query 사용 범위

TanStack Query는 모든 서버 데이터를 대체하는 도구가 아니라, 클라이언트 상호작용이 필요한 부분에 집중해서 사용한다.

권장 사용처:

- 댓글 목록 조회
- 댓글 추가 후 캐시 갱신
- 좋아요 토글
- 좋아요 수 갱신
- 태그 목록의 동적 조회

권장하지 않는 사용처:

- 정적 게시글 상세 페이지 전체를 무조건 클라이언트에서 패칭
- 라우트 초기 로딩을 전부 TanStack Query로만 처리

## 16. TypeScript 타입 설계

TypeScript를 전제로 하므로 데이터와 액션의 형태를 명확히 나눈다.

### 16.1 공통 상태 타입

```ts
export type PostStatus = 'draft' | 'published';
export type FileType = 'thumbnail' | 'content_image';
export type LoginProvider = 'google' | 'github';
```

### 16.2 게시글 타입

```ts
export interface Post {
  id: string;
  slug: string;
  title: string;
  content: string;
  status: PostStatus;
  authorId: string;
  thumbnailUrl: string | null;
  createdAt: string;
  updatedAt: string;
  publishedAt: string | null;
  viewCount: number;
  likeCount: number;
}
```

### 16.3 태그 타입

```ts
export interface Tag {
  id: string;
  name: string;
  slug: string;
  description?: string;
  color?: string;
}
```

### 16.4 댓글 타입

```ts
export interface Comment {
  id: string;
  postId: string;
  userId: string;
  nickname: string;
  content: string;
  createdAt: string;
  updatedAt: string;
  deletedAt: string | null;
  isEdited: boolean;
}
```

### 16.5 좋아요 타입

```ts
export interface Like {
  id: string;
  postId: string;
  userId: string;
  createdAt: string;
}
```

### 16.6 파일 타입

```ts
export interface StoredFile {
  id: string;
  postId: string | null;
  storagePath: string;
  publicUrl: string;
  type: FileType;
  createdAt: string;
}
```

### 16.7 서버 액션 결과 타입

```ts
// 모든 Server Action의 기본 응답 형태
type ActionResult = { ok: true } | { ok: false; message: string };
// ok: true 분기에 필요한 데이터가 있으면 필드를 추가한다
// 예: { ok: true; postId: string } | { ok: false; message: string }
```

예시:

```ts
export type SaveDraftResult =
  | { ok: true; postId: string }
  | { ok: false; message: string };
```

### 16.8 서버/클라이언트 응답 타입

```ts
export interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  pageSize: number;
}
```

### 16.9 상태 저장소 타입

Zustand는 작성 중인 글의 임시 상태에만 사용한다.

예시:

```ts
export interface DraftPostState {
  title: string;
  content: string;
  tags: string[];
  thumbnail: File | string | null;
  status: PostStatus;
}
```

## 17. 운영 원칙

- 관리 기능은 서버에서 반드시 권한 검증을 한다.
- 댓글과 좋아요는 로그인 계정 단위로 추적한다.
- 초안은 저장하되 삭제 시 관련 이미지도 함께 제거한다.
- 모든 공개 데이터는 읽기 전용 경로로 안전하게 노출한다.
- 에러 상황에서는 사용자에게 명확한 메시지를 준다.

## 18. 최종 권장안

현재 조건에서 가장 균형이 좋은 안은 다음과 같다.

- 읽기는 누구나 가능
- 댓글과 좋아요는 소셜로그인 필요
- 회원가입은 없음
- 관리자는 본인 1명
- 게시판은 1개
- 태그 중심 탐색
- `slug` 기반 라우팅 사용
- SEO는 필수 기능으로 격상
- Markdown 글쓰기
- 이미지 붙여넣기 업로드 지원
- 초안 삭제 시 Storage 이미지 정리
- 작성용 에디터는 `MDXEditor`
- 읽기용 뷰어는 `react-markdown + remark-gfm + rehype-highlight`
- TanStack Query는 댓글/좋아요 등 상호작용 데이터에만 집중

이 구조는 구현 복잡도와 운영 편의성 사이의 균형이 좋다.
개인 블로그로서도 충분히 단단하고, 나중에 기능을 더 붙이기도 쉽다.
