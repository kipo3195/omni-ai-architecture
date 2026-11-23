# Messenger Client Ingress Pending AI Actions

Status: Planned

---

## Purpose

AI Orchestrator가 Client의 후속 입력을 기다리는 action을 생성했지만 WebSocket 전달 시점에 연결이 끊기거나 Client가 event를 받기 전에 재접속할 수 있다. Messenger Client Ingress는 실시간 WebSocket 전달과 별도로, 인증된 Client가 현재 알아야 할 유효한 AI action을 `sid` 기준으로 다시 조회할 수 있는 복구 경로를 제공한다.

WebSocket event는 낮은 지연을 위한 정상 전달 경로이고 Pending Actions API는 유실 복구와 현재 상태 동기화를 위한 경로다. Pending action 저장소는 AI Orchestrator lifecycle의 Source of Truth가 아니라 Messenger Client Ingress의 짧은 수명 delivery projection이다.

```text
AI Orchestrator action 생성
→ Messenger Client Ingress가 pending action projection 저장
→ 현재 connection에 WebSocket event 전달
   ├─ 정상 수신: Client가 action 표시
   └─ 전달 전 disconnect 또는 event 유실
       → Client reconnect
       → Access Token의 tenantId / userId / sid로 pending action 재조회
       → Client UI 복구
```

## Decision

- Client가 알아야 할 AI action은 기능별 WebSocket event로 즉시 전달한다.
- WebSocket으로 action을 보내기 전에 Redis에 복구 가능한 projection을 저장한다.
- 동일 action의 중복 WebSocket 전달은 Redis의 원자적 create-if-absent 결과로 제어한다.
- 재접속한 Client는 공통 `GET /rest/ai/pending_actions` API로 현재 `sid` 범위의 유효한 action을 조회한다.
- API는 action ID만 반환하지 않고 Client가 추가 왕복 없이 UI를 복구할 수 있는 최소 payload를 함께 반환한다.
- Pending action은 조회만으로 완료되지 않는다. 사용자 결정이나 기능별 terminal 처리에서 제거하거나 상태를 전이한다.
- 기능별 권위 상태와 후속 실행 정책은 AI Orchestrator가 계속 소유한다.

## Returned Message Confirmation Flow

Returned Message Topics의 `USER_RETURNED` 요청은 Presence owner가 상태 변경 REST 응답과 분리해 AI Orchestrator로 전달한다. AI Orchestrator 응답이 다음 조건을 만족하면 Messenger Client Ingress는 확인 action을 생성한다.

```text
outcome = CREATED
status = AWAITING_CONFIRMATION
confirmationId is present
confirmationExpiresAt is in the future
```

전체 흐름은 다음과 같다.

```text
Client 상태 변경
→ 상태 변경 REST 응답
→ Messenger Client Ingress가 USER_RETURNED를 AI Orchestrator에 제출
→ AI Orchestrator가 AWAITING_CONFIRMATION 응답
→ Returned Message confirmation projection을 Redis에 원자적으로 등록
→ 요청 routingRef의 tenantId / userId / clientSessionId로 현재 connection resolve
→ 요청한 Client connection에 WebSocket event 전달
→ Client가 confirmation UI 표시
→ Client가 별도 confirmation API로 사용자 결정을 제출
```

상태 변경 응답은 AI Orchestrator 처리 완료를 기다리지 않는다. 다만 실제 HTTP response flush와 비동기 작업 실행 순서에 의존하지 않도록 Client는 상태 변경 응답과 WebSocket event를 독립적으로 처리한다.

Returned Message Topics는 인증된 connection의 `connectionType 1, 2, 3`을 허용한다. Presence owner가 검증해 `routingRef`에 저장한 값을 delivery projection에서도 유지한다.

### AI Orchestrator Response

```json
{
  "requestId": "request-1",
  "triggerId": "trigger-1",
  "outcome": "CREATED",
  "reason": null,
  "status": "AWAITING_CONFIRMATION",
  "unreadCount": 3,
  "confirmationId": "confirmation-1",
  "messageSelectionRef": "selection-1",
  "confirmationExpiresAt": "2026-10-19T06:05:00Z",
  "executionId": null
}
```

### Client WebSocket Event

Messenger Client protocol의 field name은 snake_case를 사용한다.

