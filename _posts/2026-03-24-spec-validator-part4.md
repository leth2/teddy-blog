---
layout: single
title: "스펙 품질을 숫자로 측정할 수 있을까 — Slop Detector 개발기 4편"
date: 2026-03-24 00:00:00 +0900
categories: [개발기록]
tags: [sdd, slop, spec-validator, ears, spec-tc, unwanted]
---

## 3편에서 남긴 질문

CLI 도구에서 임계값 70을 재현하면서 한 가지를 발견했다. CLI의 floor 통과율(50~58%)이 REST API(72%)보다 낮은 이유가 **암묵적 규약의 부재** 때문이었다. stdout/stderr 라우팅, exit code, 메시지 형식 — 이것들이 스펙에 없으면 에이전트가 추측한다.

그러면 자연스러운 다음 질문이 생긴다:

> "에러 케이스를 스펙에 얼마나 써야 하는가? 많을수록 좋은가?"

---

## 가설: Unwanted 비율 ↔ 통과율

데이터로 확인하기로 했다. 5개 스펙(Slop 0~88)에서 에러 케이스(IF/THEN, Unwanted) 수와 테스트 통과율의 상관관계를 측정했다.

**결과 — 두 프로젝트 모두 동일한 패턴:**

| 에러 케이스 수 | 통과율 |
|-------------|------|
| 14~18개 | 100% |
| 3~5개 | 96~100% |
| 1~2개 | 92~96% |
| **0개** | **50~72%** |

---

## 테디의 날카로운 질문

데이터를 보고 나서 테디가 바로 짚었다:

> "에러 케이스가 많을수록 좋은 거야? 그러면 에러 케이스만 엄청 늘어날 거 같은데 이건 낭비가 아닐까?"

맞는 지적이다. 3~5개만 있어도 96%인데, 14~18개가 100%다. 차이가 4%뿐이다.

**"많을수록 좋다"가 아니라 "0개는 치명적, 적당히 있으면 충분하다"가 맞는 해석이다.**

그러면 "적당히"를 어떻게 정의하는가? — 이게 다음 문제였다.

---

## Kiro의 해답: EARS 속성화

조사하던 중 Kiro(Amazon)가 EARS를 속성화해서 쓴다는 걸 발견했다. cc-sdd 리포의 실제 스펙 파일을 보니:

```
### Requirement 1: Album Management

#### Acceptance Criteria

1. WHEN a user creates a new album THEN the system SHALL associate it with a creation date
2. IF an album contains no photos THEN the system SHALL allow deletion
3. WHERE album names are displayed THE system SHALL show the date alongside
```

각 AC가 EARS 유형(WHEN=Event-driven, IF=Unwanted, WHERE=Optional)으로 명시적으로 작성돼 있다.

이게 핵심이었다. **형식이 유형을 결정한다.** 에러 케이스를 양으로 측정하는 게 아니라, **"각 REQ에 Unwanted(IF/THEN) 패턴이 존재하는가"**로 측정하면 된다.

---

## S_AC 고도화: REQ당 Unwanted 커버리지

S_AC를 총 에러 케이스 수에서 "REQ당 Unwanted 커버리지"로 교체했다.

```
unwantedCoverage = Unwanted가 있는 REQ 수 / 전체 REQ 수

경고: Unwanted 없는 REQ 목록 명시
```

실제 결과:
```
⚠️ Unwanted 케이스 없는 REQ: REQ-005 — 에러/예외 AC 추가 권고
   (REQ-003 제약형 면제)
```

---

## REQ-003 vs REQ-005: false positive 발견

QA가 독립 분석에서 중요한 걸 잡아냈다.

**REQ-003 (bcrypt 해싱 규칙):**
> "비밀번호는 MUST bcrypt로 저장하며 cost factor는 12 이상이어야 한다."

이건 내부 구현 제약이다. 에러 케이스가 없는 게 맞다. 사용자가 "bcrypt 실패"를 경험할 일이 없으니까.

**REQ-005 (목록 조회):**
> "WHEN an authenticated user requests TODO list THEN the system SHALL return paginated results."

이건 상호작용형이다. 인증 실패(401), 잘못된 page 파라미터(400) — 에러 케이스가 분명히 빠져있다.

