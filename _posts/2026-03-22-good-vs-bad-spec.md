---
layout: single
title: "좋은 스펙과 나쁜 스펙 — requirements, design, tasks 단계별 예시"
date: 2026-03-22 00:00:00 +0900
categories: [인사이트]
tags: [sdd, spec-quality, rfc2119, design-by-contract, gherkin]
---

스펙 품질을 숫자로 측정하는 도구를 만들면서 가장 먼저 해야 할 일이 생겼다. **"좋은 스펙"의 예시를 만드는 것.** 측정 도구는 기준이 있어야 작동한다.

이 글은 todo-api를 예시로 requirements, design, tasks 세 단계별로 나쁜 스펙과 좋은 스펙을 대비한다. 기준은 RFC 2119, Design by Contract, Gherkin BDD, Divio 문서 시스템이다.

---

## 1. requirements.md

### ❌ 나쁜 스펙

```markdown
# TODO API 요구사항

## 기능 요구사항

- 사용자는 TODO를 만들 수 있어야 한다.
- 목록을 볼 수 있어야 한다.
- TODO를 수정할 수 있어야 한다.
- 삭제도 가능해야 한다.
- 인증이 필요하다.
- 토큰은 적절한 시간 동안 유효해야 한다.
- 성능이 좋아야 한다.
```

**왜 나쁜가:**
- `MUST/SHALL` 없음 — 모든 요구사항이 선택적처럼 읽힘
- AC(수락 기준) 없음 — 개발자가 "완료"를 정의할 수 없음
- "적절한 시간", "성능이 좋아야" — 측정 불가 표현
- 에러 케이스 전무 — 실패 시나리오 없음
- REQ ID 없음 — 추적 불가

**Slop Score: ~88 🔴 SLOP**

---

### ✅ 좋은 스펙

```markdown
# TODO API 요구사항

## REQ-001: TODO 생성

사용자는 인증 후 TODO 항목을 생성할 수 있다.

**제약 조건:**
- 제목(title)은 MUST 1~200자 이내
- 설명(description)은 MAY 0~1000자
- 생성 시 상태는 MUST `pending`으로 초기화

**수락 기준 (AC):**
- AC-001: 유효한 요청 → 201 Created + 생성된 TODO 객체 반환
- AC-002: 제목 누락 → 400 Bad Request + `{"error": "title_required"}`
- AC-003: 제목 200자 초과 → 400 Bad Request + `{"error": "title_too_long"}`
- AC-004: 인증 토큰 없음 → 401 Unauthorized
- AC-005: 만료된 토큰 → 401 Unauthorized + `{"error": "token_expired"}`

---

## REQ-002: 인증 (JWT)

시스템은 MUST JWT 기반 인증을 사용한다.

**제약 조건:**
- 토큰 만료 시간: MUST 7일 (604800초)
- 알고리즘: MUST HS256
- Payload: MUST `{user_id, email, exp}` 포함

**수락 기준 (AC):**
- AC-006: 유효한 자격증명 → 200 OK + `{token, expires_at}`
- AC-007: 잘못된 비밀번호 → 401 Unauthorized + `{"error": "invalid_credentials"}`
- AC-008: 존재하지 않는 이메일 → 401 Unauthorized (사용자 열거 방지)
- AC-009: 만료된 토큰으로 API 접근 → 401 + `{"error": "token_expired"}`
```

**왜 좋은가:**
- REQ-NNN ID로 추적 가능
- MUST/MAY로 강도 명시 (RFC 2119)
- 각 REQ마다 수락 기준(AC) 존재
- 에러 케이스 구체적으로 명시
- "7일", "1~200자" 등 수치로 측정 가능

**Slop Score: ~12 🟢 OK**

---

## 2. design.md

### ❌ 나쁜 스펙

```markdown
# TODO API 설계

## 아키텍처

Express 서버를 사용한다. SQLite를 데이터베이스로 쓴다. JWT로 인증한다.

## API

- POST /todos
- GET /todos
- PUT /todos/:id
- DELETE /todos/:id

## 데이터베이스

users 테이블과 todos 테이블이 있다.
```

**왜 나쁜가:**
- "왜 Express인가", "왜 SQLite인가" — 설계 근거 없음
- API 엔드포인트만 나열 — 입출력 타입 없음
- 코드처럼 생겼지만 Reference 레이블 없음
- 트레이드오프 없음 — 다른 선택지 고려했는가?

**Slop Score: ~71 🔴 SLOP**

---

### ✅ 좋은 스펙

