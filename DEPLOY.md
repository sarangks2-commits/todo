# 구동 방법

## 배포 URL (GitHub Pages)

```
https://sarangks2-commits.github.io/todo/
```

> GitHub Pages는 `main` 브랜치 루트를 기준으로 서빙됩니다.
> Supabase 소셜 로그인 redirect URL도 이 주소로 설정되어 있습니다.

---

## GitHub Pages 활성화 방법

1. GitHub 리포지토리 → **Settings → Pages**
2. Source: **Deploy from a branch**
3. Branch: `main` / `/ (root)` → **Save**
4. 약 1~2분 후 위 URL에서 접속 가능

---

## 로컬 실행

```bash
cd src/exercise/kangsoo.lee/day03/todo
python3 -m http.server 8766
# → http://localhost:8766/index.html
```

> WSL 환경이라면 Windows 브라우저에서도 동일한 URL로 접속 가능합니다.
> 소셜 로그인(GitHub/Google)은 redirect URL이 GitHub Pages로 설정되어 있어
> 로컬 테스트 시에는 이메일 로그인을 사용하세요.

---

## 서버 종료

```bash
kill $(lsof -ti:8766)
```

---

## 사전 조건 — Supabase 설정

자세한 내용은 [SUPABASE.md](./SUPABASE.md)를 참고하세요.

### 1. 테이블 생성 (SQL Editor에서 실행)

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

### 2. 소셜 로그인 Provider 설정

**GitHub:**
```
Authentication → Providers → GitHub → Enable
→ Client ID / Secret: GitHub OAuth App에서 발급
→ Callback URL: https://tfakzxaygrvvzmtgqwhr.supabase.co/auth/v1/callback
```

**Google:**
```
Authentication → Providers → Google → Enable
→ Client ID / Secret: Google Cloud Console에서 발급
→ Authorized redirect URI: https://tfakzxaygrvvzmtgqwhr.supabase.co/auth/v1/callback
```

### 3. Redirect URL 허용 등록

```
Authentication → URL Configuration → Redirect URLs → Add URL
→ https://sarangks2-commits.github.io/todo/
```

### 4. Confirm email 설정 (선택)

```
Authentication → Providers → Email
→ "Confirm email" 토글 OFF → Save   (개발용: 즉시 로그인)
```

### 5. API 키

| 항목 | 값 |
|------|-----|
| Project URL | `https://tfakzxaygrvvzmtgqwhr.supabase.co` |
| Region | Southeast Asia (Singapore) |
| anon key | `index.html` 내 `SUPABASE_ANON` 변수 참고 |

---

## 첫 실행 순서

1. 브라우저에서 배포 URL 또는 로컬 URL 접속
2. 로그인 화면 → **이메일 로그인** 또는 **GitHub / Google 소셜 로그인**
3. 최초 사용 시 **회원가입** 탭에서 이메일 + 비밀번호 입력
4. (Confirm email ON인 경우) 메일함에서 확인 링크 클릭
5. 로그인 후 Todo 추가 시작
