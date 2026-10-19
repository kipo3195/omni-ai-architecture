# AI Orchestrator Result Router

Status: In Progress

---

## Purpose

AI Orchestrator가 trusted `routingRef`를 사용해 현재 realtime connection owner를 선택하고 Result를 전달하는 구현 책임을 정의한다. 현재 Conversation Start는 `AiExecution`을 Router에 직접 전달하며, 계획된 공유 저장소 연동에서는 `executionId`로 실행 상태와 `routingRef`를 복원한다.

공통 이벤트 계약과 전체 흐름은 [ADR 001](../../decisions/001-session-registry-result-routing.md)을 따른다. WebSocket Service MVP의 인증 세션 기반 target은 [ADR 005](../../decisions/005-websocket-realtime-client-session-result-routing.md)를 따른다.

## Current Implementation

현재 `AiResultRouter`는 execution의 trusted `routingRef`를 사용하고, `RedisRealtimeConnectionResolver`가 `CLIENT_SESSION_CURRENT` 기준으로 현재 `connectionId`와 `ownerInstanceId`를 조회한다. `RestResultDeliveryClient`가 설정된 owner instance endpoint의 `/internal/v1/ai-results`로 `DeliveryEvent`를 전송한다.

```text
AiExecution + ResultEvent
→ Redis current connection / owner resolve
→ DeliveryEvent
→ owner instance REST endpoint
```

Conversation Start가 이 경로를 사용하며 router, Redis resolver와 REST client에 대한 단위 테스트가 있다. NATS delivery adapter는 아직 Spring Bean으로 등록되지 않은 placeholder이고, 아래 Core NATS flow는 이 프로젝트의 계획된 result delivery 경로다.

---

## Target Shared Execution Correlation

Target 구조에서는 execution과 routing context를 Orchestrator instance memory가 아니라 공유 저장소에 보관한다. 현재 Conversation Start는 메모리 Repository를 사용하며, 아래 저장 구조는 아직 구현되지 않았다.

```text
ai:execution:{executionId}
→ {
    workflowType,
    status,
    supersessionKey,
    routingRef,
    createdAt,
    expiresAt
  }

ai:active:{supersessionKey}
→ executionId
```

`ai:active`는 같은 의미 범위에서 새 실행이 이전 실행을 대체해야 하는 Use Case에서만 사용한다. `supersessionKey`의 구성과 이전 execution 취소 정책은 각 Use Case가 정의한다.

AI 실행 시작 순서는 다음과 같다.

```text
1. idempotencyKey 검증
2. executionId 생성
3. execution status와 routingRef 저장
4. 필요한 경우 active execution 교체와 이전 execution 취소
5. Omni AI Server 호출
6. 호출자에게 executionId 반환
```

---

## Routing Reference Resolution

`routingRef`는 ADR의 공통 계약을 사용한다.

```text
routingRef
  tenantId
  userId
  targetRef
  connectionType (필요 시)
  scopeRef (필요 시)
```

Result Router는 Client나 Omni AI Server가 반환한 routing 위치를 신뢰하지 않는다. `executionId`로 Orchestrator가 저장한 `routingRef`를 조회한다.

`connectionType`은 특정 Client 종류로 delivery를 제한해야 하는 Use Case의 선택적 constraint다. `channelType`이 WebSocket/TCP transport를 의미하는 것과 달리, `connectionType`은 Messenger Client Ingress가 인증된 connection context에서 판정한 제품별 연결 구분이다. Returned Message Topics에서는 Desktop Client의 legacy `connectType 1, 2` 중 요청을 발생시킨 값을 trusted `routingRef`에 저장한다.

`scopeRef`가 있으면 `scopeRef.type`에 대응하는 validator로 Event가 아직 유효한지 확인한다. 예를 들어 Conversation Start는 `ROOM_SESSION`, Draft Assist는 `DRAFT_SESSION` validator를 사용할 수 있다. 별도 lifecycle이 없는 Event는 scope 검증을 생략한다.

`DEVICE_CURRENT` target은 Messenger Client Ingress가 전달한 인증된 `tenantId + userId + deviceId`를 사용해 다음과 같이 resolve한다.

