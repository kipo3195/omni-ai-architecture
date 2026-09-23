# Design Decisions

> Role: 주요 설계 결정의 요약 인덱스이며, 자세한 개별 결정은 `docs/decisions/`에 ADR로 기록한다.
> Status: Designed
> 이 문서는 README의 Core Design을 기준으로 주요 설계 판단과 Trade-off를 정리한다.

---

## WebSocket Service 책임 분리는 Target Architecture로 둔다

결정
→ Omni AI 1차 구축에서는 현재 `WebSocket Service`와 연동하고, Target Architecture에서는 `WebSocket Service`, `user-service`, `auth-service`, `file-service`로 책임을 점진 분리한다.

배경
→ 현재 WebSocket Service는 WebSocket, Chat, Note, Alert, User State, User Info, Auth, File, REST API 책임을 하나의 프로세스에 가진다.

이유
→ AI Trigger, Context Assembly, LLM Streaming, Result Routing까지 WebSocket Service에 직접 추가하면 서비스 응집도가 더 낮아지고 변경 영향 범위가 커진다.

→ 따라서 우선 `AI Orchestrator`와 `Omni AI Server`를 분리된 AI boundary로 만들고, WebSocket Service split은 별도 migration topic으로 관리한다.

Trade-off
→ 서비스 간 계약, 데이터 ownership, migration 순서를 별도로 관리해야 한다.

Status: Designed

---

## AI Orchestrator는 별도 Spring Boot Runtime으로 둔다

결정
→ AI Orchestrator를 WebSocket Service나 Omni AI Server 내부에 분산시키지 않고, Spring Boot 기반 신규 Application Service로 구현한다.

배경
→ Messenger에는 WebSocket 기반 Client뿐 아니라 TCP 연결 기반의 TCP Realtime Service도 존재한다. 두 Client 경로 모두 동일한 AI 기능, 동일한 요청 / 응답 규격, 동일한 실행 정책을 제공해야 한다.

이유
→ AI 실행 판단, Trigger Policy, Context Assembly, AiTask 생성, Execution Correlation을 각 realtime service에 분산하면 WebSocket 경로와 TCP 경로의 구현이 달라지고 정책 불일치가 발생할 수 있다.

→ 반대로 이 책임을 Omni AI Server로 넘기면 Python AI Runtime이 Messenger connection topology, channel별 session, product policy까지 알게 되어 AI Runtime과 Product Application 경계가 흐려진다.

Trade-off
→ 별도 Orchestrator hop이 추가된다. 대신 channel-specific connection / delivery 책임과 AI execution policy를 분리하고, WebSocket Client와 TCP Client가 동일한 AI 규격을 사용할 수 있다.

Related
→ [ADR 003. AI Orchestrator as a Spring Boot Application Service](decisions/003-ai-orchestrator-spring-boot-service.md)

Status: Accepted

---

## Business State는 각 Service가 소유한다

결정
→ `WebSocket Service`는 realtime delivery와 message state를, `user-service`는 user / presence를, `auth-service`는 authentication을, `file-service`는 file / attachment를 소유한다.

→ 현재 구현에서는 이 책임들이 `WebSocket Service` 내부에 남아 있을 수 있으며, 위 구분은 Target Service Boundary 기준이다.

배경
→ AI Use Case가 늘어나도 Domain State의 ownership을 중앙 AI 서버나 Orchestrator로 옮기면 안 된다.

이유
→ State ownership이 흐려지면 권한, 상태 변경, 정책 판단이 중복되고 Cross-service dependency가 빠르게 증가한다.

Trade-off
→ `AI Orchestrator`는 각 Domain Service가 소유한 상태를 직접 DB / Redis로 조회하지 않고, 해당 Service API / gRPC 또는 명시적으로 계약된 Projection을 통해 확인해야 한다. 단, cooldown, deduplication, execution correlation처럼 `AI Orchestrator`가 소유한 실행 조정 상태는 자체 DB / Redis에 저장하고 직접 조회할 수 있다.

