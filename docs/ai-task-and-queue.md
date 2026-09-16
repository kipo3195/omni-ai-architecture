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
connectionId (필요 시)
roomSessionId (필요 시)
tenantId
userId
roomId
requestedAt
priority
workloadType
routingRef
contextRef
metadata
```

필수 필드는 구현 과정에서 축소하거나 확장할 수 있다.

`connectionId`, `roomSessionId`, `routingRef`는 AI 실행의 의미를 나타내는 필드가 아니라 결과 전달 대상을 해석하기 위한 correlation / routing context이다. 최종 전달 대상은 Result push 시점의 Session Registry에서 다시 확인한다.

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

대용량 Context는 AiTask payload에 직접 싣지 않고 Context Store, Service API / gRPC, 또는 명시적으로 계약된 Projection에서 조회하는 방향으로 둔다. Domain Service가 소유한 DB / Redis를 `ai-orchestrator`가 직접 읽는 방식은 기본 전략으로 두지 않는다.

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
connectionId (필요 시)
roomSessionId (필요 시)
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

## 9. Connection Ownership Registry

AI 기능은 Trigger를 처리한 instance와 실제 WebSocket이 붙어 있는 instance가 다를 수 있다는 전제를 가진다.

`realtime-message-service`는 WebSocket 연결 시 Connection Registry에 현재 연결 owner를 등록한다.

Registry 후보 key / value:

```text
connectionId
tenantId
userId
deviceId (필요 시)
roomId (필요 시)
roomSessionId (필요 시)
ownerInstanceId
connectedAt
lastSeenAt
expiresAt
capabilities
metadata
```

`connectionId`는 WebSocket connection lifecycle을 식별한다. `roomSessionId`는 특정 사용자가 특정 room에 진입해 있는 logical room presence lifecycle을 식별한다. reconnect, room 이동, multi-device 상황에서는 둘이 다르게 갱신될 수 있다.

Registry는 다음 책임을 가진다.

- WebSocket 연결 / 해제 시 현재 `ownerInstanceId` 갱신
- `enterRoom` / `leaveRoom` 시 `roomSessionId`와 `connectionId` 연결
- heartbeat / TTL 기반 stale owner 제거
- Result push 시 현재 owner instance 조회
- Client Tool Delivery 시 현재 target connection 조회

`ownerInstanceId`는 Result 전달 시점의 현재 WebSocket owner이다. `sourceInstanceId`는 Trigger를 발행하거나 Request를 처리한 instance correlation 정보일 뿐 최종 delivery source of truth가 아니다.

Status: Designed

---

## 10. Client Explicit Request and enterRoom Correlation

`enterRoom`처럼 Client가 명시적으로 호출하는 API는 AI 실행 후보를 만들 수 있지만, 해당 API를 처리한 instance가 최종 WebSocket delivery owner라고 가정하지 않는다.

권장 흐름:

```text
Client WebSocket Connect
  ↓
realtime-message-service
  ↓
Connection Registry 등록
  - connectionId
  - ownerInstanceId
  - userId / tenantId
  - capabilities

Client enterRoom
  ↓
realtime-message-service
  ↓
roomSessionId 생성
  ↓
roomSessionId ↔ connectionId ↔ ownerInstanceId 연결
  ↓
Business Policy / Trigger Policy
  ↓
AiTask 생성
  - triggerId
  - taskId
  - executionId
  - connectionId 또는 roomSessionId
  - routingRef
  ↓
ai-orchestrator
  ↓
omni-ai-server
```

`enterRoom` 외에도 선택 메시지 요약, 현재 화면 기반 질문, Draft 보조, Client Local Context가 필요한 요청은 동일한 원칙을 따른다.

```text
Client Explicit Request
  ↓
realtime-message-service
  ↓
Authentication / Session / Permission
  ↓
connectionId / roomSessionId correlation
  ↓
ai-orchestrator
  ↓
AiTask / executionId
```

AI Task는 `triggerId` / `taskId` / `executionId`로 처리하고, Client delivery는 `connectionId` / `roomSessionId` / `routingRef`를 통해 현재 owner를 다시 resolve한다.

Status: Designed

---

## 11. Server-triggered Target Resolution

Server-triggered AI 기능도 명확한 target client resolution을 거쳐야 한다.

예:

```text
USER_RETURNED
LABEL_MATCHED
SCHEDULE_TRIGGERED
MESSAGE_CREATED
```

Server Trigger는 특정 WebSocket connection에서 시작되지 않을 수 있다. 이 경우 Trigger에는 user / tenant / room / business context를 담고, `ai-orchestrator` 또는 Service-local Handler가 Result 전달 전에 현재 target connection을 resolve한다.

```text
Business Event
  ↓
NATS JetStream (필요 시)
  ↓
ai-orchestrator
  ↓
Policy / Context Assembly
  ↓
AiTask
  ↓
omni-ai-server
  ↓
Result Push
  ↓
Session Registry에서 target connection / ownerInstanceId 확인
```

Target이 없는 경우에는 기능 성격에 따라 처리한다.

- 즉시 전달이 필요한 stream: fail / timeout / fallback
- 재접속 후 확인 가능한 결과: Conversation / Notification 저장 후 later delivery
- 이미 의미가 사라진 trigger: expiresAt / maxAge 기준으로 drop

Status: Designed

---

## 12. Core NATS Result Routing

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
→ Session Registry에서 현재 ownerInstanceId 확인
→ Core NATS
→ realtime-message-service #6
→ WebSocket Push
```

`sourceInstanceId`는 correlation 정보로 사용할 수 있지만, 최종 Routing의 절대적인 Source of Truth로 사용하지 않는다.

Instance-targeted subject 후보:

```text
ai.result.realtime.{ownerInstanceId}
ai.stream.realtime.{ownerInstanceId}
```

Result Routing 단계:

```text
1. omni-ai-server가 executionId 기준으로 stream / result 생성
2. ai-orchestrator 또는 Result Router가 triggerId / taskId / executionId로 실행 상태 확인
3. routingRef, connectionId, roomSessionId, userId / roomId로 Session Registry 조회
4. 현재 ownerInstanceId 확인
5. 해당 ownerInstanceId의 realtime-message-service instance subject로 Core NATS publish
6. 해당 instance가 local WebSocket session으로 최종 전송
```

최종 전송 직전에 local session이 사라진 경우 해당 instance는 disconnect / stale routing으로 처리하고, 필요하면 Registry 재조회 또는 fallback 정책을 적용한다.

Status: Designed

---

## 13. Stream Event

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
PROGRESS
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

Stream Event는 Client UX를 위한 token delta 또는 execution progress event다. Tool lifecycle의 source of truth로 사용하지 않는다.

Tool lifecycle event는 Tool Runtime이 관리한다.

```text
TOOL_CREATED
TOOL_DISPATCHED
TOOL_PROGRESS
TOOL_COMPLETED
TOOL_FAILED
TOOL_TIMEOUT
```

Tool lifecycle event는 `toolCallId` 기준의 공식 상태 전이에 사용하고, Stream Event는 `executionId` 기준의 표시용 진행 상태에 사용한다. 따라서 Stream Event가 Tool lifecycle event보다 먼저 도착하거나 늦게 도착해도 execution correctness가 깨지지 않도록 설계한다.

Status: Designed

---

## 14. Workload Split

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

## 15. Execution Policy

Execution Policy는 이미 승인된 Task의 실행 안정성을 다룬다.

- retry
- timeout
- rate limit
- duplicate execution 방지
- backpressure
- failure handling
- result routing timeout

Status: Designed
