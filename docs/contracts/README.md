# Common Contracts

> Role: 여러 Architecture, Concept, Use Case와 Component 문서가 공유하는 메시지·식별자 계약의 기준 이름과 확정 상태를 관리한다.

---

## Contract Authority

계약 해석 우선순위는 다음과 같다.

1. `Accepted` ADR
2. 이 문서에 정리된 공통 계약
3. Component의 현재 구현 mapping
4. Use Case별 확장 필드
5. `Proposed` ADR과 Concept의 Candidate field

하위 문서는 공통 필드의 이름이나 의미를 독립적으로 변경하지 않고 이 문서와 연결된 ADR을 참조한다. 이해를 위한 schema 반복은 허용하지만 차이가 있으면 계약 확정 상태와 mapping을 명시한다. 현재 API가 다른 필드명을 사용하면 Component 문서에 adapter mapping을 기록한다.

---

## Contract Decision Status

| Contract | Decision Status | Authority |
| --- | --- | --- |
| `routingRef` | Accepted | [ADR 001](../decisions/001-session-registry-result-routing.md) |
| `ResultEvent` | Accepted | [ADR 001](../decisions/001-session-registry-result-routing.md) |
| `DeliveryEvent` | Accepted | [ADR 001](../decisions/001-session-registry-result-routing.md) |
| WebSocket `CLIENT_SESSION_CURRENT` | Accepted | [ADR 005](../decisions/005-websocket-realtime-client-session-result-routing.md) |
| 전체 AI identifier 분리 | Proposed | [ADR 004](../decisions/004-ai-execution-and-context-identifiers.md) |
| 공통 `AiTask` schema | Proposed | [ADR 007](../decisions/007-ai-task-contract.md) |

`Proposed`와 `Draft` 계약은 구현 후보이며, Accepted 계약과 충돌하면 Accepted 계약을 우선한다.

- `Accepted`: 공식 설계 결정으로 확정된 계약
- `Proposed`: 최종 승인 전 검토 중인 설계안
- `Draft`: 구현 과정에서 변경될 수 있는 초기 초안

---

## AI Task

AI Orchestrator가 Omni AI Server에 실행을 전달하는 최소 공통 wire contract는 다음과 같다.

```json
{
  "executionId": "execution-1",
  "tenantId": "tenant-1",
  "taskType": "RETURNED_MESSAGE_TOPICS",
  "contractVersion": 1,
  "input": {}
}
```

`taskType + contractVersion` 조합이 Task별 `input`과 `result` schema를 결정한다. 기존 소비자가 처리할 수 없는 입출력 변경에서는 `contractVersion`을 증가시킨다. 구체적인 필드와 검증 규칙은 Task별 계약에서 관리한다.

공통 wire field는 Architecture 문서의 식별자 명명 규칙에 따라 `lowerCamelCase`를 사용한다. 현재 API나 구현이 다른 이름을 사용하면 Component 문서와 Service adapter에 mapping을 기록한다.

`executionId`는 AI Orchestrator가 생성하고 하나의 AI 실행과 그 결과를 연결한다. 공통 필드의 의미와 확장 범위는 [ADR 007](../decisions/007-ai-task-contract.md)을 따른다. Retry, 멱등성, 동일 요청 비교와 `input` 크기 제한은 아직 공통 계약으로 확정하지 않았다.

---

## AI Task Result

성공한 Task는 Task별 결과를 `result`에 담는다.

```json
{
  "executionId": "execution-1",
  "taskType": "RETURNED_MESSAGE_TOPICS",
  "contractVersion": 1,
  "result": {}
}
```

실패한 Task는 공통 오류를 `error`에 담는다.

```json
{
  "executionId": "execution-1",
  "taskType": "RETURNED_MESSAGE_TOPICS",
  "contractVersion": 1,
  "error": {
    "code": "AI_GENERATION_FAILED",
    "message": "Generation failed",
    "retryable": true
  }
}
```

성공 응답은 `result`, 실패 응답은 `error`만 포함한다. `accepted`는 Task 실행 성공이 아닌 비동기 접수 응답이다. Task별 결과 schema와 오류 code는 각 Task 계약에서 정의한다.

---

## Routing Reference

```text
routingRef
  tenantId
  userId
  targetRef
    type: CONNECTION | DEVICE_CURRENT | CLIENT_SESSION_CURRENT | SCOPE
    connectionId (필요 시)
    deviceId (필요 시)
    clientSessionId (필요 시)
  connectionType (필요 시)
  scopeRef (필요 시)
    type
    id
```

`routingRef`는 instance 위치가 아니라 논리 target을 표현한다. 최종 `connectionId`와 `ownerInstanceId`는 Result 전달 시점에 Realtime Connection Registry에서 다시 조회한다.

---

## Result Event

```text
ResultEvent
  executionId
  eventId
  eventType
  sequence (stream인 경우)
  routingRef (선택, non-authoritative snapshot)
  payload
```

Result Router는 Event에 포함된 `routingRef`를 authoritative 값으로 사용하지 않는다. `executionId`로 AI Orchestrator가 저장한 trusted `routingRef`를 복원한다.

Workflow별 `conversationId`, token `content`와 표시용 `metadata`는 필요한 경우 `payload` 안에 둔다.

---

## Delivery Event

```text
DeliveryEvent
  executionId
  eventId
  eventType
  connectionId
  connectionType (필요 시)
  userId
  scopeRef (필요 시)
  payload
```

`DeliveryEvent`는 Result Router가 현재 owner를 resolve한 뒤 선택된 Messenger Client Ingress instance에 보내는 계약이다.

---

## Identifier Status

`executionId`는 현재 구현에서도 AI 실행 식별자로 사용한다. `triggerId`, `taskId`, `conversationId`를 포함한 전체 lifecycle 분리는 ADR 004가 `Proposed` 상태이므로 아직 최종 공통 계약이 아니다.

현재 또는 legacy 필드명의 mapping은 [Current Service Boundary](../concepts/current-service-boundary.md)의 Identifier Naming을 따른다.
