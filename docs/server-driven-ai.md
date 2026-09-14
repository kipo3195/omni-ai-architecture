# Server-driven AI

> Role: Business Event / Client Request가 Java Messenger Backend의 Policy를 거쳐 AiTask로 전환되는 흐름을 설명하는 문서  
> Status: Designed  
> 이 문서는 Java Messenger Backend가 Business Event와 Client Request를 AI 실행 후보로 받아 `SKIP / EXECUTE`를 판단하는 구조를 설명한다.

---

## 1. Purpose

Server-driven AI의 핵심은 AI Runtime이 먼저 기능을 시작하지 않는다는 점이다. Messenger Business Logic이 Event와 Context를 평가하고, AI 실행이 필요하다고 판단한 경우에만 AiTask를 만든다.

```text
Business Event 또는 Client Request
        ↓
Java Messenger Backend
        ↓
Feature Service / Handler
        ↓
Business Policy
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
- `LABEL_MATCHED`
- `SCHEDULE_TRIGGERED`

```text
Business Event
    ↓
Feature Service / Handler
    ↓
Feature / Permission / Cooldown / Context Policy
    ↓
SKIP or EXECUTE
```

Status: Designed

### Client-explicit path

사용자가 명시적으로 AI 기능을 요청하는 경우도 Java Messenger Backend를 거친다.

예:

- 선택 메시지 요약
- 현재 화면 기반 질문
- Draft 보조
- Client Local Context가 필요한 요청

```text
Client Request
    ↓ WebSocket
Java Messenger Backend
    ↓
Authentication / Session / Permission
    ↓
Client Context Scope 확인
    ↓
SKIP or EXECUTE
```

Client Request는 직접 Omni AI Server로 전달되지 않는다.

Status: Designed

---

## 3. Business Event와 AiTask

Business Event는 "무슨 일이 발생했는가"를 의미한다.

AiTask는 "AI가 무엇을 수행해야 하는가"를 의미한다.

```text
ROOM_ENTERED
    ↓
Business Policy
    ↓
CONVERSATION_START
```

Business Event와 AiTask를 분리하면 Event 증가와 AI 기능 증가를 독립적으로 다룰 수 있다.

Status: Designed

---

## 4. Service / Handler

Feature Service / Handler는 정책을 직접 모두 구현하는 거대한 객체가 아니라, Policy를 조합하고 실행 순서를 관리하는 Orchestration 계층이다.

책임:

- Event / Request 해석
- 필요한 Business State 조회
- Policy 조합
- `SKIP / EXECUTE` 결정
- `EXECUTE`인 경우 AiTask 생성

Status: Designed

---

## 5. Policy Composition

### Conversation Start

```text
Room Enter
  ↓
ConversationStartService
  ├─ FeatureEnabledPolicy
  ├─ CooldownPolicy
  ├─ TodayHiddenPolicy
  └─ RecentContextPolicy
  ↓
AiTask(CONVERSATION_START)
```

Status: Designed

### Urgent Message Summary

```text
OFFLINE → ONLINE
  ↓
UrgentMessageSummaryService
  ├─ AwayDurationPolicy
  ├─ UnreadMessagePolicy
  ├─ UrgentMessagePolicy
  └─ PermissionPolicy
  ↓
AiTask(URGENT_MESSAGE_SUMMARY)
```

Status: Designed

### Label Multimodal Action

```text
Label Matched
  ↓
LabelActionService
  ↓
Business Policy
  ↓
AiTask(LABEL_MULTIMODAL_ACTION)
```

Status: Designed

---

## 6. Time-based Trigger

오프라인 시각과 온라인 복귀 시각처럼 과거 상태가 필요한 경우에도 별도 Trigger Queue가 항상 필요한 것은 아니다.

```text
USER_ONLINE
  ↓
previousOfflineAt 조회
  ↓
awayDuration 계산
  ↓
Policy
  ↓
AiTask
```

시간 경과 자체가 Trigger가 되어야 하는 경우에만 Scheduler / Delayed Trigger를 고려한다.

```text
USER_OFFLINE
  ↓
Delayed Trigger / Scheduler
  ↓
N시간 후 상태 재검증
  ↓
Policy
  ↓
AiTask
```

Status: Designed

---

## 7. Business Policy와 Execution Policy

Business Policy는 Queue 이전에 끝난다.

- Feature Enable
- Permission
- Cooldown
- Today Hidden
- User State
- Room / Label / Business State
- Client Context Scope

Execution Policy는 Queue 이후의 실행 안정성을 다룬다.

- retry
- timeout
- rate limit
- duplicate execution 방지
- backpressure

Status: Designed
