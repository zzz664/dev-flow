# 환경 변수 설계 문서

## 1. 필수 환경 변수

### 1.1 Supabase

| 변수명 | 노출 범위 | 설명 |
|--------|-----------|------|
| `NEXT_PUBLIC_SUPABASE_URL` | 클라이언트 + 서버 | Supabase 프로젝트 URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | 클라이언트 + 서버 | Supabase 공개 키 (RLS 적용됨) |
| `SUPABASE_SERVICE_ROLE_KEY` | **서버 전용** | Supabase 서비스 롤 키 (RLS 우회). 절대 클라이언트에 노출 금지 |

### 1.2 관리자 인증

| 변수명 | 노출 범위 | 설명 |
|--------|-----------|------|
| `ADMIN_EMAIL` | **서버 전용** | 관리자로 허용할 이메일 주소. 환경 변수 기반 관리자 식별에 사용 |

### 1.3 사이트 메타

| 변수명 | 노출 범위 | 설명 |
|--------|-----------|------|
| `NEXT_PUBLIC_SITE_URL` | 클라이언트 + 서버 | 배포된 사이트 URL (예: `https://myblog.com`). sitemap, canonical, OG 태그에 사용 |

---

## 2. 선택 환경 변수

| 변수명 | 노출 범위 | 설명 |
|--------|-----------|------|
| `NEXT_PUBLIC_SITE_NAME` | 클라이언트 + 서버 | 사이트 이름 (기본값: `Tech Vibe`) |
| `NEXT_PUBLIC_SITE_DESCRIPTION` | 클라이언트 + 서버 | 사이트 설명 (SEO 기본 description에 사용) |

---

## 3. `.env.local` 예시

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxxxxxxxxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# 관리자
ADMIN_EMAIL=your@email.com

# 사이트 URL (필수 — sitemap, canonical, OG 태그에 사용)
NEXT_PUBLIC_SITE_URL=http://localhost:3000

# 사이트 메타 (선택)
NEXT_PUBLIC_SITE_NAME=Tech Vibe
NEXT_PUBLIC_SITE_DESCRIPTION=개인 기술 블로그
```

---

## 4. 보안 원칙

- `SUPABASE_SERVICE_ROLE_KEY`와 `ADMIN_EMAIL`은 절대 `NEXT_PUBLIC_` 접두사를 붙이지 않는다.
- 서버 액션, Route Handler, Server Component에서만 사용한다.
- `.env.local`은 `.gitignore`에 반드시 포함되어야 한다.

---

## 5. Supabase OAuth 설정

환경 변수와 별도로 Supabase Dashboard에서 설정해야 한다.

**Authentication → Providers:**
- Google OAuth: Client ID, Client Secret 등록
- GitHub OAuth: Client ID, Client Secret 등록

**Authentication → URL Configuration:**
- Site URL: `NEXT_PUBLIC_SITE_URL` 값과 동일하게 설정
- Redirect URLs: `{NEXT_PUBLIC_SITE_URL}/auth/callback`
