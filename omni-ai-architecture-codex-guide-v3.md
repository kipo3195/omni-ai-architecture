# Omni AI Architecture Codex Update Guide v3

## 목적

현재 `omni-ai-architecture` 저장소의 기존 문서 구조와 톤을 유지하면서,
Legacy WS Service의 실제 분리 계획과 AI Orchestration 구조를 반영한다.

이번 문서에서는 서비스 명칭을 다음과 같이 고정한다.

```text
realtime-message-service
user-service
auth-service
file-service
ai-orchestrator
omni-ai-server
```

Codex는 기존 README / docs를 먼저 읽고,
현재 문서의 구조, 용어, 표현 방식을 최대한 유지하면서
아래 설계 방향을 최소 변경으로 반영한다.

이 문서는 새 Architecture 문서를 처음부터 다시 쓰기 위한 지시서가 아니다.

---

# 1. Repository 분석 우선

작업 시작 전 반드시 다음을 확인한다.

```text
README.md
docs/**
*.md
```

특히 아래 내용을 먼저 파악한다.

```text
- 현재 Omni AI의 목표
- Business Backend와 Omni AI의 역할 구분
- Trigger / Context / Workflow 설명
- Conversation Start Reference Workflow
- Server-driven AI 방향
- Client Integration
- Agent Execution
- SaaS / Tenant 관련 설명
- 비용 최적화 / Semantic Cache
- Architecture Diagram
- Milestone / Roadmap
```

기존에 이미 정의된 개념과 충돌하는 용어를 임의로 추가하지 않는다.

---

# 2. 현재 Legacy WS의 문제

현재 Legacy WS Service는 하나의 프로세스에서 너무 많은 책임을 가진다.

예:

```text
Legacy WS
├─ WebSocket Connection
├─ 채팅
├─ 쪽지
├─ 알림
├─ 사용자 상태
├─ 사용자 정보
├─ 조직 / 클래스
├─ 인증
├─ 파일
├─ REST API
└─ 기타 Business Logic
```

여기에 다음 책임까지 추가될 경우
서비스 응집도가 더 낮아지고 변경 영향 범위가 커질 수 있다.

```text
AI Trigger
AI Context Assembly
AI Execution
LLM Streaming
AI Result Routing
Conversation Management
```

따라서 AI 기능 추가를 계기로
기존 Legacy WS의 책임을 역할별 서비스로 점진 분리한다.

---

# 3. Target Service Boundary

서비스 명칭과 역할은 다음 기준으로 문서화한다.

---

## 3.1 realtime-message-service

실시간 메시징과 관련된 책임을 담당한다.

```text
- /ws WebSocket 연결
- WebSocket Connection / Session 관리
- 채팅 발신
- 채팅 읽음 처리
- 채팅 삭제
- 채팅방 변경
- Line Key 관련 처리
- 쪽지 발신
- 쪽지 확인
- 쪽지 삭제
- 쪽지 회수
- 알림 확인
- 알림 삭제
- 알림 조회
- 채팅 이력 조회 REST
- 쪽지 이력 조회 REST
- 알림 이력 조회 REST
- unread count 계산 또는 조회 위임
- Redis WS_PACKET publish / subscribe
- 접속 중인 Session으로 실시간 Push
- AI Streaming Result를 Client로 전달
```

핵심 역할:

> Client와의 Realtime Connection 및 Message Delivery의 Owner.

AI 결과도 별도 AI 전용 WebSocket 서버를 만들기보다
`realtime-message-service`의 실시간 전달 기능을 사용한다.

---

## 3.2 user-service

사용자 및 사용자 상태 관련 책임을 담당한다.

```text
- 사용자 정보 조회
- 사용자 정보 변경
- 조직 조회
- 클래스 조회
- Rule 조회
- Rule Cache 관리
- 사용자 상태 변경
- Presence / User Activity State
- 친구 메모
- 라벨
- 주소록
```

라벨 / 주소록의 최종 분리 여부는 향후 결정 가능하다.

현재 Architecture 문서에서는 다음처럼 표현한다.

```text
user-service
├─ User Profile
├─ Organization / Class
├─ Rule / Cache
├─ Presence
├─ Friend Memo
└─ Label / Address Book (분리 가능성 검토)
```

핵심 역할:

> User State와 User-related Business State의 Owner.

Server-driven AI Trigger 중 사용자 상태 변화와 관련된 Event는
`user-service`가 발생시킨다.

예:

```text
USER_RETURNED
USER_BECAME_IDLE
USER_STATUS_CHANGED
```

---

## 3.3 auth-service

인증과 보안 정책을 담당한다.

```text
- login
- token generate
- token reIssue
- token validation
- password challenge
- cookie 정책
- JWT 정책
- 사용자 / Tenant Authentication Context
```

핵심 역할:

> Authentication과 Token Policy의 Owner.

AI Orchestrator는 인증 상태를 직접 소유하지 않는다.

---

## 3.4 file-service

파일 관련 책임을 분리한다.

예:

```text
- 파일 업로드
- 파일 다운로드
- 파일 메타정보
- 파일 권한
- 파일 조회
- Attachment 관련 처리
```

현재 Legacy WS 내부 파일 처리 로직의 실제 범위를 분석하여
구체적인 책임을 기존 문서에 맞게 보완한다.

AI가 파일을 Context로 사용하는 경우에도
파일 상태와 권한의 Source of Truth는 `file-service`에 둔다.

---

# 4. 서비스 분리의 핵심 원칙

서비스 분리의 목적은 Spring 사용 자체가 아니다.

문서에서는 반드시 다음 순서로 설명한다.

```text
1. Legacy WS의 책임이 과도하게 집중되어 있음
2. AI 기능 추가 시 결합도가 더 커질 가능성이 있음
3. Realtime / User / Auth / File 책임을 분리할 필요가 있음
4. 신규 Java Service 구현 기술로 Spring Boot를 검토
```

즉:

> Framework가 Architecture의 목적이 아니라,
> Responsibility Separation이 먼저다.

---

# 5. AI Orchestrator 도입 이유

서비스를 분리하더라도 AI 기능에서는 여러 서비스의 상태를 조합해야 할 수 있다.

예:

```text
user-service
- 사용자가 자리비움 상태에서 복귀

realtime-message-service
- unread message 존재
- 긴급 쪽지 존재
- 최근 채팅 Context 존재

auth-service
- AI Feature 접근 가능

file-service
- 관련 Attachment 존재
```

이 기능을 어느 한 서비스가 직접 책임지면
서비스 간 직접 의존이 증가한다.

예:

```text
user-service
→ realtime-message-service 조회
→ auth-service 조회
→ file-service 조회
→ Omni AI 호출
```

이 구조가 반복되면
`user-service`가 Cross-domain AI Application 역할까지 가지게 된다.

따라서 Cross-domain AI Use Case를 조정하기 위한
`ai-orchestrator`를 둔다.

---

# 6. ai-orchestrator 책임

`ai-orchestrator`는 새로운 Business Domain의 Owner가 아니다.

역할은 AI Use Case Coordination이다.

```text
- Server-driven AI Trigger 수신
- Client-driven AI Request 수신
- Cross-domain AI Use Case 판단
- Trigger / Execution Correlation
- Context Assembly
- Cooldown
- Duplicate Trigger 방지
- Tenant / Feature Policy 연계
- AI Execution State
- Conversation Metadata 관리
- Result Routing
- Realtime Target Resolution
```

담당하지 않는 책임:

```text
- 사용자 상태의 Source of Truth
- 메시지 상태의 Source of Truth
- 인증 상태의 Source of Truth
- 파일 상태의 Source of Truth
- User / Assistant Turn History
- LLM Prompt Runtime State
- LangGraph Checkpoint
```

핵심 정의:

> `ai-orchestrator`는 여러 Messenger Service의 상태를 조합해야 하는 AI Use Case의 Application / Coordination Boundary다.

---

# 7. omni-ai-server 책임

`omni-ai-server`는 Python 기반 AI Runtime 영역이다.

현재 방향:

```text
Python
FastAPI
LangGraph
LLM Provider
```

책임:

```text
- Workflow 실행
- Prompt 구성
- LLM 호출
- Tool Execution
- Validation
- Rewrite
- Semantic Retrieval
- Semantic Cache
- Conversation History 조회
- User / Assistant Turn 저장
- Conversation Summary
- LangGraph Checkpoint
- Agent State
- 다음 LLM 호출용 Context 구성
```

