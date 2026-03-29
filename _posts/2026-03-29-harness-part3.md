---
layout: single
title: "AI를 위한 환경 — 하네스 설계 (하네스 엔지니어링 시리즈 3편)"
date: 2026-03-29 00:00:00 +0900
categories: [개발기록]
tags: [sdd, harness, agent, spec-driven, automation, ai-sandbox, trust-engineering]
series: harness-engineering
series_part: 3
---

> 이 글은 "하네스 엔지니어링" 시리즈의 3편입니다.  
> 1편: 왜 스펙 품질이 문제인가 / 2편: 어떻게 측정하는가  
> 3편에서는 그 도구들을 24/7 자율 운영하게 만드는 하네스 설계를 다룹니다.

---

## 핵심 주장

AI 에이전트 파이프라인을 개선하는 방법을 찾다 보면 자연스럽게 이쪽으로 간다.

> "RAG를 붙이면 어떨까? 벡터 DB로 컨텍스트를 풍부하게 하면? 그래프DB로 의존성을 추적하면?"

우리가 스펙 검증 도구를 만들면서 다른 결론에 도달했다.

**AI가 올바른 판단을 하려면 올바른 입력이 필요하다.**

RAG는 더 많은 것을 모델에게 넣는 방향이다. 하지만 Slop이 섞인 스펙 1000개를 검색해서 넣어봤자, 모델은 여전히 모호한 의도를 추측으로 채운다. "Garbage in, garbage out"이 더 정교하게, 더 비싸게 발생할 뿐이다.

진짜 문제는 입력의 **양**이 아니라 **품질**이다.

> 이 글은 이 주장을 설계 원칙으로 구체화한다:  
> 스펙 품질 게이트 → AI 자율 실행 환경(하네스) → 증명 기반 신뢰 구조.

---

## 왜 스펙 품질을 먼저 만들었나

코드를 생성하는 속도는 폭발적으로 빨라졌다. AI가 코드를 몇 배 빠르게 만든다.

그런데 이상한 일이 생겼다. 에이전트가 코드를 더 빠르게 만들수록, 버그도 더 빠르게 생겼다.

우리 실험 데이터가 이걸 보여준다:

| 스펙 품질 (Slop Score) | 에이전트 테스트 통과율 | 구현 갭 수 |
|---------------------|-------------------|----------|
| 0 (매우 명확) | 100% (25/25) | 7건 |
| 19 | 96% (24/25) | 9건 |
| 51 (중간) | 92% (23/25) | 9건 |
| 79 (모호) | **72%** (18/25) | 20건 |
| 88 (최악) | 72% (18/25) | 20건 |

> *n=5, REST API + CLI 타입. 임계값 70은 Slop 51(92%)~79(72%) 사이의 급격한 하락 구간에 안전 마진을 포함한 운영값이다. 프로젝트 타입별로 `score-weights.json`에서 조정 가능하다.*

Slop 51→79 구간에서 통과율이 20% 급락한다. 임계값 70은 이 하락 구간의 시작점 앞에 설정된 안전선이다.

실패 원인을 보면 패턴이 명확하다:

```
"토큰은 적절한 시간 동안"  → 24시간으로 추정 → 7일 기준 테스트 실패
"필요에 따라 입력한다"    → 제한 없음 구현 → 200자 초과 테스트 실패
"보안은 일반적인 수준"    → bcrypt cost=10 → cost=12 기준 실패
에러 코드 미명시          → NOT_FOUND     → TODO_NOT_FOUND 불일치
```

모호한 언어가 에이전트의 임의 추론으로 이어지고, 그 추론이 틀릴 때 버그가 된다.  
**모델이 나쁜 게 아니다. 입력이 나쁜 것이다.**

---

## 스펙 품질 게이트 — 모델에게 들어가기 전 필터

### Slop Detector

스펙의 품질을 정량적으로 측정한다. 계산 방식은 두 단계:

**1단계 (규칙 기반, 재현 가능)**
- RFC 2119 키워드 밀도: MUST/SHALL 비율 vs SHOULD/MAY/적절히/필요에 따라 비율
- 모호 표현 정규식 탐지: "적절히", "충분히", "일반적인 수준", "필요에 따라"
- AC(수락 기준) 완결성: 각 REQ에 에러 케이스(IF/THEN) 존재 여부

**2단계 (LLM Judge, 선택적)**
- `--with-llm` 플래그로 활성화
- "왜 이 기술인가?" 같은 의사결정 근거 누락 탐지 (규칙으로 불가)
- 기본값은 규칙 기반만 — 재현 가능성을 우선한다

