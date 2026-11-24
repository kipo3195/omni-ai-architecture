# Messenger Client AI Feature Event Contract

Status: Proposed

---

## Purpose

Messenger Client가 AI 기능별 WebSocket 메시지를 서로 다른 최상위 구조와 식별자 규칙으로 해석하지 않도록 공통 Client event envelope를 정의한다.

이 계약은 `AI Orchestrator → Messenger Client Ingress`의 서버 간 `ResultEvent` / `DeliveryEvent`를 대체하지 않는다. Messenger Client Ingress가 인증된 local connection으로 최종 전달하기 직전에 서버 이벤트를 Client protocol로 변환하는 계약이다.

```text
Omni AI Server execution result
→ AI Orchestrator domain lifecycle / ResultEvent
→ Result Router / DeliveryEvent
→ Messenger Client Ingress
→ Client AI Feature Event 변환
→ WebSocket 또는 TCP Client
```

## Boundary

- AI Orchestrator는 기능 lifecycle, 내부 상태와 revision의 authoritative owner다.
- Event producer는 `eventId`와 `occurredAt`을 생성하고 재전송 시 같은 값을 유지한다.
- Messenger Client Ingress는 서버 내부 상태를 Client 공개 `type`, `state`, `payload`로 변환한다.
- Omni AI Server는 Client event type, Client state와 Client schema version을 알지 않는다.
- Client는 `executionId`, `triggerId`, `messageSelectionRef`, `scopeRef` 같은 서버 correlation을 화면 상태의 주 식별자로 사용하지 않는다.

## Common Envelope

Messenger Client protocol의 field name은 `snake_case`를 사용한다.

