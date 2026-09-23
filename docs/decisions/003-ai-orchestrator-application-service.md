# 003. AI Orchestrator as an Application Service

Status: Accepted

---

## Context

초기에는 별도 AI Orchestrator layer를 두지 않고, AI 실행 판단과 Context Assembly 책임을 WebSocket Service와 Omni AI Server에 분산시키는 방안을 검토하였다.

하지만 Messenger에는 WebSocket 기반 Client뿐 아니라 TCP 연결 기반의 TCP Realtime Service도 존재한다. 두 Client 경로 모두 동일한 AI 기능, 동일한 요청 / 응답 규격, 동일한 실행 정책을 제공해야 한다.

AI 실행 판단, Trigger Policy, Context Assembly, AiTask 생성, Execution Correlation을 각 realtime service에 분산하면 WebSocket 경로와 TCP 경로의 구현이 달라지고, 기능 추가 시 중복 구현과 정책 불일치가 발생할 수 있다.

또한 이 책임을 Omni AI Server로 넘기면 Python AI Runtime이 Messenger connection topology, channel별 session, product policy까지 알게 되어 AI Runtime과 Product Application 경계가 흐려진다.

AI Orchestrator 자체도 향후 Trigger, Policy, Context Assembly, Tool Relay, Execution Management 등 여러 책임이 추가될 수 있으므로, 단순한 기술 레이어 중심 구조보다는 업무 기능의 경계를 먼저 정의하고 각 기능 내부에서 역할을 분리할 필요가 있다.

---

## Decision

독립적인 AI Orchestrator Application Service를 둔다.

AI Orchestrator는 Spring Boot 기반 신규 Application Service로 구현한다.

WebSocket Service와 TCP Realtime Service는 각자의 connection, session, delivery 책임을 유지한다.

AI 실행 판단과 실행 규격은 AI Orchestrator layer에서 공통화한다.

AI Orchestrator는 WebSocket Client와 TCP Client 모두에 대해 동일한 AI Use Case, AiTask schema, executionId, policy, context assembly, result contract를 제공한다.

Omni AI Server는 channel 종류를 알지 않고, 실행이 확정된 AiTask를 처리하는 Python AI Runtime으로 유지한다.

초기 Result Router는 `AI Orchestrator` 내부의 application module로 둔다. 이 module은 `routingRef`로 현재 Realtime Connection Registry owner를 resolve하고 Core NATS owner instance subject를 선택한다. Realtime session의 소유나 Client까지의 token stream proxy는 이 module의 책임이 아니며, 해당 책임은 WebSocket Service와 TCP Realtime Service에 남긴다.

고빈도 streaming의 독립적인 확장 또는 장애 격리가 필요해지면 Result Router는 같은 `ResultEvent` 계약을 유지한 채 별도 `Realtime Delivery / Result Router` 배포 단위로 분리할 수 있다.

AI Orchestrator 내부 구조는 **업무 기능을 기준으로 1차 분리하고, 각 기능 내부에서 역할에 따라 레이어를 정의하는 방식**을 따른다.

패키지 구조를 설계할 때 다음 두 가지 질문을 기준으로 한다.

1. 이 코드는 어떤 업무를 수행하는가?
2. 그 업무 안에서 어떤 역할을 수행하는가?

예를 들어 AI 실행과 관련된 코드는 먼저 `execution` 영역에 위치시키고, 이후 역할에 따라 `application`, `domain`, `infrastructure` 등의 레이어로 분리한다.

```text
execution/
├── application
├── domain
└── infrastructure
```

프로젝트 전체를 `controller`, `service`, `repository`와 같은 기술 레이어 기준으로 먼저 분리하지 않는다.

대신 다음과 같이 업무 기능의 경계를 먼저 정의한다.

```text
execution/
context/
policy/
tool/
conversation/
result-routing/
```

각 기능 내부에서는 필요한 경우에만 `application`, `domain`, `infrastructure` 등의 레이어를 구성하며, 모든 기능에 동일한 레이어 구조를 강제하지 않는다.

