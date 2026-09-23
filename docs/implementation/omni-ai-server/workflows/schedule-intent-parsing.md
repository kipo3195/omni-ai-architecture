# Schedule Intent Parsing Workflow

Status: Planned

사용자의 자연어 Schedule 요청을 Phase 3에서 정의한 ScheduleSpec 후보로 변환한다.

## Initial Contract

- 입력: 자연어 요청, 사용자 timezone, 지원하는 task type 목록
- 출력: ScheduleSpec 후보 또는 추가 확인이 필요한 필드

## Boundary

출력은 실행 명령이 아니라 검증 대상 구조화 데이터다. permission, policy, 사용자 확인, 저장과 Scheduler 등록은 AI Orchestrator가 담당한다.

## Related

- [Server Schedule Intent Use Case](../../use-cases/04-server-schedule-intent/README.md)

