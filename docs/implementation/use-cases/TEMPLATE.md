# NN. Use Case Name

> 이 파일을 새 Use Case 디렉터리의 `README.md`로 복사하여 사용한다.

Status: Planned | In Progress | Implemented

---

## User Outcome

사용자가 어떤 상황에서 어떤 결과를 받는지 한 문단으로 설명한다.

## Trigger

- 시작 이벤트 또는 사용자 요청
- 실행에 필요한 선행 상태
- 실행하지 않는 조건

## End-to-End Flow

```text
Trigger
→ Business Logic
→ AI Orchestrator
→ Omni AI Server
→ Result Delivery
→ Client
```

## Responsibility

| Component | Responsibility |
| --- | --- |
| Business Service | |
| AI Orchestrator | |
| Omni AI Server | |
| Realtime Service | |
| Client | |

## Contracts

Event, request, `AiTask`, structured result와 필요한 식별자를 기록한다.

```text
triggerId
taskId
executionId
```

## State and Idempotency

- 실행 상태 전이
- 중복 이벤트와 중복 결과 처리 기준
- 취소와 오래된 결과 판정 기준

## Failure and Deferred Execution

- 외부 호출 실패
- timeout과 retry
- Client offline/disconnect
- 실행 보류와 재개 조건

## Scope

- 이번 단계에서 구현하는 항목

## Non-scope

- 명시적으로 다음 단계로 미루는 항목

## Acceptance Criteria

- 입력부터 사용자 결과까지 검증 가능한 완료 조건
- 실패, 중복, 취소 경로의 완료 조건

## Current Implementation Gap

현재 코드에서 동작하는 부분과 목표 사이의 차이를 기록한다. 구현이 변경될 때 함께 갱신한다.

## Open Questions

- 아직 결정되지 않은 설계 또는 정책

## Related Documents

- Roadmap
- 서비스별 Implementation 문서
- 관련 Architecture / ADR

