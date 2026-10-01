# 005. WebSocket Realtime Service Result Routing by Authenticated Client Session

Status: Accepted

## Context

WebSocket Realtime Service MVP 대상 Client는 신뢰할 수 있는 별도 `deviceId`를 제공하지 않는다. Browser fingerprint나 Electron runtime 정보는 안정적인 식별자가 아니며 privacy와 보안 측면에서도 Realtime routing key로 사용하기 적합하지 않다.

현재 로그인 흐름은 REST 인증 요청으로 JWT를 발급하고, WebSocket 연결과 이후 REST 요청에서 같은 JWT를 검증해 인증 주체를 확인한다. 이 구조에서 Client가 별도 식별자를 생성하고 영구 저장하도록 요구하면 다음 정책이 Client 구현에 분산된다.

- 식별자 생성 방식과 저장 위치
- 로그아웃 또는 계정 변경 시 초기화 정책
- Client storage 삭제와 private browsing 처리
- multi-tab 또는 multi-process 공유 방식
- 식별자 유실, 중복, 재설치 처리
- Browser, Electron, Native, TCP Client별 구현 차이

MVP에서는 동일 로그인 세션에 하나의 활성 WebSocket connection만 허용한다. AI 실행 중 reconnect가 발생하더라도 Result는 같은 로그인 세션의 새로운 connection으로 전달되어야 한다.

## Decision

WebSocket Realtime Service MVP는 인증 서버가 로그인 세션 단위의 `clientSessionId`를 생성하고 JWT의 `sid` claim으로 발급한다.

```text
REST Login
→ clientSessionId 생성
→ JWT 발급
  - tenantId
  - userId
  - sid: clientSessionId
```

`sid`는 JWT 자체나 access token별 `jti`가 아니다. 같은 로그인 세션에서 access token을 refresh하면 기존 `sid`를 유지하고, 새로운 로그인 세션을 만들 때만 새로운 `sid`를 발급한다.

Realtime Service는 Client가 request body, header 또는 WebSocket message로 제출한 별도 routing identifier를 신뢰하지 않는다. 검증된 JWT에서 `tenantId`, `userId`, `sid`를 추출하고 서버 내부에서는 `sid`를 `clientSessionId`로 사용한다.

WebSocket Realtime Service의 공통 routing target에 `CLIENT_SESSION_CURRENT`를 사용한다.

```text
routingRef
  tenantId
  userId
  targetRef
    type: CLIENT_SESSION_CURRENT
    clientSessionId
```

`clientSessionId`는 connection 자체가 아니라 현재 connection을 찾기 위한 논리적 pointer다. WebSocket 연결마다 재사용하지 않는 `connectionId`를 새로 생성한다. `leaseId`는 향후 connection ownership 갱신이나 인계가 필요한 경우 사용할 수 있는 확장 필드이며 MVP에서는 사용하지 않는다.

```text
rt:client-session-current:{tenantId}:{userId}:{clientSessionId}
→ connectionId

rt:connection:{tenantId}:{connectionId}
→ {
    userId,
    clientSessionId,
    ownerServiceType,
    ownerInstanceId,
    leaseId,
    connectedAt,
    lastSeenAt,
    capabilities
  }
```

동일한 `clientSessionId`로 새 WebSocket이 연결되면 새 connection이 current pointer를 원자적으로 교체한다. 이전 connection에는 가능한 경우 `SESSION_REPLACED`를 전달한 뒤 종료한다. 새 연결을 거절하지 않는 이유는 Client reload나 네트워크 복구 과정에서 stale connection 때문에 정상 reconnect가 막히는 것을 피하기 위해서다.

Heartbeat와 disconnect는 자신이 current connection의 소유자인 경우에만 current pointer를 갱신하거나 삭제한다. 이전 connection의 늦은 heartbeat나 disconnect가 새 connection 상태를 덮어쓰면 안 된다.

### Connection lifecycle and TTL

`CLIENT_SESSION_CURRENT` pointer와 connection 상세정보에는 TTL을 적용한다. TTL은 정상 disconnect 처리를 대신하지 않으며, Process crash, network partition 또는 callback 누락으로 명시적인 정리가 수행되지 못했을 때 stale connection 정보를 제거하기 위한 최종 안전장치다.

정상 disconnect에서는 종료되는 connection의 상세정보를 즉시 삭제한다. Current pointer는 pointer가 가리키는 `connectionId`가 종료되는 connection과 일치할 때만 삭제한다. 동일한 `clientSessionId`로 reconnect가 완료된 후 이전 connection의 disconnect가 늦게 실행되더라도 새로운 current pointer를 삭제해서는 안 된다.

Heartbeat가 확인된 활성 connection은 주기적으로 다음 정보를 갱신한다.

- current pointer TTL
- connection 상세정보 TTL
- `lastSeenAt`

Heartbeat callback마다 Redis를 갱신할 필요는 없다. WebSocket Server는 connection별 마지막 heartbeat 수신 시각과 마지막 Redis 갱신 시각을 관리하고, 설정된 갱신 주기가 지난 활성 connection만 TTL을 연장한다. TTL과 갱신 주기는 고정된 protocol 값이 아니라 운영 환경에 따라 조정할 수 있는 설정값으로 관리한다.

Current pointer 확인과 TTL 갱신 또는 삭제는 하나의 원자적 Redis 연산으로 처리한다. 비교와 변경을 별도 명령으로 실행하면 비교 직후 reconnect가 발생하여 이전 connection이 새로운 current connection의 상태를 갱신하거나 삭제할 수 있다.

