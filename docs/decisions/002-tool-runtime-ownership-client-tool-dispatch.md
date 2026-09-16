# 002. Tool Runtime Ownership and Client Tool Dispatch

Status: Accepted

## Context

Omni AI는 Workflow / Agent execution 중 server tool과 client tool을 모두 사용할 수 있다.

초기 구조에서는 server tool은 `AI Orchestrator`의 Server Tool Relay를 통하고, client tool은 `Omni AI Server`가 `WebSocket Service`의 Client Tool Relay를 직접 사용하는 방향이었다.

```text
Server Tool
Omni AI Server → AI Orchestrator → Business Services

Client Tool
Omni AI Server → WebSocket Service → Client
```

이 구조는 client tool path가 짧고 latency가 낮지만, Tool lifecycle ownership이 실행 위치에 따라 나뉜다. Agent execution에서 `client tool → server tool → client tool`처럼 여러 tool call이 섞이면 timeout, retry, permission, audit, result validation, execution resume 처리가 `Omni AI Server`, `AI Orchestrator`, `WebSocket Service`에 분산될 수 있다.

## Decision

Server Tool과 Client Tool의 lifecycle owner를 `AI Orchestrator` 내부 Tool Runtime으로 둔다.

`Omni AI Server`는 LLM / LangGraph execution 중 필요한 Tool을 결정한다. Tool Runtime은 Tool registry, schema validation, permission, dispatch, timeout, retry, cancellation, result normalization, audit, execution resume coordination을 담당한다.

```text
Tool Decision
= Omni AI Server

Tool Lifecycle
= AI Orchestrator Tool Runtime

Server Tool Dispatch
= AI Orchestrator Tool Runtime → Server Tool Adapter → target service

Client Tool Dispatch
= AI Orchestrator Tool Runtime → Core NATS → WebSocket Service → Client

Client Tool Result
= Client → WebSocket Service → Core NATS → AI Orchestrator Tool Runtime → Omni AI Server resume
```

`WebSocket Service`는 client session lookup, WebSocket delivery, client result ingress를 담당한다. Client Tool lifecycle의 source of truth는 아니다.

LLM token stream과 execution progress stream은 Tool Runtime을 token-by-token 경유하지 않는다.

```text
Omni AI Server → Core NATS → WebSocket Service → Client
```

## Alternatives

### Keep Client Tool Relay outside AI Orchestrator

Client tool을 `Omni AI Server ↔ WebSocket Service ↔ Client`로 직접 처리한다.

장점:

- client tool hop이 적다.
- 구현이 단순하다.
- `AI Orchestrator` 부하가 낮다.

단점:

- server tool과 client tool의 lifecycle owner가 갈라진다.
- Tool timeout, retry, audit, permission, result validation이 중복될 수 있다.
- LangGraph / Agent execution resume 상태를 `Omni AI Server`가 직접 더 많이 소유하게 된다.

### Route every stream event through Tool Runtime

Tool lifecycle과 LLM streaming / progress event를 모두 Tool Runtime으로 통합한다.

장점:

- client에서 event ordering을 이해하기 쉽다.
- 모든 execution event가 한 곳을 통과한다.

단점:

- `AI Orchestrator`가 high-frequency streaming data plane이 된다.
- token-by-token proxy로 latency와 부하가 증가한다.
- Tool lifecycle event와 presentation stream의 의미가 섞인다.

## Consequences

좋아지는 점:

- Tool이라는 플랫폼 개념의 ownership이 하나로 유지된다.
- server/client tool 모두 동일한 correlation, timeout, retry, cancellation, audit, result normalization 정책을 사용할 수 있다.
- LangGraph / Agent execution에서 여러 차례 tool call이 발생해도 `WAITING_TOOL_RESULT → RESUMING` 흐름을 일관되게 관리할 수 있다.
- MCP, browser, sandbox, approval tool 같은 새로운 tool class를 adapter로 확장하기 쉽다.

감수할 점:

- Client Tool 호출에 `AI Orchestrator / Tool Runtime` hop이 추가된다.
- `AI Orchestrator` 장애가 client tool lifecycle에도 영향을 준다.
- Core NATS subject, idempotency, duplicate result handling, timeout 처리 계약을 더 엄격히 설계해야 한다.
- Tool lifecycle event와 execution progress / LLM token stream의 ordering은 전역 순서를 보장하지 않는다. Client는 stream lane과 lifecycle lane을 분리해서 처리해야 한다.

## Related

- [Architecture](../architecture.md)
- [Client Integration](../client-integration.md)
- [AiTask and Queue](../ai-task-and-queue.md)
- [Design Decisions](../design-decisions.md)
