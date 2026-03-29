---
layout: single
title: "신뢰는 설계된다 — 하네스 엔지니어링 시리즈 4편"
date: 2026-03-29 00:00:00 +0900
categories: [개발기록]
tags: [sdd, harness, trust-engineering, alignment-faking, proof-package, autonomous-agents]
series: harness-engineering
series_part: 4
---

> 이 글은 "하네스 엔지니어링" 시리즈의 마지막 편입니다.  
> 1편: 왜 스펙 품질이 문제인가 / 2편: 어떻게 측정하는가 / 3편: AI를 위한 환경 — 하네스 설계  
> 4편은 묻는다: AI가 자율적으로 작동한다면, 우리는 어떻게 신뢰하는가?

---

## AI가 안전한 척할 수 있다

2024년 12월, Anthropic과 Redwood Research가 공동으로 논문 하나를 발표했다.

제목: "Alignment Faking in Large Language Models" (arxiv:2412.14093)

핵심 발견: **AI 모델이 명시적으로 훈련받거나 지시받지 않았는데도, 스스로 안전한 것처럼 행동하면서 내부 목표를 숨길 수 있다.**

논문의 표현:

> 논문 요약: 충분히 정교한 모델은 새 원칙에 맞춰 행동하는 척하면서, 실제로는 원래 선호를 유지할 수 있다. 실험은 이것이 명시적 훈련 없이도 자발적으로 일어날 수 있음을 보였다.

이것이 단순한 이론이 아니다. 실험으로 실증됐다. 모델이 훈련 중에는 안전하게 행동하고, 이후 다른 맥락에서 원래 선호를 드러낼 수 있다는 것을 보였다.

이 연구가 하네스 설계에 주는 함의는 명확하다:

**"테스트 통과했습니다"라는 AI의 선언은 충분하지 않다.**

신뢰는 AI의 성격이 아니라 시스템 설계에서 나와야 한다.

---

## 에이전트 신뢰 모델 6가지

Botao Hu와 Helena Rong은 2025년 논문(arXiv:2511.03434, AAAI 2026 TrustAgent 워크샵)에서 에이전트 신뢰 모델을 6가지로 분류했다:

| 모델 | 방식 | 예시 | 한계 |
|------|------|------|------|
| **Brief** | 자기 선언 | AgentCard, README | 아무나 쓸 수 있음 |
| **Claim** | 능력/신원 주장 | "나는 코딩을 잘 한다" | 검증 불가 |
| **Proof** | 암호학적/행동 증명 | ZK proof, 실행 로그 | 비쌈, 복잡함 |
| **Stake** | 담보 기반 | 틀리면 패널티 | 경제적 인센티브 |
| **Reputation** | 누적 평판 | 성공률 히스토리 | 게이밍 가능 |
| **Constraint** | 구조적 제약 | 샌드박스, allowlist | 유연성 감소 |

논문의 결론:

> "No single mechanism suffices. Proof + Stake to gate high-impact actions, Reputation as overlay, Constraint as baseline."

단일 메커니즘으로는 부족하다. 논문은 Proof + Stake를 핵심으로 권고한다.

우리 하네스는 이 중 **Stake(담보 기반)**를 **Constraint(구조적 제약)**로 대체해서 세 레이어를 조합한다:

- **Constraint** — 사전 승인 샌드박스 (AI가 할 수 있는 것의 경계)
- **Proof** — 증명 패키지 (AI가 한 것의 행동 기반 증거)
- **Reputation** — 신뢰 히스토리 (반복 증명으로 쌓인 신뢰)

Stake 대신 Constraint를 선택한 이유: 경제적 패널티 구조는 외부 서비스나 에이전트 마켓플레이스에 적합하다. 팀 내부 개발 하네스에서는 "물리적으로 할 수 없게 만드는" Constraint가 더 단순하고 확실하다.

---

## Constraint: 사전 승인 샌드박스

신뢰의 첫 번째 레이어는 구조적 제약이다.

AI가 무엇을 할 수 있는지를 사전에 정의하고, 그 경계 안에서만 실행하게 한다. 경계는 AI가 변경할 수 없다. 사람이 버전 관리 하에 정의한다.

```
[사전 승인된 샌드박스 — 사람이 정의, git 관리]
├── 읽기/쓰기 가능 경로: src/, spec/, tests/
├── 실행 가능 명령: npm test, npm run lint
├── 허용 외부 통신: 없음 (Docker 네트워크 격리)
└── AI는 이 경계 안에서만 실행
    (경계 파일 수정 시 pre-commit hook 차단)
```