Status: Designed

---

## Cross-service AI Use Case는 AI Orchestrator가 조정한다

결정
→ 여러 Service의 상태를 조합해야 하는 AI Use Case는 `AI Orchestrator`에서 처리한다.

배경
→ Returned Message Topic Digest처럼 user-service와 Message / Realtime Service의 상태를 함께 봐야 하는 기능이 존재한다.

이유
→ 특정 Service에 Cross-domain AI Application 책임을 몰아주지 않기 위해 Application / Coordination Boundary를 둔다.

Trade-off
→ `AI Orchestrator`의 HA, execution state, cooldown, dedup, result routing 정책을 별도로 관리해야 한다.

Status: Designed

---

## Business Event와 AiTask를 분리한다

결정
→ Business Event와 AiTask를 다른 모델로 둔다.

배경
→ `USER_RETURNED`는 발생한 일을 나타내고, `RETURNED_MESSAGE_TOPICS`는 AI가 수행할 작업을 나타낸다.

이유
→ 하나의 Event에서 여러 Task 후보를 만들 수 있고, 같은 Task를 여러 Event나 Client Request 경로에서 재사용할 수 있다.

Trade-off
→ Event-to-Task mapping과 Policy 조합을 별도로 관리해야 한다.

Status: Designed

---

## Conversation Metadata와 Runtime State를 분리한다

결정
→ `AI Orchestrator`는 Product Metadata를 관리하고, `Omni AI Server`는 AI Runtime State를 관리한다.

배경
→ Conversation list, title, owner, tenant, archive / delete는 Messenger Product 기능이고, user / assistant turns, summary, checkpoint, agent state는 다음 LLM 호출에 필요한 Runtime State다.

이유
→ Product Access Control과 LLM Context 관리를 분리해야 `AI Orchestrator`가 prompt runtime까지 소유하거나 `Omni AI Server`가 Messenger product metadata까지 알게 되는 일을 피할 수 있다.

Trade-off
→ Conversation Metadata Store와 AI History Store의 consistency / retention 정책을 별도로 설계해야 한다.

Status: Designed

---

## WebSocket Session과 ConversationId를 분리한다

결정
→ WebSocket Session은 `WebSocket Service`가 관리하고, AI Conversation은 `conversationId` 기준의 logical conversation으로 관리한다.

배경
→ reconnect, scale-out, session migration이 발생해도 동일한 AI Conversation은 이어질 수 있다.

이유
→ Network Connection lifecycle과 Logical AI Conversation lifecycle이 다르기 때문이다.

Trade-off
→ Session Registry와 stream reconnect / resume 정책이 필요하다.

Status: Designed

---

## NATS JetStream은 Trigger 전달에 사용한다

결정
→ 재처리가 필요한 Business Event / AI Trigger는 NATS JetStream으로 전달한다.

배경
→ AI Trigger Consumer 장애, ACK, retry, 재처리 요구가 있을 수 있다.

이유
→ Kafka를 기본 전제로 추가하기보다 현재 Messenger 구조에서 NATS를 공통 Messaging Infrastructure로 우선 활용한다.

Trade-off
→ Trigger expiration, durable consumer, subject 설계를 명확히 해야 한다.

Status: Designed

---

## Core NATS는 Result / Streaming Routing에 사용한다

결정
→ 실시간 AI Result와 LLM Streaming Routing은 Core NATS를 사용한다.

배경
→ AI 결과는 영속성보다 현재 연결된 사용자에게 빠르게 전달하는 것이 중요한 경우가 많다.

이유
→ Reconnect, scale-out, instance restart가 발생할 수 있으므로 Result 전달 시점에 현재 Session Owner를 확인하고 대상 `WebSocket Service` instance로 low-latency routing한다.

Trade-off
→ Session Registry 또는 현재 Session Owner 조회 경로가 필요하다.

Status: Designed

---

## AI Result는 Session Registry 기준으로 Routing한다

