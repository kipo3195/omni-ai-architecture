# AI Orchestrator Implementation

Status: Planned

---

## Implementation Areas

| Area | 기록 범위 |
| --- | --- |
| [Spring Architecture](spring-architecture/README.md) | Spring 서비스 구성, 패키지 경계, 공통 기반 |
| [Server Trigger](server-trigger/README.md) | Business Event 수신부터 실행 판단, Context Assembly, AiTask 생성까지 |
| [Client Request](client-request/README.md) | Client AI 요청 수신과 공통 실행 흐름 |
| [Tool Runtime](tool-runtime/README.md) | Tool lifecycle, 권한 확인, dispatch와 결과 연결 |

각 영역에는 공통 구현 구조를 기록하고, Conversation Start와 같은 개별 기능은 적용 사례로 연결한다.

---

## Scope

AI Orchestrator는 Spring Boot 기반 신규 Application Service로 구현할 계획이다.

주요 책임:

- Business Event 수신
- Client AI Request 수신
- Trigger Policy 판단
- Context Assembly
- AiTask 생성
- Omni AI Server 호출
- Execution Correlation
- Conversation Metadata 관리
- Tool Runtime lifecycle 관리

AI Orchestrator는 WebSocket Service나 TCP Realtime Service의 connection / session owner가 아니다. 또한 LLM prompt 실행, LangGraph checkpoint, user / assistant turn history를 직접 소유하지 않는다.

---

## Internal Structure

```text
Client / Event
  ↓
TriggerController / EventConsumer
  ↓
AiExecutionUseCase
  ├─ TriggerPolicy
  ├─ ContextAssembler
  ├─ AiTaskFactory
  ├─ ExecutionCorrelationStore
  └─ OmniAiClient
```

---

## Component Responsibility

| Component | Responsibility |
| --- | --- |
| TriggerController | Client explicit AI request 수신 |
| EventConsumer | Business Event / AI Trigger 수신 |
| AiExecutionUseCase | AI 실행 판단과 실행 흐름 조정 |
| TriggerPolicy | feature, permission, cooldown, dedup 판단 |
| ContextAssembler | 각 Domain Service가 소유한 데이터를 조회하고, AI 실행에 필요한 Context 형태로 조합 |
| AiTaskFactory | Omni AI Server에 전달할 AiTask 생성 |
| ExecutionCorrelationStore | triggerId, taskId, executionId 기준 실행 상관관계 저장 |
| OmniAiClient | Omni AI Server 호출, timeout, error mapping |

---

## Channel Integration

AI Orchestrator는 WebSocket Service와 TCP Realtime Service가 동일한 AI Use Case와 규격을 사용할 수 있도록 channel-independent application boundary를 제공한다.

```text
WebSocket Client
→ WebSocket Service
→ AI Orchestrator
→ Omni AI Server

TCP Client
→ TCP Realtime Service
→ AI Orchestrator
→ Omni AI Server
```

WebSocket Service와 TCP Realtime Service는 channel-specific connection, session, delivery를 담당한다.

AI Orchestrator는 다음 공통 책임을 담당한다.

- AI execution decision
- Trigger Policy
- Context Assembly
- AiTask schema
- executionId / taskId / triggerId correlation
- result routing contract

---

## Non-responsibilities

- WebSocket connection ownership
- TCP connection ownership
- Client session source of truth
- Domain state source of truth
- LLM prompt execution
- LangGraph runtime state
- User / assistant turn history
- High-frequency token streaming proxy