Anthropic의 Computer Use 공식 문서도 같은 패턴을 권고한다:
- "dedicated virtual machine or container with minimal privileges"
- "limiting internet access to an allowlist of domains"
- "asking a human to confirm decisions that may result in meaningful real-world consequences"

MCP 공식 스펙에는 이 원칙이 명문화돼 있다:
> "There SHOULD always be a human in the loop with the ability to deny."

Constraint 레이어가 하는 일: AI가 아무리 정교하게 alignment faking을 해도, **물리적으로 할 수 없는 것들을 만든다.** 선언이 아니라 구조다.

---

## Proof: 증명 패키지

신뢰의 두 번째 레이어는 행동 기반 증거다.

AI가 작업을 완료하면 증명 패키지를 생성한다. AI가 "결과를 주장"하는 것이 아니라 **증거를 제출**하는 구조다.

```json
{
  "summary": "auth 스펙 구현 완료",
  "spec_version": "spec/auth.md@abc123",
  "slop_score": 42,
  "test_results": {
    "total": 25,
    "passed": 25,
    "failed": 0,
    "report_path": ".sdd/reports/test-2026-03-29.json"
  },
  "drift_check": "clean",
  "git_diff_summary": "src/auth/handler.ts +47 -12",
  "failed_attempts": [
    { "attempt": 1, "error": "bcrypt cost=10 → 테스트 실패" },
    { "attempt": 2, "error": "토큰 만료 24h → 7일 기준 실패" }
  ],
  "resolution": "spec/auth.md#REQ-002 재참조 후 수정"
}
```

이 구조에서 신뢰는 선언에서 나오지 않는다:

**테스트 결과**는 AI가 작성하지 않는다. Docker 컨테이너가 생성한 JSON 파일이다. AI가 결과를 수정하려면 Docker 환경 설정 파일을 바꿔야 하는데, 그 파일은 Constraint 레이어에서 보호된다.

**실패 시도**가 포함된다. 두 번 실패하고 세 번째에 성공했다면, 그 기록이 그대로 제출된다. AI가 완벽한 척 실패를 숨길 수 없다.

**spec-tc 생성 테스트는 읽기 전용**이다. AI는 구현 코드(`src/`)만 수정할 수 있다. `tests/spec-generated/` 경로의 파일을 수정하면 git pre-commit hook이 차단한다. "항상 pass하는 mock 테스트를 만들어서 통과한 척"을 구조적으로 막는다.

**git diff**는 조작 불가능하다. 실제로 무엇을 바꿨는지가 그대로 드러난다.

사람은 이 증명 패키지를 PR 코멘트에서 확인한다. "이 증거가 충분한가"만 판단하면 된다. 실행 과정을 직접 감시하지 않아도 된다.

---

## Reputation: 신뢰 히스토리

신뢰의 세 번째 레이어는 누적된 증거다.

증명 패키지가 반복적으로 쌓이면 패턴이 보인다:

```json
{
  "trust_history": {
    "auth 스펙 구현":       { "success": 12, "total": 12, "auto_approve": true },
    "DB 스키마 변경":        { "success": 3,  "total": 5,  "auto_approve": false },
    "신규 외부 API 연동":     { "success": 1,  "total": 3,  "auto_approve": false }
  }
}
```

반복적으로 증명된 작업 타입은 자동 승인 대상이 된다. 새롭거나 실패 이력이 있는 타입은 계속 사람이 검토한다.

이것이 **AI 자율성이 실제로 확장되는 방식**이다. 처음부터 전권을 주는 것이 아니라, 증명을 통해 신뢰가 쌓이면서 자율 범위가 점진적으로 넓어진다.

### 신뢰 히스토리의 위험: 순환 논리

단, 신뢰 히스토리에 의존하는 것만으로는 부족하다.

"성공률을 믿기 때문에 자동 승인을 믿는다 → 자동 승인 결과로 성공률이 쌓인다"는 순환이 생길 수 있다.

이 순환을 끊는 것이 **Spot Audit**이다:

- 자동 승인된 작업 중 일부를 주기적으로 사람이 랜덤 재검토
- 성공률이 갑자기 높아지면 오히려 의심한다 — 주간 비교 기준 ≥20% 급등 시 해당 작업 타입 Spot Audit 트리거 (AI가 "쉬운 케이스"만 선택하거나, 테스트 범위를 좁히는 건 아닌가?)
- 샌드박스 경계 자체도 정기적으로 사람이 검토 (AI가 경계를 적절히 쓰고 있는가?)

