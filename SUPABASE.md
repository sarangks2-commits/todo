# Supabase Todo App — 설정 & 기술 문서

---

## 기술 스택

| 구분 | 기술 |
|------|------|
| **Frontend** | Vanilla HTML / CSS / JavaScript (프레임워크 없음) |
| **폰트** | Inter (Google Fonts) |
| **Backend / DB** | Supabase (PostgreSQL) |
| **인증** | Supabase Auth — Email+Password / GitHub OAuth / Google OAuth |
| **클라이언트 SDK** | `@supabase/supabase-js v2` (CDN) |
| **보안** | Row Level Security (RLS) |
| **배포** | GitHub Pages (단일 HTML 파일) |

---

## 동작 로직

### 인증 흐름

```
앱 로드
  └── sb.auth.getSession()
        ├── 세션 있음 → onSignedIn(user) → Todo 목록 로드
        └── 세션 없음 → 로그인 오버레이 표시

로그인 오버레이
  ├── [이메일 로그인]  signInWithPassword(email, password)
  ├── [회원가입]       signUp(email, password)
  │                     ├── Confirm email OFF → 즉시 로그인
  │                     └── Confirm email ON  → 확인 메일 발송
  ├── [GitHub 로그인]  signInWithOAuth({ provider: 'github', redirectTo })
  │                     └── GitHub 인증 → Supabase callback → redirectTo로 복귀
  └── [Google 로그인]  signInWithOAuth({ provider: 'google', redirectTo })
                         └── Google 인증 → Supabase callback → redirectTo로 복귀

onAuthStateChange('SIGNED_IN') → onSignedIn(user) 자동 호출
  └── 오버레이 닫기 → 유저바 표시 → loadTodos()
```

### Todo CRUD 흐름

```
모든 쓰기 작업 (추가 / 수정 / 삭제 / 완료 토글)
  └── Supabase REST API 호출 (.insert / .update / .delete)
        ├── 성공 → loadTodos() 재호출 → render()
        └── 실패 → 상태바에 에러 메시지 표시

loadTodos()
  └── SELECT * FROM todos
        WHERE user_id = auth.uid()
        ORDER BY position ASC
```

### 순서 변경 흐름

```
드래그 & 드롭 / ▲▼ 버튼
  └── 클라이언트 배열 재정렬 → render() (즉시 반영)
        └── 각 항목 position 값 UPDATE (병렬 Promise.all)
              └── loadTodos() 재호출 → 서버 순서와 동기화
```

### 보안 구조

```
브라우저 (anon key 포함)
  └── Supabase API 요청
        └── RLS Policy 검사
              └── auth.uid() = todos.user_id 일치 여부 확인
                    ├── 일치 → 요청 허용
                    └── 불일치 → 거부 (타인 데이터 접근 불가)
```

---

## 주요 기능

| 기능 | 설명 |
|------|------|
| 이메일 회원가입/로그인 | 이메일 + 비밀번호, 탭 전환 UI |
| GitHub 소셜 로그인 | OAuth 2.0, GitHub Pages로 redirect |
| Google 소셜 로그인 | OAuth 2.0, GitHub Pages로 redirect |
| 세션 유지 | 새로고침 후에도 로그인 상태 유지 |
| 로그아웃 | 세션 삭제 후 로그인 오버레이 재표시 |
| Todo 추가 | 텍스트 + 우선순위(높음/보통/낮음) 선택 후 저장 |
| 완료 토글 | 체크박스 클릭 → done 값 반전 저장 |
| 수정 | 텍스트 더블클릭 또는 ✏️ 버튼 → 인라인 편집 |
| 삭제 | 🗑️ 버튼 → 즉시 삭제 |
| 순서 변경 | 드래그 & 드롭 또는 ▲▼ 버튼 → position 서버 저장 |
| 우선순위 정렬 | 툴바 버튼 토글 → 클라이언트 정렬 |
| 필터 | 전체 / 미완료 / 완료 |
| 진행률 표시 | 완료 개수 / 전체 퍼센트 프로그레스 바 |
| 완료 일괄 삭제 | 하단 버튼 → 완료 항목 전체 삭제 |

