# Components

여러 Use Case가 공유하는 서비스별 구현 책임과 내부 구조를 기록한다. 기능 하나에만 해당하는 흐름과 AI 입출력은 [Use Cases](../use-cases/README.md)에 둔다.

## Component Map

| Component | Responsibility | Implementation Areas |
| --- | --- | --- |
| [AI Orchestrator](ai-orchestrator/README.md) | 실행 정책, Context Assembly, correlation, Tool Runtime | [Policy](ai-orchestrator/policy.md), [Server Trigger](ai-orchestrator/server-trigger.md), [Client Request](ai-orchestrator/client-request.md), [Schedule](ai-orchestrator/schedule-management.md), [Tool Runtime](ai-orchestrator/tool-runtime.md), [Result Router](ai-orchestrator/result-router.md) |
| [Omni AI Server](omni-ai-server/README.md) | Workflow·Agent 실행, LLM, Runtime State | [Task Execution](omni-ai-server/task-execution.md), [Tool Calling](omni-ai-server/tool-calling.md), [Result Event](omni-ai-server/result-event.md) |
| [Realtime Service](realtime-service/README.md) | Connection 소유, session registry, Client delivery | [Result Delivery](realtime-service/result-delivery.md), [User AI Settings](realtime-service/user-ai-settings.md) |

## Boundary

```text
WebSocket Service / TCP Realtime Service
→ connection / session / channel delivery

AI Orchestrator
→ policy / context assembly / AiTask / execution correlation

Omni AI Server
→ prompt / workflow / agent / LLM runtime
```

Use Case를 먼저 End-to-End로 구현하고, 둘 이상의 기능에서 재사용이 확인된 구조만 Component 문서로 승격한다.