단순한 기능에는 불필요한 추상화나 계층을 추가하지 않고, 비즈니스 규칙과 외부 시스템 의존성이 커지는 영역에 대해서만 명확한 architectural boundary를 둔다.

---

## Alternatives

### Distribute orchestration into WebSocket Service and TCP Realtime Service

각 realtime service가 AI Trigger Policy, Context Assembly, AiTask 생성을 직접 처리한다.

장점:

* 별도 service hop이 줄어든다.
* channel별 session context에 바로 접근할 수 있다.

단점:

* WebSocket 경로와 TCP 경로에 AI 정책이 중복된다.
* 동일 기능의 request / response 규격이 channel별로 달라질 수 있다.
* 기능 추가 시 realtime service마다 구현과 테스트가 반복된다.

### Move orchestration into Omni AI Server

Omni AI Server가 Business Event, Client Request, Trigger Policy, Context Assembly까지 담당한다.

장점:

* AI Runtime 내부에서 prompt / workflow와 가까운 위치에서 판단할 수 있다.
* Spring service를 추가하지 않아도 된다.

단점:

* Python AI Runtime이 Messenger product policy와 connection topology를 알게 된다.
* authentication, permission, tenant, session, delivery boundary가 흐려진다.
* Omni AI Server가 AI Runtime이 아니라 product application service가 된다.

### Organize AI Orchestrator by technical layer

프로젝트 전체를 `controller`, `service`, `repository`, `domain`과 같은 기술 레이어 기준으로 구성한다.

장점:

* 전통적인 Spring Layered Architecture와 유사하여 초기 구조가 단순하다.
* 각 클래스의 기술적 역할을 빠르게 구분할 수 있다.

단점:

* 하나의 업무 기능과 관련된 코드가 여러 패키지에 분산된다.
* 기능 수가 증가할수록 각 기술 레이어 패키지가 비대해질 수 있다.
* 업무 경계보다 Framework / technical concern 중심의 구조가 되기 쉽다.
* 특정 기능을 이해하거나 변경할 때 여러 패키지를 오가야 한다.

---

## Consequences

좋아진 점:

* WebSocket Client와 TCP Client가 동일한 AI 기능 규격을 사용할 수 있다.
* Channel-specific connection / delivery 책임과 AI execution policy를 분리할 수 있다.
* AI 기능 추가 시 WebSocket Service와 TCP Realtime Service에 정책을 중복 구현하지 않아도 된다.
* Omni AI Server가 Messenger channel topology를 알 필요가 없다.
* AI Runtime은 Prompt / Workflow / Agent 실행에 집중할 수 있다.
* Java / Spring 기반 backend 운영 모델에서 policy, context assembly, execution correlation을 관리할 수 있다.
* 업무 기능 단위로 코드가 응집되어 특정 AI Use Case의 변경 범위를 파악하기 쉬워진다.
* 기능 내부에서만 필요한 레이어를 구성할 수 있어 과도한 계층화를 줄일 수 있다.
* 외부 시스템 연동과 핵심 비즈니스 규칙의 경계를 분리하기 쉬워진다.
* 프로젝트 규모가 커져도 기술 레이어 단위 패키지에 코드가 집중되는 문제를 줄일 수 있다.

감수할 점:

* 별도 orchestrator layer가 추가되어 service hop이 늘어난다.
* channel별 adapter contract와 result routing contract를 명확히 관리해야 한다.
* execution correlation, timeout, retry, idempotency 설계가 필요하다.
* AI Orchestrator의 HA, observability, failure handling을 별도로 설계해야 한다.
* 기능 경계를 잘못 정의하면 package 간 의존 관계가 복잡해질 수 있다.
* 작은 기능까지 과도하게 분리하지 않도록 지속적으로 경계를 조정해야 한다.

---

## Related

* [Architecture](../architecture.md)
* [Service Boundary and Migration](../service-boundary-and-migration.md)
* [AI Orchestrator Implementation](../implementation/ai-orchestrator/README.md)
