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
> 이번 편에서는 n=30으로 확장하고, Saleor라는 다른 코드베이스에서도 측정했다.

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

커밋 선택 기준: 프로덕션 Python 파일 1~4개 변경, 전체 2~7개, merge 커밋 제외.

---

## 결과 1: Zulip n=30 전체 방식 비교

| 방식 | F1 | Precision | Recall | F1>0 케이스 |
|------|-----|-----------|--------|------------|
| **v3** (ripgrep + AI) | **0.345** | 0.313 | 0.461 | **15/30** |
| v4 (+ 도메인 지식) | 0.269 | 0.218 | 0.394 | 13/30 |
| RAG (벡터) | 0.196 | 0.191 | 0.267 | 11/30 |
| AI 단독 | 0.141 | 0.148 | 0.144 | 6/30 |
| C-det (결정론 그래프) | 0.102 | 0.078 | 0.172 | 6/30 |

### 핵심 발견 ①: v3가 가장 높다, 하지만 30개의 절반은 실패

v3 F1=0.345이지만 30개 중 15개(50%)가 F1=0이다. 잘 되는 케이스와 전혀 안 되는 케이스가 명확히 갈린다.

**v3가 잘 되는 패턴:**
```
"mattermost_importer: Handle export output from newer mmctl"
→ zerver/data_import/mattermost.py  (F1=1.0)
이유: 커밋 메시지의 "mattermost" → 파일명 직결
```

**v3가 실패하는 패턴:**
```
"streams: Allow read access for archived channels"
→ version.py, zerver/views/streams.py  (F1=0)
이유: "version.py"를 예측할 단서가 커밋 메시지에 없음
```

### 핵심 발견 ②: v4가 v3보다 낮다 (0.269 < 0.345)

도메인 지식(파일별 docstring)을 주입하면 오히려 성능이 떨어진다. 이 패턴은 n=10에서도 관찰됐고(v3 0.430 > v4 0.355), n=30에서도 재현됐다.

원인: Zulip은 공개 프로젝트라 Claude가 내부 구조를 이미 알고 있다. 우리가 제공한 docstring이 AI의 기존 지식과 충돌하거나 노이즈로 작용한다.

**통제 실험 결과 (1편):**
- AI 단독 (힌트 없음): F1=0.204
- AI 단독 (Zulip 구조 힌트 추가): F1=0.211

힌트 유무 차이가 0.007. "더 많은 컨텍스트 = 더 좋은 성능"이 성립하지 않는 케이스다.

### 핵심 발견 ③: C-det (결정론 그래프)이 가장 낮다

C-det F1=0.102, 6/30만 성공. import 관계 그래프에서 연결 수가 높은 파일이 "수정할 파일"과 일치하지 않는다.

```
연결 수 높은 파일 = 많이 import되는 파일 = 중요한 인프라 파일
수정할 파일 = 기능과 직접 관련된 파일
```

이 두 집합은 거의 겹치지 않는다. 그래프 구조 자체는 유용하지만, 파일 예측 태스크에서는 ripgrep + AI보다 낮다.

---

## 결과 2: 코드베이스 비교 (Zulip vs Saleor)

| 방식 | Zulip n=30 | Saleor n=26 | 배율 |
|------|-----------|------------|------|
| v3 (ripgrep + AI) | **0.345** | **0.146** | 2.4× |
| AI 단독 (파일목록 제공) | 0.141 | 0.054 | 2.6× |
| AI 순수 (파일목록 없이) | - | 0.026 | - |

### Saleor에서 성능이 크게 떨어지는 이유

**① 파일명과 기능의 비직관적 연결**

Saleor는 GraphQL 중심 구조다. 비즈니스 로직이 `mutations/`, `types.py`, `resolvers.py` 등 generic한 파일에 분산된다.

```
커밋: "Make refundSettings field nullable on RefundSettingsUpdate"
정답: saleor/graphql/shop/mutations/refund_settings_update.py

커밋: "Migrate shipping methods to checkout deliveries"
정답: saleor/celeryconf.py  ← ripgrep이 예측하기 어려움
      saleor/checkout/migrations/...
```

`celeryconf.py`는 커밋 메시지의 어떤 키워드와도 연결되지 않는다.

