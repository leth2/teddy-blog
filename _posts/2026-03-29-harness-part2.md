---
layout: single
title: "스펙 품질을 어떻게 측정하는가 — 하네스 엔지니어링 시리즈 2편"
date: 2026-03-29 00:00:00 +0900
categories: [개발기록]
tags: [sdd, slop-detector, ears, spec-tc, rfc2119, spec-quality]
series: harness-engineering
series_part: 2
---

> 이 글은 "하네스 엔지니어링" 시리즈의 2편입니다.  
> 1편에서는 코드 속도가 병목이 아니라 스펙 품질이 문제라는 것을 이야기했습니다.  
> 2편에서는 그 품질을 어떻게 측정하는지를 다룹니다.

---

## 스펙 품질이 측정 가능한가?

"좋은 스펙"은 직관적으로 느껴진다. 읽다 보면 "이건 구현 가능하다"는 느낌이 오는 것. 반대로 나쁜 스펙을 읽으면 "이걸로 뭘 만들라는 건지"라는 답답함이 온다.

문제는 이 직관을 기계가 측정할 수 없다는 거다. 직관으로는 게이트를 만들 수 없다.

**측정 가능해야 자동화할 수 있다. 자동화해야 24/7 작동한다.**

우리가 만든 Slop Detector는 이 질문에서 시작했다: 스펙의 나쁜 품질 — 우리는 이걸 "Slop(슬롭)"이라고 부른다 — 을 어떻게 정량화할 것인가?

---

## 나쁜 스펙 vs 좋은 스펙 — 실제로 어떻게 다른가

같은 기능을 두 가지 방식으로 써보자. REST API의 사용자 인증 기능이다.

### ❌ 나쁜 스펙 (Slop Score: 88)

```markdown
## 인증

- 사용자는 로그인할 수 있어야 한다.
- 토큰은 적절한 시간 동안 유효해야 한다.
- 보안은 일반적인 수준으로 한다.
- 실패 시 에러를 반환한다.
```

읽으면 말이 된다. 근데 AI 에이전트에게 이걸 주면:
- "적절한 시간" → 24시간으로 추정 (실제 기준이 7일이면 실패)
- "일반적인 보안" → bcrypt cost=10 선택 (기준이 12면 실패)
- "에러를 반환한다" → 어떤 형식? 어떤 코드? 추측한다

### ✅ 좋은 스펙 (Slop Score: 12)

```markdown
## REQ-002: 인증 (JWT)

시스템은 MUST JWT 기반 인증을 사용한다.

**제약 조건:**
- 토큰 만료: MUST 7일 (604800초)
- 알고리즘: MUST HS256
- 비밀번호 해싱: MUST bcrypt, cost factor ≥ 12

**수락 기준:**
1. WHEN 유효한 자격증명으로 로그인 요청 시
   THEN 200 OK + {"token": "...", "expires_at": "..."}

2. IF 잘못된 비밀번호
   THEN 401 + {"error": "INVALID_CREDENTIALS"}

3. IF 존재하지 않는 이메일
   THEN 401 + {"error": "INVALID_CREDENTIALS"}
   (사용자 열거 방지 — 이메일 존재 여부를 구분하지 않는다)

4. IF 만료된 토큰으로 API 접근
   THEN 401 + {"error": "TOKEN_EXPIRED"}
```

둘의 차이는 "말이 되느냐"가 아니다. **AI가 추측해야 하는 항목의 수**다.

나쁜 스펙: 추측 항목 7개 이상. 좋은 스펙: 추측 항목 0개.

---

## Slop의 세 가지 신호

반복해서 보니 나쁜 스펙에는 세 가지 패턴이 있었다.

### S1. 결정 밀도가 낮다 — RFC 2119 기준

IETF가 1997년에 만든 RFC 2119는 MUST / SHOULD / MAY의 강도를 법적 수준으로 정의한다.

```
MUST / SHALL     → 절대적 요구사항. 반드시 구현해야 한다.
SHOULD           → 권고사항. 특별한 이유 없으면 따라야 한다.
MAY              → 선택사항. 구현해도 되고 안 해도 된다.
```

나쁜 스펙에는 MUST/SHALL 대신 "해야 한다", "가능해야 한다", "적절히"가 가득하다.

