# day03 / Todo List

kangsoo.lee의 day03 실습 — 단일 HTML 파일 Todo 앱.

## 실행

```bash
# 브라우저에서 직접 열기
open index.html

# 또는 로컬 서버
python3 -m http.server 8766
# → http://localhost:8766/index.html
```

## 주요 기능

- 할 일 추가 / 수정(더블클릭 또는 ✏️) / 삭제
- 우선순위 설정 (높음 · 보통 · 낮음) 및 우선순위순 정렬
- 드래그 & 드롭 / ▲▼ 버튼으로 순서 변경
- 완료 체크 · 필터(전체 / 미완료 / 완료)
- 진행률 프로그레스 바
- `localStorage` 저장 (새로고침 후에도 유지)

## 기술 스택

- Vanilla HTML / CSS / JS — 외부 라이브러리 없음
- 다크 테마 디자인 (Inter 폰트, CSS 변수)
