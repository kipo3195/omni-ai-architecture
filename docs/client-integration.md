# Client Integration

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
├─ Server Context / Server Tool
└─ Client Integration
```

Workflow Execution과 Agent Execution 모두 필요한 경우 Client Integration을 사용할 수 있다.

Status: Designed

---

## 2. Why Not Direct Connection

Client는 Omni AI Server와 직접 연결하지 않는다.

```text
Client
  ↕ WebSocket
Java Messenger Server
  ↕
Omni AI Runtime
```

Java Messenger Server가 이미 다음 책임을 갖고 있기 때문이다.

- Authentication
- User Session
- WebSocket Connection
- Permission
- Device State

Status: Designed

---

## 3. Java Tool Gateway

Client Context나 Client Tool이 필요한 경우 Java Messenger Server를 Gateway로 사용한다.

```text
Omni AI Runtime
   ↓ Client Context / Tool 필요
Java Tool Gateway
   ↓ WebSocket
Client Tool
   ↓
Java Tool Gateway
   ↓
Omni AI Runtime
```

Java Tool Gateway 책임:

- Client Connection 조회
- Permission / Capability Check
- Tool Request 전달
- Tool Response correlation
- timeout / disconnect 처리

Status: Designed

---

## 4. Client Tool Registry

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

## 5. Minimal Context Principle

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

## 6. Context Filtering Responsibility

```text
Client
→ Scope Reduction

Java Server
→ Business Filtering

Omni AI
→ Semantic Filtering / Ranking / Summary
```

Client는 의미적 판단보다 범위 축소와 deterministic filtering에 집중한다.

Status: Designed

---

## 7. Correlation

Agent Execution에서는 한 Task 안에서 여러 Tool Call이 발생할 수 있다.

공통 식별자 후보:

```text
taskId
executionId
toolCallId
```

Client Tool Response는 최소한 위 식별자로 원 요청과 연결될 수 있어야 한다.

Status: Designed

---

## 8. Workflow and Agent Examples

Workflow Execution에서도 Client Context가 필요할 수 있다.

```text
Workflow Execution
→ Client Context 필요
→ Java Tool Gateway
→ Client Tool
→ Context 확보
→ Workflow 계속 실행
```

Agent Execution에서는 Runtime 중 Tool 사용 여부를 동적으로 결정할 수 있다.

```text
Agent Execution
→ Tool Decision
→ Client Tool 선택
→ Java Tool Gateway
→ Client Tool
→ Tool Result
→ Agent Resume
```

Status: Designed

---

## 9. Timeout / Disconnect

Client Tool Calling은 Client 상태에 영향을 받는다.

고려 대상:

- Client disconnect
- Tool Result timeout
- Device capability mismatch
- 사용자가 화면을 이동한 경우
- Partial Result 또는 Fallback

Status: Planned

