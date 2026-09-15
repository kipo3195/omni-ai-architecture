# Phase 1. Server-driven 기본 구조 Notes

> Role: Phase 1 개발 중 구현 메모, 열린 질문, 회고를 기록하는 문서
> Status: Planned

---

## Scope

Phase 1의 목표는 Service Boundary와 Server-driven 기본 구조, `ai-orchestrator` boundary를 작게 E2E로 검증하는 것이다.

대상 흐름:

```text
ROOM_ENTERED
  ↓
realtime-message-service
  ↓
Business Policy
  ↓
AiTask(CONVERSATION_START)
  ↓
omni-ai-server Task Router
  ↓
Workflow Execution
  ↓
Structured Result
  ↓
ai-orchestrator 또는 realtime-message-service delivery
  ↓
Client Suggestion
```

Cross-domain Use Case와 NATS JetStream / Core NATS Result Routing은 Phase 2에서 검증한다.

---

## Implementation Notes

아직 작성된 구현 메모는 없다.

기록 예:

```md
## YYYY-MM-DD. 메모 제목

### Context
무엇을 구현하던 중이었는가.

### Note
어떤 판단이나 발견이 있었는가.

### Follow-up
추가로 확인할 것은 무엇인가.
```

---

## Open Questions

- `ai-orchestrator`를 Phase 1에서 thin boundary로 둘지, Conversation Start는 service-local handler로만 둘지
- Session Registry와 Result Routing을 Phase 1에서 어느 수준까지 stub 처리할지
- `triggerId / taskId / executionId / conversationId` correlation 필드를 Phase 1 계약에 포함할지

---

## Retrospective

Phase 1 완료 후 다음을 기록한다.

- 계획과 달라진 점
- 재사용 가능한 구조
- 다음 Phase 전에 정리할 점
- ADR로 승격할 결정
