# Implementation

> Role: 설계 문서에서 정의한 책임 경계를 실제 runtime과 application structure로 연결한다.
> Status: In Progress

---

## Purpose

이 디렉터리는 Omni AI Platform의 서비스별 구현 구조를 정리한다.

상위 Architecture 문서는 책임 경계와 목표 실행 흐름을 설명한다. 이 디렉터리는 각 서비스의 실제 코드 구조와 실행 방식을 기록한다. 먼저 서비스별로 나누고, 각 서비스 안에서는 여러 AI 기능이 공유할 수 있는 실행 경로와 공통 구성 요소를 기준으로 문서를 둔다. `Conversation Start` 같은 개별 기능은 해당 구조의 적용 사례로 다룬다.

---

## Document Structure

| Service | Area | 현재 문서 내용 |
| --- | --- | --- |
| [AI Orchestrator](ai-orchestrator/README.md) | [Spring Architecture](ai-orchestrator/spring-architecture/README.md) | Spring 서비스 구조의 기록 범위 |
| AI Orchestrator | [Server Trigger](ai-orchestrator/server-trigger/README.md) | Conversation Start의 현재 실행 흐름과 구현 경계 |
| AI Orchestrator | [Client Request](ai-orchestrator/client-request/README.md) | Client 요청 경로의 기록 범위 |
| AI Orchestrator | [Tool Runtime](ai-orchestrator/tool-runtime/README.md) | Tool 관리·라우팅의 기록 범위 |
| [Omni AI Server](omni-ai-server/README.md) | [Task Execution](omni-ai-server/task-execution/README.md) | AiTask 실행 구조의 기록 범위 |
| Omni AI Server | [Tool Calling](omni-ai-server/tool-calling/README.md) | Runtime Tool 호출의 기록 범위 |

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
