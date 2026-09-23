# Implementation

> Role: 설계 문서에서 정의한 책임 경계를 실제 runtime과 application structure로 연결한다.
> Status: In Progress

---

## Purpose

이 디렉터리는 Omni AI Platform의 End-to-End Use Case와 서비스별 구현 구조를 정리한다.

상위 Architecture 문서는 책임 경계와 목표 실행 흐름을 설명한다. 이 디렉터리는 사용자 기능의 전체 흐름과 각 서비스의 실제 코드 구조를 기록한다.

- `use-cases`: 사용자 결과를 기준으로 여러 서비스에 걸친 Vertical Slice와 완료 상태를 기록한다.
- `ai-orchestrator`, `omni-ai-server`: 여러 Use Case가 공유하는 서비스 내부 구조와 Horizontal Capability를 기록한다.

Use Case를 먼저 E2E로 구현하고, 둘 이상의 기능에서 재사용이 확인된 구조를 서비스별 공통 문서로 정리한다.

---

## End-to-End Use Cases

| 순서 | Use Case | Status |
| --- | --- | --- |
| 01 | [Conversation Start](use-cases/01-conversation-start/README.md) | In Progress |
| 02 | [Returned Message Topics](use-cases/02-returned-message-topics/README.md) | Planned |
| 03 | [Scheduled Weekly Report](use-cases/03-scheduled-weekly-report/README.md) | Planned |
| 04 | [Server Schedule Intent](use-cases/04-server-schedule-intent/README.md) | Planned |
| 05 | [Tool-assisted AI](use-cases/05-tool-assisted-ai/README.md) | Planned |

전체 순서와 작성 규칙은 [Use Cases](use-cases/README.md)에서 관리한다.

---

## Service Implementation Structure

| Service | Area | 현재 문서 내용 |
| --- | --- | --- |
| [AI Orchestrator](ai-orchestrator/README.md) | [Spring Architecture](ai-orchestrator/spring-architecture/README.md) | Spring 서비스 구조의 기록 범위 |
| AI Orchestrator | [Server Trigger](ai-orchestrator/server-trigger/README.md) | Conversation Start의 현재 실행 흐름과 구현 경계 |
| AI Orchestrator | [Client Request](ai-orchestrator/client-request/README.md) | Client 요청 경로의 기록 범위 |
| AI Orchestrator | [Schedule Management](ai-orchestrator/schedule-management/README.md) | Schedule과 실행 회차 관리의 기록 범위 |
| AI Orchestrator | [Tool Runtime](ai-orchestrator/tool-runtime/README.md) | Tool 관리·라우팅의 기록 범위 |
| [Omni AI Server](omni-ai-server/README.md) | [Task Execution](omni-ai-server/task-execution/README.md) | AiTask 실행 구조의 기록 범위 |
| Omni AI Server | [Workflows](omni-ai-server/workflows/README.md) | Use Case별 AI Workflow 구현 범위 |
| Omni AI Server | [Tool Calling](omni-ai-server/tool-calling/README.md) | Runtime Tool 호출의 기록 범위 |

Use Case 상태와 서비스 내부 capability 상태는 독립적으로 관리한다. 예를 들어 Conversation Start가 구현되어도 Tool Calling은 `Planned`일 수 있다.

---

## Boundary

```text
WebSocket Service / TCP Realtime Service
→ connection / session / channel delivery

AI Orchestrator
→ policy / context assembly / AiTask / execution correlation

Omni AI Server
→ prompt / workflow / agent / LLM runtime
```

Realtime channel이 WebSocket인지 TCP인지와 무관하게 AI 실행 판단과 요청 / 응답 규격은 AI Orchestrator에서 공통화한다.
