# Decision Summary

이 문서는 주요 설계 결정의 빠른 인덱스다. 결정의 배경, 대안과 Trade-off는 개별 ADR 또는 연결된 기준 문서에서 관리하며 여기에서 반복하지 않는다.

## Architecture Decision Records

| ADR | Decision | Status |
| --- | --- | --- |
| [001](001-session-registry-result-routing.md) | AI Result는 Trigger source가 아니라 현재 Realtime connection owner를 기준으로 routing한다. | Accepted |
| [002](002-tool-runtime-ownership-client-tool-dispatch.md) | Server/Client Tool lifecycle은 AI Orchestrator의 Tool Runtime이 소유한다. | Accepted |
| [003](003-ai-orchestrator-application-service.md) | AI Orchestrator를 channel-independent Application Service로 분리한다. | Accepted |
| [004](004-ai-execution-and-context-identifiers.md) | trigger, task, execution, conversation과 tool identifier의 역할을 분리한다. | Proposed |
| [005](005-websocket-realtime-client-session-result-routing.md) | WebSocket MVP는 인증 세션을 current connection의 논리적 target으로 사용한다. | Accepted |
| [006](006-ai-feature-enablement-policy-and-user-preference.md) | Global, Feature와 User AI Policy의 source와 조회 전략을 분리한다. | Accepted |

ADR 상태는 `Proposed → Accepted → Superseded`만 사용한다.

## Supporting Design Directions

아래 항목은 현재 Architecture와 Concept 문서의 기준 방향이다. 구현 진행 상태는 이 문서가 아니라 [Roadmap](../roadmap.md)에서 관리한다.

| Direction | Source of truth |
| --- | --- |
| 현재 Client ingress는 `WebSocket Service (WS)`와 `TCP Service (NS)`이며, 두 서비스를 포괄하는 논리적 경계는 `Messenger Client Ingress`다. | [Current Service Boundary](../concepts/current-service-boundary.md) |
| Business State는 현재 해당 기능을 구현한 Messenger Service가 소유하고 Orchestrator는 Cross-domain AI Use Case를 조정한다. | [Architecture Overview](../overview.md) |
| Business Event와 AiTask를 분리하고 정책을 통과한 실행만 Task로 만든다. | [AiTask and Queue](../concepts/ai-task-and-queue.md) |
| Conversation Product Metadata와 AI Runtime State를 분리한다. | [Architecture Overview](../overview.md) |
| Durable Trigger는 JetStream, 계획된 realtime delivery는 Core NATS를 사용한다. | [Architecture Overview](../overview.md) |
| Client Tool은 실행 방식이 아니라 Workflow/Agent가 사용하는 Provider다. | [Client Tool Integration](../concepts/client-integration.md) |
| Client는 Omni AI Server와 직접 연결하지 않고 `Messenger Client Ingress`의 인증된 session을 사용한다. | [Client Tool Integration](../concepts/client-integration.md) |
| 예측 가능한 기능은 Workflow로 시작하고 Agent Runtime과 Worker 분리는 검증된 필요에 따라 확장한다. | [Roadmap](../roadmap.md) |
