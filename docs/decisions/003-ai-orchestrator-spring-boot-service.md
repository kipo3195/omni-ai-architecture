# 003. AI Orchestrator as a Spring Boot Application Service

Status: Accepted

---

## Context

초기에는 별도 AI Orchestrator layer를 두지 않고, AI 실행 판단과 Context Assembly 책임을 WebSocket Service와 Omni AI Server에 분산시키는 방안을 검토하였다.

하지만 Messenger에는 WebSocket 기반 Client뿐 아니라 TCP 연결 기반의 TCP Realtime Service도 존재한다. 두 Client 경로 모두 동일한 AI 기능, 동일한 요청 / 응답 규격, 동일한 실행 정책을 제공해야 한다.

AI 실행 판단, Trigger Policy, Context Assembly, AiTask 생성, Execution Correlation을 각 realtime service에 분산하면 WebSocket 경로와 TCP 경로의 구현이 달라지고, 기능 추가 시 중복 구현과 정책 불일치가 발생할 수 있다.

또한 이 책임을 Omni AI Server로 넘기면 Python AI Runtime이 Messenger connection topology, channel별 session, product policy까지 알게 되어 AI Runtime과 Product Application 경계가 흐려진다.

---

## Decision

독립적인 AI Orchestrator Application Service를 둔다.

AI Orchestrator는 Spring Boot 기반 신규 Application Service로 구현한다.

WebSocket Service와 TCP Realtime Service는 각자의 connection, session, delivery 책임을 유지한다.

AI 실행 판단과 실행 규격은 AI Orchestrator layer에서 공통화한다.

AI Orchestrator는 WebSocket Client와 TCP Client 모두에 대해 동일한 AI Use Case, AiTask schema, executionId, policy, context assembly, result contract를 제공한다.

Omni AI Server는 channel 종류를 알지 않고, 실행이 확정된 AiTask를 처리하는 Python AI Runtime으로 유지한다.

---

## Alternatives

### Distribute orchestration into WebSocket Service and TCP Realtime Service

각 realtime service가 AI Trigger Policy, Context Assembly, AiTask 생성을 직접 처리한다.

장점:

- 별도 service hop이 줄어든다.
- channel별 session context에 바로 접근할 수 있다.

단점:

- WebSocket 경로와 TCP 경로에 AI 정책이 중복된다.
- 동일 기능의 request / response 규격이 channel별로 달라질 수 있다.
- 기능 추가 시 realtime service마다 구현과 테스트가 반복된다.

### Move orchestration into Omni AI Server

Omni AI Server가 Business Event, Client Request, Trigger Policy, Context Assembly까지 담당한다.

장점:

- AI Runtime 내부에서 prompt / workflow와 가까운 위치에서 판단할 수 있다.
- Spring service를 추가하지 않아도 된다.

단점:

- Python AI Runtime이 Messenger product policy와 connection topology를 알게 된다.
- authentication, permission, tenant, session, delivery boundary가 흐려진다.
- Omni AI Server가 AI Runtime이 아니라 product application service가 된다.

---

## Consequences

좋아진 점:

- WebSocket Client와 TCP Client가 동일한 AI 기능 규격을 사용할 수 있다.
- Channel-specific connection / delivery 책임과 AI execution policy를 분리할 수 있다.
- AI 기능 추가 시 WebSocket Service와 TCP Realtime Service에 정책을 중복 구현하지 않아도 된다.
- Omni AI Server가 Messenger channel topology를 알 필요가 없다.
- AI Runtime은 Prompt / Workflow / Agent 실행에 집중할 수 있다.
- Java / Spring 기반 backend 운영 모델에서 policy, context assembly, execution correlation을 관리할 수 있다.

감수할 점:

- 별도 orchestrator layer가 추가되어 service hop이 늘어난다.
- channel별 adapter contract와 result routing contract를 명확히 관리해야 한다.
- execution correlation, timeout, retry, idempotency 설계가 필요하다.
- AI Orchestrator의 HA, observability, failure handling을 별도로 설계해야 한다.

---

## Related

- [Architecture](../architecture.md)
- [Service Boundary and Migration](../service-boundary-and-migration.md)
- [AI Orchestrator Implementation](../implementation/ai-orchestrator.md)
