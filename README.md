# Omni AI Platform

> **Messenger Business Event를 AI 실행의 시작점으로 확장하고,
> 현재 `WS service`와 연동되는 `ai-orchestrator`, `omni-ai-server`를 중심으로
> 다양한 AI 기능을 공통 구조에서 실행하기 위한 플랫폼 설계**

![Status](https://img.shields.io/badge/status-designed-blue)
![Architecture](https://img.shields.io/badge/focus-architecture-informational)
![Backend](https://img.shields.io/badge/backend-Java-orange)
![AI](https://img.shields.io/badge/AI-Python-purple)

---

## Why Omni AI?

기존 Messenger AI는 대부분 `/요약`, `/번역`, `/질문`처럼 **사용자가 AI 기능을 직접 호출하는 방식**으로 동작한다.

이 방식에서는 사용자가 먼저 상황을 인지하고, 어떤 AI 기능이 필요한지 판단한 뒤, 직접 AI에게 요청해야 한다.

하지만 Messenger Backend는 이미 사용자의 업무 흐름을 이해할 수 있는 많은 정보를 가지고 있다.

* 어떤 채팅방에 진입했는지
* 얼마나 많은 메시지를 읽지 않았는지
* 사용자가 자리를 비웠다가 복귀했는지
* 어떤 메시지가 특정 Label이나 업무 조건에 해당하는지
* 어떤 대화가 장시간 응답되지 않았는지
* 어떤 Business Event가 발생했는지

즉, **AI에게 다시 상황을 설명하지 않아도 Messenger는 이미 사용자의 현재 상태와 업무 Context를 알고 있다.**

Omni AI는 이러한 정보를 AI의 입력 데이터로만 활용하는 것이 아니라,
**AI가 필요한 순간을 Messenger가 판단하고 적절한 실행으로 연결하는 것**을 목표로 한다.

```text
Omni AI

Business Event / User State / Business Data / User Request
        ↓
Messenger Business Logic
        ↓
AI execution decision
        ↓
Context-aware AI
        ↓
Popup / Push / Navigation / Voice / Suggestion
```

예를 들어,

* 채팅방 진입
  → 현재 대화 Context를 기반으로 대화 시작 문장 추천

* OFFLINE → ONLINE 상태 변경
  → 자리비움 동안 쌓인 메시지 중 확인이 필요한 내용 요약

* 여러 대화방에서 특정 업무 조건 발생
  → 중요 상황을 판단하고 사용자에게 선제적으로 안내

* Label Matched
  → AI 판단 후 Popup / Push / Navigation / Voice 등 적절한 방식으로 전달

* `/요약`, `/질문`과 같은 사용자 직접 요청
  → 동일한 AI 실행 구조를 통해 처리

이 구조에서 AI는 독립적인 기능의 시작점이 아니다.

**Messenger가 관리하는 Business Logic과 Business State를 중심으로 실행 필요성을 판단하고, AI는 필요한 Context를 해석하여 결과를 생성한다.**

Omni AI의 목적은 AI 기능을 단순히 더 많이 추가하는 것이 아니라,

> **User Request뿐 아니라 Messenger가 이미 알고 있는 Business Event, User State, Business Data를 하나의 AI 실행 구조로 연결하는 것**

이다.


---

## Current Scope

현재 Messenger Backend는 `WS service`가 인증, 파일, 실시간 채팅, 쪽지, 알림, 사용자 상태, REST API 등을 함께 처리하는 구조다.

Omni AI Architecture의 우선 범위는 이 WS service를 즉시 분리하는 것이 아니라, 기존 WS service와 연동되는 `ai-orchestrator`와 `omni-ai-server`를 정의하고 구축하는 것이다.

아래 Architecture Diagram은 현재 배포 구조가 아니라, WS service 책임을 점진적으로 분리했을 때의 **Target Architecture / Evolution Direction**이다.

서비스 분리 방향은 [Service Boundary and Migration](docs/service-boundary-and-migration.md)에서 별도로 다룬다.

---

## Target Architecture / Evolution Direction

```mermaid
flowchart TD
    A[Messenger Client] <-->|WebSocket| B[realtime-message-service]
    A -->|Client AI Request| G[ai-orchestrator]
    A --> C[service<br/> user / auth / file]

    C --> F[NATS JetStream<br/>Business Event / AI Trigger]
    B --> F
    F --> G

    G --> H{AI 실행 필요?}
    H -- No --> I[Skip]
    H -- Yes --> J[AiTask / executionId]
    J --> K[omni-ai-server]

    K <--> M[Workflow / Agent Execution]
    K -->|Tool Request| G
    G --> L[Tool Runtime<br/>inside ai-orchestrator]
    L --> N[Server Tool Adapter]
    N <--> B
    N <--> C

    L -->|Client Tool Dispatch| R[Core NATS]
    R --> O[Client Tool Delivery<br/>inside realtime-message-service]
    O --> P[Client Tool]
    P --> O
    O --> R
    R --> L
    L -->|Tool Result / Resume| K

    K --> Q[Streaming / Structured Result]
    Q --> R
    R --> B
```

### Responsibility

아래 책임표는 Target Architecture 기준이다. 현재는 일부 책임이 `WS service` 안에 함께 존재할 수 있다.

| Layer | Responsibility |
| --- | --- |
| **realtime-message-service** | WebSocket Connection, Chat / Note / Alert, History REST, Realtime Push, AI Streaming Result 전달 |
| **user-service** | User Profile, Organization / Class, Rule / Cache, Presence, Friend Memo, Label / Address Book |
| **auth-service** | Authentication, Token Policy, JWT / Cookie Policy, User / Tenant Authentication Context |
| **file-service** | File Upload / Download, Metadata, Permission, Attachment |
| **ai-orchestrator** | Cross-service AI coordination, Trigger Policy, Context Assembly, Conversation Metadata, Tool Runtime, Tool Lifecycle, Result Routing |
| **Tool Runtime** | `ai-orchestrator` 내부 책임. Tool registry, schema validation, permission, dispatch, timeout, retry, result normalization, execution resume 조정 |
| **Server Tool Adapter** | Tool Runtime의 server-side adapter. Server-side tool/context 요청을 `auth-service`, `file-service`, `user-service`, `realtime-message-service`로 중계 |
| **omni-ai-server** | Workflow / Agent 실행, Prompt / LangGraph / LLM / Tool Decision, Conversation History / Runtime State |
| **Client Tool Delivery** | `realtime-message-service` 내부 책임. Client Tool 요청 전달, Session lookup, WebSocket delivery, Client result ingress |
| **NATS JetStream** | 재처리가 필요한 Business Event / AI Trigger 전달 |
| **Core NATS** | LLM Streaming / Execution Progress / Client Tool dispatch를 현재 Session Owner 기준으로 low-latency routing |

---

## Core Principles

### 1. Business Event와 AiTask를 분리

```text
USER_RETURNED
= 무슨 일이 발생했는가

URGENT_MESSAGE_SUMMARY
= AI가 무엇을 수행해야 하는가
```

Business Event가 발생했다고 해서 항상 AI Task가 생성되는 것은 아니다. Policy를 통과해 실행이 확정된 경우에만 `AiTask`를 만든다.

### 2. Service Ownership 유지

각 Service는 자신의 Business State와 Policy의 Source of Truth를 유지한다.

```text
realtime-message-service → WebSocket, Chat / Note / Alert, Realtime Delivery
user-service             → User, Presence, Rule, Label
auth-service             → Authentication, Token Policy
file-service             → File, Attachment, Permission
```

`ai-orchestrator`는 Domain State를 소유하지 않는다. 여러 Service의 상태를 조합해야 하는 AI Use Case만 조정한다.

### 3. AI Runtime과 Messenger Application 분리

```text
ai-orchestrator
→ Product-facing Metadata, Policy, Execution Correlation, Tool Runtime, Result Routing

omni-ai-server
→ Prompt, Workflow, LangGraph, LLM Execution, Tool Decision, Conversation History, Agent State
```

`omni-ai-server`가 Messenger topology, WebSocket routing, Authentication, Product Metadata를 직접 소유하지 않도록 한다.

---

## AI Function Models

| Model | Trigger | Main Flow | Result |
| --- | --- | --- | --- |
| Server-driven AI | Server Business Event | `user-service / realtime-message-service → NATS JetStream → ai-orchestrator → omni-ai-server` | Realtime Push |
| Client-driven Command | `/요약`, `/일정`, `/번역` | `Client → ai-orchestrator → omni-ai-server` 또는 `Client → realtime-message-service → ai-orchestrator → omni-ai-server` | Streaming |
| Stateful Chatbot | `conversationId + message` | `Client → ai-orchestrator → omni-ai-server` 또는 WebSocket 경유 | Streaming / Multi-turn |

---

## Conversation Ownership

Conversation은 Product Metadata와 AI Runtime History를 분리한다.

```text
Conversation List
Client → ai-orchestrator → Conversation Metadata Store → Client

Conversation Detail History
Client → ai-orchestrator access check → omni-ai-server → History Store → ai-orchestrator → Client
```

`omni-ai-server`가 title 같은 metadata 후보를 생성할 수는 있지만, 최종 저장 owner는 `ai-orchestrator`다.

---

## Tool Runtime Boundary

Server Tool과 Client Tool은 같은 Tool lifecycle을 공유하고, 실제 실행 위치만 adapter로 분리한다.

```text
Server Tool
omni-ai-server
→ Server Tool이 필요하다고 판단
→ ai-orchestrator Tool Runtime
→ Server Tool Adapter
→ target service
→ ai-orchestrator Tool Runtime
→ omni-ai-server

Client Tool
omni-ai-server
→ Client Tool이 필요하다고 판단
→ ai-orchestrator Tool Runtime
→ Core NATS
→ realtime-message-service Client Tool Delivery
→ Client Tool
→ realtime-message-service Client Tool Delivery
→ Core NATS
→ ai-orchestrator Tool Runtime
→ omni-ai-server
```

`ai-orchestrator`는 Server Tool과 Client Tool의 lifecycle owner다. `realtime-message-service`는 Client Tool의 session lookup, WebSocket delivery, result ingress를 담당하지만, `toolCallId`, timeout, retry, result validation, execution resume 판단은 Tool Runtime에서 관리한다.

LLM token stream과 execution progress는 Tool lifecycle과 분리한다. 고빈도 streaming event는 `ai-orchestrator`가 token-by-token proxy하지 않고 `omni-ai-server → Core NATS → realtime-message-service → Client` 경로로 전달한다.

---

## Communication

```text
Business Event / AI Trigger
= NATS JetStream

Execution Control
= Client / realtime-message-service → ai-orchestrator → omni-ai-server

LLM Streaming
= omni-ai-server → Core NATS → realtime-message-service → Client

Execution Progress
= omni-ai-server → Core NATS → realtime-message-service → Client

Tool Call Control
= omni-ai-server → ai-orchestrator Tool Runtime

Server Tool Dispatch
= ai-orchestrator Tool Runtime → target service

Client Tool Dispatch
= ai-orchestrator Tool Runtime → Core NATS → realtime-message-service → Client

Client Tool Result
= Client → realtime-message-service → Core NATS → ai-orchestrator Tool Runtime → omni-ai-server resume
```

공통 correlation 후보:

```text
triggerId
taskId
executionId
conversationId
toolCallId
toolAttempt
idempotencyKey
```

---

## Documentation

| Document | Role |
| --- | --- |
| [Architecture](docs/architecture.md) | 전체 구조, Layer 책임, 실행 축 |
| [Service Boundary and Migration](docs/service-boundary-and-migration.md) | 현재 WS service 현실과 target service split / migration 방향 |
| [Server-driven AI](docs/server-driven-ai.md) | Business Event / Client Request가 AiTask로 전환되는 흐름 |
| [AiTask and Queue](docs/ai-task-and-queue.md) | AiTask, Trigger, Result Routing, Stream Event |
| [Client Integration](docs/client-integration.md) | Client Tool Delivery와 Client Context |
| [Design Decisions](docs/design-decisions.md) | 주요 설계 결정 요약 |
| [Roadmap](docs/roadmap.md) | Phase와 구현 순서 |

---

## Roadmap

```text
Phase 1
Current WS service 연동과 Omni AI 기본 구조

Phase 2
Cross-domain Trigger / Result Routing

Phase 3
Tool Runtime / Server Tool Adapter / Client Tool Delivery

Phase 4
Stateful Chatbot / Agent Runtime

Phase 5
AI Context Projection / Cache / Observability

Future
WS service responsibility split
```

---

## Summary

현재 1차 범위는 `WS service`와 연동되는 `ai-orchestrator`, `omni-ai-server` 구축이다.

Target Architecture에서는 `WS service`의 책임을 `realtime-message-service`, `user-service`, `auth-service`, `file-service`로 점진 분리한다. 여러 Service의 상태를 조합해야 하는 AI Use Case는 `ai-orchestrator`가 실행을 조정하고, `omni-ai-server`는 Workflow, LLM Execution, Conversation History와 Agent State를 관리한다.
