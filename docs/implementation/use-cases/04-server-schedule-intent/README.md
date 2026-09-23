# 04. Server Schedule Intent

Status: Planned

---

## User Outcome

Client LLM이 없는 환경에서도 사용자가 자연어로 반복 요약 일정을 요청하고, 해석된 내용을 확인한 뒤 Schedule을 등록할 수 있다.

## Trigger

- Client가 자연어 Schedule 요청을 AI Orchestrator로 전송한다.

## End-to-End Flow

```text
자연어 요청
→ AI Orchestrator
→ Omni AI Server: Schedule Intent Parsing
→ ScheduleSpec 후보
→ AI Orchestrator: 검증
→ Client 확인
→ Schedule 등록
```

## Responsibility

| Component | Responsibility |
| --- | --- |
| Client | 자연어 요청과 확인 결과 전달 |
| AI Orchestrator | 인증, AI 호출, 결과 검증, 확인, Schedule 등록 |
| Omni AI Server | 자연어를 Phase 3의 ScheduleSpec으로 변환 |
| Scheduler | 등록된 Schedule 실행 |

## Contracts

Phase 3의 ScheduleSpec을 변경 없이 사용한다. 해석할 수 없는 필드와 추가 확인이 필요한 필드를 표현하는 오류 계약을 별도로 둔다.

## State and Idempotency

Intent parsing과 Schedule 등록을 분리한다. 동일 parse 결과가 여러 번 제출되어도 하나의 Schedule만 등록되도록 request idempotency를 적용한다.

## Failure and Deferred Execution

불명확한 요청은 추측해서 등록하지 않고 확인이 필요한 항목을 반환한다. LLM 실패는 Schedule 등록 실패와 구분한다.

## Scope

- Server LLM 기반 Schedule Intent Parsing
- ScheduleSpec 검증과 사용자 확인
- Phase 3 실행 구조 재사용

## Non-scope

- Omni AI Server의 Schedule 저장 또는 Scheduler 직접 제어
- 범용 대화형 Agent
- Runtime Tool Calling

## Acceptance Criteria

- Client LLM 없이 Phase 3과 동일한 ScheduleSpec을 생성한다.
- LLM 출력만으로 Schedule이 직접 등록되지 않는다.
- Phase 3의 Scheduler와 deferred execution을 변경 없이 재사용한다.

## Current Implementation Gap

Schedule Intent Workflow와 자연어 요청 ingress가 아직 구현되지 않았다.

## Open Questions

- 불명확한 날짜·timezone을 Client 확인으로 해소하는 응답 형식
- Schedule 수정 요청과 신규 등록 요청을 구분하는 기준

## Related Documents

- [Roadmap](../../../roadmap.md)
- [Client Request](../../ai-orchestrator/client-request/README.md)
- [Schedule Management](../../ai-orchestrator/schedule-management/README.md)
- [Schedule Intent Parsing Workflow](../../omni-ai-server/workflows/schedule-intent-parsing.md)