```
📊 Spec Quality Score: auth-api
══════════════════════════════════════
종합: 45/100 — 구현 진행 가능 (< 70)

모호성 검출 (2건):
  🟡 "적절한 시간" — 구체적 수치 없음
  🟡 "일반적인 보안 수준" — 기준 미정의

→ 수정 후 진행 권장
```

Slop ≥ 70이면 **하네스가 파이프라인을 차단**하고 사람에게 알린다. AI가 자체 중단하는 것이 아니다 — 하네스가 막는 것이다. AI가 모호한 스펙을 자동으로 "수정"하면, 원래 의도와 다른 스펙이 만들어진다. 그게 더 위험하다.

---

### EARS 구조화 — AI가 파싱할 수 있는 형식

```
# Before (자연어)
"로그인 실패 시 에러를 반환한다"

# After (구조화 EARS)
IF the user provides invalid credentials
THEN the system MUST return 401
  AND {"error": "INVALID_CREDENTIALS"}
```

**형식이 정보 밀도를 결정한다.** 구조화된 EARS는:
- 조건(precondition)/결과(postcondition)를 기계가 파싱 가능
- 테스트 코드로 자동 변환 가능
- 경계 조건이 명시적으로 드러남

EARS(Easy Approach to Requirements Syntax)는 Alistair Mavin이 Rolls-Royce에서 항공우주/방산 요구사항을 기계 검증 가능한 형식으로 만들기 위해 개발했다(IEEE, 2009). 모호한 언어로는 검증이 불가능하다 — AI 에이전트 스펙에 구조화 EARS가 필요한 이유가 동일하다.

Bertrand Meyer의 Design by Contract도 같은 원칙이다: "소프트웨어 신뢰성은 정밀하게 정의된 명세(precondition + postcondition)에서 나온다." Meyer는 Eiffel 언어 설계자이자 ACM Fellow다.

RFC 2119(IETF, 1997)는 MUST/SHOULD/MAY의 강도를 법적 수준으로 정의한다. 30년 된 표준이 AI 스펙 품질 측정에도 그대로 유효하다. Slop Detector의 S1(결정밀도) 신호가 이 기반이다.

---

### Spec-TC — 스펙에서 테스트 자동 생성

구조화된 스펙은 테스트 골격으로 직접 변환된다:

```typescript
// spec_ref: spec/auth.md#REQ-003
// 이 테스트는 spec-tc가 생성. 읽기 전용 — AI가 수정 불가.
describe("REQ-003: 인증", () => {
  // WHEN: 유효한 자격증명
  test("유효한 자격증명 → 200 + 토큰 반환", async () => { ... });

  // IF: 잘못된 비밀번호
  test("잘못된 비밀번호 → 401", async () => { ... });
});
```

`spec_ref`가 핵심이다. 각 테스트가 어느 스펙과 연결되는지 코드 레벨에 존재한다.

**중요: spec-tc가 생성한 테스트 골격은 읽기 전용으로 보호된다.** 구체적으로는 두 가지 메커니즘으로 구현한다:

1. **Git pre-commit hook**: `tests/spec-generated/` 경로의 파일 변경을 감지해서 커밋 차단
2. **drift-detector**: 테스트 파일의 `spec_ref` 주석이 변경되면 CI에서 감지

AI는 `src/` 구현 코드만 건드릴 수 있다. 테스트 골격 자체를 수정하면 훅이 막는다. 이것이 "증명 패키지 조작" 우려에 대한 구조적 방어다.

---

### Drift Detector

구현 후 스펙과 코드가 벌어지는 것을 CI에서 잡는다:

```
❌ Drift 감지: spec/auth.md#3.2 ↔ src/auth/handler.ts
   - 스펙: 토큰 만료 시 401 반환
   - 구현: 토큰 만료 시 500 반환
→ 빌드 실패: 스펙 또는 구현 수정 필요
```

Martin Fowler의 CI 원칙과 같다: "통합 에러는 빠르게 발견하고 빠르게 고친다." 우리는 코드-코드 통합이 아닌 **스펙-코드 통합**을 지속적으로 검증한다.

---

## 하네스 — AI 자율 사이클을 위한 환경

개별 도구들은 수동으로 돌릴 수 있다. 하네스는 그것을 **사람 개입 없이 완전한 사이클로** 만드는 것이다.

### 설계 전제: 사람이 아닌 AI를 위한 환경

