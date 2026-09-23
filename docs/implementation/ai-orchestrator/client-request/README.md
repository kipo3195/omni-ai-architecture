# Client Request

Status: Planned

Client의 명시적 AI 요청을 처리하는 AI Orchestrator의 공통 구조를 기록한다.

요청 수신, 실행 판단, Context 구성, AiTask 생성과 결과 계약을 실제 구현 기준으로 정리한다. 채널별 연결과 전달 책임은 각 Realtime Service의 경계로 둔다.

## First Applications

### Phase 3. Client-generated ScheduleSpec

Client LLM이 생성한 ScheduleSpec을 수신한다. Client 출력은 신뢰하지 않고 authentication, schema, 지원 task type, permission, timezone과 idempotency를 검증한 뒤 Schedule Management로 전달한다.

### Phase 4. Natural-language Schedule Request

사용자의 자연어 요청을 수신해 Omni AI Server의 Schedule Intent Parsing Workflow로 전달한다. 반환된 ScheduleSpec 후보를 다시 검증하고 사용자 확인 후 등록한다.

두 경로는 동일한 ScheduleSpec과 Schedule Management를 사용한다. 차이는 ScheduleSpec 후보를 Client LLM과 Server LLM 중 누가 생성하는가뿐이다.

## Boundary

- Client Request는 channel connection을 소유하지 않는다.
- LLM 결과가 Schedule 또는 다른 product state를 직접 변경하지 않는다.
- 데이터 변경은 application validation과 필요한 사용자 확인을 거친다.

## Related

- [Scheduled Weekly Report](../../use-cases/03-scheduled-weekly-report/README.md)
- [Server Schedule Intent](../../use-cases/04-server-schedule-intent/README.md)
- [Schedule Management](../schedule-management/README.md)