```json
{
  "type": "returned_message_topic",
  "data": {
    "schema_version": 1,
    "lifecycle_id": "01M85Y481XN1G5RH59YFWENM4R",
    "event_id": "f028716d-6027-493d-91ae-92f15cc26e7b",
    "revision": 1,
    "state": "AWAITING_CONFIRMATION",
    "occurred_at": "2026-11-24T06:08:49.955831Z",
    "payload": {}
  }
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `type` | Y | Client가 기능을 구분하는 stable lower snake case key |
| `data.schema_version` | Y | 해당 `type`의 state와 payload를 포함한 Client wire contract version |
| `data.lifecycle_id` | Y | 하나의 기능 lifecycle을 처음부터 terminal 상태까지 식별하는 opaque ID |
| `data.event_id` | Y | 하나의 Client event를 식별하는 globally unique ID |
| `data.revision` | Y | 같은 `type + lifecycle_id` 안에서 단조 증가하는 상태 event 순서 |
| `data.state` | Y | 해당 `type`에서 정의한 Client 공개 상태 |
| `data.occurred_at` | Y | event가 최초 생성된 서버 시각의 UTC RFC 3339 timestamp |
| `data.payload` | Y | `type + schema_version + state`가 결정하는 object. 전달할 값이 없어도 `{}` 사용 |

`schema_version`은 서버 간 Task contract version과 별개의 Client contract version이다. 기존 Client가 처리할 수 없는 필드 제거, 타입 변경, state 의미 변경 또는 payload 구조 변경은 version을 증가시킨다. 선택 필드 추가처럼 기존 해석을 깨지 않는 변경은 같은 version에서 허용할 수 있다.

## Identity and Ordering

Client는 다음 key로 기능 상태를 관리한다.

```text
(type, lifecycle_id)
```

Client는 `event_id`로 중복 event를 제거하고, 같은 lifecycle에서 현재 저장한 revision 이하의 event를 무시한다.

```text
AWAITING_CONFIRMATION  event_id=A  revision=1
PROCESSING             event_id=B  revision=2
TOPICS_READY           event_id=C  revision=3
```

- 서로 다른 상태 event는 서로 다른 `event_id`를 사용한다.
- 같은 event를 retry 또는 reconnect recovery로 다시 전달할 때는 `event_id`, `revision`, `occurred_at`, `state`, `payload`를 변경하지 않는다.
- revision은 증가만 보장하며 연속성을 보장하지 않는다. Client는 revision gap만으로 실패를 판단하지 않고 필요한 경우 상태 복구 API를 호출한다.
- `occurred_at`은 Messenger Client Ingress가 직렬화한 시각이 아니라 producer가 상태 event를 생성한 시각이다.

## Lifecycle ID Mapping

`lifecycle_id`는 모든 기능에서 같은 물리 ID 종류를 강제하는 필드가 아니다. `type`별 기능 lifecycle을 나타내며, 서로 lifecycle이 다른 식별자를 합치지 않는다.

| Client type | Current source | Rule |
| --- | --- | --- |
| `returned_message_topic` | AI Orchestrator `returned_message_request.request_id` | 복귀 감지부터 confirmation과 Topic 결과 전달까지 유지 |
| `conversation_start` | AI Orchestrator `executionId` | 현재 구현은 하나의 Conversation Start와 하나의 AI execution이 1:1이므로 사용 |

Conversation Start의 `roomSessionId`는 `ROOM_SESSION` scope ID이며 `lifecycle_id`로 바꾸지 않는다. 향후 하나의 Conversation Start 기능 lifecycle이 여러 AI execution을 소유하게 되면 별도 lifecycle ID를 추가하고 mapping을 contract version과 함께 갱신한다.

## Client State Rules

`state`는 Orchestrator persistence enum을 자동 직렬화하지 않는다. 같은 문자열을 사용하더라도 Client 공개 enum을 별도로 관리하고 adapter에서 명시적으로 매핑한다.

`type`이 이미 기능 namespace이므로 state에 기능명을 반복하지 않는다.

```text
권장:   type=conversation_start, state=PROCESSING
비권장: type=conversation_start, state=CONVERSATION_PROCESSING
```

### Returned Message Topic

| Client state | Source lifecycle | Payload |
| --- | --- | --- |
| `AWAITING_CONFIRMATION` | `AWAITING_CONFIRMATION` | confirmation ID, unread count, expiration |
| `PROCESSING` | `PROCESSING` | `{}` |
| `TOPICS_READY` | Topic result가 저장된 `DELIVERY_PENDING` | topics, message assessments, generated time |
| `DECLINED` | `DECLINED` | `{}` |
| `EXPIRED` | `EXPIRED` | `{}` |
| `UNAVAILABLE` | `BLOCKED_BY_POLICY` 등 실행 불가 | 안전한 reason code |
| `FAILED` | `FAILED` 또는 `DELIVERY_FAILED` | 안전한 error object |
| `CANCELLED` | `CANCELLED` | `{}` |

`TOPICS_READY`는 AI 결과가 준비되어 Client에 전달되는 event라는 의미다. Client acknowledgement가 없는 현재 delivery에서는 `TOPICS_DELIVERED` 또는 `CLIENT_RECEIVED`를 사용하지 않는다.

Confirmation event 예시:

```json
{
  "type": "returned_message_topic",
  "data": {
    "schema_version": 1,
    "lifecycle_id": "request-1",
    "event_id": "57b7bd77-f6e5-4adc-9e3b-3f16f964ac5f",
    "revision": 1,
    "state": "AWAITING_CONFIRMATION",
    "occurred_at": "2026-11-24T06:01:00Z",
    "payload": {
      "confirmation_id": "confirmation-1",
      "unread_count": 3,
      "confirmation_expires_at": "2026-11-24T06:05:00Z"
    }
  }
}
```

Topic result event 예시:

```json
{
  "type": "returned_message_topic",
  "data": {
    "schema_version": 1,
    "lifecycle_id": "request-1",
    "event_id": "ca3ba282-716a-46ab-b023-1b757880cfe2",
    "revision": 3,
    "state": "TOPICS_READY",
    "occurred_at": "2026-11-24T06:08:49.955831Z",
    "payload": {
      "topics": [
        {
          "topic": "운영 점검",
          "summary": "운영 점검 관련 요청입니다.",
          "source_message_refs": ["message-1"]
        }
      ],
      "message_assessments": [
        {
          "message_ref": "message-1",
          "priority": "NORMAL",
          "priority_reason": "일반 업무",
          "reply_required": false,
          "reply_reason": "명시적인 회신 요청 없음",
          "action_required": true,
          "action_reason": "확인 작업 요청"
        }
      ],
      "generated_at": "2026-11-24T06:08:49.955831Z"
    }
  }
}
```

### Conversation Start

| Client state | Source execution | Payload |
| --- | --- | --- |
| `SCHEDULED` | `SCHEDULED` | 필요한 경우 room display context |
| `PROCESSING` | `EXECUTING` | `{}` |
| `SUGGESTIONS_READY` | `COMPLETED` | suggestion result |
| `CANCELLED` | `CANCELLED` | `{}` |
| `UNAVAILABLE` | `REJECTED` | 안전한 reason code |
| `FAILED` | `FAILED` | 안전한 error object |

현재 Conversation Start는 Client 재접속 후 결과 재전송과 durable revision을 보장하지 않는다. 이 기능에 recovery를 추가하기 전까지 revision은 살아 있는 execution lifecycle 안에서만 유효하며, process restart를 넘는 보장을 문서화하지 않는다.

```json
{
  "type": "conversation_start",
  "data": {
    "schema_version": 1,
    "lifecycle_id": "bb62bc0b-8a3f-4ec2-8de3-a88c00cf7f88",
    "event_id": "aa01754c-f7d9-4617-bcbb-fc0ff2fd954d",
    "revision": 2,
    "state": "PROCESSING",
    "occurred_at": "2026-11-24T06:08:40Z",
    "payload": {}
  }
}
```

## Error Payload

Client에 실패 상태를 전달하는 기능은 내부 exception, credential, 외부 응답 원문을 노출하지 않고 stable code를 사용한다.

```json
{
  "code": "AI_EXECUTION_FAILED",
  "message": "AI 결과를 생성하지 못했습니다.",
  "retryable": false
}
```

`message`는 표시용이며 Client 분기는 `code`와 `state`를 기준으로 한다.

## Returned Message Confirmation Request

Target Client API는 lifecycle을 path에서 식별하고 body에는 confirmation action과 결정을 전달한다.

```http
POST /rest/ai/returned-message-topics/{lifecycleId}/confirmations
Authorization: Bearer {accessToken}
Content-Type: application/json
```

```json
{
  "confirmation_id": "confirmation-1",
  "decision": "YES"
}
```

Messenger Client Ingress는 다음 값을 검증한다.

```text
path.lifecycleId == pending projection.requestId
body.confirmationId == pending projection.confirmationId
JWT tenantId / userId / sid == projection owner
current time < confirmationExpiresAt
projection status == AWAITING_CONFIRMATION
```

AI Orchestrator 호출에 필요한 `absencePeriodId`, `messageSelectionRef`, `routingRef`는 인증 context와 pending projection에서 복원한다. Client가 제출한 값을 신뢰하지 않는다.

같은 lifecycle, confirmation과 decision의 retry는 같은 현재 상태를 반환하는 멱등 요청으로 처리한다. 이미 반영된 confirmation에 반대 decision이 들어오면 `409 Conflict`를 반환한다. 성공 body가 필요한 경우 같은 Client event envelope를 사용해 `PROCESSING`, `DECLINED` 또는 현재 terminal state를 반환한다.

## Pending Action Recovery

Pending Actions API의 목록 wrapper는 pagination metadata를 유지하고, `items[]`는 WebSocket과 같은 Client event envelope를 반환한다.

```json
{
  "result_code": 0,
  "result_message": "success",
  "data": {
    "server_time": "2026-11-24T06:01:00Z",
    "next_cursor": null,
    "items": [
      {
        "type": "returned_message_topic",
        "data": {
          "schema_version": 1,
          "lifecycle_id": "request-1",
          "event_id": "57b7bd77-f6e5-4adc-9e3b-3f16f964ac5f",
          "revision": 1,
          "state": "AWAITING_CONFIRMATION",
          "occurred_at": "2026-11-24T06:01:00Z",
          "payload": {
            "confirmation_id": "confirmation-1",
            "unread_count": 3,
            "confirmation_expires_at": "2026-11-24T06:05:00Z"
          }
        }
      }
    ]
  }
}
```

Pending projection은 Client event의 `event_id`, `revision`, `occurred_at`을 함께 저장해야 하며 reconnect recovery에서 새 값으로 다시 만들지 않는다.

## Server Contract Mapping

Accepted `ResultEvent` / `DeliveryEvent` 계약은 유지한다.

```text
DeliveryEvent
  executionId
  eventId
  eventType
  connectionId
  userId
  scopeRef
  payload