**② 코드베이스 규모 대비 파일당 책임 분산**

Saleor Python 4232개 파일 중 비슷한 이름의 파일이 많다. ripgrep으로 "shipping"을 검색하면 수십 개의 파일이 나오고, AI가 그 중 정답을 고르기 어렵다.

**③ AI 내부 지식 차이**

- AI 단독 (파일목록 제공): Zulip 0.141, Saleor 0.054
- AI 순수 (파일목록 없이): Saleor 0.026

AI 순수 방식에서 Saleor F1=0.026은 사실상 추측 수준이다. Zulip은 공개 프로젝트로 Claude 학습 데이터에 포함됐을 가능성이 높지만, Saleor는 그 정도가 낮거나 GraphQL 구조가 더 복잡해서 예측이 어렵다.

---

## n=10 vs n=30 수치 변화

| 방식 | n=10 | n=30 | 변화 | 해석 |
|------|------|------|------|------|
| v3 | 0.430 | **0.345** | ↓ −0.085 | n=10이 유리한 케이스에 편향 |
| v4 | 0.355 | **0.269** | ↓ −0.086 | 동일 패턴 재현 |
| C-det | 0.292 | **0.102** | ↓ −0.190 | 가장 큰 하락, n=10 측정이 부정확했음 |
| RAG | 0.125 | **0.196** | ↑ +0.071 | n이 늘수록 안정화 |
| AI 단독 | 0.204 | **0.141** | ↓ −0.063 | n=10 과대평가 |

**C-det의 0.292 → 0.102 하락이 가장 크다.** n=10에서 운 좋게 연결 수가 높은 파일이 정답과 겹쳤던 것으로 보인다. n=30에서는 그래프 방식이 파일 예측 태스크에 적합하지 않음이 명확해졌다.

---

## 데이터 신뢰도 평가

**지지되는 주장:**
- v3(ripgrep + AI)가 n=30 기준 F1=0.345로 가장 높다 ✓
- 도메인 지식 주입(v4)이 v3보다 낮다 — n=10, n=30 모두에서 재현 ✓
- 코드베이스 구조에 따라 성능이 2배 이상 차이난다 ✓
- C-det(그래프)이 파일 예측 태스크에서 약하다 — n=30에서 명확 ✓

**여전히 한계:**
- n=30도 통계적으로 작다 (±0.05 오차 범위)
- Saleor RAG 미측정 — Chroma 인덱스 미구축
- 탐색 태스크(구조 이해) F1 미측정 — 파일 예측만
- 커밋 선택 기준이 특정 유형에 편향될 수 있음

---

## 방식별 결론

| 방식 | Zulip F1 | Saleor F1 | 적합한 상황 |
|------|---------|----------|------------|
| v3 (ripgrep + AI) | 0.345 | 0.146 | 파일명이 기능과 직결될 때 |
| v4 (+ 도메인 지식) | 0.269 | - | 공개 프로젝트에서는 역효과 |
| RAG (벡터) | 0.196 | - | 언어 경계 넘기, 의미 기반 탐색 |
| AI 단독 | 0.141 | 0.054 | 도구 없을 때 하한선 |
| C-det (그래프) | 0.102 | - | 파일 예측보다 구조 탐색에 적합 |

**코드베이스 구조에 따라 최적 방식이 달라진다.** Zulip 같이 파일명이 직관적인 코드베이스는 v3가 효과적이다. Saleor처럼 GraphQL 구조가 분산된 코드베이스에서는 모든 방식이 낮아진다.

---

## 다음 단계

남은 논리적 빈틈:

1. **Saleor RAG 측정** — Chroma 인덱스 구축 후 Zulip과 비교
2. **탐색 태스크 정량화** — "이 코드가 어떻게 동작하나?" F1 설계
3. **n 확대** — 50개 이상이어야 신뢰도 주장 가능
4. **세 번째 코드베이스** — odoo, netbox 등 다른 유형

---

*실험 환경: Raspberry Pi 4B × 2, cc-proxy (Claude API)*  
*데이터: Zulip n=30, Saleor n=26*  
*증적: [teddy-team-sync/context/experiments/n30-comparison](https://github.com/leth2/teddy-team-sync/tree/main/context/experiments/n30-comparison)*