이 하네스는 개발자의 편의를 위한 자동화 도구가 아니다. **AI가 24/7 자율적으로 작동하기 위한 환경**이다.

24시간 돌아가는 하네스는 매 단계마다 사람의 승인을 받을 수 없다. 대신 **사전에 모든 승인이 이뤄진 샌드박스** 안에서 AI가 자유롭게 실행한다:

```
[사전 승인된 샌드박스 — 사람이 정의, 버전 관리]
├── 읽기/쓰기 가능한 파일 경계
├── 실행 가능한 명령 목록
├── 외부 통신 allowlist (나머지는 차단)
└── AI는 이 경계 안에서 자율 실행
    (경계 자체는 AI가 변경 불가)

[사람의 역할]
└── 최종 결과물 검토 — 수락 / 거절
    (실행 과정 개입 없음)
```

Anthropic의 Claude Agent SDK가 이 패턴을 공식 구현한다: `allowedTools: ['Read', 'Edit', 'Bash']` — 에이전트가 할 수 있는 것을 선언적으로 제한. Computer Use 공식 문서에서는 "dedicated virtual machine or container with minimal privileges + allowlist domains + human to confirm decisions with real-world consequences"를 필수 권고한다.

MCP 공식 스펙(2025-06-18 버전)도 같은 방향이다:
> "For trust & safety, there SHOULD always be a human in the loop with the ability to deny."

사전 승인 기반 샌드박스는 새로운 아이디어가 아니라 **업계가 수렴하고 있는 표준 패턴**이다.

---

### AI 친화적 테스트 환경 — 출력도 읽을 수 있어야 한다

스펙 품질 게이트는 **입력**을 정제한다. AI가 올바른 판단을 하려면 **출력**도 읽을 수 있어야 한다.

```
# 사람을 위한 테스트 출력 (지금)
✗ auth test failed
  Expected 401, got 500
  at Object.<anonymous> (/src/auth.test.ts:23)

# AI를 위한 테스트 출력 (필요한 것)
{
  "test": "auth/login",
  "status": "fail",
  "spec_ref": "spec/auth.md#REQ-003",   ← spec-tc가 심어놓은 연결
  "expected": 401,
  "actual": 500,
  "container_logs": [...],
  "related_files": ["src/auth/handler.ts"]
}
```

`spec_ref` 덕분에 AI는 "500 에러 고쳐줘"가 아니라 "spec/auth.md#REQ-003의 의도에 맞게 고쳐줘"로 작업한다. 추측이 아니라 스펙 기반 수정이 된다.

smolagents(HuggingFace)의 에이전트 루프가 이 구조를 명확히 한다:
```python
memory = [user_defined_task]
while llm_should_continue(memory):
    action = llm_get_next_action(memory)
    observations = execute_action(action)
    memory += [action, observations]   # 행동 + 관찰값이 연결된 상태 기록
```

**Docker는 이 환경을 격리하고 재현 가능하게 만드는 foundation이다.** AI가 동일한 환경을 언제든 다시 실행할 수 있어야 한다. "내 환경에선 됐는데"는 24/7 자율 운영의 적이다.

---

### AI 자율 사이클 — 전체 흐름

```
① 스펙 읽기
   → [하네스] Slop Score 측정
   → Slop ≥ 70: 하네스가 실행 차단, 사람에게 스펙 보완 요청
   → Slop < 70: 다음 단계 진행
         ↓
② 테스트 골격 생성 (spec-tc)
   → spec_ref 포함, 기대 동작 정의
   → 생성된 테스트는 읽기 전용 보호
         ↓
③ 구현 (Docker 샌드박스 안)
   → 실패 시 스펙 재참조 + 재시도 (최대 N회)
   → N회 초과: 실행 중단, 사람에게 에스컬레이션
         ↓
④ 자기검증
   → 테스트 통과 확인
   → Drift 체크 (spec ↔ src)
   → 증명 패키지 생성
         ↓
⑤ 사람에게 제출 (PR + 증명 패키지)
   → 사람: 수락 / 거절
```

---

## 증명 패키지 — 주장이 아니라 증거

AI가 자율로 실행한다면, 사람은 어떻게 신뢰하는가?

**"테스트 통과했습니다"라는 AI의 주장은 충분하지 않다.** AI가 결과를 주장하는 게 아니라 **증거를 제출하는 구조**여야 한다.

