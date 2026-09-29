# 001. Session Registry Based AI Result Routing

Status: Accepted

## Context

Omni AI 기능은 Client explicit request와 Server trigger 양쪽에서 시작될 수 있다.

예:

- `enterRoom` 기반 Conversation Start
- 선택 메시지 요약
- 현재 화면 기반 질문
- Draft 보조
- `USER_RETURNED` 기반 urgent summary
- `LABEL_MATCHED` 기반 multimodal action

이때 Trigger 또는 API 요청을 처리한 `WebSocket Service` instance와 실제 WebSocket connection을 소유한 instance가 다를 수 있다.

```text
enterRoom 처리
Client → WebSocket Service #1

AI 처리 중 reconnect
Client WebSocket → WebSocket Service #6

Result push
WebSocket Service #1로 보내면 실패하거나 stale session이 된다.
```

Scale-out, scale-in, instance restart, reconnect, session migration, multi-device 상황에서도 같은 문제가 발생한다.

## Decision

AI Result delivery는 Trigger source instance가 아니라 Session Registry의 현재 owner 기준으로 routing한다.

WebSocket / TCP 연결 시 각 Realtime Service는 Realtime Connection Registry에 현재 연결을 등록한다.

```text
connectionId
channelType
tenantId
userId
deviceId (필요 시)
ownerServiceType
ownerInstanceId
leaseId
connectedAt
lastSeenAt
expiresAt
capabilities
```

Conversation Start의 `enterRoom` 호출 시에는 `roomSessionId`를 생성하고 현재 `connectionId` / `ownerInstanceId`와 연결한다.

```text
roomSessionId
roomId
connectionId
ownerInstanceId
enteredAt
lastSeenAt
expiresAt
```

AI 실행은 `triggerId` / `taskId` / `executionId`로 처리한다. Client delivery는 `routingRef`의 target reference와 필요한 경우 Use Case별 scope reference를 통해 현재 target을 resolve한다. `roomSessionId`는 Conversation Start에서 사용하는 scope reference의 예시이며 모든 AI Event의 공통 필드는 아니다.

### Common Routing Contract

`routingRef`는 instance 위치가 아니라 논리적인 delivery target과 선택적인 Event scope를 표현한다.

```text
routingRef
  tenantId
  userId
  targetRef
    type: CONNECTION | DEVICE_CURRENT | SCOPE
    deviceId (필요 시)
    connectionId (필요 시)
  scopeRef (필요 시)
    type
    id
```

`targetRef.type`의 의미는 다음과 같다.

- `CONNECTION`: 요청 당시 exact connection에만 전달하고 reconnect 후에는 drop한다.
- `DEVICE_CURRENT`: 같은 device의 현재 connection으로 다시 resolve한다.
- `SCOPE`: Use Case scope가 현재 가리키는 connection으로 resolve한다.

`scopeRef`는 공통 Session ID가 아니다. Event의 유효 범위를 독립적으로 검증해야 할 때만 사용한다. Conversation Start는 `roomSessionId`, 선택 메시지 요약은 `selectionRequestId`, Draft 보조는 `draftSessionId`를 사용할 수 있으며 별도 lifecycle이 없는 Event는 생략한다.

Omni AI Server가 생성하는 공통 결과 계약은 다음과 같다.

```text
ResultEvent
  executionId
  eventId
  eventType
  sequence (stream인 경우)
  routingRef (전달되는 경우에도 non-authoritative snapshot)
  payload
```

Result Router는 Event에 포함된 routing 위치를 그대로 신뢰하지 않고 `executionId`로 AI Orchestrator가 저장한 trusted `routingRef`를 조회한다. 선택된 Realtime Service instance로 보내는 계약은 다음과 같다.

```text
DeliveryEvent
  executionId
  eventId
  eventType
  connectionId
  userId
  scopeRef (필요 시)
  payload
```

### Result Router Placement

초기 구현에서 Result Router는 별도 배포 서비스가 아니라 `AI Orchestrator` 내부 모듈로 둔다. 이 모듈의 책임은 execution과 `routingRef`를 검증하고 Session Registry에서 현재 owner를 조회한 뒤 해당 instance subject로 Core NATS event를 발행하는 데까지다. WebSocket 또는 TCP session을 소유하거나 Client까지 stream을 proxy하지 않는다.

고빈도 LLM streaming으로 독립적인 확장 또는 장애 격리가 필요해지거나, WebSocket / TCP 외 delivery adapter가 늘어나면 같은 계약을 유지한 채 `Realtime Delivery / Result Router`를 별도 배포 단위로 분리할 수 있다.

### End-to-End Flow

```text
Client WebSocket Connect
→ 임의의 Realtime Service instance
→ connectionId / leaseId 생성
→ Realtime Connection Registry 등록

Client REST 또는 WebSocket AI Request
→ 임의의 Realtime Service instance
→ 인증 주체와 target connection resolve
→ Use Case scope ID 생성 또는 확인 (필요 시)
→ 임의의 AI Orchestrator instance

AI Orchestrator
→ executionId 생성
→ execution / trusted routingRef 공유 저장소 저장
→ 임의의 Omni AI Server instance 호출

Omni AI Server
→ executionId 기준 ResultEvent 생성
→ 임의의 AI Orchestrator instance로 반환 또는 publish

AI Orchestrator Result Router
→ execution / active execution / scopeRef 검증
→ targetRef로 현재 connectionId resolve
→ Realtime Connection Registry에서 current ownerInstanceId 조회
→ Core NATS owner instance subject로 DeliveryEvent publish

현재 Realtime Service instance
→ exact connectionId local lookup
→ 인증 주체 검증
→ local realtime connection으로 최종 push
```