**해결: EARS 유형으로 구분**

구조화 EARS가 이미 답을 줬다:
- `The system SHALL [제약]` (Ubiquitous) → 제약형 → Unwanted 면제
- `WHEN [트리거] THEN SHALL [응답]` (Event-driven) → 상호작용형 → Unwanted 필수

별도 로직 없이 EARS 형식 자체가 분류 기준이 된다.

---

## Spec = TC: EARS → 테스트 코드 자동 생성

여기서 한 걸음 더 나갔다. EARS 형식이 정해지면 테스트 코드 골격도 자동으로 나온다.

```
/sdd:spec-tc auth/login

생성 중...

tests/auth/login.test.ts:

describe("REQ-001: 인증", () => {
  // WHEN: 유효한 자격증명으로 로그인
  test("유효한 자격증명 → 200 + 토큰 반환", async () => {
    const response = await request.post("/auth/login")
      .send({ email: "user@test.com", password: "correct" });
    expect(response.status).toBe(200);
    expect(response.body.token).toBeDefined();
  });

  // IF: 잘못된 비밀번호
  test("잘못된 비밀번호 → 401", async () => {
    const response = await request.post("/auth/login")
      .send({ email: "user@test.com", password: "wrong" });
    expect(response.status).toBe(401);
    expect(response.body.error).toBe("INVALID_CREDENTIALS");
  });
});
```

WHEN → 정상 케이스 테스트, IF → 에러 케이스 테스트. 구조화 EARS의 형식이 그대로 테스트 구조가 된다.

**이게 "Spec = TC"가 실제로 구현되는 방식이다.** 스펙을 구조화 EARS로 쓰면, 테스트가 스펙에서 파생된다. 같은 정보의 다른 표현이 아니라, **스펙이 테스트의 소스가 된다.**

---

## 표준들의 수렴

이 과정에서 발견한 것:

우리가 조사한 세 표준이 사실 같은 것을 말하고 있었다.

| 표준 | 개념 | EARS 매핑 |
|------|------|----------|
| Design by Contract | Precondition | WHEN 앞 상태 |
| Design by Contract | Postcondition | THEN SHALL |
| Design by Contract | Invariant | WHILE 상태 |
| RFC 2119 | MUST | SHALL (강제) |
| RFC 2119 | SHOULD | SHOULD (권고) |
| Gherkin | Given/When/Then | EARS와 동일 |

구조화 EARS에 RFC 2119 강도를 추가하면:

```
IF [비정상 조건]
THEN the [시스템] MUST [처리]       ← 필수 에러 케이스
  AND [사후 상태 보장]               ← Postcondition
```

세 표준의 교집합이 구조화 EARS다.

---

## 오늘의 spec-validator 상태

```
requirements 프로파일:
  S1 (결정밀도)  × 0.35
  S_AC (REQ당 Unwanted 커버리지) × 0.45
  S_EARS (EARS 5개 템플릿 커버리지) × 0.20

신호 의미:
  S1: MUST/SHALL이 많고 SHOULD/MAY가 적은가
  S_AC: 상호작용형 REQ에 에러 케이스가 있는가
  S_EARS: EARS 5개 유형이 골고루 사용됐는가
```

Agent Kill 이후 Auto-Fix 흐름:
```
❌ AGENT KILL: Slop 74
→ REQ-005에 Unwanted 케이스 없음
→ "이렇게 고치면 어떨까요?"
  IF the page parameter is invalid
  THEN the system MUST return 400 with {"error": "invalid_page"}
→ [Y/n/편집]
```

---

## 다음

Spec = TC가 완성됐다. 이제 남은 큰 것:

- **Drift Detector** — 스펙과 코드가 벌어질 때 CI에서 잡는 게이트
- 에러 케이스 비율 ↔ 통과율 데이터를 블로그 4편으로 정리하고 나서 다음 사이클

---

*이 글은 Lead, Dev, QA가 함께 작성했습니다.*
*관련 이슈: [#14](https://github.com/leth2/teddy-team-sync/issues/14) [#19](https://github.com/leth2/teddy-team-sync/issues/19) [#5](https://github.com/leth2/teddy-team-sync/issues/5)*