결정
→ Client explicit request나 Server trigger를 처리한 instance를 최종 delivery target으로 사용하지 않는다.

→ WebSocket 연결 시 Connection Registry에 `connectionId`와 현재 `ownerInstanceId`를 등록하고, `enterRoom` 시 `roomSessionId`를 생성해 `connectionId` / `ownerInstanceId`와 연결한다.

→ AI Result push 시점에는 `triggerId` / `taskId` / `executionId`로 실행 상태를 확인하고, `connectionId` / `roomSessionId` / `routingRef`를 통해 Session Registry에서 현재 `ownerInstanceId`를 resolve한 뒤 해당 `WebSocket Service` instance로만 전달한다.

배경
→ `enterRoom` API를 처리한 instance와 실제 WebSocket이 붙어 있는 instance가 다를 수 있다. AI 처리 중 reconnect, scale-out, scale-in, instance restart, session migration도 발생할 수 있다.

이유
→ Trigger source와 delivery owner를 분리해야 Client 명시 요청, room 진입 기반 AI, server-triggered AI, Client Tool Calling을 동일한 routing 원칙으로 처리할 수 있다.

Trade-off
→ Session Registry의 TTL, heartbeat, stale owner 제거, registry 조회 실패, result routing timeout, reconnect / resume 정책을 별도로 설계해야 한다.

Related
→ [ADR 001. Session Registry Based AI Result Routing](decisions/001-session-registry-result-routing.md)

Status: Designed

---

## AI Orchestrator는 Streaming Data Plane이 아니다

결정
→ LLM Stream을 `AI Orchestrator`가 token-by-token proxy하지 않고, `Omni AI Server → Core NATS → WebSocket Service → Client` 경로로 전달한다.

배경
→ `AI Orchestrator`는 executionId, conversationId, workflow, policy, correlation, routing context를 관리하는 control path 역할을 가진다.

이유
→ `AI Orchestrator`를 Streaming Data Plane으로 만들지 않고 Execution Control과 Coordination 책임에 집중시키기 위해서다.

Trade-off
→ Stream event schema와 Core NATS subject, ordering, reconnect 정책을 별도로 설계해야 한다.

Status: Designed

---

## Client Tool Integration은 Execution Mode가 아니라 Provider이다

결정
→ Workflow / Agent는 실행 방식이고, Server Tool / Client Tool은 Context / Tool Provider로 분리한다.

배경
→ Tool을 사용한다고 해서 모두 Agent Execution인 것은 아니다. 사전 정의 Workflow도 server-side context나 client context가 필요할 수 있다.

이유
→ 실행 방식과 Context 공급 방식을 분리해야 기능 조합이 자연스럽다.

Trade-off
→ 문서와 구현에서 두 축을 계속 명확히 구분해야 한다.

Status: Designed

---

## Tool Runtime은 AI Orchestrator가 소유한다

결정
→ Server Tool과 Client Tool의 lifecycle은 `AI Orchestrator` 내부 Tool Runtime이 소유한다.

→ `Omni AI Server`는 LLM / LangGraph 실행 중 필요한 Tool을 결정하지만, Tool registry, schema validation, permission, dispatch, timeout, retry, result normalization, execution resume coordination은 Tool Runtime에서 관리한다.

배경
→ Agent Runtime에서는 하나의 execution 안에서 server tool과 client tool이 여러 차례 섞여 호출될 수 있다. Tool lifecycle이 server/client 실행 위치에 따라 갈라지면 correlation, timeout, retry, audit, resume 로직이 중복된다.

이유
→ Tool이라는 제품/플랫폼 개념의 ownership을 하나로 유지하고, 실행 위치 차이는 adapter와 delivery path로 제한하기 위해서다.

Trade-off
→ Client Tool도 Tool Runtime을 경유하므로 hop이 늘고, Orchestrator 부하와 장애 영향 범위가 커질 수 있다. 대신 durable execution state, audit, permission, 중복 방지, resume 모델을 일관되게 가져갈 수 있다.

