# 007. Common AI Task Contract

Status: Proposed

## Context

AI Orchestrator가 여러 AI 기능을 동일한 방식으로 Omni AI Server에 전달하려면 기능과 무관하게 해석할 수 있는 최소 공통 계약이 필요하다.

기능별 입력을 공통 계약에 모두 포함하면 새 기능이 추가될 때마다 공통 schema가 변경된다. 반대로 공통 필드 없이 기능별 요청만 사용하면 실행 식별, tenant 격리와 계약 버전 관리 방식이 기능마다 달라질 수 있다.

초기에는 앞으로 추가될 Task와 각 Task의 중복 처리 기준을 모두 예측하기 어렵다. 따라서 공통 계약은 실행 전달에 반드시 필요한 필드만 정의하고, Task별 입력과 실행 정책은 별도 계약과 후속 결정으로 확장한다.

## Decision

### Request Contract

AI Orchestrator는 Omni AI Server에 다음 공통 `AiTask` 계약으로 실행을 전달한다.

```json
{
  "executionId": "execution-1",
  "tenantId": "tenant-1",
  "taskType": "RETURNED_MESSAGE_TOPICS",
  "contractVersion": 1,
  "input": {}
}
```

공통 필드의 의미는 다음과 같다.

| Field | Meaning |
| --- | --- |
| `executionId` | 하나의 AI 실행을 식별한다. AI Orchestrator가 생성하고 상태와 결과 correlation에 사용한다. |
| `tenantId` | Tenant 간 권한, 실행과 데이터 격리 기준이다. |
| `taskType` | Omni AI Server가 실행할 Workflow를 선택하는 값이다. |
| `contractVersion` | 같은 `taskType`의 입출력 계약 버전을 구분하는 정수다. |
| `input` | `taskType`별 실행 입력이다. 각 Task가 자신의 schema를 정의한다. |

`taskType + contractVersion` 조합이 Task의 `input`과 `result` schema를 결정한다. 기존 소비자가 처리할 수 없는 입출력 변경에서는 `contractVersion`을 증가시킨다. Omni AI Server는 지원하지 않는 조합이나 해당 schema에 맞지 않는 `input`을 실행하지 않는다.

공통 JSON wire contract의 필드명은 Architecture 문서의 식별자 명명 규칙에 따라 `lowerCamelCase`를 사용한다. 현재 API나 구현이 `snake_case`를 사용하면 Service adapter에서 mapping한다.

`triggerId`, `taskId`, `routingRef`와 기능별 scope reference는 AI Orchestrator가 실행 상태와 결과 전달을 위해 관리할 수 있지만, 공통 AiTask의 필수 필드로 전송하지 않는다. Omni AI Server가 실제 실행에 필요한 값만 각 `input` schema에 명시한다.

크기가 제한되지 않는 대화 원문, 메시지 본문과 파일 내용은 공통 필드로 추가하지 않는다. 각 Task는 필요한 범위의 입력을 `input`에 직접 담거나 별도의 context reference를 사용하는 방식을 자신의 계약에서 결정한다.

### Result Contract

Task 실행이 성공하면 다음 공통 envelope의 `result`에 Task별 결과를 담는다.

```json
{
  "executionId": "execution-1",
  "taskType": "RETURNED_MESSAGE_TOPICS",
  "contractVersion": 1,
  "result": {}
}
```

Task 실행이 실패하면 같은 공통 필드와 `error`를 반환한다.

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

| Error field | Meaning |
| --- | --- |
| `code` | 호출자가 분기 처리할 수 있는 안정적인 오류 코드다. |
| `message` | 운영 확인과 제한적인 사용자 표시를 위한 오류 설명이다. |
| `retryable` | 동일 작업을 다시 시도할 수 있는 오류인지 나타낸다. 실제 retry 정책은 별도 결정으로 관리한다. |

성공 응답은 `result`, 실패 응답은 `error`만 포함한다. `result`와 `error`를 동시에 포함하지 않는다. 비동기 API가 Task를 접수했다는 `accepted` 응답은 실행 성공 결과가 아니며 별도의 접수 응답으로 취급한다.

## Deferred Decisions

다음 항목은 Task와 실행 요구사항이 구체화된 후 별도 결정으로 정의한다.

- dispatch retry와 중복 실행 방지 기준
- Task별 idempotency key
- 동일 요청 비교와 payload fingerprint 생성 방식
- 같은 실행 ID로 다른 요청이 들어왔을 때의 응답 정책
- `input` 크기 제한과 대용량 Context 전달 방식

## Alternatives

### Task별로 서로 다른 요청 구조 사용

각 기능의 초기 구현은 단순하지만 공통 실행 추적, tenant 격리와 버전 관리 방식이 기능별로 달라진다.

### 모든 실행 metadata를 공통 계약에 포함

`triggerId`, `taskId`, `routingRef`, `contextRef` 등을 한 번에 전달하면 확장 정보가 풍부하지만 초기 계약이 커지고 Omni AI Server가 Orchestrator의 내부 실행·전달 모델에 결합된다.

## Consequences

좋아지는 점:

- 모든 AI 기능이 동일한 최소 실행 계약을 사용한다.
- 기능 추가 시 공통 envelope을 변경하지 않고 `input` schema를 추가할 수 있다.
- Omni AI Server가 Orchestrator의 routing과 Trigger 내부 모델에 불필요하게 결합되지 않는다.
- 아직 요구사항이 없는 retry와 중복 방지 정책을 성급하게 공통화하지 않는다.

감수할 점:

- `taskType + contractVersion`별 `input`과 `result` schema 및 지원 상태를 관리해야 한다.
- 현재 API나 구현이 snake_case를 사용하면 lowerCamelCase wire field와의 mapping을 adapter에서 유지해야 한다.
- retry, 멱등성과 대용량 Context 처리 방식은 후속 결정이 필요하다.

## Related

- [Common Contracts](../contracts/README.md)
- [AiTask and Queue](../concepts/ai-task-and-queue.md)
- [AI Execution and Context Identifier Separation](004-ai-execution-and-context-identifiers.md)
- [AI Orchestrator as an Application Service](003-ai-orchestrator-application-service.md)
- [Omni AI Server Task Execution](../components/omni-ai-server/task-execution.md)