```

`executionId`, 내부 `eventType`과 `scopeRef`는 routing, correlation과 stale result 검증에 계속 사용한다. Messenger Client Ingress의 기능별 adapter가 이를 Client envelope로 변환하며 서버 내부 필드를 그대로 노출하지 않는다.

AI Orchestrator가 Client 변환에 필요한 lifecycle metadata를 제공하는 경우 `DeliveryEvent.payload`의 기능별 projection에 다음 값을 포함할 수 있다.

```text
lifecycleId
lifecycleRevision
lifecycleStatus
stateChangedAt
result or action payload
```

Messenger Client Ingress는 내부 `lifecycleStatus`를 이 문서의 Client `state`로 명시적으로 매핑한다. Omni AI Server의 Task request / response schema는 이 Client contract 때문에 변경하지 않는다.

## Delivery and Acknowledgement

WebSocket send queue 등록, owner instance REST 2xx 또는 Core NATS publish 성공은 Client 수신 성공을 의미하지 않는다. 현재 envelope의 `event_id`는 중복 제거용이며 Client acknowledgement가 아니다.

Client 수신 보장이 필요해지면 다음을 별도 계약으로 정의한다.

```text
Client receives event_id
→ Client acknowledgement
→ delivery acknowledged state
```

Acknowledgement가 도입되기 전에는 `DELIVERED`, `CLIENT_RECEIVED`, `ACKNOWLEDGED` 같은 Client state를 사용하지 않는다.

## Compatibility and Rollout

기존 숫자 WebSocket type 또는 기능별 전용 event type을 공통 envelope로 바꾸는 것은 breaking Client change다. 연결 capability 또는 명시적인 Client protocol version으로 serializer를 선택한다.

초기 capability key는 다음과 같이 정의한다.

```text
ai_feature_event_v1
```

```text
capability present
→ 이 문서의 공통 envelope만 전송

capability absent
→ 기존 Client event만 전송
```

같은 Client connection에 legacy event와 신규 event를 동시에 보내지 않는다. `schema_version`은 신규 envelope 내부 version이며, envelope 자체를 인식하지 못하는 legacy Client를 구분하는 수단은 아니다.

## Non-responsibilities

- Client envelope가 AI Orchestrator의 domain lifecycle을 대체하지 않는다.
- `lifecycle_id`가 `executionId`, `roomSessionId`, `connectionId`를 하나의 공통 lifecycle로 합치지 않는다.
- Messenger Client Ingress가 기능별 authoritative 상태를 새로 판단하지 않는다.
- Omni AI Server가 Client protocol을 생성하거나 Client revision을 소유하지 않는다.
- Client event success가 Client 수신 acknowledgement를 의미하지 않는다.

## Related

- [Messenger Client Ingress Result Delivery](result-delivery.md)
- [Messenger Client Ingress Pending AI Actions](pending-actions.md)
- [ADR 001. Session Registry Based AI Result Routing](../../decisions/001-session-registry-result-routing.md)
- [Returned Message Topics](../../use-cases/02-returned-message-topics.md)
- [Conversation Start](../../use-cases/01-conversation-start.md)

