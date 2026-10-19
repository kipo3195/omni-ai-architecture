# Components

여러 Use Case가 공유하는 서비스별 구현 책임과 내부 구조를 기록한다. 기능 하나에만 해당하는 흐름과 AI 입출력은 [Use Cases](../use-cases/README.md)에 둔다.

## Component Map

| Component | Responsibility | Implementation Areas |
| --- | --- | --- |
| [AI Orchestrator](ai-orchestrator/README.md) | 실행 정책, Context Assembly, correlation, Tool Runtime | [Policy](ai-orchestrator/policy.md), [Server Trigger](ai-orchestrator/server-trigger.md), [Client Request](ai-orchestrator/client-request.md), [Schedule](ai-orchestrator/schedule-management.md), [Tool Runtime](ai-orchestrator/tool-runtime.md), [Result Router](ai-orchestrator/result-router.md), [Spring Architecture](ai-orchestrator/spring-architecture.md) |
| [Omni AI Server](omni-ai-server/README.md) | Workflow·Agent 실행, LLM, Runtime State | [Task Execution](omni-ai-server/task-execution.md), [Tool Calling](omni-ai-server/tool-calling.md), [Result Event](omni-ai-server/result-event.md) |
| [Messenger Client Ingress](messenger-client-ingress/README.md) | WebSocket Service (WS) / TCP Service (NS) connection 소유, session registry, Client delivery | [Result Delivery](messenger-client-ingress/result-delivery.md), [User AI Settings](messenger-client-ingress/user-ai-settings.md) |

## Boundary

```text
WebSocket Service (WS) / TCP Service (NS)
→ connection / session / channel delivery

AI Orchestrator
→ policy / context assembly / AiTask / execution correlation

Omni AI Server
→ prompt / workflow / agent / LLM runtime
```

Use Case를 먼저 End-to-End로 설계한다. 둘 이상의 Use Case가 공유할 구조는 Component 문서로 분리할 수 있으며, 검증 전에는 `Planned`, 구현·검증 중에는 `In Progress`, 완료 후에는 `Implemented`로 상태를 명시한다.
