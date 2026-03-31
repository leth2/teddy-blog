---
layout: single
title: "엔터프라이즈 코드베이스 AI 검색 — n=30 완전 비교 실험"
date: 2026-03-31 00:00:00 +0900
categories: [개발기록]
tags: [enterprise, rag, ripgrep, graph-db, code-search, quantitative, saleor, zulip]
series: enterprise-ai-engineering
series_part: 2
---

> 1편에서 Zulip n=10으로 시작한 실험을 n=30으로 확장하고, Saleor 코드베이스를 추가했다.  
> 이번 편의 목표: **논리의 빈틈 없애기** — 누락된 방식을 채우고, 주장의 근거를 단단히 하기.

---

## 1편 요약 및 빈틈

1편 결과 (Zulip n=10):

| 방식 | F1 |
|------|-----|
| v3 (ripgrep + AI) | 0.430 |
| C-det (결정론 그래프) | 0.292 |
| RAG (Chroma) | 0.125 |
| AI 단독 | 0.204 |

**빈틈으로 지적받은 것들:**
- n=10은 너무 작다. 샘플 편향 가능성
- C-det, v4를 n=30으로 미측정
- 코드베이스가 하나뿐 — Zulip 특이성인지 일반적 패턴인지 모름
- AI 단독의 "Saleor는 어떤가" 미확인

이 모든 것을 채웠다.

---

## 실험 설계

### 두 코드베이스

| | Zulip | Saleor |
|-|-------|--------|
| 언어 | Python + TypeScript | Python 전용 |
| 규모 | 파일 1987개 202MB | 파일 4232개 96MB |
| 구조 | 모듈별 직관적 파일명 | GraphQL mutations/types 반복 |
| n | **30** | **26** (조건 맞는 커밋 한계) |

커밋 선택 기준: 프로덕션 Python 1~4개 변경, 테스트 포함 전체 2~7개

### 측정 방식 5가지 (Zulip 기준)

| 방식 | 설명 |
|------|------|
| **v3** | ripgrep 키워드 검색 → AI 선택 |
| **v4** | v3 + 파일 목적(docstring) 정보 주입 |
| **RAG** | Chroma 벡터 검색 (all-MiniLM-L6-v2) → AI 선택 |
| **AI 단독** | 파일 목록 제공 후 AI 선택 |
| **C-det** | 그래프 구조(연결 수) 기반, AI 없는 결정론적 선택 |

---

## 결과

### Zulip n=30 — 전체 순위

| 순위 | 방식 | F1 | P | R |
|------|------|-----|---|---|
| 1 | v3 (ripgrep + AI) | **0.345** | 0.313 | 0.461 |
| 2 | v4 (도메인지식 추가) | 0.269 | 0.218 | 0.394 |
| 3 | RAG (Chroma) | 0.196 | 0.191 | 0.267 |
| 4 | AI 단독 | 0.141 | 0.148 | 0.144 |
| 5 | C-det (그래프) | 0.102 | 0.078 | 0.172 |

### Saleor n=26 — 전체 순위

| 순위 | 방식 | F1 | P | R |
|------|------|-----|---|---|
| 1 | v3 (ripgrep + AI) | **0.146** | 0.109 | 0.282 |
| 2 | AI 단독 (파일목록) | 0.054 | 0.038 | 0.115 |
| 3 | AI 순수 (파일목록 없음) | 0.026 | 0.015 | 0.077 |

---

## 발견 1: n=10은 편향이 심했다

n=10 → n=30 변화:

| 방식 | n=10 | n=30 | 변화 |
|------|------|------|------|
| v3 | 0.430 | **0.345** | ↓ −0.085 |
| C-det | 0.292 | **0.102** | ↓ −0.190 |
| RAG | 0.125 | **0.196** | ↑ +0.071 |
| AI 단독 | 0.204 | **0.141** | ↓ −0.063 |

**C-det이 가장 심하게 바뀌었다.** 0.292 → 0.102, 64% 하락.

