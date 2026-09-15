# Client Integration

> Role: Client Context / Client Tool을 Workflow 또는 Agent가 사용할 수 있는 Provider 계층으로 설명하는 문서
> Status: Designed
> 이 문서는 Client Integration을 Workflow / Agent와 같은 실행 방식이 아니라 Context / Tool Provider 계층으로 설명한다.

---

## 1. Position

Client Integration은 별도의 AI 실행 방식이 아니다.

```text
Execution Mode
├─ Workflow Execution
└─ Agent Execution

Context / Tool Provider
├─ Server Tool Request via ai-orchestrator
└─ Client Integration
```

Workflow Execution과 Agent Execution 모두 필요한 경우 Client Integration을 사용할 수 있다.

Status: Designed

---

## 2. Why Not Direct Connection

Client는 `omni-ai-server`와 직접 연결하지 않는다.

```text
Client
  ↕ WebSocket
realtime-message-service
  ↕
ai-orchestrator / omni-ai-server
```

`realtime-message-service`가 이미 다음 책임을 갖고 있기 때문이다.

- Authentication
- User Session
- WebSocket Connection
- Permission
- Device State

Status: Designed

---

## 3. Client Tool Relay

Client Context나 Client Tool이 필요한 경우 `realtime-message-service` 내부의 Client Tool Relay를 사용한다.

```text
omni-ai-server
   ↓ Client Context / Tool 필요
Client Tool Relay
   ↓ WebSocket
Client Tool
   ↓
Client Tool Relay
   ↓
omni-ai-server
```

Client Tool Relay 책임:

- Client Connection 조회
- Permission / Capability Check
- Tool Request 전달
- Tool Response correlation
- timeout / disconnect 처리

Client Tool Relay는 별도 서비스가 아니라 `realtime-message-service`의 responsibility다.

`ai-orchestrator`가 실행을 조정하는 경우에도 Client Tool 요청의 session, permission, connection 책임은 Client Tool Relay에 남긴다. Tool result payload는 `omni-ai-server`로 반환하고, `ai-orchestrator`에는 started / completed / failed 같은 lifecycle event만 보고할 수 있다.

Client Tool Relay는 요청을 보낸 instance가 아니라 Session Registry의 현재 `ownerInstanceId`를 기준으로 target WebSocket session을 찾는다. Agent 실행 중 reconnect나 room 이동이 발생할 수 있으므로 `connectionId`, `roomSessionId`, `toolCallId`, `executionId`를 함께 사용해 현재 client location과 tool response를 연결한다.

Status: Designed

---

## 4. Server Tool Relay와의 구분

Server-side Context나 Tool이 필요한 경우에는 Server Tool Relay를 사용한다.

```text
omni-ai-server
→ ai-orchestrator
→ Server Tool Relay
→ auth-service / file-service / user-service / realtime-message-service
```

Client-side Context나 Tool이 필요한 경우에는 Client Tool Relay를 사용한다.

```text
omni-ai-server
→ Client Tool Relay inside realtime-message-service
→ Client Tool
```

Server Tool Relay는 `ai-orchestrator` 내부 책임으로 service API / gRPC 호출, service capability, policy-aware access를 다룬다. Client Tool Relay는 `realtime-message-service` 내부 책임으로 WebSocket session lookup, client capability, request / response correlation을 다룬다.

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

realtime-message-service
→ Business Filtering

ai-orchestrator
→ Cross-domain Context Assembly / Trigger Filtering

omni-ai-server
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

Status: Designed

---

## 9. Workflow and Agent Examples

Workflow Execution에서도 Client Context가 필요할 수 있다.

```text
Workflow Execution
→ Client Context 필요
→ Client Tool Relay
→ Client Tool
→ Context 확보
→ Workflow 계속 실행
```

Agent Execution에서는 Runtime 중 Tool 사용 여부를 동적으로 결정할 수 있다.

```text
Agent Execution
→ Tool Decision
→ Client Tool 선택
→ Client Tool Relay
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

Status: Planned

---

## 11. Streaming Result Delivery

LLM Streaming Result는 `ai-orchestrator`가 token-by-token proxy하지 않는다.

```text
omni-ai-server
→ Core NATS
→ realtime-message-service
→ Client WebSocket
```

`realtime-message-service` 책임:

- Core NATS AI Stream 수신
- Session Registry 또는 local session map을 통한 Target WebSocket Session 탐색
- 현재 ownerInstanceId가 자신인지 확인
- Messenger Client WebSocket Protocol로 변환
- Client에 Stream Push

`realtime-message-service`가 담당하지 않는 책임:

- AI Trigger Policy
- Cross-domain Context Assembly
- LLM Workflow
- Conversation Runtime State

Status: Designed
