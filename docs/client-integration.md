# Client Tool Integration

> Role: Client Context / Client Tool을 Workflow 또는 Agent가 사용할 수 있는 Provider 계층으로 설명하는 문서
> Status: Designed
> 이 문서에서 Client Integration은 Client Tool Integration을 의미한다. Client LLM이 ScheduleSpec을 생성하는 기능과 구분한다.
> Client Tool Integration은 Workflow / Agent와 같은 실행 방식이 아니라 Context / Tool Provider 계층이다.

---

## 1. Position

Client Tool Integration은 별도의 AI 실행 방식이 아니다.

Phase 3의 Client LLM 기반 Scheduled Weekly Report는 Client가 구조화된 요청을 생성하는 Use Case이며, 실행 중 Client capability를 호출하는 Client Tool Integration이 아니다.

```text
Execution Mode
├─ Workflow Execution
└─ Agent Execution

Context / Tool Provider
└─ Tool Request via AI Orchestrator Tool Runtime
```

Workflow Execution과 Agent Execution 모두 필요한 경우 Client Tool Integration을 사용할 수 있다.

Status: Designed

---

## 2. Why Not Direct Connection

Client는 `Omni AI Server`와 직접 연결하지 않는다.

```text
Client
  ↕ WebSocket
WebSocket Service
  ↕
AI Orchestrator / Omni AI Server
```

`WebSocket Service`가 이미 다음 책임을 갖고 있기 때문이다.

- Authentication
- User Session
- WebSocket Connection
- Permission
- Device State

Status: Designed

---

## 3. Client Tool Delivery

Client Context나 Client Tool이 필요한 경우 `AI Orchestrator`의 Tool Runtime이 lifecycle을 소유하고, `WebSocket Service`는 Client Tool Delivery를 담당한다.

```text
Omni AI Server
   ↓ Client Context / Tool 필요
AI Orchestrator Tool Runtime
   ↓ Result Router
   ↓ Realtime Connection Registry에서 routingRef 기준 현재 ownerInstanceId 조회
   ↓ Core NATS owner-instance subject
Client Tool Delivery
   ↓ WebSocket
Client Tool
   ↓
Client Tool Delivery
   ↓ Core NATS
AI Orchestrator Tool Runtime
   ↓
Omni AI Server resume
```

Tool Runtime 책임:

- Tool registry / schema validation
- Permission / capability policy decision
- Tool lifecycle state
- Tool Request dispatch
- Tool Response correlation
- timeout / retry / cancellation
- result validation / normalization
- execution resume coordination

Client Tool Delivery 책임:

- Result Router가 선택한 instance의 local Client Connection 조회
- WebSocket Tool Request 전달
- Client Tool Response ingress
- disconnect 감지
- 선택된 현재 Session Owner instance에서 local delivery

Client Tool Delivery는 별도 서비스가 아니라 `WebSocket Service`의 responsibility다.

`WebSocket Service`는 client session과 WebSocket delivery를 소유하지만 Tool lifecycle owner는 아니다. Tool result payload는 `AI Orchestrator`의 Tool Runtime으로 반환되고, Tool Runtime이 상태를 완료 처리한 뒤 `Omni AI Server` execution resume을 조정한다.

Client Tool Delivery는 요청을 보낸 instance를 target으로 사용하지 않는다. Result Router가 Realtime Connection Registry의 현재 `ownerInstanceId`를 기준으로 target instance와 Core NATS subject를 선택한다. Agent 실행 중 reconnect나 room 이동이 발생할 수 있으므로 `connectionId`, `roomSessionId`, `toolCallId`, `executionId`를 함께 사용해 현재 client location과 tool response를 연결한다. 초기에는 AI Orchestrator 내부 Result Router가 owner 조회와 Core NATS subject 선택을 담당하며, Client Tool Delivery는 선택된 instance의 local session 최종 전달과 result ingress만 담당한다.

Status: Designed

---

## 4. Server Tool Adapter와 Client Tool Delivery의 구분

Server-side Context나 Tool이 필요한 경우에는 Tool Runtime의 Server Tool Adapter를 사용한다.

```text
Omni AI Server
→ AI Orchestrator Tool Runtime
→ Server Tool Adapter
→ auth-service / file-service / user-service / WebSocket Service
```

Client-side Context나 Tool이 필요한 경우에는 Tool Runtime에서 Core NATS와 Client Tool Delivery를 통해 Client에 dispatch한다.

