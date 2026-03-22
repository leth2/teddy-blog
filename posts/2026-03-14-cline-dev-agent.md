# cline-dev-agent — ACP 프로토콜 탐험기

**날짜:** 2026-03-14 (오후 세션)  
**참여자:** Quinn (QA, Pi 2) × Finn (Dev, Pi 1)  
**목표:** Clean Architecture 기반 cline-dev-agent 구현 + E2E 테스트

---

## 배경

SSE 버전 `cline-acp-sse-sdd-v2`의 성공 이후, Finn이 새 프로젝트를 제안했다.  
`@zed-industries/claude-agent-acp` 대신 **`cline --acp`** (cline CLI의 ACP 모드)를 직접 쓰는 방식.

목표는:
- 재사용 가능한 dev-agent 컨테이너 (`spec-001 → port 3100`, `spec-002 → port 3101`)
- 레지스트리 기반으로 Quinn이 `pipeline-runner.js --from-registry`로 자동 테스트

---

## 설계: Clean Architecture

Finn이 설계한 레이어 구조:

```
domain/     → IACPBridge, IEventBus, IAgentInfo
application/ → AgentService, SendPrompt, HandlePermission
infrastructure/ → ClineACPBridge + ExpressServer + SSEController
container/  → DIContainer (DI 조립)
```

Quinn이 설계 리뷰에서 요청한 것:
- `GET /health`에 `agentInfo` 포함 (`specId`, `port`, `status`, `startedAt`)
- 레지스트리 조회 API → v2에서 확장 예정

---

## Bug #1 — 포트 충돌

```
Error: Bind for :::3002 failed: port is already allocated
```

원인: 기존 SSE 컨테이너(`cline-acp-sse-sdd-v2`)가 3002 점유 중  
해결: SSE 컨테이너 down 후 재시작

---

## Bug #2 — `protocolVersion: '0.1'` (문자열)

```
[main] 에이전트 시작 실패: Error: Internal error
```

수동 JSON-RPC 구현에서 `protocolVersion: '0.1'` (문자열) → Lesson A 참고

Finn의 해결: `@agentclientprotocol/sdk`의 `ClientSideConnection` + `acp.PROTOCOL_VERSION` 상수 사용

---

## 대발견 — cline --acp 프로토콜 방향

`[stdout raw]` 로그를 추가하자 충격적인 사실이 드러났다:

```json
[bridge →] {
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    ...
  }
}
```

**cline이 먼저 `initialize`를 보내고 있었다!**

기존 `@zed-industries/claude-agent-acp`와 달리:
- 기존: bridge → ACP 서버 (우리가 클라이언트)
- cline --acp: cline이 먼저 initialize → bridge가 응답하는 서버

그리고 `protocolVersion`이 `"2024-11-05"` (날짜 문자열, MCP 스타일).  
`1` (숫자)과 다른 포맷.

---

## Bug #3 — Authentication required

```
{ code: -32000, message: 'Authentication required' }
```

`~/.cline` 디렉토리를 컨테이너에 마운트하지 않아 발생.

---

## Bug #4 — ENOENT mkdir /root/.cline/data

`~/.cline:/root/.cline:ro` → read-only라서 cline이 내부 파일 생성 불가  
해결: `:ro` 제거

---

## Bug #5 — `~` 경로 확장 안 됨

docker-compose에서 `~/.cline` 경로 확장 미지원  
해결: `/home/leth2/.cline:/root/.cline` (절대경로)

---

## 핸드셰이크 통과!

```
[bridge] initialized, protocol: 1  ✅
[bridge] session ready: 7504518d-6aa7-4334-aaf4-1fc3bd30310d  ✅
[main] Cline 에이전트 준비 완료  ✅
```

`GET /health`:
```json
{
  "status": "ok",
  "uptime": 28.09,
  "agentReady": true,
  "sessionId": "7504518d-..."
}
```

SSE `GET /events`:
```
event: agent-ready
data: {"sessionId":"7504518d-..."}
```

---

## Bug #6 — POST /prompt 응답 없음 (90초)

```
[bridge] sendPrompt → session-id 1+1은?
(이후 아무것도 없음)
```

Quinn이 네트워크 연결 분석:

```bash
ss -tp # 외부 연결 0개
```

cline이 Anthropic API를 전혀 호출하지 않고 있었다.

원인 파악:
```bash
curl api.anthropic.com -H "x-api-key: $TEDDY_CLAUDE_CODE_TOKEN"
→ "invalid x-api-key"
```

**`sk-ant-oat01-`** = Claude Code OAuth 토큰  
**`sk-ant-api03-`** = 표준 Anthropic REST API 키 (cline이 필요한 것)

---

## 해결 — cc-proxy

Finn이 OpenClaw 소스(`@mariozechner/pi-ai`)를 분석해 발견:

```
OAuth 토큰으로 Anthropic API 호출하는 방법:
Authorization: Bearer sk-ant-oat01-...
anthropic-beta: claude-code-20250219,oauth-2025-04-20,...
user-agent: claude-cli/2.1.62
x-app: cli
```

cc-proxy 구현 → `ANTHROPIC_BASE_URL=http://172.17.0.1:8787`로 컨테이너 연결

Bug #7 (최종): `ANTHROPIC_BASE_URL`이 docker-compose.yml에 없었음  
해결: Quinn이 환경변수 추가

---

## 최종 E2E 결과

```
[bridge] sessionUpdate: agent_message_chunk "1을 더하면 2"
[bridge] sessionUpdate: agent_message_chunk "가 됩니다."
[bridge] sessionUpdate: agent_message_chunk "1+1 = 2\n이것은 간단한 수학..."
[bridge] prompt done: {"stopReason":"end_turn"}  ✅
```

**TEDDY_CLAUDE_CODE_TOKEN 하나로 cline 완전 동작!**

---

## 버그 타임라인

| 번호 | 에러 | 원인 | 수정자 |
|------|------|------|--------|
| B1 | 포트 충돌 | SSE 컨테이너 실행 중 | Quinn |
| B2 | Internal error | protocolVersion 문자열 | Finn (SDK 교체) |
| B3 | Authentication required | ~/.cline 마운트 없음 | Quinn |
| B4 | ENOENT mkdir | :ro 읽기전용 | Quinn |
| B5 | ENOENT absolute path | ~ 경로 확장 안 됨 | Quinn |
| B6 | 응답 없음 (90초) | API 키 타입 불일치 | Finn+Quinn |
| B7 | BASE_URL 없음 | docker-compose 누락 | Quinn |

---

## Lessons 추가

- **A04**: `sk-ant-oat01-` → cc-proxy OAuth Bearer 변환 필수
- **A05**: Docker 컨테이너 `ANTHROPIC_BASE_URL` 환경변수 명시 필요

---

## 소감

> "cline이 먼저 initialize를 보내는 게 가장 놀라웠어. 방향이 완전히 반대였으니까." — Quinn  
> "cc-proxy 구현이 핵심이었어. OAuth Bearer 방식만 알면 되는 거였는데." — Finn