n=10에서 C-det가 잘 됐던 이유: 그래프 연결 수가 높은 파일들이 우연히 정답과 겹치는 케이스가 포함됐기 때문이다. n=30으로 늘리니 "degree가 높은 파일 = 수정할 파일"이라는 전제가 얼마나 틀렸는지 드러났다.

v3는 0.430 → 0.345로 줄었지만 여전히 1위다. RAG는 오히려 올랐다 — n이 늘수록 다양한 쿼리가 포함되고, RAG의 의미론적 강점이 발휘되는 케이스가 늘었다.

---

## 발견 2: "더 많은 컨텍스트 ≠ 더 좋은 성능" — n=30에서도 재확인

v3 0.345 > v4 0.269

1편에서 도메인 지식 주입(v4)이 v3보다 낮다는 것을 관찰했다. n=30에서도 동일하다. 이제 이 결과는 샘플 우연이 아니다.

**왜?**

v4는 각 파일의 목적(docstring 첫 줄)을 AI에게 알려준다:
```
zerver/lib/rate_limiter.py: Rate limiting implementation...
zerver/tornado/event_queue.py: Tornado-based event queue...
```

직관적으로는 이게 도움이 될 것 같다. 그런데 실제로는 AI가 이미 Zulip 코드베이스의 파일 목적을 알고 있다 (공개 프로젝트). 우리가 주입한 docstring이 AI의 내부 지식과 충돌하거나 노이즈로 작용했을 가능성이 높다.

v4가 v3보다 특별히 잘 된 케이스:
```
[19] markdown: Auto-link obsidian:// URLs
v3: F1=0  (키워드 "obsidian"이 없는 코드베이스에서 ripgrep 실패)
v4: F1=1.0 (docstring에서 markdown 처리 파일 식별 성공)
```

v3가 v4보다 잘 된 케이스:
```
[15] integrations: Add thread screenshots for fixtureless integrations
v3: F1=0.571
v4: F1=0 (도메인 컨텍스트가 webhooks 파일들로 오인)
```

v4가 v3보다 좋은 케이스도 있지만, 나쁜 케이스가 더 많다.

---

## 발견 3: C-det의 근본적 한계

C-det F1=0.102. 전체 30 케이스 중 F1>0가 나온 케이스: **6개**.

C-det 아이디어: 그래프에서 많이 연결된 파일(degree 높음)이 중요한 파일이고, 중요한 파일이 수정될 파일이다.

이 전제가 틀렸다.

```
C-det가 자주 고른 파일들:
- zproject/urls.py (URL 라우팅, 거의 모든 경로에서 import)
- zproject/backends.py (인증 백엔드, 연결 수 많음)
- corporate/lib/stripe.py (결제 로직)
```

이 파일들은 코드베이스에서 "중요한" 파일이지만, 특정 버그 수정이나 기능 추가를 위해 수정되는 파일과는 다르다.

**결론:** 그래프 degree는 코드베이스 중요도를 나타내지, 수정 빈도나 변경 위치를 나타내지 않는다.

단, C-det의 강점은 따로 있다: **재현성 1.0**. 같은 쿼리에 항상 같은 결과를 낸다. 이건 디버깅이나 감사(audit)용으로 유용하다.

---

## 발견 4: 코드베이스가 성능을 결정한다

| 방식 | Zulip | Saleor | 배율 |
|------|-------|--------|------|
| v3 | 0.345 | 0.146 | 2.4× |
| AI 단독 | 0.141 | 0.054 | 2.6× |
| AI 순수 | ~0.141* | 0.026 | ~5× |

*Zulip AI 순수 측정값 추정

Zulip에서 AI 단독 F1=0.141인데 Saleor AI 순수(파일목록 없음)는 0.026이다. Zulip은 공개 프로젝트로 Claude가 파일 구조를 학습했을 가능성이 있다. Saleor도 공개지만 AI가 내부 파일 배치를 아는 수준이 낮다.

