# NN. Use Case Name

> 이 파일을 `NN-short-use-case-name.md` 형식의 새 파일로 복사하여 사용한다.

진행 상태는 [Roadmap](../roadmap.md)에서 관리한다.

## User Outcome

사용자가 어떤 상황에서 어떤 결과를 받는지 한 문단으로 설명한다.

## Trigger

- Source owner가 소유한 상태와 Business Event 성립 조건
- Source owner가 확정해야 하는 식별자, 시각, trusted context
- AI Orchestrator가 확정된 Event에 적용할 기능별 실행 / skip 조건
- Source Event 확정 조건과 AI 기능 정책을 같은 정책으로 표현하지 않음

## End-to-End Flow

```text
Raw State Change / Client Request
→ 현재 Business owner 또는 Messenger Client Ingress: Business Event / trusted context 확정
→ AI Orchestrator: source event contract 검증과 AI 기능 정책
→ Omni AI Server
→ Result Delivery
→ Client
```

## Responsibility

| Component | Responsibility |
| --- | --- |
| 현재 Business owner / Messenger Client Ingress | 소유한 상태 전이, Business Event, trusted context 확정 |
| AI Orchestrator | Source Event를 재판정하지 않고 contract 검증, AI 기능 정책, Context Assembly, AiTask lifecycle 관리 |
| Omni AI Server | |
| Messenger Client Ingress | `WebSocket Service (WS)` 또는 `TCP Service (NS)`의 ingress / delivery 책임 |
| Client | |

## Contracts

Event, request, `AiTask`, structured result와 필요한 식별자를 기록한다. Event contract에는 source owner가 확정한 사실과 Orchestrator가 적용할 기능 정책 입력을 구분한다.

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

