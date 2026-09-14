# Roadmap

> Status: Planned  
> 이 문서는 README의 Roadmap을 확장한 구현 순서이다. 완료된 구현 목록이 아니다.

---

## Phase 1. Server-driven 기본 구조

Status: Planned

목표:

- Java Handler / Service
- Business Policy 조합
- `SKIP / EXECUTE` 판단
- AiTask 생성
- AI Task Queue 연결
- Omni AI Task Router
- Conversation Start E2E

검증 포인트:

- Business Event와 AiTask 분리
- Business Policy가 Queue 이전에 끝나는지
- Queue가 Event Bus가 아니라 실행 경계로 동작하는지

---

## Phase 2. 공통 구조 재사용 검증

Status: Planned

목표:

- 두 번째 Server-driven Use Case 적용
- 상태 변경 기반 긴급 메시지 요약 또는 Label 기반 기능
- 공통 AiTask / Queue / Workflow 구조 재사용 검증

검증 포인트:

- Task Type 추가 시 구조 변경이 작은지
- Policy 조합이 기능별로 분리되는지
- Structured Result가 Delivery와 잘 연결되는지

---

## Phase 3. Client Integration

Status: Planned

목표:

- Client Tool Registry
- Java Tool Gateway
- WebSocket Tool Request / Response
- Permission / Capability Check
- `taskId / executionId / toolCallId` correlation

검증 포인트:

- Client가 Omni AI와 직접 연결되지 않는지
- Client Tool이 최소 범위 Context만 반환하는지
- Workflow Execution에서도 Client Integration을 사용할 수 있는지

---

## Phase 4. Agent Runtime

Status: Planned

목표:

- Runtime Tool Calling
- 여러 차례 Tool Call
- Agent Execution Lifecycle
- Timeout / Retry
- gRPC 또는 Internal RPC
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

---

## Phase 5. Context Store / Cache / Observability

Status: Planned

목표:

- Context Store 검토
- workload별 Queue / Worker 분리 검토
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
| Phase 1 | Server-driven 기본 구조와 Conversation Start E2E | Planned |
| Phase 2 | 두 번째 Use Case로 공통 구조 재사용 검증 | Planned |
| Phase 3 | Client Integration | Planned |
| Phase 4 | Agent Runtime | Planned |
| Phase 5 | Context Store / Cache / Observability | Planned |

