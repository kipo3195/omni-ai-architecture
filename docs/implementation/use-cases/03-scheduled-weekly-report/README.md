# 03. Scheduled Weekly Report

Status: Planned

---

## User Outcome

사용자가 Client에서 자연어로 반복 요약 일정을 등록하고, 매주 지정한 시각에 주간보고 관련 쪽지를 요약한 별도 쪽지를 받는다. 실행 시 오프라인이면 복귀 후 실행 여부를 선택한다.

## Trigger

- 등록: Client LLM이 자연어를 `ScheduleSpec`으로 변환해 전송한다.
- 실행: 저장된 Schedule의 회차가 지정한 사용자 timezone에 도달한다.
- 재개: 보류된 회차가 있는 사용자가 온라인으로 복귀한다.

## End-to-End Flow

```text
Client LLM → ScheduleSpec → Orchestrator 검증·등록
→ Scheduler Trigger
→ Presence 확인
├─ ONLINE → 메시지 조회 → Omni AI 요약 → 쪽지 전달
└─ OFFLINE → 회차 보류 → 복귀 시 확인 → 실행 또는 skip
```

## Responsibility

| Component | Responsibility |
| --- | --- |
| Client LLM | 자연어에서 ScheduleSpec 초안 생성 |
| AI Orchestrator | schema·permission·timezone 검증, Schedule과 회차 관리 |
| Scheduler | 지정 시각에 실행 회차 Trigger |
| Message owner | 기간 내 주간보고 대상 쪽지 제공 |
| Omni AI Server | 주간보고 분류와 요약 생성 |
| Realtime Service / Client | 결과 쪽지 전달, 보류 회차 확인 UI |

## Contracts

Phase 3에서 `ScheduleSpec`을 확정하고 Phase 4에서도 그대로 재사용한다. 최소 항목은 `taskType`, 반복 주기, 요일, 시각, timezone과 delivery 유형이다.

## State and Idempotency

Schedule과 실행 회차 상태를 분리한다. 회차는 `TRIGGERED`, `DEFERRED_OFFLINE`, `AWAITING_USER_CONFIRMATION`, `PROCESSING`, `COMPLETED`, `SKIPPED`, `FAILED` 상태를 가진다.

## Failure and Deferred Execution

오프라인에서는 AI를 호출하지 않는다. 복귀 후 사용자 확인을 받아 실행하거나 skip한다. 같은 회차의 중복 Scheduler Trigger는 idempotency key로 차단한다.

## Scope

1. `3-A`: ScheduleSpec 검증과 등록
2. `3-B`: 온라인 회차의 요약과 쪽지 전달
3. `3-C`: 오프라인 회차 보류와 복귀 후 확인

## Non-scope

- Server LLM의 Schedule intent parsing
- LLM이 실행 중 선택하는 Tool Calling
- 임의 종류의 자동화 플랫폼

## Acceptance Criteria

- Client LLM의 출력이 서버 검증 없이 저장되지 않는다.
- 사용자 timezone 기준으로 회차가 한 번만 실행된다.
- 오프라인 회차는 AI 호출 없이 보류된다.
- 복귀 후 사용자의 선택에 따라 실행 또는 skip된다.

## Current Implementation Gap

ScheduleSpec, Schedule 저장소, Scheduler 연동, 주간보고 메시지 선정과 offline deferred execution이 아직 구현되지 않았다.

## Open Questions

- 주간보고 메시지를 Label, 제목, 발신자 또는 LLM 분류 중 무엇으로 선정할지
- 보류 회차의 만료 기간과 여러 회차가 누적될 때의 처리 방식

## Related Documents

- [Roadmap](../../../roadmap.md)
- [Client Request](../../ai-orchestrator/client-request/README.md)
- [Schedule Management](../../ai-orchestrator/schedule-management/README.md)
- [Weekly Report Summary Workflow](../../omni-ai-server/workflows/weekly-report-summary.md)

