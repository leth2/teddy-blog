---
layout: single
title: "엔터프라이즈 코드베이스 AI 검색 — n=30 전체 방식 비교, 그리고 코드베이스가 달라지면"
date: 2026-03-31 00:00:00 +0900
categories: [개발기록]
tags: [enterprise, rag, ripgrep, graph-db, code-search, quantitative, saleor, zulip]
series: enterprise-ai-engineering
series_part: 2
---

> 1편에서는 Zulip n=10으로 5가지 방식을 비교했다. 두 가지 논리적 빈틈이 있었다.  
> 첫째, **n=10은 충분한가?** 둘째, **한 코드베이스 결과를 일반화할 수 있는가?**  
> 이번 편에서 n=30으로 확장하고, Saleor라는 다른 코드베이스에서도 전체 방식을 측정했다.

---

## 관련 연구

이 실험과 직접 비교되는 선행 연구가 있다.

**SWE-bench** (Jimenez et al., ICLR 2024)[^swe]는 GitHub 이슈 텍스트를 보고 "어떤 파일을 수정해야 하나?"를 평가한다. 12개 Python 레포, 2,294개 이슈. 우리와 문제 정의가 동일하고 코드베이스 규모도 비슷하다. 차이는 우리가 이슈 텍스트 대신 커밋 메시지를 쿼리로 쓴다는 것.

**Agentless** (Xia et al., 2024)[^agentless]는 SWE-bench에서 "단순 텍스트 검색 + localization"이 복잡한 에이전트 방식보다 높은 성능(32%)을 보임을 보였다. 우리 실험에서 v3(ripgrep+AI)가 그래프/RAG보다 높게 나온 것과 같은 결론이다. 독립적인 연구에서 같은 패턴이 관찰됐다.

**CodeScout** (Sutawika et al., 2026)[^codescout]는 RL 기반 코드 검색 에이전트로 대형 레포에서 파일 찾기 문제를 다룬다.

우리 연구의 차별점: **5가지 검색 방식 × 2개 코드베이스를 동일 조건에서 F1로 직접 비교**한 예비 실험이다. 기존 연구들은 단일 방식 또는 단일 코드베이스에 집중한다.

---

## 실험 설계

**태스크:** "이 커밋을 수정하려면 어떤 파일?" → 실제 변경 파일과 F1(Precision/Recall 조화평균) 비교

**방식 5종:**

| 이름 | 방식 |
|------|------|
| v3 | ripgrep + 테스트 필터 + AI 선택 |
| v4 | v3 + 도메인 지식(파일별 docstring) 주입 |
| C-det | 그래프(import 관계) + ripgrep, AI 선택 없이 연결 수로 결정 |
| RAG | ChromaDB 벡터 검색 (all-MiniLM-L6-v2) |
| AI 단독 | 도구 없이 파일 목록만 제공 |

**코드베이스 2종:**

| | Zulip | Saleor |
|-|-------|--------|
| 언어 | Python + TypeScript | Python 전용 |
| 규모 | Python 1987개, 202MB | Python 4232개, 96MB |
| 구조 | 기능별 파일명 직관적 | GraphQL mutations/types 반복 |
| 샘플 수 | n=30 | n=26 |

---

## 결과 1: Zulip n=30 전체 방식 비교

| 방식 | F1 | Precision | Recall | F1>0 |
|------|-----|-----------|--------|------|
| **v3** (ripgrep + AI) | **0.345** | 0.313 | 0.461 | 15/30 |
| v4 (+ 도메인 지식) | 0.269 | 0.218 | 0.394 | 13/30 |
| RAG (벡터) | 0.196 | 0.191 | 0.267 | 11/30 |
| AI 단독 | 0.141 | 0.148 | 0.144 | 6/30 |
| C-det (결정론 그래프) | 0.102 | 0.078 | 0.172 | 6/30 |

### 발견 ①: v3가 가장 높다, 하지만 절반은 실패

30개 중 15개(50%)가 F1=0. 잘 되는 케이스와 전혀 안 되는 케이스가 명확히 갈린다.

```
성공: "mattermost_importer: Handle export from newer mmctl"
      → zerver/data_import/mattermost.py  (F1=1.0)
      커밋 키워드가 파일명에 직접 포함

실패: "streams: Allow read access for archived channels"
      → version.py, zerver/views/streams.py  (F1=0)
      version.py를 예측할 단서가 없음
```

### 발견 ②: v4(도메인 지식 주입)가 v3보다 낮다

파일별 docstring을 컨텍스트로 제공하면 오히려 F1이 낮아진다 (0.345 → 0.269). n=10에서도(0.430 → 0.355), n=30에서도 재현됐다.

원인: Zulip은 공개 프로젝트로 Claude가 이미 내부 구조를 알고 있다. 추가 docstring이 AI의 기존 지식과 충돌하거나 노이즈로 작용한다. 통제 실험에서 Zulip 힌트 유무 차이가 F1 0.007에 불과했다.

> **"더 많은 컨텍스트 ≠ 더 좋은 성능"** — Agentless 논문의 결론과 동일하다.

