# Server-driven AI

> Role: Business Event / Client Request가 Messenger Service와 AI Orchestrator의 Policy를 거쳐 AiTask로 전환되는 흐름을 설명하는 문서
> Status: Designed
> 이 문서는 Messenger Application 영역이 Business Event와 Client Request를 AI 실행 후보로 받아 `SKIP / EXECUTE`를 판단하는 구조를 설명한다.

---

## 1. Purpose

Server-driven AI의 핵심은 AI Runtime이 먼저 기능을 시작하지 않는다는 점이다. Messenger Business Logic과 Business State가 Event와 Context를 평가하고, AI 실행이 필요하다고 판단한 경우에만 AiTask를 만든다.

```text
Business Event 또는 Client Request
        ↓
WebSocket Service / user-service
        ↓
AI Orchestrator
        ↓
Business Policy / Trigger Policy
        ↓
SKIP or EXECUTE
```

Status: Designed

---

## 2. Entry Points

### Server-triggered path

Messenger 내부의 Business Event가 AI 실행 후보가 된다.

예:

- `ROOM_ENTERED`
- `USER_STATUS_CHANGED`
- `USER_RETURNED`
- `USER_BECAME_IDLE`
- `MESSAGE_CREATED`
- `LABEL_MATCHED`
- `SCHEDULE_TRIGGERED`

```text
Business Event
    ↓
WebSocket Service 또는 user-service
    ↓
NATS JetStream (필요 시)
    ↓
AI Orchestrator
    ↓
Feature / Permission / Cooldown / Context Policy
    ↓
SKIP or EXECUTE
```

WebSocket Service와 user-service는 자신이 소유한 Business Event와 routing context를 제공한다. AI 실행 판단, Trigger Policy, Context Assembly, AiTask 생성은 `AI Orchestrator`에서 공통화한다.

Status: Designed

### Client-explicit path

사용자가 명시적으로 AI 기능을 요청하는 경우는 `WebSocket Service`를 거친다.

예:

- 선택 메시지 요약
- 현재 화면 기반 질문
- Draft 보조
- `/요약`, `/일정`, `/번역`
- Client Local Context가 필요한 요청

```text
Client Request
    ↓ WebSocket
WebSocket Service
    ↓
Authentication / Session / Permission
    ↓
Client Context Scope 확인
    ↓
AI Orchestrator
    ↓
SKIP or EXECUTE
```

Client Request는 직접 `Omni AI Server`로 전달되지 않는다.

Client Request를 처리한 `WebSocket Service` instance가 최종 WebSocket delivery owner라고 가정하지 않는다. `enterRoom` 같은 요청에서는 `roomSessionId`를 만들고 현재 `connectionId` / `ownerInstanceId`와 연결하되, AI Result push 시점에는 Session Registry에서 현재 owner를 다시 확인한다.

Status: Designed

---

## 3. Business Event와 AiTask

Business Event는 "무슨 일이 발생했는가"를 의미한다.

AiTask는 "AI가 무엇을 수행해야 하는가"를 의미한다.

```text
USER_RETURNED
    ↓
Business Policy / Trigger Policy
    ↓
RETURNED_MESSAGE_TOPICS
```

Business Event와 AiTask를 분리하면 Event 증가와 AI 기능 증가를 독립적으로 다룰 수 있다.

Status: Designed

---

## 4. Service / Orchestrator Boundary

WebSocket Service와 domain service는 connection, session, domain state의 Source of Truth를 유지한다.

Service 책임:

- Event / Request 해석
- Authentication / Session / Permission 1차 확인
- connectionId / roomSessionId / routingRef correlation 제공
- Domain state 조회 API 또는 Projection 제공
- Business Event / AI Trigger 발행

`AI Orchestrator` 책임:

- Trigger Policy 판단
- 필요한 Business State 조회 조정
- Context Assembly
- `SKIP / EXECUTE` 결정
- `EXECUTE`인 경우 AiTask 생성
- Trigger / Execution correlation
- Cooldown / duplicate trigger 방지
- AI Execution State 관리
- Conversation Metadata 관리
- `Omni AI Server` 호출
- Result Routing / Realtime Target Resolution

`AI Orchestrator`가 가지지 않는 책임:

- 사용자 상태의 Source of Truth
- 메시지 상태의 Source of Truth
- 인증 상태의 Source of Truth
- 파일 상태의 Source of Truth
- User / Assistant Turn History
- LLM Prompt Runtime State
- LangGraph Checkpoint

Status: Designed

---

## 5. Policy Composition

### Conversation Start