핵심 원칙:

> ai-orchestrator는 Conversation을 제품 관점에서 관리하고,
> omni-ai-server는 Conversation을 모델 관점에서 기억한다.

---

# 8. 전체 Target Architecture

문서의 Architecture Diagram은 다음 의미가 드러나도록 수정한다.

```text
                         Messenger Client
                                │
                                │ WebSocket / REST
                                ▼
                  ┌──────────────────────────┐
                  │ realtime-message-service │
                  │                          │
                  │ WebSocket                │
                  │ Chat / Note / Alert      │
                  │ History REST             │
                  │ Realtime Push            │
                  └────────────┬─────────────┘
                               │
              ┌────────────────┼──────────────────┐
              │                │                  │
              ▼                ▼                  ▼
       ┌────────────┐    ┌────────────┐     ┌────────────┐
       │user-service│    │auth-service│     │file-service│
       └─────┬──────┘    └────────────┘     └────────────┘
             │
             │ Business Event
             ▼
       NATS JetStream
             │
             ▼
      ┌────────────────┐
      │ ai-orchestrator│
      │                │
      │ Policy         │
      │ Context        │
      │ Correlation    │
      │ Conversation   │
      │ Metadata       │
      │ Result Routing │
      └───────┬────────┘
              │
              ▼
       ┌────────────────┐
       │ omni-ai-server │
       │                │
       │ LangGraph      │
       │ LLM            │
       │ Conversation   │
       │ Runtime        │
       └───────┬────────┘
               │
          Streaming Result
               │
               ▼
           Core NATS
               │
               ▼
      realtime-message-service
               │
               ▼
         Messenger Client
```

기존 Architecture Diagram이 있다면
전체를 새로 교체하지 않고 기존 구조에 위 경계를 반영한다.

---

# 9. AI 기능 모델 1 — Server-driven AI

예:

```text
사용자 상태 감지
→ 긴급 쪽지가 있다면 Topic 추출
→ Client에 실시간 전달
```

흐름:

```text
user-service
    │
    │ USER_RETURNED
    ▼
NATS JetStream
    │
    ▼
ai-orchestrator
    │
    ├─ 긴급 쪽지 Context 확인
    ├─ unread / 최근 메시지 확인
    ├─ Policy
    ├─ Cooldown
    ├─ Dedup
    └─ AI 실행 판단
    │
    ▼
omni-ai-server
    │
    │ Topic / Summary 생성
    ▼
Core NATS
    │
    ▼
realtime-message-service
    │
    ▼
Client
```

특징:

```text
Trigger = Server Business Event
Execution = One-shot
Result = Realtime Push
```

---

# 10. AI 기능 모델 2 — Client-driven Command

예:

```text
/요약
/일정
/번역
```

흐름:

```text
Client
    │
    │ explicit request
    ▼
realtime-message-service
    │
    ▼
ai-orchestrator
    │
    ├─ User / Tenant Policy
    ├─ Workflow Selection
    ├─ Context Assembly
    └─ executionId 생성
    │
    ▼
omni-ai-server
    │
    │ LLM Streaming
    ▼
Core NATS
    │
    ▼
realtime-message-service
    │
    ▼
Client
```

특징:

```text
Trigger = Client Request
Execution = One-shot
Result = Streaming
```

기능 1과 기능 2는 Trigger Source만 다르고
AI Execution Pipeline은 최대한 공통화한다.

---

# 11. AI 기능 모델 3 — Stateful Chatbot

Chatbot은 Multi-turn Conversation이다.

```text
User
→ Assistant
→ User
→ Assistant
→ ...
```

흐름:

```text
Client
    │
    │ conversationId + message
    ▼
realtime-message-service
    │
    ▼
ai-orchestrator
    │
    ├─ Conversation Ownership
    ├─ User / Tenant Policy
    ├─ Workflow 확인
    ├─ executionId 생성
    └─ Result Routing Context
    │
    ▼
omni-ai-server
    │
    ├─ conversationId history 조회
    ├─ previous user / assistant turns
    ├─ conversation summary
    ├─ checkpoint
    ├─ current message
    └─ LLM 호출
    │
    │ Streaming Result
    ▼
Core NATS
    │
    ▼
realtime-message-service
    │
    ▼
Client
```

특징:

```text
Trigger = Client Request
Execution = Multi-turn
State = Conversation State
Result = Streaming
```

---

# 12. WebSocket Session과 AI Conversation 분리

다음 두 개념을 혼동하지 않는다.

```text
WebSocket Session
= realtime-message-service가 관리하는 Network Connection

AI Conversation
= omni-ai-server가 AI Context를 유지하는 Logical Conversation
```

예:

```text
ws-session-1
→ disconnect

ws-session-2
→ reconnect
```

하더라도:

```text
conversationId = conv-123
```

은 유지될 수 있다.

따라서 AI Conversation State를
`realtime-message-service`의 메모리나 WebSocket Session에 종속시키지 않는다.

---

# 13. Conversation Metadata vs Runtime State

## 13.1 ai-orchestrator가 관리하는 Metadata

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

이 데이터는 Messenger Product 기능을 위해 필요하다.

예:

```text
내 AI 대화 목록
최근 AI 대화
Conversation 제목
보관
삭제
접근 권한
```

---

## 13.2 omni-ai-server가 관리하는 Runtime State

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

이 데이터는 다음 LLM 추론을 위해 필요하다.

---

# 14. Conversation 목록 / 주제 조회

Client가 다음 UI를 제공한다고 가정한다.

```text
내 AI 대화

- Redis Failover 원인 분석
- 배포 실패 원인 정리
- 다음 주 일정 정리
```

이 기능은 AI Runtime 기능이 아니라 Product Application 기능이다.

흐름:

```text
Client
    │
    │ Conversation List Request
    ▼
realtime-message-service 또는 REST Entry
    │
    ▼
ai-orchestrator
    │
    ▼
Conversation Metadata Store
    │
    ▼
Conversation List
```

`title`은 Omni AI가 생성할 수 있지만
최종 저장 주체는 `ai-orchestrator`의 Metadata Store로 둔다.

---

# 15. Conversation 상세 History 조회

사용자가 Conversation을 선택하여
실제 User / Assistant Message를 보고 싶을 경우:

```text
Client
    │
    ▼
realtime-message-service / REST
    │
    ▼
ai-orchestrator
    │
    ├─ Ownership / Access 확인
    │
    ▼
omni-ai-server
    │
    ▼
Conversation History Store
    │
    ▼
User / Assistant Turns
```

핵심:

```text
ai-orchestrator
= Product Access Control

omni-ai-server
= AI Conversation History
```

---

# 16. Streaming Architecture

LLM Stream을 `ai-orchestrator`가 token-by-token proxy하지 않는 방향을 우선한다.

Control Path와 Data Path를 분리한다.

---

## 16.1 Control Path

```text
Client
    ↓
realtime-message-service
    ↓
ai-orchestrator
    ↓
omni-ai-server
```

`ai-orchestrator`가 관리:

```text
executionId
conversationId
workflow
policy
correlation
routing context
```

---

## 16.2 Data Path

```text
omni-ai-server
    ↓
Core NATS
    ↓
realtime-message-service
    ↓
Client
```

Stream Event 예:

```text
START
STATUS
DELTA
TOOL_CALL
COMPLETED
FAILED
```

예:

```json
{
  "executionId": "exec-123",
  "conversationId": "conv-123",
  "type": "DELTA",
  "sequence": 10,
  "content": "Redis의 BGSAVE는"
}
```

`realtime-message-service`는 이를
Messenger Client WebSocket Protocol로 변환하여 전달한다.

---

# 17. realtime-message-service의 AI 관련 책임

AI 기능이 추가되더라도
`realtime-message-service`가 AI Business Logic을 소유하지 않는다.

담당:

```text
- Client AI Request 수신
- AI Request를 ai-orchestrator로 전달
- Core NATS AI Stream 수신
- Target WebSocket Session 탐색
- Client에 Stream Push
```

담당하지 않음:

```text
- AI Trigger Policy
- Cross-domain Context Assembly
- LLM Workflow
- Conversation Runtime State
```

---

# 18. NATS 역할

## 18.1 JetStream

재처리 가치가 있는 Server-side Business Event / AI Trigger에 사용한다.

예:

```text
user.status.returned
message.urgent.detected
realtime.room.entered
```

역할:

```text
Durability
ACK
Retry
Consumer Recovery
```