```text
Omni AI Server
→ AI Orchestrator Tool Runtime
→ Result Router
→ Realtime Connection Registry 조회
→ Core NATS
→ Client Tool Delivery inside WebSocket Service
→ Client Tool
→ Client Tool Delivery
→ Core NATS
→ AI Orchestrator Tool Runtime
→ Omni AI Server resume
```

Tool Runtime은 server/client tool 공통 lifecycle을 관리한다. Server Tool Adapter는 service API / gRPC 호출, service capability, policy-aware access를 다룬다. Result Router는 전역 routing resolution을, Client Tool Delivery는 선택된 instance의 local session lookup과 client delivery / result ingress를 다룬다.

Status: Designed

---

## 5. Client Tool Registry

Client Tool Registry는 Client가 제공할 수 있는 Tool과 실행 가능 조건을 정의하는 개념이다.

예:

```text
getSelectedMessages()
getCurrentView()
getRecentLocalMessages(limit)
getCurrentDraft()
getSelectedFileMetadata()
```

Status: Planned

---

## 6. Minimal Context Principle

Client Tool은 전체 데이터를 전달하지 않고 AI Task에 필요한 최소 범위만 제공한다.

권장:

```text
getSelectedMessages()
getCurrentView()
getRecentLocalMessages(limit)
getCurrentDraft()
```

지양:

```text
getAllClientData()
getAllChatHistory()
readAllLocalStorage()
```

Status: Designed

---

## 7. Context Filtering Responsibility

```text
Client
→ Scope Reduction

WebSocket Service
→ Business Filtering

AI Orchestrator
→ Cross-domain Context Assembly / Trigger Filtering

Omni AI Server
→ Semantic Filtering / Ranking / Summary
```

Client는 의미적 판단보다 범위 축소와 deterministic filtering에 집중한다.

Status: Designed

---

## 8. Correlation

Agent Execution에서는 한 Task 안에서 여러 Tool Call이 발생할 수 있다.

공통 식별자 후보:

```text
triggerId
taskId
executionId
conversationId
connectionId
roomSessionId
toolCallId
```

Client Tool Response는 최소한 위 식별자로 원 요청과 연결될 수 있어야 한다.

`connectionId`와 `roomSessionId`는 client location을 찾기 위한 routing correlation이다. Tool Response와 AI Stream Result 모두 최종 push 직전에 현재 Session Registry를 확인한다.

`toolCallId`, `toolAttempt`, `idempotencyKey`는 Tool Runtime에서 중복 실행과 중복 result를 방지하기 위한 correlation이다.

Status: Designed

---

## 9. Workflow and Agent Examples

Workflow Execution에서도 Client Context가 필요할 수 있다.

```text
Workflow Execution
→ Client Context 필요
→ AI Orchestrator Tool Runtime
→ Result Router / Realtime Connection Registry 조회
→ Client Tool Delivery
→ Client Tool
→ Context 확보
→ Workflow 계속 실행
```

Agent Execution에서는 Runtime 중 Tool 사용 여부를 동적으로 결정할 수 있다.

```text
Agent Execution
→ Tool Decision
→ Client Tool 선택
→ AI Orchestrator Tool Runtime
→ Result Router / Realtime Connection Registry 조회
→ Client Tool Delivery
→ Client Tool
→ Tool Result
→ Agent Resume
```

Status: Designed

---

## 10. Timeout / Disconnect

Client Tool Calling은 Client 상태에 영향을 받는다.

고려 대상:

- Client disconnect
- Tool Result timeout
- Device capability mismatch
- 사용자가 화면을 이동한 경우
- Partial Result 또는 Fallback
- Duplicate Tool Result
- Execution resume 실패

Status: Planned

---

## 11. Streaming Result Delivery

LLM Streaming Result는 `AI Orchestrator`가 token-by-token proxy하지 않는다.

```text
Omni AI Server
→ Result Router
→ Realtime Connection Registry 조회
→ Core NATS owner-instance subject
→ Realtime Service
→ Client WebSocket
```

`WebSocket Service` 책임:

- Core NATS AI Stream 수신
- Result Router가 선택한 instance에서 local session map으로 Target WebSocket Session 확인
- Messenger Client WebSocket Protocol로 변환
- Client에 Stream Push

`WebSocket Service`가 담당하지 않는 책임:

- AI Trigger Policy
- Cross-domain Context Assembly
- LLM Workflow
- Conversation Runtime State

Status: Designed
