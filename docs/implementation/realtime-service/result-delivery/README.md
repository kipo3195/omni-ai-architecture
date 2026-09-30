# Realtime Service Result Delivery

Status: Designed

---

## Purpose

Realtime Service가 여러 instance로 실행되는 환경에서 Client connection의 현재 owner를 등록하고, AI Orchestrator가 선택한 delivery event를 정확한 local connection으로 전달하는 구현 책임을 정의한다.

공통 `routingRef`, `ResultEvent`, `DeliveryEvent` 계약과 End-to-End 흐름은 [ADR 001](../../../decisions/001-session-registry-result-routing.md)을 따른다. WebSocket Realtime Service MVP의 인증 세션 기반 target과 단일 활성 connection 정책은 [ADR 005](../../../decisions/005-websocket-realtime-client-session-result-routing.md)를 따른다.

---

## Connection Registry

WebSocket Realtime Service MVP는 인증된 로그인 세션별 단일 활성 connection을 유지한다.

```text
rt:client-session-current:{tenantId}:{userId}:{clientSessionId}
→ connectionId

rt:connection:{tenantId}:{connectionId}
→ {
    userId,
    clientSessionId,
    channelType,
    ownerServiceType,
    ownerInstanceId,
    leaseId,
    connectedAt,
    lastSeenAt,
    capabilities
  }
```

`connectionId`와 `leaseId`는 연결할 때마다 새로 생성한다. `connectionId`는 UUIDv7 또는 ULID처럼 충돌 가능성이 낮은 값을 사용하고, process-local object hash나 timestamp 단독 값은 사용하지 않는다.

Native 또는 TCP Client가 안정적인 설치나 장치 식별자를 제공하면 별도의 `rt:device-current:{tenantId}:{userId}:{deviceId}` pointer와 `DEVICE_CURRENT` target을 사용할 수 있다. `deviceId`와 `clientSessionId` 모두 connection ID가 아니라 현재 `connectionId`를 찾기 위한 논리적 pointer다.

WebSocket Realtime Service MVP는 동일 `clientSessionId`에 여러 활성 connection을 허용하지 않는다. 향후 복수 connection 또는 복수 channel의 동시 연결을 허용하면 단일 pointer 대신 connection set이나 연결별 `connectionToken`을 도입하고 명시적인 target-selection policy를 적용한다.

---

## Authentication Session Contract

WebSocket Realtime Service Client의 로그인과 token refresh 계약은 다음과 같다.

```text
REST Login
→ 인증 서버가 clientSessionId 생성
→ JWT 발급
  - tenantId
  - userId
  - sid: clientSessionId

Access Token Refresh
→ 같은 로그인 세션이면 기존 sid 유지

New Login Session
→ 새로운 sid 발급
```

`sid`는 access token 자체나 token별 `jti`와 구분한다. `jti`는 token 재발급 때 변경될 수 있으므로 current connection pointer로 사용하지 않는다.

REST와 WebSocket 요청을 처리하는 Realtime Service는 JWT의 서명, 만료, issuer, audience를 검증한 뒤 `tenantId`, `userId`, `sid`를 추출한다. Client가 별도 header, request body 또는 WebSocket message로 보낸 `clientSessionId`를 routing 권한의 근거로 사용하지 않는다.

로그아웃, 인증 세션 만료 또는 강제 session revocation 시 해당 `clientSessionId`의 current pointer와 활성 connection을 무효화한다. WebSocket을 JWT 만료 시점에 종료할지, 재인증을 요구할지는 Authentication 정책과 함께 정하되 Registry 상태와 불일치하지 않게 처리한다.

---

## Connect and Reconnect

연결 등록 순서는 다음과 같다.

```text
1. 검증된 JWT에서 tenantId / userId / sid 추출
2. connectionId / leaseId 생성
3. local connection map에 connectionId → socket 등록
4. rt:connection:{tenantId}:{connectionId} 저장
5. rt:client-session-current:{tenantId}:{userId}:{sid}를 새 connectionId로 원자적 교체
6. 이전 connection이 있으면 SESSION_REPLACED를 전달하고 종료
```

Reconnect는 기존 connection을 수정하지 않고 새 connection을 등록한다.

```text
Before
client session current → connection-A
connection-A           → realtime-instance-1

After reconnect
client session current → connection-B
connection-A           → realtime-instance-1
connection-B           → realtime-instance-5
```

새 connection이 current pointer를 교체하도록 하는 이유는 Client reload나 네트워크 복구 과정에서 stale connection 때문에 정상 reconnect가 거절되는 것을 피하기 위해서다. 오래된 connection record는 disconnect 처리 또는 TTL 만료 전까지 잠시 존재할 수 있다. 현재 target을 선택할 때는 ADR의 `targetRef` 정책과 current pointer를 사용한다.