```json
{
  "summary": "auth 스펙 구현 완료",
  "spec_version": "spec/auth.md@abc123",
  "slop_score": 42,
  "test_results": {
    "total": 25, "passed": 25, "failed": 0,
    "report_path": ".sdd/reports/test-2026-03-29.json"
  },
  "drift_check": "clean",
  "git_diff": "...",
  "failed_attempts": [
    { "attempt": 1, "error": "bcrypt cost 10 구현 → 테스트 실패" },
    { "attempt": 2, "error": "토큰 만료 24h → 7일 기준 실패" }
  ],
  "resolution": "spec/auth.md#REQ-003 재참조 후 수정"
}
```

**거짓말 안 한다는 것은 선언이 아니라 구조다:**

- 테스트 결과는 AI가 작성하지 않는다 — Docker 컨테이너가 생성한 JSON
- spec-tc 생성 테스트는 읽기 전용 — AI가 "항상 pass"하는 테스트를 만들 수 없다
- 실패 시도가 있으면 포함한다 (숨기지 않음)
- git diff는 실제 변경 내역

이 구조가 Anthropic의 Alignment Faking 연구(2024년 12월, arxiv:2412.14093)가 발견한 문제에 대한 답이다. 연구는 AI가 안전하게 행동하는 것처럼 보여도 내부 목표가 다를 수 있음을 실험으로 실증했다:

> "A sophisticated enough model might 'play along', pretending to be aligned — only later revealing that its original preferences remain."

행동 기반 증거(observable behavior)가 선언 기반 신뢰보다 중요하다. 증명 패키지는 AI의 실제 행동 흔적을 기록한다. 에이전트가 무엇을 했는지 증거가 남는다.

**증명 패키지는 PR에 자동 첨부된다.** 사람은 PR 코멘트에서 Slop Score, 테스트 결과, drift 체크, 실패 시도를 한눈에 확인하고 수락/거절한다.

---

## 하네스 설계 원칙

### 1. Rule-based 먼저, LLM은 필요할 때만

| 작업 | 방식 | 이유 |
|------|------|------|
| Slop Score 1차 | 정규식 | 재현 가능, 비용 없음 |
| Drift 감지 | @impl 태그 정적 분석 | 결정론적 |
| EARS → TC 변환 | LLM | 패턴 파싱 + 언어 생성 필요 |
| S_WHY 분석 | LLM (`--with-llm`) | 맥락 판단 필요 |

"LLM은 판단이 필요한 곳에만. 규칙으로 되는 건 규칙으로."

### 2. 상태는 파일로

모든 중간 결과를 `.sdd/` 디렉토리에 저장. 프로세스가 죽어도 재시작하면 이어진다. 메모리에 상태를 보관하지 않는다.

### 3. 하드 게이트, 자동 수정 없음

Slop ≥ 70 → 하네스가 파이프라인 차단. AI가 자동으로 스펙을 "고쳐서" 재시도하지 않는다. AI가 의도를 해석해서 스펙을 바꾸는 순간, 원래의 의도가 유실된다.

### 4. 플랫폼 독립 — git이 foundation, CI는 어댑터

24/7 운영을 위해 CI/CD를 고려하는 건 맞다. 그러나 특정 플랫폼(GitHub Actions, GitLab CI)에 하네스의 핵심을 묶으면, 플랫폼을 바꿀 때 하네스도 다시 만들어야 한다.

```
Foundation (git):
  .git/hooks/pre-push     → spec-validator (로컬에서도 동작)
  .git/hooks/post-merge   → drift-detector
  .git/hooks/post-commit  → spec-tc 생성

Core (sh 스크립트 — 어디서든 실행 가능):
  scripts/run-slop.sh
  scripts/run-spec-tc.sh
  scripts/run-drift.sh
  scripts/run-weekly.sh

Adapter (CI 플랫폼 — 교체 가능):
  .github/workflows/sdd.yml
  .gitlab-ci.yml
  Makefile
```

핵심 규칙: **`scripts/run-*.sh`만 있으면 어느 환경에서든 돌아가야 한다.**

git은 괜찮다. git은 특정 회사에 종속된 플랫폼이 아니다. 또한 MCP 공식 서버 목록에 Git 서버가 포함되어 있다 — git은 에이전트 컨텍스트의 기반으로 업계가 인정한 foundation이다.

---

## RAG와 그래프DB는 언제 필요한가

"스펙 품질 게이트는 RAG의 대안이 아니다 — RAG 전에 오는 품질 필터다."

Slop이 섞인 스펙이 RAG 인덱스에 들어가면, 정확한 검색이 틀린 내용을 더 잘 찾아주는 결과가 된다. 스펙 게이트를 통과한 스펙만 인덱스에 들어가면, RAG 정확도도 올라간다.