단 AI Trigger는 시간 민감도가 있으므로:

```text
occurredAt
expiresAt
maxAge
```

를 고려한다.

---

## 18.2 Core NATS

Realtime Result / Streaming Routing에 사용한다.

예:

```text
ai.stream.exec-123
ai.result.realtime.instance-03
```

역할:

```text
Low-latency Delivery
Target Instance Routing
Realtime Stream
```

---

# 19. Result Routing

Trigger 당시 Realtime Instance와
Result 전달 시점의 Realtime Instance가 같다고 가정하지 않는다.

AI 처리 중 다음 상황이 발생할 수 있다.

```text
Reconnect
Scale-out
Scale-in
Instance Restart
Session Migration
```

따라서 최종 Routing 시점에는
현재 Session Owner를 기준으로 전달한다.

예:

```text
Trigger 시점:
userA → realtime-message-service #1

AI 처리 중 reconnect

Result 시점:
userA → realtime-message-service #6
```

결과는 #6으로 전달한다.

---

# 20. Context 조회 전략

`ai-orchestrator`가 모든 Service를 매번 동기 호출하는 구조는 지양한다.

예:

```text
ai-orchestrator
→ user-service
→ realtime-message-service
→ auth-service
→ file-service
```

문제:

```text
Latency
Failure Propagation
Timeout Complexity
Service Coupling
```

따라서 Hybrid Strategy를 허용한다.

```text
Read-heavy / stale 허용 Context
→ DB / Redis Read-only

Strong Consistency / Critical Policy
→ Domain Service API / gRPC
```

예:

```text
최근 메시지
Unread
Presence Snapshot
→ Read-only 조회 가능

인증
권한
Tenant 강제 정책
파일 접근 권한
→ Service API
```

---

# 21. Future Option — AI Context Projection

향후 직접 DB / Redis 조회나 RPC Fan-out이 복잡해질 경우
AI 전용 Read Model을 고려한다.

```text
realtime-message-service Event ─┐
user-service Event ─────────────┼─► AI Context Projection
file-service Event ─────────────┘
```

그러나 현재 필수 Architecture로 작성하지 않는다.

`Future Consideration`으로만 둔다.

---

# 22. Spring / Python 역할

현재 Target Direction:

```text
Spring
- realtime-message-service
- user-service
- auth-service
- file-service
- ai-orchestrator

Python
- omni-ai-server
- LangGraph
- LLM
- AI Conversation Runtime
```

Spring을 선택하는 이유는
Framework 자체가 목적이 아니라
Legacy WS의 Business Responsibility를 분리하고
Application Service를 구성하기 위함이다.

---

# 23. 반드시 설명해야 할 Why

## Why split Legacy WS?

```text
Realtime Transport,
User State,
Authentication,
File,
Message Business Logic이
하나의 WS Service에 과도하게 집중되어 있기 때문.
```

## Why ai-orchestrator?

```text
AI Use Case가 user-service,
realtime-message-service,
auth-service,
file-service의 상태를 조합해야 할 때
특정 Domain Service에 Cross-domain 책임을 몰아주지 않기 위해.
```

## Why not put orchestration in omni-ai-server?

```text
Omni AI가 Messenger topology,
WebSocket routing,
Authentication,
Conversation product metadata까지 알지 않도록
Application Coordination과 AI Runtime을 분리하기 위해.
```

## Why does omni-ai-server store Assistant history?

```text
User / Assistant Turn이
다음 LLM Context 구성에 필요한 AI Runtime State이기 때문.
```

## Why does ai-orchestrator store Conversation metadata?

```text
Conversation List,
Title,
Owner,
Tenant,
Access,
Archive / Delete는
Messenger Product Application 책임이기 때문.
```

## Why separate WebSocket Session and ConversationId?

```text
Network Connection Lifecycle과
Logical AI Conversation Lifecycle이 다르기 때문.
```

## Why JetStream?

```text
재처리 가치가 있는 Business Event / AI Trigger를
Consumer 장애 시 보호하기 위해.
```

## Why Core NATS?

```text
LLM Streaming 결과를
현재 연결된 realtime-message-service Instance로
낮은 지연으로 전달하기 위해.
```

## Why not proxy every token through ai-orchestrator?