---

## Heartbeat and Disconnect

WebSocket ping/pong 또는 TCP heartbeat에 맞춰 `lastSeenAt`과 TTL을 갱신한다. Redis write를 줄이기 위해 debounce할 수 있지만 TTL 만료 전에 충분한 여유를 두고 갱신한다.

Heartbeat는 현재 record의 `connectionId + leaseId`가 자신의 값과 일치할 때만 갱신한다. Current pointer의 TTL도 여전히 자신의 `connectionId`를 가리킬 때만 연장한다.

Disconnect는 current pointer를 무조건 삭제하지 않는다.

```text
if client-session-current == disconnectingConnectionId:
    delete client-session-current

delete connection:{tenantId}:{disconnectingConnectionId}
```

비교와 갱신 또는 삭제는 Redis transaction이나 Lua script로 원자적으로 처리한다. 이전 connection의 heartbeat나 disconnect가 새 connection 상태를 덮어쓰거나 삭제하면 안 된다.

---

## Client Request Resolution

WebSocket 요청은 local connection context에서 `connectionId`를 얻는다. Client가 `connectionId`나 `ownerInstanceId`를 별도로 전달할 필요가 없다.

REST 요청은 Load Balancer에 의해 connection owner가 아닌 instance로 들어올 수 있다.

```text
Client REST Request
  - Authorization: Bearer <JWT>
→ 임의의 Realtime Service instance
→ JWT 검증 후 tenantId / userId / sid 추출
→ Connection Registry에서 current connectionId resolve
→ AI Orchestrator 호출
```

WebSocket Realtime Service MVP의 단일 활성 connection 정책은 `tenantId + userId + clientSessionId`로 조회한다. Realtime Service는 검증된 JWT의 `sid`로 `clientSessionId`를 구성하므로 Client가 별도 routing ID를 전달할 필요가 없다.

현재 인증 세션의 connection으로 결과를 전달하는 요청은 Realtime Service가 다음 `routingRef`를 구성해 AI Orchestrator에 전달한다.

```text
routingRef
  tenantId
  userId
  targetRef
    type: CLIENT_SESSION_CURRENT
    clientSessionId
  scopeRef (필요 시)
```

Realtime Service는 현재 `ownerInstanceId`를 최종 routing 값으로 확정해 전달하지 않는다. 요청 시점의 connection 조회는 인증과 요청 검증에 사용하고, 최종 owner는 AI Orchestrator가 Result 전달 시점에 Registry에서 다시 조회한다. 로그인 세션 경계를 유지하기 때문에 기존 세션에서 시작한 Result를 로그아웃 후 생성된 다른 `clientSessionId`로 자동 전달하지 않는다.

Connection-bound Event라면 `targetRef.type`을 `CONNECTION`으로 설정하고 검증된 `connectionId`를 전달한다. 향후 요청한 exact connection을 지정해야 한다면 WebSocket 연결별 짧은 수명의 opaque `connectionToken`을 발급하고 REST 요청에서 검증하는 정책을 추가한다.

AI Event별 `scopeRef`가 필요하다면 해당 Use Case owner가 생성하거나 검증한다. 예를 들어 Conversation Start의 `roomSessionId`는 하나의 scope ID일 뿐 Realtime Result Delivery의 공통 필수가 아니다.

---

## Local Delivery

Realtime Service는 instance subject에서 받은 `DeliveryEvent.connectionId`로 local connection을 찾는다.

```text
1. exact connectionId local lookup
2. event.userId와 local connection의 인증 주체 비교
3. 필요한 경우 scopeRef를 shared Registry로 재검증
4. Client protocol로 변환하여 전송
```

`clientSessionId`나 `deviceId`만 일치한다는 이유로 다른 local connection에 전달하지 않는다. `DeliveryEvent.connectionId`와 local connection의 인증 주체가 모두 일치해야 한다. Local connection이 없으면 delivery miss로 처리한다. Acknowledgement 경로가 있다면 AI Orchestrator가 Registry를 재조회할 수 있도록 miss를 반환하고, 없다면 metric과 structured log를 남긴다.

---

## Related

- [ADR 001. Session Registry Based AI Result Routing](../../../decisions/001-session-registry-result-routing.md)
- [ADR 005. WebSocket Realtime Service Result Routing by Authenticated Client Session](../../../decisions/005-websocket-realtime-client-session-result-routing.md)
- [AI Orchestrator Result Router](../../ai-orchestrator/result-router/README.md)
- [Omni AI Server Result Event](../../omni-ai-server/result-event/README.md)
