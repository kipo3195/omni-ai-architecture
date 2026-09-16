# Roadmap

> Role: 구현 순서와 진행 상태를 관리하는 문서
> Status: Planned
> 이 문서는 README의 Roadmap을 확장한 구현 순서이다. 완료된 구현 목록이 아니다.

---

## Phase 1. Current WS 연동과 Omni AI 기본 구조

Status: Planned

목표:

- 현재 WS service 연동
- `ai-orchestrator` boundary 정의
- `omni-ai-server` Task Router
- Business Policy 조합
- `SKIP / EXECUTE` 판단
- AiTask 생성
- Conversation Start E2E

검증 포인트:

- Business Event와 AiTask 분리
- 현재 WS service와 연동해도 `ai-orchestrator` / `omni-ai-server` 경계가 유지되는지
- Single-domain Use Case가 과도하게 `ai-orchestrator`에 의존하지 않는지
- `omni-ai-server`가 Business Rule을 소유하지 않는지

---

## Phase 2. Cross-domain Trigger / Result Routing 검증

Status: Planned

목표:

- 상태 변경 기반 긴급 메시지 요약 또는 Label 기반 기능
- NATS JetStream 기반 Business Event / AI Trigger 전달
- `ai-orchestrator`의 Context Assembly
- Cooldown / dedup / trigger expiration 정책
- Core NATS 기반 Result Routing
- `omni-ai-server → Core NATS → realtime-message-service` Streaming Data Path

검증 포인트:

- Trigger Consumer 장애 후 재처리가 가능한지
- Trigger가 오래된 경우 실행을 skip할 수 있는지
- Reconnect / scale-out 이후 현재 Session Owner로 결과가 전달되는지
- Structured Result가 Delivery와 잘 연결되는지
- `ai-orchestrator`가 token stream을 proxy하지 않는지

---

## Phase 3. Client Integration

Status: Planned

목표:

- Tool Runtime inside `ai-orchestrator`
- Tool Registry / Tool Lifecycle State
- Server Tool Adapter
- Client Tool Registry
- Client Tool Delivery
- WebSocket Tool Request / Response
- Permission / Capability Check
- Tool timeout / retry / cancellation
- Tool result normalization
- `triggerId / taskId / executionId / conversationId / toolCallId / toolAttempt / idempotencyKey` correlation

검증 포인트:

- Client가 `omni-ai-server`와 직접 연결되지 않는지
- Tool Runtime이 server/client tool lifecycle을 일관되게 관리하는지
- Server Tool Adapter가 `auth-service` / `file-service` / `user-service` / `realtime-message-service` 경계를 지키는지
- Client Tool Delivery가 현재 Session Owner 기준으로 target WebSocket session을 찾는지
- Client Tool이 최소 범위 Context만 반환하는지
- Workflow Execution에서도 Client Integration을 사용할 수 있는지
- LLM token stream과 execution progress가 Tool Runtime을 token-by-token 경유하지 않는지

---

## Phase 4. Stateful Chatbot / Agent Runtime

Status: Planned

목표:

- Runtime Tool Calling
- Conversation Metadata Store
- AI History Store
- `conversationId + message` flow
- 여러 차례 Tool Call
- Agent Execution Lifecycle
- Timeout / Retry
- NATS-based async tool result handling
- Agent Resume

상태 후보:

```text
CREATED
QUEUED
PROCESSING
WAITING_TOOL_RESULT
COMPLETED
FAILED
TIMEOUT
CANCELLED
```

검증 포인트:

- Tool Decision이 Runtime 중 동적으로 가능한지
- Tool Result correlation이 안정적인지
- Client disconnect와 timeout을 처리할 수 있는지
- WebSocket Session과 Conversation lifecycle이 분리되는지

---

## Phase 5. AI Context Projection / Cache / Observability

Status: Planned

목표:

- Service API / gRPC 기반 AI Context 조회 계약 검토
- Event-driven AI Context Projection 적용 범위 검토
- workload별 Queue / Consumer / Worker 분리 검토
- Exact Cache
- Semantic Cache 적용 범위 검토
- Observability
- AI Evaluation
- Execution Budget

측정 후보:

- AI 호출 횟수
- Workflow별 호출량
- Token 사용량
- 비용
- Latency
- Success / Failure
- Timeout
- Trigger Delay
- Queue Delay
- Cache Hit
- Tool Call 횟수
- 실제 사용자 Action 연결 여부

Execution Budget 후보:

```text
maxToolCalls
maxExecutionTime
maxTokens
```

---

## Status Summary

| Phase | Goal | Status |
| --- | --- | --- |
| Phase 1 | Current WS 연동과 Omni AI 기본 구조 | Planned |
| Phase 2 | Cross-domain Trigger / Result Routing 검증 | Planned |
| Phase 3 | Client Integration | Planned |
| Phase 4 | Stateful Chatbot / Agent Runtime | Planned |
| Phase 5 | AI Context Projection / Cache / Observability | Planned |
| Future | WS service responsibility split | Planned |
