# Omni AI Server Result Event

Status: Designed

---

## Purpose

Omni AI Server가 Workflow / Agent 실행 결과를 channel-independent `ResultEvent`로 생성하는 구현 책임을 정의한다.

공통 이벤트 계약과 전체 전달 흐름은 [ADR 001](../../../decisions/001-session-registry-result-routing.md)을 따른다.

---

## Input Correlation

Omni AI Server는 AI Orchestrator가 전달한 `executionId`를 모든 Stream / Result Event에 포함한다.

```text
AiTask
  executionId
  workflowType
  input
```

Omni AI Server는 `deviceId`, `connectionId`, `ownerInstanceId`를 해석하거나 Realtime target을 선택하지 않는다. Routing context의 source of truth는 AI Orchestrator가 소유한다.

---

## Event Production

공통 `ResultEvent` 계약 안에서 다음 필드를 생성한다.

```text
ResultEvent
  executionId
  eventId
  eventType
  sequence (stream인 경우)
  payload
```

- `executionId`: 어떤 AI 실행에서 생성된 Event인지 식별
- `eventId`: 중복 Event 제거를 위한 고유 ID
- `eventType`: progress, stream, completed, failed 등 Event 의미
- `sequence`: 동일 execution의 stream ordering 검증에 사용
- `payload`: Workflow별 structured result 또는 stream fragment

Omni AI Server가 `routingRef`를 transport 과정에서 echo할 수는 있지만, Result Router는 이를 authoritative routing 정보로 사용하지 않고 `executionId`로 trusted `routingRef`를 다시 조회한다.

---

## Event Lifecycle

하나의 execution은 0개 이상의 progress / stream event와 하나의 terminal event를 생성할 수 있다.

```text
STARTED
→ PROGRESS / STREAM 반복
→ COMPLETED | FAILED | CANCELLED
```

동일 execution의 terminal event를 여러 번 생성하지 않도록 상태 전이를 보호한다. Retry로 Event가 중복될 수 있으므로 동일 논리 Event는 안정적인 `eventId`를 재사용하거나 Consumer가 dedup할 수 있는 idempotency 정보를 제공한다.

Stream ordering이 필요한 Workflow는 execution 단위로 증가하는 `sequence`를 부여한다. Event 재전송이 가능하다면 같은 Event의 `eventId`와 sequence를 유지한다.

---

## Failure Contract

- Workflow 실패와 timeout은 terminal `FAILED` Event로 정규화한다.
- Result 발행 실패는 제한된 retry와 idempotent publish 정책을 적용한다.
- Routing target 부재나 Client disconnect는 Omni AI Server가 처리하지 않는다.
- Delivery 성공 여부는 AI 결과 생성 성공 여부와 분리한다.

---

## Non-responsibilities

- Realtime Connection Registry 조회
- `routingRef` 유효성 검증
- 현재 `ownerInstanceId` 선택
- Core NATS instance subject 선택
- Client connection으로 최종 전송

---

## Related

- [ADR 001. Session Registry Based AI Result Routing](../../../decisions/001-session-registry-result-routing.md)
- [AI Orchestrator Result Router](../../ai-orchestrator/result-router/README.md)
- [Realtime Service Result Delivery](../../realtime-service/result-delivery/README.md)