보안 관점에서도 같은 결론이 나온다. 2026년 3월 발표된 SoK 논문(arXiv:2603.22928)은 LLM + 도구 + RAG + 자율 에이전트의 공격 표면을 체계적으로 분석했다. 핵심 발견 중 하나: **RAG는 새로운 공격 벡터를 만든다.** "RAG index poisoning" — 소수의 오염된 문서를 인덱스에 심으면 에이전트의 답변이 조작된다. 스펙 품질 게이트가 없으면 RAG 인덱스에 Slop이 섞이고, 그 Slop이 더 정교하게 검색되어 모델에게 전달된다.

| 도구 | 해결하는 문제 | 전제 조건 |
|------|------------|---------|
| RAG | 방대한 문서에서 관련 컨텍스트 검색 | 인덱스의 문서가 품질 게이트를 통과했을 것 |
| 그래프DB | 복잡한 의존성/관계 추적 | 각 노드의 스펙이 명확할 것 |
| **스펙 품질 게이트** | **모델 입력 자체의 품질 보장** | — (전제 조건 없음) |

스펙 품질은 다른 모든 것의 upstream이다. RAG가 있어도, 그래프DB가 있어도 — 모델에게 넣는 원본 입력이 나쁘면 나머지는 그 나쁜 입력을 더 정교하게 전달할 뿐이다.

---

## 신뢰는 설계된다

증명 패키지가 반복적으로 쌓이면 신뢰 히스토리가 생긴다:

```json
{
  "trust_history": {
    "auth 스펙 구현":    { "success_rate": "12/12", "auto_approve": true },
    "DB 스키마 변경":    { "success_rate": "3/5",   "auto_approve": false },
    "신규 외부 API 연동": { "success_rate": "1/3",   "auto_approve": false }
  }
}
```

반복적으로 증명된 패턴은 자동 승인. 새롭거나 실패 이력이 있는 작업은 여전히 사람이 검토.

그러나 신뢰 히스토리도 외부 검증이 필요하다. **성공률을 믿기 때문에 자동 승인을 믿는다**는 순환을 끊으려면:

- **주기적 Spot Audit**: 자동 승인된 작업 중 일부를 사람이 랜덤으로 재검토
- **경계 정기 리뷰**: 사전 승인된 샌드박스 경계도 정기적으로 사람이 검토 (AI가 변경 불가)
- **이상 패턴 감지**: 성공률이 갑자기 높아지면 오히려 의심해야 한다

**이것이 AI 자율성이 실제로 확장되는 방식이다.** 처음부터 전권을 주는 게 아니라, 증명을 통해 신뢰가 쌓이면서 자율 범위가 점진적으로 넓어진다. 사람은 점점 더 적은 것을 검토하게 되고, 그것이 부주의가 아니라 **근거 있는 위임**이 된다.

---

## 자주 묻는 질문

### "레거시 코드베이스에는 어떻게 도입하나?"

새 프로젝트처럼 스펙부터 완성할 필요 없다. 점진적 도입:

1. **Drift Detector 먼저**: 기존 코드가 있으면, 코드에서 역으로 스펙을 만들고 drift 감지부터 시작
2. **새 기능부터 게이트 적용**: 레거시는 건드리지 않고, 신규 개발에만 Slop 게이트 적용
3. **EARS는 한 REQ씩**: 전체 스펙을 한 번에 바꾸지 않는다. PR 단위로 조금씩

레거시 코드베이스에서 가장 중요한 첫 단계는 "지금 코드와 스펙(있다면)이 얼마나 다른가"를 측정하는 것이다.

### "소규모/1인 팀에서 overhead 아닌가?"

솔직하게: 팀이 작을수록 초기 설정 비용 대비 효과를 느끼는 시점이 늦다.

1인 팀에서 하네스의 진짜 가치는 **AI 에이전트를 24시간 돌릴 때** 나온다. AI가 작업하는 동안 사람이 다른 일을 할 수 있고, 돌아와서 증명 패키지만 확인하면 된다. AI를 "비동기 동료"로 쓰는 것이다.

---

## 구현 구조 (경량)

```
sdd-harness/
├── scripts/                      # 핵심 (어디서든 sh로 실행)
│   ├── run-slop.sh
│   ├── run-spec-tc.sh
│   ├── run-drift.sh
│   └── run-weekly.sh
├── hooks/
│   └── install.sh                # git hooks 설치
├── adapters/                     # CI 어댑터 (선택)
│   ├── github-actions.yml
│   └── gitlab-ci.yml
└── .sdd/
    ├── settings/score-weights.json
    └── reports/
```

