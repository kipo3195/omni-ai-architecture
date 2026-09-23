# Schedule Management

Status: Planned

사용자가 등록한 AI Schedule과 각 실행 회차의 lifecycle을 관리한다.

## Responsibility

- ScheduleSpec schema, permission과 지원 task type 검증
- Schedule 등록, 변경, 비활성화와 삭제
- 사용자 timezone 기준 다음 실행 시각 계산
- Scheduler 등록과 Trigger 수신
- 실행 회차 생성과 idempotency
- 온라인 상태 확인
- 오프라인 회차 보류
- 온라인 복귀 후 사용자 확인
- 실행, skip, failure 결과 기록

## Boundary

Client LLM과 Omni AI Server는 ScheduleSpec 후보를 생성할 수 있지만 Schedule을 직접 저장하거나 Scheduler를 변경하지 않는다. 최종 validation과 product state 변경은 AI Orchestrator가 담당한다.

## Schedule and Occurrence

반복 규칙인 Schedule과 특정 시각에 발생한 실행 회차를 분리한다.

```text
Schedule
→ REGISTERED / PAUSED / CANCELLED

Occurrence
→ TRIGGERED
→ DEFERRED_OFFLINE
→ AWAITING_USER_CONFIRMATION
→ PROCESSING
→ COMPLETED / SKIPPED / FAILED
```

## Related

- [Scheduled Weekly Report](../../use-cases/03-scheduled-weekly-report/README.md)
- [Server Schedule Intent](../../use-cases/04-server-schedule-intent/README.md)
