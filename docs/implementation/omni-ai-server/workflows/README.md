# Workflows

> Role: End-to-End Use Case에서 Omni AI Server가 담당하는 AI 처리 흐름, Prompt, 모델 호출과 Structured Result를 기록한다.
> Status: In Progress

---

| 순서 | Workflow | Status | Use Case |
| --- | --- | --- | --- |
| 01 | [Conversation Start](conversation-start.md) | In Progress | Conversation Start |
| 02 | [Returned Message Topics](returned-message-topics.md) | Planned | Returned Message Topics |
| 03 | [Weekly Report Summary](weekly-report-summary.md) | Planned | Scheduled Weekly Report |
| 04 | [Schedule Intent Parsing](schedule-intent-parsing.md) | Planned | Server Schedule Intent |
| 05 | [Tool-assisted AI](tool-assisted-ai.md) | Planned | Tool-assisted AI |

Workflow 문서는 기능 고유 입력, Prompt와 결과 계약을 다룬다. 공통 routing, execution state와 result envelope은 [Task Execution](../task-execution/README.md), 동적 Tool suspend/resume은 [Tool Calling](../tool-calling/README.md)에서 관리한다.