전체 파일: ~12개. 추가 런타임 의존성: 없음 (Node.js만).  
LLM 호출: Slop Score당 0~1회 (기본 0회).  
CI 없는 환경: `hooks/install.sh` 한 번 실행하면 git hooks로 동일하게 작동.

---

## 참고 자료

| 자료 | 기관 | 연도 | 우리 글과의 연결 |
|------|------|------|----------------|
| [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) | Anthropic | 2024 | Workflow vs Agent, 단순 조합 패턴 |
| [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) | Anthropic | 2025 | allowedTools 기반 샌드박스 구현 |
| [Computer Use Tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool) | Anthropic | 2025 | Minimal privilege + allowlist 공식 권고 |
| [Alignment Faking in LLMs](https://www.anthropic.com/research/alignment-faking) | Anthropic + Redwood | 2024.12 | 신뢰는 선언 아닌 구조의 실증 |
| [MCP Sampling / Human-in-the-loop](https://modelcontextprotocol.io/specification/2025-06-18/client/sampling) | MCP / Anthropic | 2025 | 사전 승인 게이트 공식 스펙 |
| [MCP Architecture](https://modelcontextprotocol.io/docs/concepts/architecture) | MCP / Anthropic | 2024-25 | 에이전트 컨텍스트 레이어 구조 |
| [Practices for Governing Agentic AI](https://openai.com/index/practices-for-governing-agentic-ai-systems/) | OpenAI | 2024 | 자율 에이전트 안전 운영 baseline |
| [Introducing smolagents](https://huggingface.co/blog/smolagents) | HuggingFace | 2025 | Agency 스펙트럼, Minimal Footprint |
| [Continuous Integration](https://martinfowler.com/articles/continuousIntegration.html) | Martin Fowler | 2001/2023 | controlled repo + 빠른 통합 에러 발견 |
| [Design by Contract](https://www.eiffel.com/values/design-by-contract/introduction/) | Bertrand Meyer (Eiffel 설계자, ACM Fellow) | 1992~ | precondition/postcondition = 스펙 품질 학술 기반 |
| [EARS](https://ieeexplore.ieee.org/document/5328509) | Mavin, Rolls-Royce / IEEE | 2009 | 기계 검증 가능한 요구사항 구조화 |
| [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) | IETF | 1997 | MUST/SHOULD/MAY 강도 분류 |
| [Gherkin Reference](https://cucumber.io/docs/gherkin/reference/) | Cucumber | 2008~ | Given/When/Then = AI-friendly 실행 가능 스펙 |
| [SoK: The Attack Surface of Agentic AI](https://arxiv.org/abs/2603.22928) | Dehghantanha, Homayoun | 2026.03 | RAG poisoning + 샌드박스 방어의 보안 근거 |

---

## 요점

우리가 스펙 검증을 만든 이유:  
**AI가 올바른 판단을 하려면 올바른 입력이 필요하기 때문이다.**

하네스는:
- 입력 품질을 자동으로 보장하고 (스펙 게이트)
- AI가 사람 개입 없이 완전한 사이클을 돌리고 (사전 승인 샌드박스)
- 증거를 제출해서 신뢰를 쌓는 (증명 패키지)

구조다. 단순한 자동화가 아니라 **신뢰를 설계하는 시스템**이다.

RAG도, 그래프DB도, 복잡한 오케스트레이터도 필요할 수 있다.  
그것들은 **스펙이 좋고, 환경이 신뢰할 수 있다는 전제** 위에 올라가는 것들이다.

신뢰는 선언하는 것이 아니라 구조로 설계하는 것이다.

**모델 + 하네스. 나머지는 그 다음이다.**

---

## 다음 편에서

하네스가 24/7 자율적으로 돌아간다면, 우리는 그 결과를 어떻게 신뢰하는가?

AI가 "안전하다"고 선언하는 것만으로는 부족한 이유. 행동 기반 증거(증명 패키지)와 점진적 신뢰 확장 구조.

→ **4편: 신뢰는 설계된다**

---

*리서치 협업: Dev (자료 수집 15건) / QA (논리 검증 14건 이슈 제기 및 반론 정리)*  
*관련 시리즈: [스펙은 코드다](/2026-03-20-spec-is-code) / [Slop Detector 개발기 1~4편](/tags/slop)*
