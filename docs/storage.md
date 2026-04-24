# Supabase Storage 설계 문서

## 1. 개요

Supabase Storage를 사용해 게시글 썸네일과 본문 이미지를 관리한다.

- 버킷: `images` 단일 버킷
- 접근 정책: public read / admin write
- 파일 형식: WebP 변환 후 저장 (클라이언트 사이드 변환)
- 최대 파일 크기: 5MB (업로드 전 클라이언트 검사)

---

## 2. 버킷 구성

### 2.1 버킷 생성

```sql
INSERT INTO storage.buckets (id, name, public)
VALUES ('images', 'images', true);
```

- `public = true`: 파일 URL이 인증 없이 접근 가능 (게시글 이미지 노출에 필요)

---

### 2.2 폴더 구조

```
images/
├── thumbnails/
│   └── {uuid}.webp
└── posts/
    ├── {postId}/
    │   └── {uuid}.webp
    └── orphan/
        └── {uuid}.webp     ← postId 없는 임시 이미지
```

| 경로 | 용도 |
|------|------|
| `thumbnails/` | 게시글 대표 이미지 |
| `posts/{postId}/` | 본문에 삽입된 이미지 |
| `posts/orphan/` | 초안 생성 전 업로드된 이미지 (postId 없음) |

---

## 3. Storage RLS 정책

```sql
-- 누구나 images 버킷에서 파일 조회 가능 (public bucket)
CREATE POLICY "images: public read"
ON storage.objects FOR SELECT
USING (bucket_id = 'images');

-- 관리자만 업로드
CREATE POLICY "images: admin insert"
ON storage.objects FOR INSERT
WITH CHECK (
  bucket_id = 'images'
  AND auth.email() = current_setting('app.admin_email', true)
);

-- 관리자만 삭제
CREATE POLICY "images: admin delete"
ON storage.objects FOR DELETE
USING (
  bucket_id = 'images'
  AND auth.email() = current_setting('app.admin_email', true)
);
```

> 서버 액션에서는 `supabaseAdmin` (Service Role Key)을 사용하므로 RLS 우회가 가능하다.
> 클라이언트에서 직접 업로드가 필요한 경우 위 정책이 적용된다.

---

## 4. 업로드 흐름

### 4.1 클라이언트 전처리

1. 파일 형식 검사: `jpg`, `jpeg`, `png`, `webp`만 허용
2. 파일 크기 검사: 5MB 초과 시 거부
3. WebP 변환: `browser-image-compression` 또는 Canvas API로 변환
4. 리사이즈: 너비 최대 1200px로 축소 (비율 유지)

```ts
// 클라이언트 사이드 WebP 변환 예시
async function convertToWebP(file: File): Promise<File> {
  const canvas = document.createElement('canvas');
  const img = await createImageBitmap(file);
  const MAX_WIDTH = 1200;
  const scale = Math.min(1, MAX_WIDTH / img.width);
  canvas.width = img.width * scale;
  canvas.height = img.height * scale;
  canvas.getContext('2d')!.drawImage(img, 0, 0, canvas.width, canvas.height);
  const blob = await new Promise<Blob>((resolve) =>
    canvas.toBlob((b) => resolve(b!), 'image/webp', 0.85)
  );
  return new File([blob], `${crypto.randomUUID()}.webp`, { type: 'image/webp' });
}
```

### 4.2 서버 업로드 (Server Action)

```ts
// app/actions/upload.ts
'use server';

export async function uploadImage(formData: FormData) {
  const file = formData.get('file') as File;
  const postId = formData.get('postId') as string | null;
  const type = formData.get('type') as 'thumbnail' | 'content_image';

  const uuid = crypto.randomUUID();
  const storagePath = type === 'thumbnail'
    ? `thumbnails/${uuid}.webp`
    : postId
      ? `posts/${postId}/${uuid}.webp`
      : `posts/orphan/${uuid}.webp`;

  const { error } = await supabaseAdmin.storage
    .from('images')
    .upload(storagePath, file, { contentType: 'image/webp', upsert: false });

  if (error) return { ok: false, message: error.message };

  const { data: { publicUrl } } = supabaseAdmin.storage
    .from('images')
    .getPublicUrl(storagePath);

  const { data: fileRecord } = await supabaseAdmin
    .from('files')
    .insert({ post_id: postId, storage_path: storagePath, public_url: publicUrl, type })
    .select('id')
    .single();

  return { ok: true, publicUrl, fileId: fileRecord!.id };
}
```