```typescript
// 모호 표현 감지 패턴 (Node.js/TypeScript 구현)
const SLOP_PATTERNS = [
  /적절히|적절한|충분히|필요에 따라|일반적인 수준/,
  /가능해야|할 수 있어야/,
  /등$|기타|여러\s/,  // 열거 미완성
];
```

### S2. 에러 케이스가 없다

우리 실험에서 발견한 것: 에러 케이스(IF/THEN) 수와 테스트 통과율 사이에 강한 상관관계가 있다.

| 에러 케이스 수 | 통과율 |
|-------------|------|
| 14~18개 | 100% |
| 3~5개 | 96~100% |
| 1~2개 | 92~96% |
| **0개** | **50~72%** |

중요한 발견: "많을수록 좋다"가 아니다. **0개가 치명적**이다. 3~5개면 충분하다. 14~18개와 3~5개의 통과율 차이는 4%p에 불과하다 — 에러 케이스를 늘리는 데 드는 비용 대비 수익이 급격히 줄어든다. 에러 케이스가 전혀 없는 REQ는 AI가 에러 처리를 완전히 추측으로 구현한다.

> *동일 실험 데이터(n=10, REST API + CLI 타입) 기반.*

### S3. 결정 근거가 없다

"왜 Express인가?", "왜 bcrypt인가?" — 이런 질문에 답이 없는 설계 문서는 규칙 기반으로 탐지가 불가능하다. AI가 "그냥 쓰이는 걸로" 선택하게 된다.

예시: 설계 문서에 "Express 4.x를 사용한다"만 있고 이유가 없으면, AI는 Fastify나 Koa를 고려하지 않고 주어진 것을 그대로 쓴다. 나중에 "왜 Express를 골랐는가?"라는 질문에 팀은 아무도 답을 못한다.

이 신호는 LLM Judge만 잡을 수 있다. 기본 실행에서는 건너뛰고, `--with-llm` 플래그로 선택적으로 활성화한다.

---

## EARS — 항공우주에서 온 구조화 요구사항

S1이 "무엇을 써야 하는가(강도)"를 감지한다면, EARS는 "어떻게 써야 하는가(구조)"를 제공한다. 자연어의 모호함이 Slop의 근본 원인 중 하나다. "로그인 실패 시 에러를 반환한다"라는 문장은 조건, 주체, 결과가 뒤섞여 있다.

2009년 Alistair Mavin은 Rolls-Royce에서 항공우주/방산 요구사항을 기계 검증 가능한 형식으로 만드는 방법을 연구했다 (IEEE). 항공기 소프트웨어에서 모호한 요구사항은 단순히 테스트 실패가 아니라 인명 사고로 이어질 수 있기 때문이다.

그가 만든 EARS(Easy Approach to Requirements Syntax)는 5가지 패턴으로 요구사항을 구조화한다:

| 유형 | 패턴 | 언제 쓰나 |
|------|------|---------|
| Ubiquitous | `The [시스템] SHALL [동작]` | 항상 적용되는 기본 요구사항 |
| Event-driven | `WHEN [트리거] THEN SHALL [응답]` | 특정 이벤트에 반응할 때 |
| Unwanted | `IF [비정상 조건] THEN SHALL [처리]` | 에러/예외 케이스 |
| State-driven | `WHILE [상태] SHALL [동작]` | 특정 상태 유지 중 |
| Optional | `WHERE [포함 시] SHALL [동작]` | 선택적 기능 |

항공우주 요구사항 표준이 AI 에이전트 스펙에 적합한 이유는 동일하다: **모호한 언어는 검증이 불가능하다.**

### 구조화 전후

```
# Before (자연어)
"인증 실패 시 에러를 반환한다"

# After (구조화 EARS)
IF the user provides invalid credentials
THEN the system MUST return 401
  AND {"error": "INVALID_CREDENTIALS"}
```

차이는 "예쁜 형식"이 아니다. 후자는 **기계가 파싱할 수 있다**. 조건(IF), 주체(the system), 강도(MUST), 결과(401 + 형식)가 명확히 분리되어 있다.

---

## Spec = TC — 스펙이 테스트가 된다

구조화 EARS가 중요한 이유가 하나 더 있다. WHEN/IF/THEN 패턴은 테스트 코드 구조와 1:1로 대응된다.