```text
compare-and-refresh:
  current == connectionId
    → current TTL 갱신
    → connection TTL 갱신
    → lastSeenAt 갱신

compare-and-delete:
  current == connectionId
    → current 삭제
  종료 대상 connection 상세정보 삭제
```

Redis Lua script 또는 동일한 원자성을 제공하는 방식을 사용한다. 원자적 연산이 실행되는 동안 다른 connection의 reconnect, heartbeat 또는 disconnect 명령은 중간에 끼어들지 않으며 연산 단위로 직렬 실행된다.

MVP에서는 WebSocket 연결마다 재사용하지 않는 ULID `connectionId`를 생성하므로 current connection의 소유권 비교에 `connectionId`를 사용한다. 향후 connection ownership을 별도로 갱신하거나 인계해야 하는 요구가 생기면 `leaseId`를 발급하고 `connectionId + leaseId`를 함께 비교한다.

로그아웃, 인증 세션 만료 또는 강제 session revocation 시 해당 `clientSessionId`의 current pointer와 활성 connection을 무효화한다. 기존 인증 세션에서 시작한 AI Result를 이후 생성된 다른 로그인 세션으로 자동 전달하지 않는다.

Native 또는 TCP Client가 안정적인 설치 또는 장치 식별자를 제공하는 경우 기존 `DEVICE_CURRENT` 정책을 계속 사용할 수 있다. Request 당시 exact connection에만 전달해야 하는 Use Case는 `CONNECTION`을 사용한다.

## Alternatives

### Client가 생성하고 저장한 deviceId

Client가 UUID를 생성해 local storage에 보관할 수 있다. 하지만 storage 선택, 복수 실행 환경 간 공유, 삭제, private browsing, 로그아웃 처리 정책이 Client별로 분산되고 Client 동시 배포가 필요하다.

### Client fingerprint

별도 저장 없이 Browser나 Electron runtime 특성으로 device를 추정할 수 있지만 값이 안정적이지 않고 privacy 위험이 있다. 인증이나 정확한 Result routing target으로 사용하지 않는다.

### JWT jti 사용

`jti`는 access token을 새로 발급할 때 변경될 수 있다. token refresh가 Realtime routing identity 변경으로 이어지므로 로그인 세션의 current connection pointer로 사용하지 않는다.

### WebSocket connectionToken 발급

WebSocket 연결별 opaque token을 발급하고 REST 요청에서 함께 전달하면 요청한 exact connection을 식별할 수 있다. 다만 Client가 token을 수신, 저장, 전달해야 하며 reconnect 시 Result 연속성 정책도 추가로 필요하다. MVP의 단일 활성 connection 정책에는 사용하지 않고 향후 복수 connection 또는 exact-connection routing 요구가 생길 때 도입한다.

### tenantId + userId만 사용

구현은 단순하지만 로그아웃 후 다른 장치나 새로운 로그인 세션으로 이전 실행의 Result가 넘어갈 수 있다. 인증 세션 경계를 유지하기 위해 `clientSessionId`를 포함한다.

## Consequences

좋아지는 점:

- Client는 플랫폼별 device identifier를 추출, 생성, 저장, 동기화할 필요가 없다.
- Client는 별도 routing identifier를 REST 또는 WebSocket payload로 전달하지 않는다.
- REST와 WebSocket이 기존 JWT 인증 흐름을 그대로 사용한다.
- WebSocket, Native, TCP Client의 식별 기능 차이가 공통 인증 계약으로 전파되는 것을 줄인다.
- 식별자 발급과 검증이 서버의 authentication trust boundary 안에서 처리된다.
- 서버 중심으로 단계적으로 배포할 수 있어 Client 동시 배포 의존성이 낮다.
- 동일 로그인 세션의 reconnect 후에도 현재 connection으로 Result를 전달할 수 있다.

감수해야 할 점:

- Authentication Session과 Realtime Connection lifecycle 사이의 결합이 증가한다.
- access token refresh에서 `sid`를 유지하고, logout과 session revocation을 Connection Registry 및 활성 WebSocket과 연동해야 한다.
- JWT 검증 자체는 stateless하게 유지할 수 있지만 current connection 조회를 위한 Registry 상태와 운영 책임이 필요하다.
- Heartbeat 기반 TTL 갱신과 disconnect 정리는 current connection 소유권을 원자적으로 확인해야 하며, Redis 장애 시 stale 정보는 TTL 만료까지 남을 수 있다.
- `clientSessionId`는 물리 장치나 Client 설치를 식별하지 않으므로 신뢰 기기 관리, 장치 감사, push token 또는 장기 분석 ID로 사용할 수 없다.
- 동일 JWT를 공유하는 복수 connection은 구분할 수 없다. MVP에서는 새 connection이 기존 connection을 교체해 하나의 활성 connection만 유지한다.
- 향후 복수 connection, exact-connection delivery 또는 실제 device management가 필요하면 `connectionToken`, connection set 또는 별도 device identifier가 필요하다.
- `sid` 발급이나 refresh 구현 오류가 Realtime routing 실패로 이어질 수 있으므로 Authentication과 Realtime Service 사이의 계약 테스트가 필요하다.

## Related

- [ADR 001. Session Registry Based AI Result Routing](001-session-registry-result-routing.md)
- [Realtime Service Result Delivery](../implementation/realtime-service/result-delivery/README.md)
- [AI Orchestrator Result Router](../implementation/ai-orchestrator/result-router/README.md)