> **정상 글쓰기 플로우**: `createDraft()` → 에디터 진입 → `uploadImage({ postId, ... })` 순서로 호출하면 `postId`가 항상 존재한다. `posts/orphan/` 경로는 이 순서가 깨지는 에지 케이스용 안전장치다.
```

---

## 5. 삭제 흐름

### 5.1 단일 파일 삭제

```ts
async function deleteImage(fileId: string) {
  const { data: file } = await supabaseAdmin
    .from('files')
    .select('storage_path')
    .eq('id', fileId)
    .single();

  await supabaseAdmin.storage.from('images').remove([file!.storage_path]);
  await supabaseAdmin.from('files').delete().eq('id', fileId);
}
```

### 5.2 게시글 삭제 시 일괄 정리

```ts
async function deletePostWithFiles(postId: string) {
  // 1. 연결된 파일 목록 조회
  const { data: files } = await supabaseAdmin
    .from('files')
    .select('id, storage_path')
    .eq('post_id', postId);

  // 2. Storage 파일 일괄 삭제
  if (files && files.length > 0) {
    const paths = files.map((f) => f.storage_path);
    const { error } = await supabaseAdmin.storage.from('images').remove(paths);
    if (error) throw new Error('Storage 파일 삭제 실패');
  }

  // 3. files 레코드 삭제 (files.post_id는 ON DELETE SET NULL이므로 명시적으로 삭제)
  if (files && files.length > 0) {
    await supabaseAdmin.from('files').delete().in('id', files.map((f) => f.id));
  }

  // 4. posts 레코드 삭제 (CASCADE로 post_tags, comments, likes 연쇄 삭제)
  await supabaseAdmin.from('posts').delete().eq('id', postId);
}
```

---

## 6. 업로드 제한 요약

| 항목 | 정책 |
|------|------|
| 허용 파일 형식 | jpg, jpeg, png, webp |
| 최대 파일 크기 | 5MB |
| 저장 형식 | WebP (클라이언트 변환) |
| 최대 너비 | 1200px (비율 유지 축소) |
| 파일명 | UUID 기반 (원본 파일명 미사용) |
| 접근 방식 | Public URL (인증 불필요) |

---

## 7. 고아 파일 정리 정책

### 7.1 발생 조건

정상 플로우(`createDraft` → 이미지 업로드)에서는 항상 `postId`가 있으므로 orphan이 발생하지 않는다.

orphan 발생 경로:
- 초안 삭제 → `files.post_id SET NULL` (FK `ON DELETE SET NULL` 발동)
- 비정상 케이스: `postId` 없이 업로드 후 초안 생성 없이 이탈 (`posts/orphan/` 경로)

### 7.2 정리 방법

`files` 테이블에서 `post_id IS NULL`이고 `created_at`이 일정 기간 이전인 파일을 조회해 삭제한다.

```ts
// 관리자 /admin/images 페이지에서 수동 실행 또는 배치 처리
const { data: orphans } = await supabaseAdmin
  .from('files')
  .select('id, storage_path')
  .is('post_id', null)
  .lt('created_at', new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString());

const paths = orphans!.map((f) => f.storage_path);
await supabaseAdmin.storage.from('images').remove(paths);
await supabaseAdmin.from('files').delete().in('id', orphans!.map((f) => f.id));
```

권장 주기: 주 1회 수동 또는 Supabase Edge Function cron으로 자동화

---

## 8. /admin/images 페이지 데이터

관리자 이미지 관리 페이지에서 표시할 데이터:

```ts
// 전체 파일 목록 (게시글 제목 포함)
const { data } = await supabaseAdmin
  .from('files')
  .select(`
    id,
    storage_path,
    public_url,
    type,
    created_at,
    post_id,
    posts ( title, slug, status )
  `)
  .order('created_at', { ascending: false });
```

표시 항목:
- 파일 미리보기 (썸네일)
- 파일 타입 (thumbnail / content_image)
- 연결된 게시글 제목 (없으면 "미연결")
- 업로드 일시
- 삭제 버튼
