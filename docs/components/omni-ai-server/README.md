# Omni AI Server Implementation

Status: In Progress

---

## Implementation Areas

| Area | 기록 범위 |
| --- | --- |
| [Task Execution](task-execution.md) | AiTask 수신, 라우팅, Workflow / Agent 실행과 결과 생성 |
| [Use Case Workflows](../../use-cases/README.md) | 기능별 Prompt, AI 처리 흐름과 Structured Result |
| [Tool Calling](tool-calling.md) | Runtime의 Tool Decision, 호출과 실행 재개 흐름 |
| [Result Event](result-event.md) | executionId 기반 Result / Stream Event 생성과 lifecycle |

`Task Execution`과 `Tool Calling`에는 공통 Runtime 구조를 기록하고, 개별 AI 기능의 입출력과 Prompt 품질 조건은 해당 Use Case에 기록한다. 상위 상태는 공통 Runtime의 현재 구축 상태이며 앞으로 추가될 모든 Workflow의 완료를 뜻하지 않는다.

## Scope

Omni AI Server는 Python 기반 AI Runtime으로 구현한다.

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
- Messenger Business State ownership
- Channel-specific result delivery

Omni AI Server는 channel 종류를 알지 않고, AI Orchestrator가 전달한 AiTask와 tool result를 기준으로 AI Runtime 실행에 집중한다.