---

## 1. 프로젝트 생성

1. [https://supabase.com](https://supabase.com) → GitHub 계정으로 로그인
2. **New project** 클릭
   - Name: `kangsoo-todo`
   - Region: Southeast Asia (Singapore) ← 현재 설정
3. **Create new project** → 약 1~2분 대기

> Region은 프로젝트 생성 후 변경 불가합니다.

---

## 2. SQL — SQL Editor에서 실행

```sql
create table public.todos (
  id          uuid        primary key default gen_random_uuid(),
  user_id     uuid        not null references auth.users(id) on delete cascade,
  text        text        not null,
  done        boolean     not null default false,
  priority    text        not null default 'medium'
                          check (priority in ('high', 'medium', 'low')),
  position    integer     not null default 0,
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);

create or replace function public.set_updated_at()
returns trigger language plpgsql as $$
begin new.updated_at = now(); return new; end;
$$;

create trigger todos_updated_at
  before update on public.todos
  for each row execute function public.set_updated_at();

create index todos_user_position on public.todos(user_id, position);

alter table public.todos enable row level security;

create policy "Users can manage own todos"
  on public.todos for all
  using  (auth.uid() = user_id)
  with check (auth.uid() = user_id);
```

---

## 3. 이메일 인증 설정

```
Authentication → Providers → Email
  → "Confirm email" 토글 OFF → Save   (개발용: 즉시 로그인)
```

> 운영 시에는 ON 권장

---

## 4. GitHub 소셜 로그인 설정

### 4-1. GitHub OAuth App 생성

1. GitHub → **Settings → Developer settings → OAuth Apps → New OAuth App**
   - Application name: `kangsoo-todo`
   - Homepage URL: `https://sarangks2-commits.github.io/todo/`
   - Authorization callback URL: `https://tfakzxaygrvvzmtgqwhr.supabase.co/auth/v1/callback`
2. **Register application** → **Client ID** 와 **Client Secret** 복사

### 4-2. Supabase에 등록

```
Authentication → Providers → GitHub → Enable
→ Client ID: (복사한 값)
→ Client Secret: (복사한 값)
→ Save
```

---

## 5. Google 소셜 로그인 설정

### 5-1. Google Cloud Console OAuth 클라이언트 생성

1. [console.cloud.google.com](https://console.cloud.google.com) → 프로젝트 선택 또는 생성
2. **APIs & Services → Credentials → Create Credentials → OAuth client ID**
   - Application type: **Web application**
   - Authorized redirect URIs: `https://tfakzxaygrvvzmtgqwhr.supabase.co/auth/v1/callback`
3. **Client ID** 와 **Client Secret** 복사

### 5-2. Supabase에 등록

```
Authentication → Providers → Google → Enable
→ Client ID: (복사한 값)
→ Client Secret: (복사한 값)
→ Save
```

---

## 6. Redirect URL 허용 등록

소셜 로그인 후 돌아올 URL을 Supabase에 등록해야 합니다.

```
Authentication → URL Configuration → Redirect URLs → Add URL
→ https://sarangks2-commits.github.io/todo/
```

---

## 7. 설정 완료 현황

| Provider | 상태 |
|----------|------|
| Email | ✅ 활성화 (Confirm email OFF) |
| GitHub | ✅ 활성화 완료 |
| Google | ✅ 활성화 완료 |
| Redirect URL | ✅ `https://sarangks2-commits.github.io/todo/` 등록 완료 |

---

## 8. API 키

**Project Settings → API**

| 항목 | 값 |
|------|-----|
| **Project URL** | `https://tfakzxaygrvvzmtgqwhr.supabase.co` |
| **anon public key** | RLS로 보호 — 프론트엔드 노출 안전 |
| **service_role key** | 절대 프론트엔드에 넣지 말 것 |
