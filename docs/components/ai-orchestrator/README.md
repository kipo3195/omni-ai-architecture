# AI Orchestrator Implementation

Status: In Progress

---

## Implementation Areas

| Area | 기록 범위 |
| --- | --- |
| [Policy](policy.md) | Global / Feature / User Policy 조회와 AI 실행 여부 판단 |
| [Server Trigger](server-trigger.md) | Business Event 수신부터 실행 판단, Context Assembly, AiTask 생성까지 |
| [Client Request](client-request.md) | Client AI 요청 수신과 공통 실행 흐름 |
| [Schedule Management](schedule-management.md) | Schedule 검증·저장과 실행 회차·보류 상태 관리 |
| [Tool Runtime](tool-runtime.md) | Tool lifecycle, 권한 확인, dispatch와 결과 연결 |
| [Result Router](result-router.md) | execution 검증, routingRef 해석, 현재 connection owner 기준 Result Routing |
| [Spring Architecture](spring-architecture.md) | Spring Boot package, application boundary와 infrastructure adapter 구성 |

각 영역에는 공통 구현 구조를 기록하고, Conversation Start와 같은 개별 기능은 [Use Cases](../../use-cases/README.md)에서 End-to-End 흐름으로 연결한다. 상위 상태는 Orchestrator 기반을 구축 중이라는 의미이며 모든 미래 기능의 완료 상태가 아니다.

---

## Scope

AI Orchestrator는 Spring Boot 기반 신규 Application Service로 구현한다.

주요 책임:

- 현재 source owner가 확정한 Business Event 수신과 contract 검증
- Client AI Request 수신
- Trigger Policy 판단
- Global / Feature / User Policy 판단
- Context Assembly
- AiTask 생성
- Omni AI Server 호출
- Execution Correlation
- Conversation Metadata 관리
- Tool Runtime lifecycle 관리

AI Orchestrator는 `WebSocket Service (WS)`나 `TCP Service (NS)`의 connection / session owner가 아니다. 또한 LLM prompt 실행, LangGraph checkpoint, user / assistant turn history를 직접 소유하지 않는다.

---

## Internal Structure

```text
Client / Event
  ↓
TriggerController / EventConsumer
  ↓
AiExecutionUseCase
  ├─ AiExecutionPolicyService
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
| EventConsumer | 현재 source owner가 확정한 Business Event / AI Trigger 수신과 contract 검증 |
| AiExecutionUseCase | AI 실행 판단과 실행 흐름 조정 |
| TriggerPolicy | Source Event를 AI 기능으로 전환하는 기능별 조건, permission, cooldown, dedup 판단. Source-owned 상태 전이는 재판정하지 않음 |
| AiExecutionPolicyService | Global, User, Feature Policy를 적용해 AI 실행 허용 여부 판단 |
| ContextAssembler | 현재 Business owner가 소유한 데이터를 조회하고, AI 실행에 필요한 Context 형태로 조합 |
| AiTaskFactory | Omni AI Server에 전달할 AiTask 생성 |
| ExecutionCorrelationStore | 실행 식별자 기준 상관관계 저장. 전체 식별자 모델은 Proposed ADR 004 확정 후 적용 |
| OmniAiClient | Omni AI Server 호출, timeout, error mapping |
| ResultRouter | execution과 routingRef 검증, 현재 connection owner resolve, owner instance subject publish |

---

## Channel Integration

AI Orchestrator는 Messenger Client Ingress의 `WebSocket Service (WS)`와 `TCP Service (NS)`가 동일한 AI Use Case와 규격을 사용할 수 있도록 channel-independent application boundary를 제공한다.

```text
WebSocket Client
→ WebSocket Service
→ AI Orchestrator
→ Omni AI Server

Native Client
→ TCP Service (NS)
→ AI Orchestrator
→ Omni AI Server
```

`WebSocket Service (WS)`와 `TCP Service (NS)`는 channel-specific connection, session, delivery를 담당한다.

AI Orchestrator는 다음 공통 책임을 담당한다.

- AI execution decision
- Trigger Policy
- Context Assembly
- AiTask schema
- 실행 식별자 correlation. `triggerId / taskId / executionId` 분리는 Proposed ADR 004의 목표 모델
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