```text
rt:device-current:{tenantId}:{userId}:{deviceId}
→ connectionId

rt:connection:{tenantId}:{connectionId}
→ ownerInstanceId
```

WebSocket Service MVP의 `CLIENT_SESSION_CURRENT` target은 WebSocket Service가 검증된 JWT의 `sid`에서 구성한 `tenantId + userId + clientSessionId`를 사용한다.

```text
rt:client-session-current:{tenantId}:{userId}:{clientSessionId}
→ connectionId

rt:connection:{tenantId}:{connectionId}
→ {
    userId,
    clientSessionId,
    connectionType,
    ownerInstanceId,
    leaseId
  }
```

Result Router는 resolved connection record의 `userId`와 `clientSessionId`가 trusted `routingRef`와 일치하는지 확인한다. ResultEvent나 Client가 전달한 별도 `clientSessionId`를 authoritative 값으로 사용하지 않는다.

trusted `routingRef.connectionType`이 있으면 resolved connection record의 값과 추가로 비교한다. 불일치하면 요청 당시 instance, 다른 connection type 또는 사용자의 다른 connection으로 fallback하지 않고 routing failure로 처리한다.

따라서 `ownerInstanceId`는 `routingRef`의 필수 입력이 아니다. 요청 시점의 instance 정보가 snapshot으로 포함되더라도 사용하지 않고 Result 처리 시점의 Registry 값을 최종 target으로 선택한다.

---

## Planned Core NATS Result Routing Flow

```text
1. ResultEvent의 executionId / eventId 검증
2. execution 상태가 Result를 수용할 수 있는지 확인
3. supersession 정책이 있으면 active executionId 일치 확인
4. scopeRef가 있으면 Use Case validator 실행
5. targetRef 정책에 따라 현재 connectionId resolve
6. Connection Registry에서 current ownerInstanceId와 connectionType resolve
7. routingRef.connectionType이 있으면 일치 여부 검증
8. owner instance subject로 DeliveryEvent publish
9. terminal event라면 execution 상태 전이와 조건부 active key 삭제
```

Core NATS subject 예시는 다음과 같다.

```text
ai.result.realtime.{ownerInstanceId}
ai.stream.realtime.{ownerInstanceId}
```

Stream event에서는 active execution을 유지한다. `COMPLETED`, `FAILED`, `CANCELLED` 같은 terminal event에서만 active key의 값이 자신의 `executionId`인지 확인한 후 삭제한다.

---

## Routing Failure

Registry 조회 직후 Client가 reconnect하면 선택한 instance에서 local connection을 찾지 못할 수 있다. Delivery acknowledgement를 지원한다면 local miss 이후 Registry를 다시 조회하고 제한된 횟수만 재라우팅한다.

Registry에 유효한 target이 없을 때 요청 당시 `sourceInstanceId`나 `ownerInstanceId`로 무조건 fallback하지 않는다. Use Case에 따라 다음 중 하나를 선택한다.

- 즉시성 stream: bounded retry 후 drop
- 재접속 후 확인 가능한 결과: durable store 또는 notification으로 전환
- scope가 종료되어 의미가 사라진 결과: 즉시 drop

Core NATS publish 성공은 Client 수신 성공을 의미하지 않는다. 수신 보장이 필요한 기능은 acknowledgement, durable event, outbox, idempotent redelivery를 별도로 설계한다.

---

## Non-responsibilities

- Realtime connection 소유
- Client protocol 변환과 socket write
- AI Workflow / LLM 실행
- Omni AI Server의 Stream 생성

---

## Related

- [ADR 001. Session Registry Based AI Result Routing](../../decisions/001-session-registry-result-routing.md)
- [ADR 005. WebSocket Service Result Routing by Authenticated Client Session](../../decisions/005-websocket-realtime-client-session-result-routing.md)
- [Messenger Client Ingress Result Delivery](../messenger-client-ingress/result-delivery.md)
- [Omni AI Server Result Event](../omni-ai-server/result-event.md)
