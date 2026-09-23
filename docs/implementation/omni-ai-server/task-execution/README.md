# Task Execution

Status: In Progress

Omni AI Server가 AiTask를 받아 AI Runtime에서 실행하는 공통 구조를 기록한다.

Task 라우팅, Context Resolution, Workflow / Agent 실행, 결과 생성과 스트림 전달 계약을 실제 구현 기준으로 정리한다. 개별 AI 기능은 적용 사례로 다룬다.

이 문서는 Conversation Start, Topic Extraction, Weekly Report Summary 같은 기능 고유 로직을 설명하지 않는다. 모든 Workflow가 공유하는 실행 계약과 lifecycle만 다룬다.

## Responsibility

- AiTask 입력과 validation
- Task type 기반 Workflow routing
- Workflow / Agent 공통 실행 interface
- Prompt와 LLM 호출 경계
- Runtime history와 checkpoint
- Structured Result와 Stream Event 생성
- timeout, cancellation, failure 처리
- `triggerId`, `taskId`, `executionId` correlation과 logging

## Base Flow

```text
AiTask
→ Validate
→ Route
→ Resolve Runtime Context
→ Execute Workflow / Agent
→ Generate Structured Result / Stream Event
```

실행 중 LLM이 Tool을 선택해 외부 결과를 기다려야 하는 흐름은 [Tool Calling](../tool-calling/README.md)에서 확장한다.

## Completion Baseline

- 하나 이상의 실제 AiTask가 공통 Router를 통해 Workflow로 전달된다.
- Workflow별 입력과 출력이 공통 envelope 안에서 validation된다.
- 성공, 실패, 취소가 execution state와 correlation identifier로 추적된다.
- Structured Result가 호출자 또는 result delivery 경로로 반환된다.

## Related

- [Workflows](../workflows/README.md)
- [Conversation Start Use Case](../../use-cases/01-conversation-start/README.md)