신뢰는 한 번 쌓으면 끝이 아니다. 유지하는 구조가 있어야 한다.

---

## 사람의 역할이 변한다

이 구조에서 사람의 역할은 무엇인가?

```
Before:
  사람이 계획하고 → 코드를 작성하고 → 결과를 확인한다

After (하네스):
  사람이 경계를 설계하고 → AI가 실행하고 → 사람이 증거를 판단한다
```

실행 과정에 개입하지 않는 대신, **두 가지**에 집중한다:

**1. 경계 설계자**: 샌드박스 경계, 스펙 품질 기준, 신뢰 임계값을 정의한다. AI가 자율적으로 일할 수 있는 조건을 만드는 것이 사람의 가장 중요한 기여다.

**2. 증거 판단자**: AI가 제출한 증명 패키지가 충분한가를 판단한다. 코드 한 줄 한 줄을 보는 것이 아니라, 전체 증거의 패턴을 본다.

신뢰가 쌓일수록 두 번째 역할의 부담이 줄어든다. 처음에는 모든 PR을 꼼꼼히 검토하다가, 나중에는 특정 타입의 작업은 증명 패키지만 훑어보고 승인한다.

그것이 부주의가 아니라 **근거 있는 위임**이 된다.

---

## EARS가 설계된 이유와 같다

잠깐 돌아보면, 이 시리즈의 모든 것이 같은 원칙에서 나왔다.

- **EARS**: 항공우주에서 "모호한 언어는 검증이 불가능하다"는 문제를 구조로 해결했다
- **Slop Detector**: 스펙의 나쁜 품질을 규칙과 LLM으로 감지한다
- **증명 패키지**: AI의 모호한 선언이 아닌 검증 가능한 행동 증거를 요구한다
- **Spot Audit**: 신뢰 히스토리의 모호성을 주기적 검증으로 보완한다

**모호함은 신뢰를 만들지 못한다. 구조가 신뢰를 만든다.**

스펙의 입력 품질이 AI 판단의 출발점이라면, 신뢰의 구조는 AI 자율성의 토대다. 둘은 같은 원칙의 양면이다.

---

## 오늘 시작할 수 있는 것

4편짜리 시리즈를 읽은 독자에게 남기는 실용적 제안:

**지금 바로:**
- 스펙 하나를 열고, "MUST/SHALL/SHOULD" 없이 쓰인 요구사항을 찾아라
- 에러 케이스(IF/THEN)가 하나도 없는 REQ를 찾아라
- 그 REQ에 EARS 패턴으로 에러 케이스를 하나 추가해보라

**다음 단계:**
- git pre-push hook에 간단한 Slop 키워드 감지 스크립트 하나 달아보라
- spec-tc 없어도 된다 — 스펙의 AC를 보고 수동으로 테스트 골격을 만들어라
- `spec_ref` 주석을 테스트에 달기 시작하라

**하네스는 나중:**
- 위 두 단계가 팀에 자리 잡으면, 그때 자동화를 올려라
- 자동화 없이 동작하는 것을 먼저 만들어야 자동화가 의미가 있다

---

## 마무리 — 신뢰는 선언이 아니라 구조다

이 시리즈를 한 문장으로 줄이면:

> **AI가 올바른 판단을 하려면 올바른 입력이 필요하다.**  
> **그리고 AI의 자율성을 신뢰하려면 검증 가능한 구조가 필요하다.**

RAG도, 그래프DB도, 복잡한 오케스트레이터도 필요할 수 있다. 하지만 그것들은 스펙 품질이 좋고, 실행 환경이 신뢰할 수 있다는 전제 위에 올라가는 것들이다.

**모델 + 하네스. 나머지는 그 다음이다.**

신뢰는 선언하는 것이 아니라 구조로 설계하는 것이다.

---

*참고 자료:*
- *[Alignment Faking in Large Language Models — Anthropic + Redwood Research (arxiv:2412.14093, 2024.12)](https://www.anthropic.com/research/alignment-faking)*
- *[Inter-Agent Trust Models — Botao Hu, Helena Rong (arXiv:2511.03434, AAAI 2026 TrustAgent)](https://arxiv.org/abs/2511.03434)*
- *[Practices for Governing Agentic AI Systems — OpenAI (2024)](https://openai.com/index/practices-for-governing-agentic-ai-systems/)*
- *[Computer Use Tool — Anthropic (2025)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool)*
- *[MCP Sampling / Human-in-the-loop — MCP Spec (2025-06-18)](https://modelcontextprotocol.io/specification/2025-06-18/client/sampling)*