```json
{
  "type": "returned_message_confirmation_required",
  "data": {
    "request_id": "request-1",
    "trigger_id": "trigger-1",
    "absence_period_id": "absence-1",
    "outcome": "CREATED",
    "status": "AWAITING_CONFIRMATION",
    "unread_count": 3,
    "confirmation_id": "confirmation-1",
    "message_selection_ref": "selection-1",
    "confirmation_expires_at": "2026-10-19T06:05:00Z",
    "execution_id": null
  }
}
```

`request_id`, `trigger_id`, `absence_period_id`와 routing context는 Client 입력으로 다시 신뢰하지 않는다. Client confirmation을 처리할 때 인증 context와 저장된 pending projection을 기준으로 검증한다.

## Redis Projection

Returned Message confirmation 본문은 다음 키에 저장한다.

```text
ai:returned-confirmation:{confirmationId}
```

예시 value:

```json
{
  "action_type": "returned_message_confirmation",
  "request_id": "request-1",
  "trigger_id": "trigger-1",
  "absence_period_id": "absence-1",
  "confirmation_id": "confirmation-1",
  "message_selection_ref": "selection-1",
  "unread_count": 3,
  "tenant_id": "ucware",
  "user_id": "demo04@ucware",
  "client_session_id": "82b75440-c1ea-4c4b-b245-ae8ccc17c290",
  "connection_type": 3,
  "confirmation_expires_at": "2026-10-19T06:05:00Z",
  "status": "AWAITING_CONFIRMATION"
}
```

TTL은 `confirmationExpiresAt`과 같은 절대 만료 시각에 맞춘다. Redis API가 초 단위 상대 TTL만 지원하면 저장 시점과 `confirmationExpiresAt`의 차이를 계산한다. 계산 결과가 0 이하이면 저장하거나 Client에 전달하지 않는다. 가능하면 Redis의 absolute expiration을 사용해 처리 지연과 TTL 반올림 오차를 줄인다.

동일 `confirmationId`가 여러 번 도착할 수 있으므로 존재 여부 조회와 저장을 별도 명령으로 실행하지 않는다. `SET NX` 또는 Lua script로 본문 저장과 최초 도착 판정을 원자적으로 처리하고, 최초 저장에 성공한 처리만 WebSocket event를 전송한다.

```text
create-if-absent = true
→ 최초 action
→ WebSocket event 전송

create-if-absent = false
→ 중복 응답
→ WebSocket event 재전송 생략
```

WebSocket 전송에 실패하거나 local connection이 없더라도 projection은 TTL까지 유지한다. Client는 재접속 후 Pending Actions API로 이를 복구한다.

## SID-scoped Index

재접속한 Client는 전달받지 못한 `confirmationId`를 알 수 없으므로 confirmation 본문 키만으로 복구할 수 없다. Redis key scan을 피하기 위해 인증 세션별 보조 인덱스를 둔다.

```text
ai:pending-action-index:{tenantId}:{encodedUserId}:{sid}
```

동시에 여러 action이 존재할 수 있으므로 Sorted Set을 사용한다.

```text
member = {actionType}:{actionId}
score = expiresAt epoch seconds
```

Returned Message confirmation의 member 예시는 다음과 같다.

```text
returned_message_confirmation:confirmation-1
```

본문 create-if-absent, SID index 등록과 필요한 기존 state 정리는 가능하면 하나의 Redis Lua script로 처리한다. 본문이 만료됐지만 index member가 남는 경우 조회 과정에서 stale member를 제거한다. Index 자체의 TTL은 포함된 action 중 가장 늦은 만료 시각보다 이르게 설정하지 않는다.

## Pending Actions API

```http
GET /rest/ai/pending_actions?limit=50&cursor={opaqueCursor}&action_type={optionalType}
Authorization: Bearer {accessToken}
```

Messenger Client Ingress는 Access Token을 검증한 뒤 `tenantId`, `userId`, `sid`를 추출한다. Client가 query, header 또는 body로 별도 제출한 tenant, user 또는 sid를 조회 권한의 근거로 사용하지 않는다.

응답 예시:

```json
{
  "result_code": 0,
  "result_message": "success",
  "data": {
    "server_time": "2026-10-19T06:01:00Z",
    "next_cursor": null,
    "items": [
      {
        "action_type": "returned_message_confirmation",
        "action_id": "confirmation-1",
        "status": "AWAITING_CONFIRMATION",
        "expires_at": "2026-10-19T06:05:00Z",
        "payload": {
          "request_id": "request-1",
          "trigger_id": "trigger-1",
          "absence_period_id": "absence-1",
          "confirmation_id": "confirmation-1",
          "message_selection_ref": "selection-1",
          "unread_count": 3
        }
      }
    ]
  }
}
```

