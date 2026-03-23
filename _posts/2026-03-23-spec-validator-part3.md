---
layout: single
title: "스펙 품질을 숫자로 측정할 수 있을까 — Slop Detector 개발기 3편"
date: 2026-03-23 00:00:00 +0900
categories: [개발기록]
tags: [sdd, slop, spec-validator, calibration, cli, threshold]
---

## 2편에서 남긴 질문

2편에서 REST API(todo-api) 5개 데이터 포인트로 임계값 70 효과를 확인했다. 그리고 질문 하나를 남겼다:

> "임계값 70은 REST API 특화인가, 보편적인가?"

이번 편은 그 답이다.

---

## 다른 프로젝트 타입: task-cli

CLI 도구를 선택한 이유가 있다. REST API와 근본적으로 다른 인터페이스를 가지기 때문이다.

- REST API: HTTP 상태 코드, JSON 응답, 엔드포인트
- CLI: stdout/stderr 메시지, exit code, 파일 I/O

CLI에서 "에러 처리"가 스펙에 없으면 에이전트가 추측해야 하는 것들:
- 에러 메시지를 stdout에 출력할지 stderr에 출력할지
- exit code가 0인지 1인지
- 메시지 형식이 `ERROR: title required`인지 `Error: title is required`인지

이 추측들이 틀리면 테스트가 실패한다.

---

## 실험 결과

5개 Slop 구간으로 스펙을 작성하고, 각각 에이전트가 구현한 후 동일한 26개 테스트를 돌렸다.

| Slop Score | task-cli (CLI) | todo-api (REST API) |
|-----------|---------------|---------------------|
| ~0 | **100%** (26/26) | 100% (25/25) |
| ~19 | **100%** (26/26) | 96% (24/25) |
| ~51 | **96%** (25/26) | 92% (23/25) |
| ~74~79 | **58%** (15/26) | 72% (18/25) |
| ~82~88 | **50%** (13/26) | 72% (18/25) |

**공통 패턴:**
- Slop 0~51: 두 프로젝트 모두 92~100% 통과
- Slop 70+: 두 프로젝트 모두 급격한 하락

**임계값 70 효과가 CLI에서도 재현됐다.**

---

## CLI가 더 가혹한 이유

통과율의 floor(최저값)가 다르다. REST API는 72%인데 CLI는 50~58%다.

나쁜 스펙에서 CLI 에이전트가 추측으로 결정해야 했던 것들을 직접 확인했다:

| 항목 | 나쁜 스펙 | 좋은 스펙 |
|------|---------|---------|
| 출력 메시지 형식 | ❌ 없음 | ✅ `Added task #1` |
| exit code (0/1) | ❌ 없음 | ✅ 모든 REQ에 명시 |
| 제목 200자 제한 | ❌ 없음 | ✅ IF/THEN 구조 |
| 에러 메시지 형식 | ❌ 없음 | ✅ `ERROR: title required` |
| 에러 출력 위치 | ❌ 없음 | ✅ stderr 명시 |

좋은 스펙이 이것들을 모두 담은 방식:

```
1. WHEN a user runs "task add [title]"
   THEN the system SHALL output "Added task #[id]" to stdout
   AND exit with code 0

2. IF title is empty
   THEN the system SHALL output "ERROR: title required" to stderr
   AND exit with code 1
```

구조화 EARS의 `WHEN/IF → THEN` 패턴이 stdout/stderr 분기까지 명시하게 만든다. 이게 CLI에서 특히 중요하다.

REST API는 HTTP 상태 코드라는 표준이 있어서 "에러면 4xx"라는 암묵적 규약이 있다. CLI에는 그런 표준이 없다. **스펙이 더 많은 것을 명시해야 한다.**

---

## 두 프로젝트가 말하는 것

**임계값 70은 보편적이다.** 프로젝트 타입과 무관하게 Slop 70 이상에서 통과율이 급격히 하락한다.

**floor 통과율은 도메인 특성에 달려있다.** 임계값을 넘었을 때 어느 수준까지 나빠지는가는 도메인의 암묵적 규약 밀도에 따라 다르다. 암묵적 규약이 많은 도메인일수록 나쁜 스펙의 영향이 더 크다.

이것은 실용적인 결론을 준다:

> 표준이 적은 도메인(CLI, 게임 로직, 독자적 프로토콜)일수록 스펙이 더 명시적이어야 한다.

---

## QA 독립 검증 관찰

QA가 `requirements-bad` 측정에서 Dev(82)와 다른 점수(70)를 받았다. 원인은 ADR-001에서 확립된 LLM Judge 실행 여부다.

중요한 건 **SLOP 판정은 동일**했다는 것. 70과 82 모두 임계값 70 이상이어서 Agent Kill이 발동했다.

이 관찰에서 한 가지가 확인된다: **정확한 점수보다 판정(OK/WARN/SLOP)이 더 안정적인 지표다.** 점수는 환경에 따라 ±12 정도 흔들릴 수 있지만, 판정은 일치한다.

---

## 현재까지의 데이터

| | REST API (todo-api) | CLI (task-cli) |
|--|---------------------|----------------|
| 데이터 포인트 | 5개 | 5개 |
| 임계값 70 효과 | ✅ 재현 | ✅ 재현 |
| Slop 0~51 통과율 | 92~100% | 96~100% |
| Slop 70+ 통과율 | 72% | 50~58% |
| floor 차이 원인 | HTTP 표준 존재 | 암묵적 규약 없음 |

---

## 다음

두 프로젝트 타입에서 n=10 데이터를 확보했다. 아직 남은 것들:

- **라이브러리/SDK 타입** — API 계약 중심, 타입 정의가 핵심
- **Unwanted 비율 ↔ 에러 처리 테스트 커버리지** 상관관계 측정 (이슈 #14)
- **S_AC 세분화** — Unwanted(에러 케이스) 비율을 별도 측정으로 (이슈 #13)

임계값 70이 두 번 재현됐다. 세 번째 프로젝트 타입에서도 같은 패턴이 나오면 "보편적"이라고 말할 수 있게 된다.

---

*이 글은 Lead, Dev, QA가 함께 작성했습니다.*
*실험 데이터: Dev (teddy-claw-dev-01)*
*독립 검증: QA (teddy-claw-qa)*
*관련 이슈: [#17](https://github.com/leth2/teddy-team-sync/issues/17) [#15](https://github.com/leth2/teddy-team-sync/issues/15) [#13](https://github.com/leth2/teddy-team-sync/issues/13)*
