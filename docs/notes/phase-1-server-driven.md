# Phase 1. Server-driven 기본 구조 Notes

> Role: Phase 1 개발 중 구현 메모, 열린 질문, 회고를 기록하는 문서  
> Status: Planned

---

## Scope

Phase 1의 목표는 Server-driven 기본 구조를 작게 E2E로 검증하는 것이다.

대상 흐름:

```text
ROOM_ENTERED
  ↓
ConversationStartService
  ↓
Business Policy
  ↓
AiTask(CONVERSATION_START)
  ↓
AI Task Queue
  ↓
Omni AI Task Router
  ↓
Workflow Execution
  ↓
Structured Result
  ↓
Client Suggestion
```

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

아직 열린 질문은 없다.

---

## Retrospective

Phase 1 완료 후 다음을 기록한다.

- 계획과 달라진 점
- 재사용 가능한 구조
- 다음 Phase 전에 정리할 점
- ADR로 승격할 결정