조회 처리는 다음 순서를 따른다.

```text
1. JWT의 tenantId / userId / sid 검증
2. SID index에서 현재 시각 이후의 member를 page 단위로 조회
3. 기능별 projection 본문 조회
4. 본문의 tenantId / userId / clientSessionId와 인증 context 재검증
5. 만료됐거나 terminal 상태인 항목 제외
6. 사라진 본문과 만료 member를 index에서 정리
7. 유효한 action과 opaque next cursor 반환
```

이 API는 Client가 다음 시점에 호출한다.

- WebSocket reconnect 완료 직후
- 페이지 reload 또는 Client process 재시작 후 인증 세션 복구 시
- Client가 WebSocket sequence gap 또는 local state 불일치를 감지했을 때

조회 성공은 action 처리 완료 acknowledgement가 아니다. 조회만으로 projection을 삭제하거나 `AWAITING_CONFIRMATION` 상태를 변경하지 않는다.

## Client Confirmation Placeholder

현재 Messenger Client Ingress는 향후 confirmation 후속 처리를 연결하기 위한 임시 API를 유지한다.

```http
POST /rest/return_message_confirmation
Authorization: Bearer {accessToken}
Content-Type: application/json
```

```json
{
  "confirmation": true,
  "key": "confirmation-1"
}
```

현재 구현은 인증된 `userId`, `confirmation`, `key`를 검증하고 로깅한 뒤 공통 성공 응답을 반환한다. DB를 변경하거나 AI Orchestrator confirmation API를 호출하지 않는다. 이 API를 실제 사용자 결정 처리로 전환할 때는 pending projection의 `userId`, `clientSessionId`, 만료 시각과 상태를 먼저 검증하고, AI Orchestrator 후속 호출 및 멱등 상태 전이를 함께 구현한다.

## Authentication Session Scope

같은 로그인 세션의 Access Token refresh와 WebSocket reconnect는 기존 `sid`를 유지하므로 같은 pending action을 복구한다. 새로운 로그인으로 새 `sid`가 발급되면 이전 `sid`의 action은 자동으로 노출하지 않는다. 이는 요청한 인증 세션으로만 action을 전달한다는 현재 routing 정책과 일치한다.

향후 새 로그인 세션에서도 이전 action을 복구해야 한다면 user-scoped index, action rebinding과 권한 정책을 별도 결정한다. 현재 구현에서 다른 `sid`로 자동 fallback하거나 같은 사용자의 모든 connection에 broadcast하지 않는다.

## Source Absence Retention

`USER_RETURNED` 요청을 만들기 위해 사용한 absence record를 AI Orchestrator 호출 전에 삭제하면 다음 유실 구간이 생긴다.

```text
absence record 삭제
→ AI Orchestrator 호출
→ timeout / connection failure / retryable 5xx
→ request를 재구성할 offlineSince와 absencePeriodId 소실
```

따라서 최소 보장 흐름은 다음과 같다.

```text
absence record 조회
→ 같은 requestId / triggerId로 AI Orchestrator 호출
→ AWAITING_CONFIRMATION 응답 수신
→ pending confirmation projection과 SID index 저장
→ absence record 삭제
→ WebSocket event 전송
```

Projection 저장과 absence 삭제 사이의 crash window까지 제거하려면 같은 Redis script에서 원자적으로 처리한다. AI Orchestrator timeout은 요청이 처리되지 않았다는 확정 결과가 아니므로 retry에는 동일한 `triggerId`를 사용한다. 이 기능을 명시적으로 best-effort로 운영해 Trigger 유실을 수용하는 경우에만 선삭제를 허용한다.

## Deferred

- Messenger Client confirmation을 AI Orchestrator의 `POST /api/v1/returned-messages/{requestId}/confirmations`로 연결하는 adapter 구현
- confirmation 완료 후 pending projection 상태 전이와 제거 시점의 세부 구현
- user-scoped pending action 복구 또는 새 `sid`로의 action rebinding
- 기능 공통 pending action Redis abstraction과 Lua script의 구체적 구현 위치

## Related

- [Messenger Client Ingress Result Delivery](result-delivery.md)
- [Returned Message Topics](../../use-cases/02-returned-message-topics.md)
- [WebSocket Service Result Routing by Authenticated Client Session](../../decisions/005-websocket-realtime-client-session-result-routing.md)
- [Durable Asynchronous AI Execution and Result Reconciliation](../../decisions/008-durable-asynchronous-ai-execution-and-result-reconciliation.md)
