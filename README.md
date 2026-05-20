# Todo App — 전체 개발 기록

**배포 URL**: https://sarangks2-commits.github.io/todo/

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [기술 스택](#2-기술-스택)
3. [기능 목록](#3-기능-목록)
4. [동작 로직](#4-동작-로직)
5. [Supabase 설정](#5-supabase-설정)
6. [소셜 로그인 설정](#6-소셜-로그인-설정)
7. [GitHub Pages 배포](#7-github-pages-배포)
8. [로컬 실행](#8-로컬-실행)
9. [개발 이력](#9-개발-이력)

---

## 1. 프로젝트 개요

바이브 코딩 과정 day03 실습 — 단일 HTML 파일 Todo 앱.  
`localStorage` 로컬 저장 → Supabase PostgreSQL 클라우드 저장으로 발전시켰으며,  
이메일 / GitHub / Google 소셜 로그인을 지원하고 GitHub Pages로 배포되었습니다.

---

## 2. 기술 스택

| 구분 | 기술 |
|------|------|
| **Frontend** | Vanilla HTML / CSS / JavaScript (프레임워크 없음) |
| **폰트** | Inter (Google Fonts CDN) |
| **Backend / DB** | Supabase (PostgreSQL) |
| **인증** | Supabase Auth — Email+Password / GitHub OAuth / Google OAuth |
| **클라이언트 SDK** | `@supabase/supabase-js v2` (CDN) |
| **보안** | Row Level Security (RLS) |
| **배포** | GitHub Pages (`sarangks2-commits/todo`) |

---

## 3. 기능 목록

### 인증
| 기능 | 설명 |
|------|------|
| 이메일 회원가입 | 이메일 + 비밀번호 입력, 확인 메일 발송 (설정에 따라) |
| 이메일 로그인 | 이메일 + 비밀번호 |
| GitHub 소셜 로그인 | OAuth 2.0 — 본인 GitHub 계정으로 인증 |
| Google 소셜 로그인 | OAuth 2.0 — 본인 Google 계정으로 인증 |
| 세션 유지 | 새로고침 후에도 로그인 상태 유지 |
| 로그아웃 | 세션 삭제 후 로그인 오버레이 재표시 |

### Todo
| 기능 | 설명 |
|------|------|
| 추가 | 텍스트 + 우선순위(높음/보통/낮음) 선택 후 저장 |
| 완료 토글 | 체크박스 → done 값 반전 저장 |
| 수정 | 텍스트 더블클릭 또는 ✏️ 버튼 → 인라인 편집 |
| 삭제 | 🗑️ 버튼 → 즉시 삭제 |
| 순서 변경 | 드래그 & 드롭 또는 ▲▼ 버튼 → position 서버 저장 |
| 우선순위 정렬 | 툴바 버튼 토글 → 클라이언트 정렬 |
| 필터 | 전체 / 미완료 / 완료 |
| 진행률 표시 | 완료 개수 / 전체 퍼센트 프로그레스 바 |
| 완료 일괄 삭제 | 하단 버튼 → 완료 항목 전체 삭제 |

---

## 4. 동작 로직

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
  └── 오버레이 닫기 → 유저바 표시(이메일) → loadTodos()
```

> **소셜 로그인 이메일 표시**: 각 사용자가 본인 소셜 계정으로 인증하면,  
> 해당 계정의 이메일이 표시됩니다. 타인 이메일이 보이는 것이 아니라  
> 본인 소셜 계정에 등록된 이메일이 자동으로 표시되는 정상 동작입니다.

### Todo CRUD 흐름

```
쓰기 작업 (추가/수정/삭제/완료 토글)
  └── Supabase REST API 호출
        ├── 성공 → loadTodos() 재호출 → render()
        └── 실패 → 상태바 에러 메시지 표시

loadTodos()
  └── SELECT * FROM todos
        WHERE user_id = auth.uid()
        ORDER BY position ASC
```

### 순서 변경 흐름

```
드래그 & 드롭 / ▲▼ 버튼
  └── 클라이언트 배열 재정렬 → render() (즉시 반영)
        └── 각 항목 position UPDATE (Promise.all 병렬)
              └── loadTodos() 재호출 → 서버 동기화
```

### 보안 구조 (RLS)

```
브라우저 (anon key 포함)
  └── Supabase API 요청
        └── RLS Policy: auth.uid() = todos.user_id
              ├── 일치 → 허용
              └── 불일치 → 거부 (타인 데이터 접근 불가)
```

---

## 5. Supabase 설정

### 프로젝트 정보

| 항목 | 값 |
|------|-----|
| Project URL | `https://tfakzxaygrvvzmtgqwhr.supabase.co` |
| Region | Southeast Asia (Singapore) |
| Callback URL | `https://tfakzxaygrvvzmtgqwhr.supabase.co/auth/v1/callback` |

> **Callback URL 확인 방법**: Supabase 대시보드 → Authentication → Providers → 각 Provider 항목 펼치기 → "Callback URL (for OAuth)" 확인

> **Region 변경**: Supabase는 프로젝트 생성 후 region 변경 불가. 변경하려면 새 프로젝트 생성 후 마이그레이션 필요.

### 테이블 구조 (`todos`)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | `uuid` | PK, `gen_random_uuid()` |
| `user_id` | `uuid` | FK → `auth.users.id` |
| `text` | `text` | 할 일 내용 |
| `done` | `boolean` | 완료 여부, default `false` |
| `priority` | `text` | `high` / `medium` / `low` |
| `position` | `integer` | 표시 순서 |
| `created_at` | `timestamptz` | 생성 시각 |
| `updated_at` | `timestamptz` | 수정 시각 (트리거 자동 갱신) |

### SQL

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

### 이메일 인증 설정

```
Authentication → Providers → Email
→ "Confirm email" 토글 OFF   ← 개발용 (즉시 로그인)
→ "Confirm email" 토글 ON    ← 운영용 (확인 메일 발송)
```

---

## 6. 소셜 로그인 설정

### 설정 완료 현황

| Provider | 상태 |
|----------|------|
| Email | ✅ 활성화 |
| GitHub | ✅ 활성화 완료 |
| Google | ✅ 활성화 완료 |
| Redirect URL | ✅ `https://sarangks2-commits.github.io/todo/` 등록 완료 |

### GitHub OAuth App 설정 방법

1. `github.com` → 프로필 → Settings → Developer settings → OAuth Apps → New OAuth App
   - Homepage URL: `https://sarangks2-commits.github.io/todo/`
   - Authorization callback URL: `https://tfakzxaygrvvzmtgqwhr.supabase.co/auth/v1/callback`
2. Client ID / Client Secret 복사
3. Supabase → `Authentication → Providers → GitHub → Enable` → 입력 후 Save

### Google OAuth 설정 방법

1. `console.cloud.google.com` → 프로젝트 생성
2. API 및 서비스 → OAuth 동의 화면 → 외부 → 앱 이름/이메일 입력
3. 사용자 인증 정보 → OAuth 클라이언트 ID 만들기
   - 애플리케이션 유형: 웹 애플리케이션
   - 승인된 리디렉션 URI: `https://tfakzxaygrvvzmtgqwhr.supabase.co/auth/v1/callback`
4. 클라이언트 ID / 보안 비밀번호 복사
5. Supabase → `Authentication → Providers → Google → Enable` → 입력 후 Save

### Redirect URL 등록

```
Authentication → URL Configuration → Redirect URLs → Add URL
→ https://sarangks2-commits.github.io/todo/
```

---

## 7. GitHub Pages 배포

### 배포 리포지토리

| 항목 | 값 |
|------|-----|
| 리포지토리 | `github.com/sarangks2-commits/todo` |
| 배포 URL | `https://sarangks2-commits.github.io/todo/` |
| 브랜치 | `main` |

### GitHub Pages 활성화

```
github.com/sarangks2-commits/todo
→ Settings → Pages
→ Source: Deploy from a branch
→ Branch: main / / (root)
→ Save
```

### 로컬에서 sarangks2-commits 리포에 push하는 방법

```bash
# SSH 키 생성 (최초 1회)
ssh-keygen -t ed25519 -C "sarangks2" -f ~/.ssh/id_sarangks2

# 공개키를 github.com/sarangks2-commits → Settings → SSH keys 에 등록
cat ~/.ssh/id_sarangks2.pub

# ~/.ssh/config 설정
Host github-sarangks2
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_sarangks2

# push
git remote add origin git@github-sarangks2:sarangks2-commits/todo.git
git push -u origin main
```

---

## 8. 로컬 실행

```bash
cd src/exercise/kangsoo.lee/day03/todo
python3 -m http.server 8766
# → http://localhost:8766/index.html
```

> 소셜 로그인은 redirectTo가 GitHub Pages로 설정되어 있어 로컬에서는 이메일 로그인을 사용하세요.

### 서버 종료

```bash
kill $(lsof -ti:8766)
```

---

## 9. 개발 이력

| 단계 | 내용 |
|------|------|
| 1 | 기본 Todo 앱 (localStorage 저장) |
| 2 | 다크 테마 프로 디자인 리뉴얼 |
| 3 | 우선순위 설정 및 정렬 기능 추가 |
| 4 | 드래그 & 드롭 / ▲▼ 순서 변경 기능 추가 |
| 5 | Supabase 연동 (localStorage → PostgreSQL) |
| 6 | Anonymous Auth → 이메일 회원가입/로그인으로 전환 |
| 7 | GitHub 소셜 로그인 추가 |
| 8 | Google 소셜 로그인 추가 |
| 9 | GitHub Pages 배포 (`sarangks2-commits/todo`) |