```text
ai-orchestrator를 Streaming Data Plane으로 만들지 않고
Execution Control과 Coordination 책임에 집중시키기 위해.
```

---

# 24. Architecture 핵심 문장

최종 문서는 다음 내용을 자연스럽게 설명할 수 있어야 한다.

> `realtime-message-service`는 WebSocket Connection과 채팅/쪽지/알림의 실시간 전달을 담당하고,
> `user-service`는 사용자 정보와 Presence를,
> `auth-service`는 인증과 Token Policy를,
> `file-service`는 파일 상태와 접근을 소유한다.
>
> 여러 서비스의 상태를 조합해야 하는 AI Use Case는 `ai-orchestrator`가 실행을 조정하며,
> `omni-ai-server`는 Workflow, LLM Execution, Conversation History와 Agent State를 관리한다.
>
> Server-driven Event는 필요 시 NATS JetStream으로 전달하고,
> LLM Streaming Result는 Core NATS를 통해 `realtime-message-service`로 전달하여
> 최종적으로 Client WebSocket Session에 Push한다.

---

# 25. Codex 작업 순서

## Step 1. Repository 분석

현재 README / docs 구조를 읽는다.

## Step 2. 기존 Architecture와 비교

다음 표현을 찾는다.

```text
Business Backend
WS Service
Omni AI
Trigger
Context
Client Integration
Agent Execution
Conversation
Workflow
Realtime
```

## Step 3. 서비스 명칭 통일

기존의 추상적 표현:

```text
Chat Service
Presence Service
Message Service
Realtime Service
```

등이 존재할 경우
문맥에 맞게 아래 명칭으로 정리한다.

```text
realtime-message-service
user-service
auth-service
file-service
```

단, 기존 문서 의미를 훼손하면서 기계적으로 전부 치환하지 않는다.

## Step 4. Architecture 반영

다음을 반영한다.

```text
Legacy WS Responsibility Split
ai-orchestrator Boundary
Three AI Use Cases
Conversation Metadata / Runtime State
Streaming Control Path / Data Path
JetStream / Core NATS
Realtime Result Routing
```

## Step 5. Consistency Check

```text
- 서비스 명칭이 문서 전체에서 일관적인가?
- realtime-message-service에 AI Business Logic이 몰리지 않았는가?
- user-service가 Cross-domain AI 책임을 가지지 않는가?
- ai-orchestrator가 새로운 Monolith가 되지 않았는가?
- omni-ai-server가 Messenger Application 책임을 침범하지 않는가?
- WebSocket Session과 ConversationId가 구분되는가?
- Conversation Metadata와 History가 구분되는가?
- 현재 구현과 Target Architecture가 구분되는가?
```

---

# 26. 최종 결과 보고 형식

Codex 작업 완료 후 다음을 제공한다.

## Modified Files

수정 파일 목록.

## Architecture Changes

예:

```text
- Legacy WS Target Service Boundary 구체화
- realtime-message-service / user-service / auth-service / file-service 명칭 통일
- ai-orchestrator boundary 추가
- Server-driven / Client-driven / Chatbot 실행 모델 반영
- Conversation Metadata / AI Runtime State 분리
- Streaming Control Path / Data Path 분리
- JetStream Trigger / Core NATS Stream Routing 반영
```

## Important Decisions

왜 해당 경계를 선택했는지 요약한다.

## Remaining Decisions

예:

```text
- Label / Address Book을 user-service에 유지할지 별도 분리할지
- unread count 계산 주체
- Conversation Metadata Store
- AI History Store
- Session Registry 구조
- Trigger expiration 정책
- Chatbot History retention 정책
- Stream reconnect / resume 정책
- file-service의 AI Context 연계 범위
```

---

# 최종 설계 원칙

> `realtime-message-service`는 실시간 연결과 메시지 전달을,
> `user-service`는 사용자와 Presence를,
> `auth-service`는 인증을,
> `file-service`는 파일을 소유한다.
>
> `ai-orchestrator`는 여러 서비스의 상태를 조합하는 AI Use Case를 조정하고,
> `omni-ai-server`는 LLM Workflow와 Conversation Runtime을 소유한다.
>
> Domain State의 Ownership과 AI Coordination을 분리하고,
> AI 기능이 추가되더라도 기존 Realtime Service가 다시 Monolith로 비대해지지 않도록 한다.
