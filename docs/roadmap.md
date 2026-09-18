# Roadmap

> Role: 구현 순서와 진행 상태를 관리하는 문서
> Status: In Progress
> 이 문서는 README의 Roadmap을 확장한 구현 순서이다. 단계의 진행 상태는 개별 항목의 구현 완료를 뜻하지 않는다.
> 실제 코드 구조와 검증 결과는 [Implementation](implementation/README.md)에 별도로 기록한다.

---

## Phase 1. Conversation Start를 통한 AI Orchestrator 기반 구축

Status: In Progress

첫 번째 구현은 Conversation Start를 적용 사례로 삼아 Spring 기반 `AI Orchestrator`의 구조와 서비스 경계를 검증한다. 아래 두 작업은 구분해서 진행 상태를 기록한다.

### 1. Spring 기반 AI Orchestrator 구조

- 독립적인 Spring Boot 서비스와 기능 중심 패키지 구조 구성
- WebSocket Service, Omni AI Server와의 요청·응답 경계 정의
- Server Trigger 흐름에 필요한 application / domain / infrastructure 역할 배치
- 오류 처리와 외부 호출 등 공통 서비스 기반 정리

### 2. Conversation Start 적용

- `ROOM_ENTERED` 등 Conversation Start 실행 계기와 요청 계약 연결
- 이 기능에 필요한 Policy 판단, Context Assembly, AiTask 생성 구현
- `Omni AI Server`의 Task Router / Workflow와 연결해 추천 결과 전달
- WebSocket Service 경로에서 Conversation Start 흐름 검증

Trigger Policy, Context Assembly, AiTask 생성은 `AI Orchestrator`의 책임이지만, Phase 1에서는 Conversation Start 흐름 안에서 필요한 범위로 구현한다. 각 책임을 독립적인 공통 기능의 완료 항목으로 취급하지 않는다. 재사용 구조가 확인되면 [Server Trigger 구현 문서](implementation/ai-orchestrator/server-trigger/README.md)에 정리한다.

검증 포인트:

- Spring 서비스 구조와 Conversation Start 기능 구현의 변경 범위를 구분할 수 있는지
- Business Event와 AiTask가 분리되는지
- WebSocket Service가 connection / delivery를, AI Orchestrator가 실행 판단을 소유하는지
- Omni AI Server가 Business Rule 없이 확정된 작업을 실행하는지
- Conversation Start의 입력부터 추천 결과까지 한 경로가 동작하는지

---

## Phase 2. 쪽지 요약 품질과 Cross-domain Trigger / Result Routing 검증

Status: Planned

목표:

- TCP Realtime Service 경로에 Phase 1의 공통 AI 실행 계약 적용
- `USER_RETURNED` 등 상태 변경을 계기로 한 쪽지 요약을 적용 사례로 구현
- NATS JetStream 기반 Business Event / AI Trigger 전달
- `AI Orchestrator`에서 요약 대상 선정과 Context Assembly
- Cooldown / dedup / trigger expiration 정책
- `Omni AI Server`에서 LLM 기반 쪽지 요약 흐름과 결과 형식 구현
- 대표 쪽지 사례로 요약 품질을 평가하고 Prompt / Context 구성을 개선
- Core NATS 기반 Result Routing
- `Omni AI Server → Core NATS → WebSocket Service` Streaming Data Path

검증 포인트:

- Trigger Consumer 장애 후 재처리가 가능한지
- Trigger가 오래된 경우 실행을 skip할 수 있는지
- Reconnect / scale-out 이후 현재 Session Owner로 결과가 전달되는지
- Structured Result가 Delivery와 잘 연결되는지
- `AI Orchestrator`가 token stream을 proxy하지 않는지
- 요약이 원문에 없는 사실을 만들어내거나 중요한 내용을 누락하지 않는지
- 주요 내용과 필요한 후속 행동이 짧고 읽기 쉬운 형태로 전달되는지
- 동일한 평가 사례에서 Prompt / Context 변경 전후의 품질을 비교할 수 있는지

---

## Phase 3. Client Integration

Status: Planned

목표:

- Tool Runtime inside `AI Orchestrator`
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

- Client가 `Omni AI Server`와 직접 연결되지 않는지
- Tool Runtime이 server/client tool lifecycle을 일관되게 관리하는지
- Server Tool Adapter가 `auth-service` / `file-service` / `user-service` / `WebSocket Service` 경계를 지키는지
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
| Phase 1 | Spring 기반 AI Orchestrator 구조와 Conversation Start 적용 | In Progress |
| Phase 2 | 쪽지 요약 품질과 Cross-domain Trigger / Result Routing 검증 | Planned |
| Phase 3 | Client Integration | Planned |
| Phase 4 | Stateful Chatbot / Agent Runtime | Planned |
| Phase 5 | AI Context Projection / Cache / Observability | Planned |
| Future | WebSocket Service responsibility split | Planned |
