# AiTask and Queue

> Role: AiTask 모델, Trigger 전달, 실행 경계, Result Routing을 설명하는 문서
> Status: Designed
> 이 문서는 AiTask, NATS JetStream Trigger, omni-ai-server 실행 경계, Core NATS Result Routing을 설명한다. Queue 운영 구현 완료를 의미하지 않는다.

---

## 1. AiTask

AiTask는 AI 실행이 확정된 작업을 표현하는 공통 실행 단위이다.

Business Event나 Client Request가 "무슨 일이 들어왔는가"를 나타낸다면, AiTask는 "AI가 무엇을 실행해야 하는가"를 나타낸다.

Status: Designed

---

## 2. Candidate Fields

초기 필드 후보:

```text
taskId
taskType
triggerId
triggerType
executionId
conversationId (필요 시)
tenantId
userId
roomId
requestedAt
priority
workloadType
contextRef
metadata
```

필수 필드는 구현 과정에서 축소하거나 확장할 수 있다.

Status: Designed

---

## 3. Not a Raw Context Carrier

AiTask를 대화 원문 운반 객체로 만들지 않는다.

권장:

```text
AiTask
  - task metadata
  - trigger / execution correlation
  - contextRef
  - small metadata
```

지양:

```text
AiTask
  - full chat history
  - large client local data
  - raw file content
```

대용량 Context는 Context Store, read-only DB / Redis, 원본 저장소, 또는 `realtime-message-service`, `user-service`, `auth-service`, `file-service` API에서 조회하는 방향으로 둔다.

Status: Designed

---

## 4. Trigger Event

NATS JetStream은 재처리 가능성이 필요한 Business Event / AI Trigger 전달에 사용한다.

예:

```text
user.status.returned
realtime.room.entered
message.urgent.detected
```

Trigger Event 후보 필드:

```text
eventId
triggerId
eventType
tenantId
userId
roomId
occurredAt
expiresAt 또는 maxAge
sourceService
sourceInstanceId (필요 시)
sessionId (필요 시)
conversationId (필요 시)
metadata
```

AI Trigger는 시간 민감도가 높을 수 있다. 오래된 Trigger가 재처리되었을 때 의미 없는 AI 결과가 생성되지 않도록 `expiresAt` 또는 `maxAge`를 둔다.

Status: Designed

---

## 5. Trigger-to-Task Condition

omni-ai-server에는 실행이 확정된 Task만 전달한다.

```text
Server-triggered path                  Client-explicit path
Business Event                         Client Request
        ↓                                      ↓
NATS JetStream (필요 시)                Permission / Context Scope / Policy
        ↓                                      ↓
ai-orchestrator 또는 Service Handler    Service Handler / ai-orchestrator
        ↓                                      ↓
SKIP / EXECUTE                         SKIP / EXECUTE
        ↓                                      ↓
        └──── EXECUTE인 경우에만 AiTask 생성 ──┘
        ↓
omni-ai-server
```

NATS JetStream은 모든 AI 실행이 확정된 Task만 받는 실행 큐가 아니다. 재처리와 ACK가 필요한 Business Event / AI Trigger를 전달하는 경계이다.

Status: Designed

---

## 6. NATS JetStream Role

NATS JetStream은 Messenger 핵심 흐름과 AI Trigger 처리를 분리한다.

- Business Event / AI Trigger 전달
- Durable Consumer
- ACK
- retry
- Consumer 장애 후 재처리
- Trigger expiration 정책 적용

Status: Designed

---

## 7. omni-ai-server Execution Boundary

`omni-ai-server`는 실행이 확정된 AiTask를 처리한다.

```text
AiTask
  ↓
Task Router
  ↓
Context Resolution
  ↓
Workflow Execution 또는 Agent Execution
  ↓
LLM / Model / Tool
  ↓
Validation
  ↓
Structured Result
```

Business Rule과 Trigger Policy 판단은 omni-ai-server 호출 이전에 끝나야 한다.

Status: Designed

---

## 8. Conversation State Boundary

`ai-orchestrator`는 Product Application을 위한 Conversation Metadata를 관리한다.

```text
conversationId
userId
tenantId
workflowType
title
createdAt
updatedAt
status
archived
access policy
```

`omni-ai-server`는 다음 LLM 호출을 위한 Runtime State를 관리한다.

```text
User Turn
Assistant Turn
Conversation History
Conversation Summary
LangGraph Checkpoint
Tool State
Agent State
Prompt Context
```

`realtime-message-service`의 WebSocket Session과 `conversationId`는 다른 lifecycle을 가진다.

Status: Designed

---

## 9. Core NATS Result Routing

Core NATS는 실시간 AI Result / LLM Streaming Routing에 사용한다.

예:

```text
ai.stream.exec-123
ai.result.realtime.instance-03
```

AI 결과 자체의 영속성보다 현재 연결된 사용자에게 빠르게 결과를 전달하는 것이 중요한 경우 Core NATS가 적합하다.

Trigger 당시 Instance가 최종 Delivery 대상이라고 가정하지 않는다.

```text
Trigger 시점
userA → realtime-message-service #1

AI 처리 중 reconnect
userA → realtime-message-service #6

Result Routing
ai-orchestrator
→ Session Registry 확인
→ Core NATS
→ realtime-message-service #6
→ WebSocket Push
```

`sourceInstanceId`는 correlation 정보로 사용할 수 있지만, 최종 Routing의 절대적인 Source of Truth로 사용하지 않는다.

Status: Designed

---

## 10. Stream Event

LLM Stream을 `ai-orchestrator`가 token-by-token proxy하지 않는 방향을 우선한다.

```text
omni-ai-server
  ↓
Core NATS
  ↓
realtime-message-service
  ↓
Client
```

Stream Event 후보:

```text
START
STATUS
DELTA
TOOL_CALL
COMPLETED
FAILED
```

공통 필드 후보:

```text
executionId
conversationId
type
sequence
content
metadata
```

Status: Designed

---

## 11. Workload Split

초기에는 Orchestrator consumer와 omni-ai-server 실행 자원을 단순하게 시작한다.

향후 실제 부하 특성이 확인되면 workload 기준으로 분리할 수 있다.

```text
REALTIME
NORMAL
BACKGROUND
HEAVY
```

Event Type은 Business 확장 단위이고, Queue / Consumer / Worker Pool은 자원 격리 단위이다.

Status: Planned

---

## 12. Execution Policy

Execution Policy는 이미 승인된 Task의 실행 안정성을 다룬다.

- retry
- timeout
- rate limit
- duplicate execution 방지
- backpressure
- failure handling
- result routing timeout

Status: Designed
