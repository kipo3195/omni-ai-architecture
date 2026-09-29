# Realtime Service Result Delivery

Status: Designed

---

## Purpose

Realtime Service가 여러 instance로 실행되는 환경에서 Client connection의 현재 owner를 등록하고, AI Orchestrator가 선택한 delivery event를 정확한 local connection으로 전달하는 구현 책임을 정의한다.

공통 `routingRef`, `ResultEvent`, `DeliveryEvent` 계약과 End-to-End 흐름은 [ADR 001](../../../decisions/001-session-registry-result-routing.md)을 따른다.

---

## Connection Registry

단일 활성 connection 정책의 Redis key 예시는 다음과 같다.

```text
rt:device-current:{tenantId}:{userId}:{deviceId}
→ connectionId

rt:connection:{tenantId}:{connectionId}
→ {
    userId,
    deviceId,
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

한 device에서 multi-tab 또는 복수 channel의 동시 연결을 허용한다면 `device-current` 단일 pointer 대신 connection set을 유지하고 명시적인 target-selection policy를 적용한다.

---

## Connect and Reconnect

연결 등록 순서는 다음과 같다.

```text
1. connectionId / leaseId 생성
2. local connection map에 connectionId → socket 등록
3. rt:connection:{connectionId} 저장
4. rt:device-current:{userId}:{deviceId}를 새 connectionId로 교체
```

Reconnect는 기존 connection을 수정하지 않고 새 connection을 등록한다.

```text
Before
device current → connection-A
connection-A   → realtime-instance-1

After reconnect
device current → connection-B
connection-A   → realtime-instance-1
connection-B   → realtime-instance-5
```

오래된 connection record는 disconnect 처리 또는 TTL 만료 전까지 잠시 존재할 수 있다. 현재 target을 선택할 때는 ADR의 `targetRef` 정책과 current pointer를 사용한다.

---

## Heartbeat and Disconnect

WebSocket ping/pong 또는 TCP heartbeat에 맞춰 `lastSeenAt`과 TTL을 갱신한다. Redis write를 줄이기 위해 debounce할 수 있지만 TTL 만료 전에 충분한 여유를 두고 갱신한다.

Heartbeat는 현재 record의 `connectionId + leaseId`가 자신의 값과 일치할 때만 갱신한다. Current pointer의 TTL도 여전히 자신의 `connectionId`를 가리킬 때만 연장한다.

Disconnect는 current pointer를 무조건 삭제하지 않는다.

```text
if device-current == disconnectingConnectionId:
    delete device-current

delete connection:{disconnectingConnectionId}
```

비교와 갱신 또는 삭제는 Redis transaction이나 Lua script로 원자적으로 처리한다. 이전 connection의 heartbeat나 disconnect가 새 connection 상태를 덮어쓰거나 삭제하면 안 된다.

---

## Client Request Resolution

WebSocket 요청은 local connection context에서 `connectionId`를 얻는다. Client가 `connectionId`나 `ownerInstanceId`를 별도로 전달할 필요가 없다.

REST 요청은 Load Balancer에 의해 connection owner가 아닌 instance로 들어올 수 있다.

```text
Client REST Request
  - Authorization
  - deviceId 또는 connectionToken
→ 임의의 Realtime Service instance
→ 인증된 tenantId / userId와 target token 검증
→ Connection Registry에서 current connectionId resolve
→ AI Orchestrator 호출
```

단일 활성 connection 정책은 `tenantId + userId + deviceId`로 조회할 수 있다. 복수 connection 정책은 WebSocket 연결 시 짧은 수명의 opaque `connectionToken`을 발급하고 REST 요청에서 이를 검증한다. Client가 보낸 내부 ID만으로 권한을 부여하지 않는다.

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

`deviceId`만 일치한다는 이유로 다른 local connection에 전달하지 않는다. Local connection이 없으면 delivery miss로 처리한다. Acknowledgement 경로가 있다면 AI Orchestrator가 Registry를 재조회할 수 있도록 miss를 반환하고, 없다면 metric과 structured log를 남긴다.

---

## Related

- [ADR 001. Session Registry Based AI Result Routing](../../../decisions/001-session-registry-result-routing.md)
- [AI Orchestrator Result Router](../../ai-orchestrator/result-router/README.md)
- [Omni AI Server Result Event](../../omni-ai-server/result-event/README.md)