```markdown
# TODO API 설계

## 기술 결정

| 결정 | 선택 | 이유 | 대안 고려 |
|------|------|------|----------|
| 런타임 | Node.js + TypeScript | 팀 익숙도, 타입 안전성 | Go (성능 좋지만 팀 학습 비용) |
| 웹 프레임워크 | Express 4.x | 생태계, 미들웨어 풍부 | Fastify (더 빠르나 현 트래픽 불필요) |
| 데이터베이스 | SQLite | 검증 목적 단순성, 파일 기반 | PostgreSQL (프로덕션 전환 시 고려) |
| 인증 | JWT (stateless) | 수평 확장 용이 | Session (stateful, 현 요구사항 오버킬) |

## API 인터페이스 계약

### POST /todos

**Precondition:** 유효한 Bearer 토큰 + title 필드 존재

**Request:**
```json
// Example — 실제 타입 정의는 src/types/todo.ts 참조
{
  "title": "string (1~200자, required)",
  "description": "string (0~1000자, optional)"
}
```

**Postcondition:**
- 성공 시: DB에 TODO 저장 + 201 반환
- 실패 시: DB 변경 없음 + 4xx 반환 (원자성 보장)

**Invariant:** user_id는 MUST 생성자 ID로 자동 설정 (사용자 입력 불가)

## 에러 처리 전략

모든 에러 응답은 MUST 다음 형식을 따른다:
```json
// Reference: 모든 에러 응답의 표준 형식
{
  "error": "snake_case_error_code",
  "message": "사람이 읽을 수 있는 설명"
}
```

이유: 클라이언트가 에러 코드로 분기 처리할 수 있도록.
```

**왜 좋은가:**
- 기술 결정마다 이유 + 대안 명시
- Pre/Postcondition/Invariant 구분
- 코드 블록에 `// Reference`, `// Example` 레이블
- 코드 블록에 "이유" 주석 포함

**Slop Score: ~18 🟢 OK**

---

## 3. tasks.md

### ❌ 나쁜 스펙

```markdown
# TODO API 태스크

1. 프로젝트 세팅
2. 인증 구현
3. TODO CRUD 구현
4. 테스트
5. 배포
```

**왜 나쁜가:**
- 완료 기준(Done criteria) 없음
- 의존성 없음 — 어떤 순서로 해야 하는가?
- 너무 커서 분해 불가 — "인증 구현"이 하루인가 한 달인가?
- 테스트가 마지막에 있음 — TDD 원칙 위반

**Slop Score: ~82 🔴 SLOP**

---

### ✅ 좋은 스펙

```markdown
# TODO API 태스크

## TASK-001: 프로젝트 초기화

**의존성:** 없음
**예상 시간:** 1h

**완료 기준:**
- [ ] TypeScript + Express 프로젝트 생성
- [ ] ESLint + Prettier 설정
- [ ] Vitest 테스트 환경 구성
- [ ] `npm test` 실행 시 테스트 스위트 통과

---

## TASK-002: JWT 인증 — 테스트 먼저

**의존성:** TASK-001 완료
**예상 시간:** 3h

**완료 기준 (TDD 순서):**
- [ ] 인증 실패 테스트 작성 (Red)
  - 잘못된 비밀번호 → 401
  - 만료 토큰 → 401
  - 토큰 없음 → 401
- [ ] 인증 로직 구현 (Green) — REQ-002 AC-006~009 통과
- [ ] 리팩터링 (Refactor)
- [ ] 단위 테스트 커버리지 ≥ 90%

**검증:** `npm test -- auth` 전체 통과

---

## TASK-003: TODO CRUD — 테스트 먼저

**의존성:** TASK-002 완료
**예상 시간:** 4h

**완료 기준 (TDD 순서):**
- [ ] REQ-001 AC-001~005 테스트 작성 (Red)
- [ ] CRUD 구현 (Green)
- [ ] 단위 테스트 커버리지 ≥ 90%
- [ ] Gherkin 통합 테스트 통과:

```gherkin
Given 인증된 사용자가
When title="오늘 할 일"로 TODO 생성 요청하면
Then 201 Created + todo.id 반환
And DB에 status="pending"으로 저장됨
```

**검증:** `npm test -- todos` 전체 통과
```

**왜 좋은가:**
- TASK-NNN ID + 의존성 명시
- 예상 시간으로 크기 가늠 가능
- Done criteria가 체크박스로 구체적
- TDD 순서(Red→Green→Refactor) 명시
- Gherkin 통합 테스트 포함
- 검증 명령어까지 명시

**Slop Score: ~9 🟢 OK**

---

## 패턴 요약

| | 나쁜 스펙 | 좋은 스펙 |
|--|----------|----------|
| **언어** | "해야 한다", "가능해야 한다" | MUST, SHALL, MAY (RFC 2119) |
| **기준** | 없음 | AC, 완료 기준, 체크박스 |
| **에러** | 언급 없음 | 각 에러 케이스 + HTTP 코드 |
| **수치** | "적절히", "충분히" | "7일", "1~200자", "≥ 90%" |
| **이유** | 없음 | 결정마다 근거 + 대안 |
| **코드** | 레이블 없이 덤프 | Reference/Example 레이블 + 주석 |
| **순서** | 임의 | 의존성 선언 + TDD 순서 |

---

## 실증 결과 (todo-api Phase A)

Phase A에서 실제로 확인한 것:

- 나쁜 스펙(위 예시 수준) → **Slop Score 92, Agent Kill 발동**
- 개선된 스펙 → **Slop Score 0~38, OK 통과**

핵심 발견: "TODO 항목"이라는 도메인 용어가 TBD 마커로 오인식되는 버그도 발견됐다. 실제 데이터로 돌려봐야 도구의 false positive가 드러난다.

---

*이 글은 spec-validator Phase 1.5 개발과 todo-api 검증 사이클의 기록입니다.*
*이슈: [teddy-team-sync #12](https://github.com/leth2/teddy-team-sync/issues/12)*