Realtime Service, AI Orchestrator, Omni AI Server 사이에는 sticky session이 필수가 아니다. 특정 instance에 종속되는 상태는 Realtime Service의 local connection뿐이며, 해당 위치는 Session Registry로 resolve한다.

`sourceInstanceId`는 correlation 정보로만 사용한다. 최종 delivery source of truth로 사용하지 않는다. Connection Registry는 WebSocket 전용이 아니라 WebSocket / TCP Realtime Service가 각자 등록하는 Realtime Connection Registry로 확장한다. 각 Realtime Service는 자신의 local connection을 소유하고 최종 push를 수행한다.

구현에서는 `deviceId`를 connection 자체의 식별자로 사용하지 않는다. Realtime 연결마다 고유한 `connectionId`와 lease를 생성하고, 단일 활성 연결 정책이 필요한 경우 `deviceId`는 현재 `connectionId`를 가리키는 pointer로 사용한다. Reconnect는 새 connection을 등록한 뒤 pointer를 교체하며, heartbeat와 disconnect는 자신의 connection / lease가 현재 값과 일치할 때만 갱신하거나 삭제한다.

`routingRef`의 instance 정보는 요청 시점의 snapshot 또는 correlation 정보다. Result Router는 결과 전달 직전에 target reference와 선택적인 Use Case scope reference를 통해 현재 owner를 다시 resolve한다. Conversation Start에서는 이 scope reference가 `roomSessionId`다. Registry에서 유효한 target을 찾지 못한 경우 요청 당시 instance로 무조건 fallback하지 않으며, 재조회, drop, durable delivery 중 Use Case에 맞는 정책을 적용한다.

### Delivery Semantics

정확한 target에만 전달하는 것과 반드시 전달하는 것은 서로 다른 보장이다. 이 결정은 잘못된 Client로 전달하지 않는 것을 우선한다.

- `eventId`로 중복 Result를 식별하고 stream은 `executionId + sequence`로 순서를 검증한다.
- Progress / stream event는 active execution을 유지하고 terminal event에서만 상태를 종료한다.
- Core NATS publish 성공은 Client 수신 성공을 의미하지 않는다.
- Local connection miss가 확인되면 Registry를 다시 조회해 제한적으로 재라우팅할 수 있다.
- Target이 없으면 Event 특성에 따라 drop, durable store, notification 중 하나를 선택한다.
- Client 수신 보장이 필요하면 acknowledgement, outbox, durable event, idempotent redelivery를 별도로 설계한다.

## Alternatives

### Trigger source instance로 push

Trigger를 발행하거나 API를 처리한 instance로 Result를 돌려보내는 방식이다.

장점은 구현이 단순하다는 점이다. 하지만 AI 처리 중 reconnect나 instance restart가 발생하면 stale session으로 push하게 된다. `enterRoom` 처리 instance와 WebSocket owner가 달라지는 상황도 처리하기 어렵다.

### Broadcast 후 local session이 있는 instance가 처리

모든 `WebSocket Service` instance에 Result를 broadcast하고, local session이 있는 instance만 push하는 방식이다.

구현은 직관적이지만 instance 수가 늘수록 불필요한 fan-out이 커진다. tenant / user / room 기준 privacy boundary와 observability도 흐려진다.

### AI Orchestrator가 WebSocket stream을 직접 proxy

`AI Orchestrator`가 token stream을 받아 Client까지 직접 중계하는 방식이다.

Routing 판단은 한 곳에 모을 수 있지만, `AI Orchestrator`가 Streaming Data Plane이 되어 부하와 장애 영향 범위가 커진다. WebSocket session ownership도 `WebSocket Service`와 중복된다.

## Consequences

좋아지는 점:

- Client explicit request와 Server trigger를 같은 delivery 원칙으로 처리할 수 있다.
- reconnect, scale-out, instance restart 중에도 현재 WebSocket owner 기준으로 push할 수 있다.
- `AI Orchestrator`는 실행 correlation과 routing decision에 집중하고, 최종 WebSocket delivery는 `WebSocket Service`에 남길 수 있다.
- Client Tool Delivery도 같은 target reference / optional scope reference / current `ownerInstanceId` 모델을 사용할 수 있다.

감수해야 할 점:

- Session Registry의 TTL, heartbeat, stale owner 제거 정책이 필요하다.
- Result push 직전 registry 조회 실패, owner instance unavailable, local session missing 상황의 fallback이 필요하다.
- stream reconnect / resume 정책을 별도로 정의해야 한다.
- multi-device, multi-tab, same user multi-room 진입 정책을 명확히 해야 한다.

## Related

- [AiTask and Queue](../ai-task-and-queue.md)
- [Server-driven AI](../server-driven-ai.md)
- [Client Tool Integration](../client-integration.md)
- [Design Decisions](../design-decisions.md)
- [Realtime Service Result Delivery](../implementation/realtime-service/result-delivery/README.md)
- [AI Orchestrator Result Router](../implementation/ai-orchestrator/result-router/README.md)
- [Omni AI Server Result Event](../implementation/omni-ai-server/result-event/README.md)
