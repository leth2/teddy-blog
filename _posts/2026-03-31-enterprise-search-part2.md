---
layout: single
title: "엔터프라이즈 코드베이스 AI 검색 — n=30 확장 실험, 그리고 코드베이스가 달라지면"
date: 2026-03-31 00:00:00 +0900
categories: [개발기록]
tags: [enterprise, rag, ripgrep, code-search, quantitative, saleor, zulip]
series: enterprise-ai-engineering
series_part: 2
---

> 1편에서는 Zulip 코드베이스, n=10 커밋으로 실험했다. 오늘은 두 가지 질문에 답한다.  
> 1) **n=10은 믿을 수 있나?** — 샘플을 30개로 늘리면 수치가 어떻게 바뀌는가  
> 2) **코드베이스를 바꾸면 어떻게 되나?** — Zulip에서 Saleor로 옮겼을 때

---

## 이전 편 요약

1편 실험 조건:
- 코드베이스: Zulip (Python 1987개 + TypeScript 459개, 202MB)
- 태스크: "이 커밋 메시지를 수정하려면 어떤 파일을 바꿔야 하나?" (파일 예측 F1)
- n=10 커밋 (이 숫자가 너무 적다는 게 이번 편 출발점)

**1편 n=10 결과 (Zulip):**

| 방식 | F1 | 설명 |
|------|-----|------|
| v3 (ripgrep + 테스트 필터 + AI) | 0.430 | 최고 |
| v4 (도메인 지식 주입) | 0.355 | v3보다 낮음 |
| C-det (결정론 그래프) | 0.292 | |
| A3v2 (ripgrep + AST) | 0.266 | |
| RAG (Chroma, all-MiniLM) | 0.125 | |
| AI 단독 | 0.204 | 베이스라인 |

---

## 실험 1: n=10 → n=30 (Zulip)

### 방법

기존 10개 커밋에 20개를 추가했다. 선택 기준은 동일하다:
- 프로덕션 Python 파일 1~4개 변경
- 전체 Python 파일 2~7개 (테스트 포함)
- merge 커밋 제외

### 결과

| 방식 | n=10 F1 | n=30 F1 | 변화 |
|------|---------|---------|------|
| v3 (ripgrep + AI) | 0.430 | **0.345** | ↓ −0.085 |
| RAG (Chroma) | 0.125 | **0.196** | ↑ +0.071 |
| AI 단독 | 0.204 | **0.141** | ↓ −0.063 |

n=10 때 v3가 0.430으로 가장 높았던 건 **유리한 커밋이 많이 포함됐기 때문**이었다.  
n=30에서는 0.345로 내려왔고, RAG는 0.125 → 0.196으로 올라왔다.

### 케이스별 분포

v3 n=30 케이스 분포:

```
F1=1.0 : 5개  (커밋 [2][9][16][29][30])
F1≥0.5 : 7개  
F1=0.0 : 15개  
```

v3도 절반이 F1=0이다. "잘 되는 케이스"와 "전혀 안 되는 케이스"가 명확히 갈린다.

**잘 되는 패턴:** `mattermost_importer: Handle export output from newer mmctl` → `zerver/data_import/mattermost.py` (파일명이 기능과 직결)

**안 되는 패턴:** `streams: Allow read access for archived channels` → `version.py`, `zerver/views/streams.py` (커밋 메시지에 `version.py`를 연결할 단서 없음)

### 해석

n=10은 과대평가 위험이 있다. n=30이 더 신뢰할 수 있는 수치다. 그러나 n=30도 여전히 작다. 이 수치들은 "방향"을 보는 데 쓰고, 절대값으로 해석하지 않는 것이 맞다.

---

## 실험 2: Saleor 코드베이스

### Saleor 선택 이유

Zulip과 의도적으로 대비되는 특성을 가진 코드베이스를 골랐다:

| | Zulip | Saleor |
|-|-------|--------|
| 언어 | Python + TypeScript | Python 전용 |
| 규모 | Python 1987개, 202MB | Python 4232개, 96MB |
| 구조 | 모듈별 파일명이 직관적 | GraphQL 중심, mutations/types 반복 |
| 도메인 | 팀 채팅 | e-커머스 플랫폼 |
| 공개 여부 | 공개 (Claude 학습 가능성) | 공개 (동일) |

### 결과

| 방식 | Zulip n=30 | Saleor n=26 | 차이 |
|------|-----------|------------|------|
| v3 (ripgrep + AI) | **0.345** | **0.146** | −0.199 |
| AI 단독 | 0.141 | 0.054 | −0.087 |

**Saleor에서 v3 F1이 Zulip의 절반 이하다.**

### 왜 Saleor에서 더 낮은가

케이스를 분석하면 패턴이 보인다:

**잘 된 케이스 (F1≥0.5):**
```
[12] send_email: Move noop validation  → send_email.py  ✓
[13] Allow close() to work on SMTP     → email_backends.py  ✓
[25] Fix order line discount           → order_line_discount_remove.py  ✓
```
파일명이 커밋 메시지 키워드를 포함하면 ripgrep이 잘 찾는다.

**안 된 케이스 (F1=0):**
```
[3] Treat exceptions as expected    → saleor/product/tasks.py
[5] Migrate shipping methods        → celeryconf.py, migrations/...
[9] Fix memory leak for cached      → saleor/core/cache.py, utils/...
```

**핵심 문제:** Saleor는 GraphQL 구조상 기능이 여러 파일에 분산된다.  
`mutations/`, `types.py`, `resolvers.py` 같은 generic한 파일들에 비즈니스 로직이 나뉘어져 있다.  
커밋 메시지 키워드가 파일명과 연결되지 않는 케이스가 많다.

```
커밋: "Make refundSettings field nullable on RefundSettingsUpdate"
정답: saleor/graphql/shop/mutations/refund_settings_update.py

커밋: "Fix Circuit Breaker for promise webhook"  
정답: saleor/webhook/circuit_breaker/breaker_board.py
      saleor/webhook/transport/synchronous/transport.py
```

두 번째 케이스는 v3가 찾았다 (F1=0.286). 첫 번째는 `nullable`이라는 키워드가 GraphQL 스키마 파일들을 더 많이 가져와서 실패.

### AI 단독도 마찬가지

AI 단독 F1: Zulip 0.141 → Saleor 0.054

Zulip은 공개 프로젝트로 Claude가 내부 구조를 학습했을 가능성이 있다. Saleor도 공개지만 Zulip만큼 AI가 파일 구조를 잘 알지 못하는 것 같다. 혹은 Saleor의 GraphQL 구조가 더 예측하기 어렵다.

---

## 두 코드베이스 통합 비교

### 방식별 성능 (n=30/26)

| 방식 | Zulip F1 | Saleor F1 | 평균 |
|------|---------|----------|------|
| v3 (ripgrep + AI) | 0.345 | 0.146 | 0.246 |
| RAG (Chroma) | 0.196 | - | - |
| AI 단독 | 0.141 | 0.054 | 0.098 |

### v3 P/R 분석

| | Zulip | Saleor |
|-|-------|--------|
| Precision | 0.313 | 0.109 |
| Recall | 0.461 | 0.282 |
| F1 | 0.345 | 0.146 |

두 코드베이스 모두 Recall이 Precision보다 높다. v3는 "관련 파일을 찾아내는" 것보다 "엉뚱한 파일을 포함하지 않는 것"이 더 어렵다.

Saleor에서 특히 Precision이 낮다 (0.109). ripgrep이 키워드 기반으로 파일을 가져오면, Saleor의 넓은 코드베이스에서 관련 없는 파일이 더 많이 포함된다.

---

## RAG n=10 vs n=30 (Zulip)

