# Omni AI Server Implementation

Status: Implemented

---

## Scope

Omni AI Server는 Python 기반 AI Runtime으로 구현하였다.

주요 책임:

- AiTask 처리
- Workflow / Agent 실행
- Prompt 구성
- LLM 호출
- Tool Decision
- Conversation Runtime History 관리
- Agent State / Checkpoint 관리

---

## Runtime Boundary

Omni AI Server는 실행이 확정된 AiTask만 처리한다.

```text
AiTask
  ↓
Task Router
  ↓
Context Resolution
  ↓
Workflow Execution / Agent Execution
  ↓
LLM / Tool Decision
  ↓
Structured Result / Stream Event
```

Business Rule, Trigger Policy, permission, cooldown, dedup 판단은 Omni AI Server 호출 이전에 끝난다.

---

## Non-responsibilities

- WebSocket / TCP connection ownership
- Messenger client session ownership
- Product-facing conversation metadata ownership
- Trigger Policy 판단
- Domain Service state ownership
- Channel-specific result delivery

Omni AI Server는 channel 종류를 알지 않고, AI Orchestrator가 전달한 AiTask와 tool result를 기준으로 AI Runtime 실행에 집중한다.