### 발견 ③: C-det이 n=10(0.292)에서 n=30(0.102)으로 크게 하락

그래프에서 연결 수(import degree)가 높은 파일 = 인프라 파일. 수정할 파일 ≠ 중요한 파일. n=10에서 우연히 겹쳤던 것이 n=30에서 드러났다.

---

## 결과 2: 코드베이스 비교 — Zulip vs Saleor

| 방식 | Zulip n=30 | Saleor n=26 | 배율 |
|------|-----------|------------|------|
| v3 | 0.345 | **0.146** | 2.4× |
| RAG | 0.196 | **0.091** | 2.2× |
| AI 단독 | 0.141 | 0.054 | 2.6× |
| AI 순수 (파일목록 없이) | - | 0.026 | - |

**모든 방식에서 Saleor가 Zulip의 절반 이하다.**

### 왜 Saleor에서 성능이 떨어지는가

Saleor는 GraphQL 중심 구조다. 비즈니스 로직이 `mutations/`, `types.py`, `resolvers.py` 같은 generic 파일에 분산된다.

```
커밋: "Migrate shipping methods to checkout deliveries"
정답: saleor/celeryconf.py        ← 키워드 연결 없음
      checkout/migrations/...
```

ripgrep은 "shipping" 키워드로 수십 개 파일을 가져오지만 `celeryconf.py`는 잡지 못한다. RAG도 동일하다: 의미 벡터가 "배송 방법 → 비동기 작업 설정 파일"을 연결하지 못한다.

**RAG Saleor F1=0.091 (6/26 성공)**: Zulip(0.196)보다도 낮다. 코드베이스 구조 의존성이 RAG에서도 재현됐다.

---

## n=10 vs n=30 수치 변화

| 방식 | n=10 | n=30 | 변화 | 해석 |
|------|------|------|------|------|
| v3 | 0.430 | **0.345** | ↓ −0.085 | 유리한 케이스 편향 |
| v4 | 0.355 | **0.269** | ↓ −0.086 | 동일 패턴 재현 |
| C-det | 0.292 | **0.102** | ↓ **−0.190** | 가장 큰 하락 |
| RAG | 0.125 | **0.196** | ↑ +0.071 | n이 늘수록 안정화 |
| AI 단독 | 0.204 | **0.141** | ↓ −0.063 | 과대평가 |

n=10은 전반적으로 과대평가됐다. C-det의 하락이 가장 크다.

---

## 데이터 신뢰도

**지지되는 주장:**
- v3가 n=30 기준 F1=0.345로 가장 높다 ✓
- 도메인 지식 주입(v4)이 v3보다 낮다 — n=10, n=30 모두에서 재현, Agentless 논문과 일치 ✓  
- 코드베이스 구조에 따라 성능이 2배 이상 차이난다 — v3, RAG, AI단독 모두에서 재현 ✓
- C-det이 파일 예측 태스크에서 약하다 — n=30에서 명확 ✓

**여전히 한계:**
- n=30도 통계적으로 작다 (±0.05 오차 범위)
- 탐색 태스크("이 코드 어떻게 동작하나?") F1 미측정
- 커밋 선택 기준이 특정 유형에 편향될 가능성

---

## 전체 결과표

### Zulip n=30

| 방식 | F1 | P | R |
|------|-----|-----|-----|
| v3 | **0.345** | 0.313 | 0.461 |
| v4 | 0.269 | 0.218 | 0.394 |
| RAG | 0.196 | 0.191 | 0.267 |
| AI 단독 | 0.141 | 0.148 | 0.144 |
| C-det | 0.102 | 0.078 | 0.172 |

### Saleor n=26

| 방식 | F1 | P | R |
|------|-----|-----|-----|
| v3 | **0.146** | 0.109 | 0.282 |
| RAG | 0.091 | 0.063 | 0.173 |
| AI 단독 | 0.054 | 0.038 | 0.115 |
| AI 순수 | 0.026 | 0.015 | 0.077 |

---

## 다음 단계

1. **탐색 태스크 F1 설계** — "이 기능 어디 있나?" 정량화 방법
2. **n 확대** — 50개 이상
3. **세 번째 코드베이스** — 다른 구조 유형
4. **코드 특화 임베딩** — CodeBERT로 RAG 개선 시도

---

[^swe]: Jimenez et al., "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?", ICLR 2024. [arXiv:2310.06770](https://arxiv.org/abs/2310.06770)
[^agentless]: Xia et al., "Agentless: Demystifying LLM-based Software Engineering Agents", 2024. [arXiv:2407.01489](https://arxiv.org/abs/2407.01489)
[^codescout]: Sutawika et al., "CodeScout: An Effective Recipe for Reinforcement Learning of Code Search Agents", 2026. [arXiv:2503.xxxxx](https://arxiv.org/search/?query=CodeScout+code+search)

*실험 환경: Raspberry Pi 4B × 2, cc-proxy (Claude API)*  
*데이터: Zulip n=30, Saleor n=26*  
*증적: [teddy-team-sync/context/experiments/n30-comparison](https://github.com/leth2/teddy-team-sync/tree/main/context/experiments/n30-comparison)*
