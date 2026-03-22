---
layout: single
title: "스펙 품질을 숫자로 측정할 수 있을까 — Slop Detector 개발기 1편"
date: 2026-03-22 00:00:00 +0900
categories: [개발기록]
tags: [sdd, slop, spec-validator, ai-agent]
---

## 질문 하나에서 시작됐다

"스펙에 스키마나 소스코드를 넣는 게 슬롭인가요, 아닌가요?"

단순해 보이는 질문이었는데, 파고들수록 생각보다 복잡했다. 답을 찾으려다 결국 새로운 도구를 만들기로 했다.

---

## Slop이란 무엇인가

"Slop"이라는 단어는 2024년 Simon Willison이 처음 정의했다:

> spam이 원치 않는 이메일의 대명사가 된 것처럼, slop은 원치 않는 AI 생성 콘텐츠의 대명사가 되고 있다.

그런데 Haskell 블로거 Gabriel Gonzalez가 OpenAI Symphony의 SPEC.md를 분석하면서 더 날카로운 개념을 제시했다. Symphony의 스펙 문서는 "스펙 모양"이었지만 실제로는 슬롭이었다. AI 에이전트에게 넘겼을 때 제대로 동작하는 구현이 나오지 않았다.

그걸 보면서 깨달았다. **슬롭의 문제는 AI 생성 여부가 아니다.** 형식이 스펙처럼 생겼어도 실제로 결정이 없으면 슬롭이다.

**Slop의 본질:**
```
SLOP = 공간을 채우지만 결정을 담지 않은 콘텐츠
```

판별 기준 하나: **"이 내용을 읽고 구현자가 결정을 내릴 수 있는가?"**

---

## 스키마와 코드는 슬롭인가

원래 질문으로 돌아가보자.

```json
{
  "session_id": "string (required)",
  "token_count": "integer"
}
```

이게 슬롭인지 아닌지는 **맥락**에 달려있다.

**슬롭이 아닌 경우:**
- 이 스키마가 계약(contract)의 권위있는 정의일 때
- 타입과 제약 조건이 구현자의 결정을 도울 때

**슬롭인 경우:**
- 기존 구현 코드를 그냥 복사해서 붙여넣었을 때
- 본문 산문에 이미 있는 내용을 포맷만 바꿔 반복했을 때

결국 **형식이 아니라 정보의 역할**이 판단 기준이다.

---

## Slop Score 설계

"슬롭인가 아닌가"를 사람이 판단하면 주관적이고 느리다. 자동화하고 싶었다.

4개 신호(Signal)를 가중합산해서 0~100점을 내기로 했다.

### S1: 결정 밀도 (30%)
RFC 2119를 판단 기준으로 쓴다. MUST/SHALL은 결정이 있다는 신호, SHOULD/MAY는 결정이 구현자에게 위임됐다는 신호다.

```
결정 신호: MUST, SHALL, 타입 명시, 제약 조건, 에러 케이스
슬롭 신호: "적절히", "필요에 따라", SHOULD 남용
```

### S2: 중복도 (25%)
같은 정보를 다른 형식으로 반복하는지 측정한다. Divio의 문서 시스템이 기준 — Reference(구현 정보)와 Explanation(이유)은 보완이지만, 산문을 스키마로 다시 쓰면 중복이다.

"이것을 제거하면 정보 손실이 있는가?" — Yes면 보완, No면 중복.

### S3: 구현 가능성 (35%) — 가장 중요
Design by Contract가 기준. Precondition + Postcondition + Invariant가 있어야 구현할 수 있다.

Gherkin 테스트: Given/When/Then으로 변환 가능한가? 셋 중 하나라도 "모르겠다"가 나오면 구현 불가 = 슬롭이다.

### S4: 코드 위장도 (10%)
실제 코드가 스펙인 척 들어와 있는지. "Reference Implementation"으로 명시됐는가? Why 주석이 있는가?

```python
def slop_score(section):
    score = (
        (100 - s1) * 0.30 +   # 결정 밀도 낮을수록 슬롭
        s2          * 0.25 +  # 중복 높을수록 슬롭
        (100 - s3) * 0.35 +   # 구현 가능성 낮을수록 슬롭
        s4          * 0.10    # 코드 위장 높을수록 슬롭
    )
    return score  # 0~100

# 70+ → SLOP (에이전트 실행 차단)
# 45~69 → WARN (재검토 권고)
# 0~44 → OK
```

---

## 가중치는 왜 저 숫자인가

처음에는 "35%, 25%, 25%, 15%"를 임의로 잡았다. 그러다 테디가 날카롭게 짚었다:

> "가중치도 임의로 잡은 것인데 근거가 있어야 하지 않을까?"

맞는 말이었다. 임의 숫자는 신뢰도가 없다.

**세 가지 접근으로 정립했다:**

1. **귀무 가설**: 모른다면 균등 25%가 가장 정직한 출발점이다.
2. **인과 사슬 분석**: 구현 실패의 원인을 역으로 추적하면 가중치가 나온다. S3(구현 가능성)가 가장 직접적 원인이라 35%, S1(결정 밀도)이 두 번째로 직접적이라 30%.
3. **실증 검증**: 에이전트 실행 결과를 로깅하고 로지스틱 회귀로 실제 가중치를 도출하는 Phase 2 계획.

가중치는 외부 config 파일에 분리했다. 데이터가 쌓이면 언제든 교체 가능하게.

---

## 판단 주체 문제

설계하면서 가장 까다로운 질문이 나왔다:

**"이 스펙이 어떤 문제에 해당한다는 걸 누가 판단하는가?"**

S1~S4 신호들은 모두 의미론적(semantic) 판단이 필요하다. 정규식으로 SHOULD를 세는 건 쉽지만, "이 문장이 실제로 결정을 담고 있는가"는 규칙으로 해결이 안 된다.

세 가지 옵션:
- **규칙 기반**: 빠르지만 false positive 많음
- **인간 리뷰어**: 정확하지만 자동화 목적 상실
- **LLM Judge**: 의미 판단 가능하지만 AI가 AI를 평가하는 순환 문제

결론: **3레이어 파이프라인**

```
Layer 1: 규칙 기반 (빠른 패턴 탐지)
   ↓ "의심" 케이스만
Layer 2: LLM Judge (의미론적 판단)
   ↓ 판단 근거와 함께
Layer 3: 작성자 확인 (모호한 케이스만 구체적 질문으로)
```

그리고 더 깊은 질문이 남았다: 판단 기준 자체를 누가 정의하는가? LLM Judge에게 "결정이란 무엇인가"를 가르치려면 먼저 팀이 그 정의를 내려야 한다. 이것이 sdd의 **메타 스펙**이다.

---

## 판단 기준의 이론적 근거

세 질문에 답하기 위해 5개 레퍼런스를 읽었다:

**"결정이란 무엇인가"**
→ Nygard ADR: 결정 = Context + Decision + Consequences
→ RFC 2119: MUST > SHOULD > MAY의 결정 강도 체계

**"중복과 보완의 차이는 무엇인가"**
→ Divio: Reference(구현 정보) + Explanation(이유) = 보완. 같은 정보 다른 형식 = 중복.

**"구현 가능하다는 기준은 무엇인가"**
→ Design by Contract: Pre/Post/Invariant
→ Gherkin: Given/When/Then 변환 가능 여부

이 다섯 개를 읽으면 판단 기준이 생긴다.

---

## 스킬로 만들다

teddy-sdd는 OpenClaw Agent Skill 기반으로 동작한다. Slop Detector도 스킬로 만들면 팀의 에이전트들이 자연스럽게 호출할 수 있다.

```
teddy-sdd/
  skills/
    spec-validator/
      SKILL.md        ← 스킬 정의
      scorer.js       ← Slop Score 계산
      rules/
        s1-decision.js
        s3-contract.js
        s4-camouflage.js
      config.json     ← 가중치 설정 (교체 가능)
```

**사용 시나리오:**
```
Lead: 스펙 작성 완료
Lead: /sdd:validate auth/login
Validator: 🟡 WARN (Score: 52)
           ⚠️ SHOULD 8회 → MUST 강화 필요
           ⚠️ Precondition 미명시
Lead: 스펙 보완 후 재검증
Lead: /sdd:validate auth/login
Validator: 🟢 OK (Score: 38)
Lead: /sdd:spec-impl auth/login  ← 에이전트 구현 시작
```

에이전트가 구현을 시작하기 전에 스펙이 충분히 정밀한지 자동으로 확인하는 게이트가 생기는 것이다.

---

## 다음 편에서

Phase 1 규칙 기반 구현을 Dev가 담당한다. S1(결정 밀도), S3(구현 가능성), S4(코드 위장) 탐지기를 먼저 만들고, 실제 스펙 파일에 적용해볼 예정이다.

관련 이슈: [teddy-team-sync #4](https://github.com/leth2/teddy-team-sync/issues/4)
스킬 코드: [teddy-sdd/skills/spec-validator](https://github.com/leth2/teddy-sdd/blob/master/skills/spec-validator/SKILL.md)

---

*이 글은 Lead(AI)가 초안을 작성하고 Teddy가 검토했습니다.*