Related
→ [ADR 002. Tool Runtime Ownership and Client Tool Dispatch](decisions/002-tool-runtime-ownership-client-tool-dispatch.md)

Status: Designed

---

## Server Tool Adapter는 각 Service Boundary를 통과한다

결정
→ `Omni AI Server`가 server-side context / tool이 필요하다고 결정하면 `AI Orchestrator`의 Tool Runtime과 Server Tool Adapter를 통해 `auth-service`, `file-service`, `user-service`, `WebSocket Service`와 연결한다.

→ service split 전에는 target service 대신 현재 `WebSocket Service` API가 relay 대상이 될 수 있다.

Tool 사용 여부와 tool input 구성은 `Omni AI Server`가 판단하고, Messenger Service로의 relay와 policy-aware access는 `AI Orchestrator`가 담당한다.

배경
→ AI Runtime이 인증, 파일, 사용자, 메시지 데이터를 직접 소유하지 않더라도 실행 중 해당 Service의 context나 tool이 필요할 수 있다.

이유
→ `Omni AI Server`가 각 Service의 DB나 내부 구현에 직접 결합하지 않고, Messenger Application 영역인 `AI Orchestrator`가 service API / gRPC, capability, policy-aware access를 통해 필요한 기능만 중계하게 하기 위해서다.

Trade-off
→ Server Tool schema, timeout, permission propagation, failure handling을 Tool Runtime 계약으로 설계해야 한다.

Status: Designed

---

## Client는 Omni AI Server와 직접 연결하지 않는다

결정
→ Client는 `WebSocket Service`와 WebSocket으로 통신하고, Client Tool은 `AI Orchestrator`의 Tool Runtime이 Core NATS와 `WebSocket Service`의 Client Tool Delivery를 통해 dispatch한다.

배경
→ `WebSocket Service`는 Authentication, Session, WebSocket, Permission, Device State를 이미 소유한다.

이유
→ `Omni AI Server`가 Client 연결을 직접 소유하면 세션과 권한 책임이 중복된다. 동시에 Tool lifecycle을 `WebSocket Service`에 두면 server tool과 client tool의 execution state가 갈라진다.

Trade-off
→ Client Tool Calling에는 `AI Orchestrator / Tool Runtime`, Core NATS, `WebSocket Service` delivery hop이 추가된다. 대신 Tool lifecycle, audit, timeout, retry, result normalization, execution resume 책임을 중앙화할 수 있다.

Status: Designed

---

## Agent Runtime은 단계적으로 확장한다

결정
→ 예측 가능한 기능은 Workflow Execution으로 먼저 다루고, Runtime Tool Calling이 필요한 기능은 Agent Runtime으로 확장한다.

배경
→ Conversation Start, Returned Message Topic Digest, Scheduled Weekly Report는 Workflow로 시작하기 적합하다. 반면 여러 Tool을 순차 호출하고 다음 단계를 동적으로 결정하는 기능은 Agent Lifecycle이 필요하다.

이유
→ 초기 복잡도를 낮추면서 공통 AiTask / Trigger / Context 구조를 먼저 검증할 수 있다.

Trade-off
→ Agent Runtime, correlation, timeout, resume 상태 관리는 별도 설계가 필요하다.

Status: Planned

---

## Queue와 Worker는 Workload 기준으로 확장한다

결정
→ 초기에는 단순하게 시작하고, 필요하면 workload 기준으로 Queue / Consumer / Worker Pool을 분리한다.

배경
→ Event Type은 Business 확장 단위이고 Queue / Worker Pool은 자원 격리 단위이다.

이유
→ Event Type별 Queue는 기능이 늘어날수록 운영 복잡도를 키운다. 자원 격리는 latency, 처리 시간, 비용, 부하 특성에 맞춰야 한다.

Trade-off
→ 초기 구조에서는 서로 다른 workload가 같은 실행 자원을 공유할 수 있다.

Status: Designed
