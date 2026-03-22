# Slop Detection — 판단 기준 레퍼런스

## 읽기 자료 목록

### Q1. "결정이란 무엇인가"

1. **Michael Nygard — Documenting Architecture Decisions (2011)**
   https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
   - 분량: ~1페이지
   - 핵심: 결정 = Context + Decision + Consequences

2. **RFC 2119 — Key words for use in RFCs**
   https://datatracker.ietf.org/doc/html/rfc2119
   - 분량: 2페이지
   - 핵심: MUST / SHOULD / MAY의 법적 수준 정의

### Q2. "중복과 보완의 차이는 무엇인가"

3. **Divio — The Documentation System**
   https://docs.divio.com/documentation-system/
   - 분량: 웹페이지
   - 핵심: Tutorials / How-to / Explanation / Reference 4가지 타입

### Q3. "구현 가능하다는 기준은 무엇인가"

4. **Design by Contract — Eiffel**
   https://www.eiffel.com/values/design-by-contract/introduction/
   - 핵심: Precondition / Postcondition / Invariant

5. **Gherkin Reference — Cucumber**
   https://cucumber.io/docs/gherkin/reference/
   - 핵심: Given / When / Then 시나리오 포맷

## 판단 기준 요약

- 결정의 언어: MUST > SHOULD > MAY (RFC 2119)
- 문서 타입 구분: Reference(구현 가능) + Explanation(왜) (Divio)
- 구현 가능성 테스트: Given/When/Then으로 변환 가능한가 (Gherkin)
- 결정의 단위: ADR 하나 = 결정 하나 (Nygard)

## 관련 이슈

- teddy-team-sync #4: Spec Quality Gate
- teddy-team-sync #5: Spec = TC 파이프라인