```text
Room Enter
  ↓
WebSocket Service
  ├─ roomSessionId 생성
  ├─ connectionId / ownerInstanceId 연결
  ↓
AI Orchestrator
  ├─ FeatureEnabledPolicy
  ├─ CooldownPolicy
  ├─ TodayHiddenPolicy
  ├─ RecentContextPolicy
  └─ Context Assembly
  ↓
AiTask(CONVERSATION_START)
```

Status: Designed

### Client Explicit AI Request

```text
Client AI Request
  ↓
WebSocket Service
  ├─ Authentication / Session / Permission
  ├─ Client Context Scope 확인
  └─ connectionId / roomSessionId correlation
  ↓
AI Orchestrator
  ├─ Trigger Policy
  ├─ Context Assembly
  └─ Execution Correlation
  ↓
AiTask(CLIENT_REQUESTED_AI)
```

명시적 Client 요청에서 생성된 AiTask도 `triggerId` / `taskId` / `executionId` 기준으로 실행한다. Result delivery는 request 처리 instance가 아니라 Session Registry의 현재 `ownerInstanceId` 기준으로 결정한다.

Status: Designed

### Returned Message Topics

```text
USER_RETURNED
  ↓
user-service
  ↓
NATS JetStream
  ↓
AI Orchestrator
  ├─ AwayDurationPolicy
  ├─ ReceivedMessagePolicy
  ├─ DuplicatePeriodPolicy
  └─ PermissionPolicy
  ↓
AiTask(RETURNED_MESSAGE_TOPICS)
```

Status: Designed

### Label Multimodal Action

```text
Label Matched
  ↓
user-service
  ↓
AI Orchestrator
  ↓
Business Policy
  ↓
AiTask(LABEL_MULTIMODAL_ACTION)
```

Status: Designed

---

## 6. Time-based Trigger

오프라인 시각과 온라인 복귀 시각처럼 과거 상태가 필요한 경우에도 모든 판단을 별도 Trigger Queue로 미룰 필요는 없다.

```text
USER_ONLINE
  ↓
user-service
  ↓
previousOfflineAt 조회
  ↓
awayDuration 계산
  ↓
AI Trigger 발행
  ↓
AI Orchestrator
  ↓
Policy
  ↓
AiTask
```

여러 Service의 상태를 조합하거나 재처리가 필요한 Trigger는 NATS JetStream을 통해 `AI Orchestrator`로 전달한다.

```text
USER_RETURNED
  ↓
NATS JetStream
  ↓
AI Orchestrator
  ↓
Unread / Room / File / Tenant Policy 조합
  ↓
AiTask
```

시간 경과 자체가 Trigger가 되어야 하는 경우에는 Scheduler / Delayed Trigger를 고려한다.

Status: Designed

---

## 7. Business Policy와 Execution Policy

Business Policy와 Trigger Policy는 `Omni AI Server` 호출 이전에 끝난다.

- Feature Enable
- Permission
- Cooldown
- Today Hidden
- User State
- Room / Label / Business State
- File Access
- Client Context Scope
- Tenant Policy
- Duplicate Trigger 방지

Execution Policy는 실행 안정성을 다룬다.

- retry
- timeout
- rate limit
- duplicate execution 방지
- backpressure

Status: Designed

---

## 8. Context 조회 전략

`AI Orchestrator`가 모든 Service를 매번 동기 RPC로 호출하는 synchronous fan-out 구조는 지양한다.

각 Domain Service가 소유한 AI Context는 Service API / gRPC 또는 명시적으로 계약된 Projection을 통해 조회한다. 일정 수준의 stale을 허용할 수 있는 Context도 `AI Orchestrator`가 Domain DB / Redis를 직접 읽는 방식이 아니라, Service가 제공하는 read API나 event-driven Projection으로 제공하는 방향을 우선한다.

예:

- 최근 메시지
- unread count
- presence snapshot
- 최근 room 목록

`AI Orchestrator`가 직접 DB / Redis를 사용할 수 있는 범위는 자신이 소유한 실행 조정 상태다.

예:

- cooldown
- deduplication key
- execution correlation
- AI task status / execution metadata
- retry / backoff state
- conversation metadata

권한, 강제 차단 상태, Tenant Policy, 파일 접근 권한처럼 강한 정합성이 필요한 판단은 해당 Service API / gRPC를 통해 확인한다.

원칙:

```text
WebSocket Service / user-service / auth-service / file-service
= Source of Truth

AI Orchestrator
= Owner of AI Coordination State / Coordination Boundary
```

Status: Designed