```typescript
// 생성: spec-tc가 spec/auth.md#REQ-002에서 자동 생성
// spec_ref: spec/auth.md#REQ-002  ← 어느 스펙인지 연결
describe("REQ-002: 인증", () => {

  // WHEN: 유효한 자격증명
  test("유효한 자격증명 → 200 + 토큰", async () => {
    const res = await post("/auth/login", { email, password: "correct" });
    expect(res.status).toBe(200);
    expect(res.body.token).toBeDefined();
  });

  // IF: 잘못된 비밀번호
  test("잘못된 비밀번호 → 401", async () => {
    const res = await post("/auth/login", { email, password: "wrong" });
    expect(res.status).toBe(401);
    expect(res.body.error).toBe("INVALID_CREDENTIALS");
  });

});
```

`spec_ref`가 핵심이다. 이 테스트가 어느 스펙에서 나왔는지 코드 레벨에 연결되어 있다.

나중에 테스트가 실패하면 AI는 단순히 "401이 나왔다"가 아니라 "spec/auth.md#REQ-002의 의도에 맞게 고쳐야 한다"를 안다. 추측이 아닌 스펙 기반 수정이 된다.

**스펙이 테스트의 소스가 된다. 같은 의도의 다른 표현이 아니라, 스펙이 테스트를 파생시킨다.**

---

## 임계값 70은 보편적인가

Slop Score를 만들고 첫 번째 질문이 생겼다: 임계값 70은 REST API에서만 통하는 건 아닌가?

CLI 도구(task-cli)로 같은 실험을 반복했다:

| Slop Score | REST API (todo-api) | CLI (task-cli) |
|-----------|---------------------|----------------|
| ~0 | 100% | 100% |
| ~51 | 92% | 96% |
| ~79 | **72%** | **58%** |
| ~88 | 72% | 50% |

**임계값 70 효과가 CLI에서도 재현됐다.** 두 타입 모두 Slop 70 이상에서 통과율이 급락한다.

CLI의 floor(바닥)이 더 낮은(58% vs 72%) 이유가 있다. REST API에는 HTTP 상태 코드라는 업계 표준이 있다 — "에러면 4xx"라는 암묵적 규약. CLI에는 그런 표준이 없다. stdout vs stderr, exit code 0 vs 1, 메시지 형식 — 전부 스펙에 없으면 에이전트가 추측한다.

**표준이 적은 도메인일수록 스펙이 더 명시적이어야 한다.**

**임계값 70은 고정된 기준이 아니다.** REST API + CLI 두 타입, n=10 실험에서 도출한 초기 경험값이다. `score-weights.json`으로 팀마다 조정 권장. 70이 의미 있는 이유는 Slop 51→79 구간에서 통과율이 급락한다는 패턴이 두 타입에서 재현됐기 때문이지, 70이라는 숫자 자체가 절대적인 것은 아니다.

---

## 파이프라인으로 묶으면

이 도구들은 개별로도 쓸 수 있지만 파이프라인으로 연결될 때 가치가 배가된다:

```
스펙 작성
    ↓
[Slop Detector] → Slop ≥ 70: 차단 + 수정 권고
    ↓ (통과)
[spec-tc] → 테스트 골격 자동 생성 (spec_ref 포함)
    ↓
AI 에이전트 구현
    ↓
[drift-detector] → 스펙 ↔ 코드 불일치 감지
    ↓
[spec-score --retro] → 사후 점수 보정, 재작성 권고
```

이 파이프라인이 **항상, 자동으로** 작동하게 만드는 것이 하네스다.

---

## 다음 편에서

파이프라인을 만들었다. 근데 이걸 24시간 돌리려면 무엇이 필요한가?

사람이 매번 트리거하는 것과 AI가 자율적으로 도는 것의 차이. 사전 승인된 샌드박스, AI를 위한 로그 설계, git을 foundation으로 쓰는 이유.

→ **3편: AI를 위한 환경 — 하네스 설계** (`2026-03-29-sdd-harness`)

---

*참고 자료:*
- *[EARS — Alistair Mavin, Rolls-Royce / IEEE (2009)](https://ieeexplore.ieee.org/document/5328509)*
- *[RFC 2119 — IETF (1997)](https://datatracker.ietf.org/doc/html/rfc2119)*
- *[Design by Contract — Bertrand Meyer (Eiffel 설계자, ACM Fellow)](https://www.eiffel.com/values/design-by-contract/introduction/)*
- *[Gherkin Reference — Cucumber](https://cucumber.io/docs/gherkin/reference/)*
- *내부 실험 데이터: spec-validator n=10 (REST API + CLI 타입)*
