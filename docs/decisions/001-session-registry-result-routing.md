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

이때 Trigger 또는 API 요청을 처리한 `realtime-message-service` instance와 실제 WebSocket connection을 소유한 instance가 다를 수 있다.

```text
enterRoom 처리
Client → realtime-message-service #1

AI 처리 중 reconnect
Client WebSocket → realtime-message-service #6

Result push
realtime-message-service #1로 보내면 실패하거나 stale session이 된다.
```

Scale-out, scale-in, instance restart, reconnect, session migration, multi-device 상황에서도 같은 문제가 발생한다.

## Decision

AI Result delivery는 Trigger source instance가 아니라 Session Registry의 현재 owner 기준으로 routing한다.

WebSocket 연결 시 `realtime-message-service`는 Connection Registry에 현재 연결을 등록한다.

```text
connectionId
tenantId
userId
deviceId (필요 시)
ownerInstanceId
connectedAt
lastSeenAt
expiresAt
capabilities
```

`enterRoom` 호출 시에는 `roomSessionId`를 생성하고 현재 `connectionId` / `ownerInstanceId`와 연결한다.

```text
roomSessionId
roomId
connectionId
ownerInstanceId
enteredAt
lastSeenAt
expiresAt
```

AI 실행은 `triggerId` / `taskId` / `executionId`로 처리한다. Client delivery는 `connectionId` / `roomSessionId` / `routingRef`를 통해 현재 target을 resolve한다.

Result push 흐름:

```text
omni-ai-server
  ↓
AI stream / result
  ↓
ai-orchestrator 또는 Result Router
  ↓
Session Registry에서 현재 ownerInstanceId 확인
  ↓
Core NATS instance subject로 publish
  ↓
해당 realtime-message-service instance
  ↓
local WebSocket session으로 최종 push
```

`sourceInstanceId`는 correlation 정보로만 사용한다. 최종 delivery source of truth로 사용하지 않는다.

## Alternatives

### Trigger source instance로 push

Trigger를 발행하거나 API를 처리한 instance로 Result를 돌려보내는 방식이다.

장점은 구현이 단순하다는 점이다. 하지만 AI 처리 중 reconnect나 instance restart가 발생하면 stale session으로 push하게 된다. `enterRoom` 처리 instance와 WebSocket owner가 달라지는 상황도 처리하기 어렵다.

### Broadcast 후 local session이 있는 instance가 처리

모든 `realtime-message-service` instance에 Result를 broadcast하고, local session이 있는 instance만 push하는 방식이다.

구현은 직관적이지만 instance 수가 늘수록 불필요한 fan-out이 커진다. tenant / user / room 기준 privacy boundary와 observability도 흐려진다.

### ai-orchestrator가 WebSocket stream을 직접 proxy

`ai-orchestrator`가 token stream을 받아 Client까지 직접 중계하는 방식이다.

Routing 판단은 한 곳에 모을 수 있지만, `ai-orchestrator`가 Streaming Data Plane이 되어 부하와 장애 영향 범위가 커진다. WebSocket session ownership도 `realtime-message-service`와 중복된다.

## Consequences

좋아지는 점:

- Client explicit request와 Server trigger를 같은 delivery 원칙으로 처리할 수 있다.
- reconnect, scale-out, instance restart 중에도 현재 WebSocket owner 기준으로 push할 수 있다.
- `ai-orchestrator`는 실행 correlation과 routing decision에 집중하고, 최종 WebSocket delivery는 `realtime-message-service`에 남길 수 있다.
- Client Tool Relay도 같은 `connectionId` / `roomSessionId` / `ownerInstanceId` 모델을 사용할 수 있다.

감수해야 할 점:

- Session Registry의 TTL, heartbeat, stale owner 제거 정책이 필요하다.
- Result push 직전 registry 조회 실패, owner instance unavailable, local session missing 상황의 fallback이 필요하다.
- stream reconnect / resume 정책을 별도로 정의해야 한다.
- multi-device, multi-tab, same user multi-room 진입 정책을 명확히 해야 한다.

## Related

- [AiTask and Queue](../ai-task-and-queue.md)
- [Server-driven AI](../server-driven-ai.md)
- [Client Integration](../client-integration.md)
- [Design Decisions](../design-decisions.md)