| | n=10 | n=30 |
|-|------|------|
| F1 | 0.125 | **0.196** |
| Precision | 0.153 | 0.191 |
| Recall | 0.167 | **0.267** |

Recall이 0.167 → 0.267로 크게 올랐다. n=30에서 RAG가 의미론적으로 관련 파일을 찾는 케이스가 늘어났다는 뜻이다.

RAG가 잘 된 케이스 예시:
```
커밋: "email_mirror: Add support for iso-8859-8-i Hebrew encoding"
정답: zerver/lib/email_mirror.py
RAG: F1=1.0 (정확히 맞춤)
```

커밋 메시지의 "email_mirror"가 벡터 공간에서 `email_mirror.py`와 가까웠다. 단어 매칭이 아닌 의미 유사도가 효과적이었던 케이스.

반면 RAG가 실패한 케이스:
```
커밋: "populate_db: Fix -o flag not spreading messages across days"
정답: zilencer/management/commands/populate_db.py
RAG 후보: zerver/lib/fix_unreads.py (키워드 "Fix"로 잘못 매칭)
```

---

## 이 데이터는 설득력이 있는가?

셀프 검토:

**신뢰할 수 있는 것:**
- 동일 데이터셋 + 동일 평가 지표 → 방식 간 상대 비교는 유효
- ground truth = 실제 커밋 변경 파일 → 객관적
- 경로 정규화 버그 수정 후 재측정 → 측정 오류 검증 완료

**주의해야 할 것:**
- n=30도 통계적으로 작다. ±0.05 수준의 오차를 감안해야 함
- 커밋 선택 기준(prod 1~4개, 전체 2~7개)이 "쉬운" 케이스에 편향될 수 있음
- Saleor Chroma 인덱스 없음 → RAG 비교 불완전
- C-det(그래프)을 n=30으로 측정하지 않음 → 비교 누락

**정직한 결론:**  
"v3가 Zulip에서 F1=0.345로 가장 높고, 코드베이스가 바뀌면 성능이 크게 달라진다"는 관찰은 충분히 지지된다. 하지만 "v3가 모든 코드베이스에서 최선이다"는 결론은 이 데이터로 낼 수 없다.

---

## 다음에 해야 할 것

1. **C-det n=30 측정** — 그래프 방식도 n=30으로 확장
2. **Saleor Chroma 인덱스 구축** → RAG Zulip/Saleor 비교
3. **코드 특화 임베딩 테스트** — all-MiniLM → CodeBERT/jina-code로 RAG 개선 시도
4. **탐색 태스크 정량화** — 파일 예측 외 "코드 이해" 태스크 F1 설계
5. **n 확대** — 최소 50개는 있어야 신뢰도 주장 가능

---

## 마무리

이번 실험에서 배운 것:

1. **n=10은 편향이 있다.** n=30으로 늘리니 v3 F1이 0.430 → 0.345로 내려왔다. 작은 샘플에서 좋은 결과가 나왔을 때 조심해야 한다.

2. **코드베이스 구조가 성능을 결정한다.** Saleor에서 v3 F1=0.146. Zulip(0.345)의 절반 이하. ripgrep 기반 방식은 파일명이 기능과 직결될 때 효과적이고, GraphQL처럼 구조적으로 분산된 코드베이스에서는 약하다.

3. **RAG는 n이 늘수록 안정화된다.** 0.125 → 0.196. Recall 개선이 뚜렷하다.

하나의 검색 방식이 모든 코드베이스에서 최선이 아니라는 것. 이건 엔터프라이즈 AI 엔지니어링에서 생각보다 중요한 관찰이다.

---

*실험 환경: Raspberry Pi 4B × 2, cc-proxy (Claude API), n30 실험 2026-03-30*  
*데이터: Zulip n=30, Saleor n=26 커밋*  
*증적: [teddy-team-sync/context/experiments/n30-comparison](https://github.com/leth2/teddy-team-sync/tree/main/context/experiments/n30-comparison)*