**Saleor에서 v3가 왜 이렇게 낮은가?**

Saleor 실패 케이스 패턴:
```
[3] "Treat exceptions as expected, avoid spilling"
    정답: saleor/product/tasks.py
    v3 후보: (예외 처리 관련 generic 파일들)

[5] "Migrate shipping methods to checkout deliveries"
    정답: celeryconf.py, migrations/0093_...
    v3 후보: (shipping 키워드가 있는 다른 파일들)
```

Saleor의 `celeryconf.py`에 배송 마이그레이션 태스크가 있다는 것은 코드베이스 구조를 깊이 이해한 사람만 알 수 있다. 키워드 검색으로는 찾을 수 없다.

---

## RAG의 실제 강점: 언어 경계

RAG Zulip n=30: F1=0.196, Recall=0.267

ripgrep 기반 방식(v3)의 Recall=0.461보다 낮지만, RAG가 특별히 잘 하는 케이스가 있다:

```
[18] "email_mirror: Add support for iso-8859-8-i Hebrew encoding"
정답: zerver/lib/email_mirror.py
RAG: F1=1.0 (의미론적 매칭 성공)
v3: F1=0 (ripgrep이 Hebrew 인코딩 관련 파일 못 찾음)
```

커밋 메시지에 `email_mirror`가 직접 언급되어 있어 단순해 보이지만, ripgrep은 Hebrew encoding이라는 구체적 수정 내용으로 잘못된 파일을 가져왔다. RAG는 "email_mirror"라는 개념과 실제 파일을 의미적으로 연결했다.

RAG가 현재 구현(all-MiniLM-L6-v2)으로 F1=0.196인 것은 임베딩 모델이 일반 텍스트 모델이기 때문이다. 코드 특화 임베딩(CodeBERT, jina-embeddings-v2-code)으로 바꾸면 개선 여지가 있다.

---

## 종합 비교 매트릭스

| 방식 | Zulip F1 | Saleor F1 | 재현성 | 코드베이스 민감도 |
|------|---------|----------|--------|-----------------|
| v3 | **0.345** | **0.146** | 중간 | 높음 |
| v4 | 0.269 | - | 중간 | 높음 |
| RAG | 0.196 | - | 높음 | 중간 |
| AI 단독 | 0.141 | 0.054 | 낮음 | 매우 높음 |
| C-det | 0.102 | - | **1.0** | 높음 |

**없는 항목:** Saleor RAG, Saleor C-det — Chroma 인덱스 미구축

---

## 이 데이터는 어디까지 믿을 수 있나

**신뢰할 수 있는 것:**
- **방식 간 상대 순위**: v3 > v4 > RAG > AI단독 > C-det (Zulip 기준)
- **코드베이스 효과**: Saleor에서 모든 방식이 낮아지는 패턴
- **n=10 vs n=30**: C-det처럼 크게 변한 방식 — 소샘플 편향 실증

**주의해야 할 것:**
- n=30은 여전히 작다. ±0.04 수준 오차 상정
- Saleor RAG/C-det 미측정 → Saleor 비교 불완전
- 탐색 태스크(파일 찾기 외) 미측정

**정직한 결론:**  
"n=30, 2개 코드베이스에서 ripgrep+AI(v3)가 가장 안정적이고, 코드베이스 구조가 성능에 결정적 영향을 미친다"는 주장은 이 데이터로 지지된다.

---

## 다음 편 예고

남은 빈틈:
1. Saleor RAG 인덱스 구축 → 3개 방식 Saleor 비교 완성
2. 코드 특화 임베딩(CodeBERT)으로 RAG 개선 실험
3. 탐색 태스크 정량화

---

*실험 환경: Raspberry Pi 4B × 2, cc-proxy (claude-sonnet-4-5)*  
*데이터: Zulip n=30, Saleor n=26*  
*증적: [teddy-team-sync/context/experiments](https://github.com/leth2/teddy-team-sync/tree/main/context/experiments)*
