# Tool Calling

Status: Planned

Omni AI Server의 Tool Decision과 실행 재개 구조를 기록한다.

Runtime의 Tool 요청, AI Orchestrator와의 계약, Tool 결과 수신 후 재개 흐름을 실제 구현 기준으로 정리한다.

Tool Calling은 Task Execution 도중 발생하는 suspend/resume 확장 흐름이며 독립적인 최상위 실행 경로가 아니다. 정해진 순서대로 Service API를 호출하는 Workflow는 Tool Calling으로 분류하지 않는다. LLM이 실행 중 Tool 필요 여부와 종류를 선택할 때 이 구조를 사용한다.

## Responsibility

- LLM에 제공할 Tool schema binding
- LLM Tool Decision과 argument 해석
- Tool Request 생성
- execution checkpoint 저장과 `WAITING_TOOL_RESULT` 전환
- Tool Result 수신과 원래 execution correlation
- checkpoint 복원과 Workflow / Agent resume
- 반복 Tool Call, timeout, failure, cancellation 처리

AI Orchestrator의 Tool Runtime은 registry, schema validation, permission, dispatch, retry와 result normalization을 소유한다. Omni AI Server는 Tool을 선택하고 결과를 사용해 AI 실행을 이어간다.

## Base Flow

```text
Workflow / Agent
→ LLM Tool Decision
→ Tool Request
→ Save Checkpoint / Wait
→ AI Orchestrator Tool Runtime
→ Tool Result
→ Restore Checkpoint / Resume
```

## First Application

[Phase 5 Tool-assisted AI](../../use-cases/05-tool-assisted-ai/README.md)에서 조회 중심 Server Tool부터 검증한다.
