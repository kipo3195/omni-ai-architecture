# 01. Conversation Start

Status: In Progress

---

## User Outcome

사용자가 대화방에 들어오면 현재 방에 적합한 대화 시작 추천을 받고, 추천이 더 이상 유효하지 않은 방에서는 오래된 결과를 받지 않는다.

## Trigger

- `ROOM_ENTERED`: 실행 후보 생성
- `ROOM_LEFT`: 예약 실행 취소 또는 이후 결과 전달 차단

## End-to-End Flow

```text
ROOM_ENTERED / ROOM_LEFT
→ Messenger Business Logic
→ AI Orchestrator: 실행 판단과 correlation
→ Omni AI Server: TRENDING Workflow
→ Realtime Service: 현재 room session 확인
→ Client Suggestion
```

## Responsibility

| Component | Responsibility |
| --- | --- |
| Messenger Business Logic | 방 입장·퇴장 사실과 session 식별자 제공 |
| AI Orchestrator | 실행 예약, 취소, Context 구성, AI 호출, 유효한 결과 판정 |
| Omni AI Server | Conversation Start 데이터 생성 |
| Realtime Service | 현재 session 조회와 Client 전달 |
| Client | 추천 표시 |

## Contracts

최소 correlation 후보는 `userId`, `roomKey`, `roomSessionId`, `triggerId`, `executionId`다. 구체 schema는 구현 과정에서 확정한다.

## State and Idempotency

같은 사용자와 방에 새 Enter 요청이 들어오면 이전 활성 실행을 취소한다. 결과 전달 전 현재 `roomSessionId`와 활성 `executionId`를 다시 확인한다.

## Failure and Deferred Execution

Omni AI Server 실패나 timeout은 실행 실패로 기록하고 추천 없이 종료한다. 이 Phase에서는 Client 재접속 후 결과 재전송을 보장하지 않는다.

## Scope

- Enter/Leave 기반 실행과 취소
- TRENDING Conversation Start 생성
- 현재 Client session으로 결과 전달

## Non-scope

- NATS 기반 cross-domain trigger
- 장기 보관과 재처리
- Tool Calling

## Acceptance Criteria

- Enter Room부터 Client Suggestion까지 E2E 경로가 동작한다.
- Leave Room 이후 예약 실행 또는 결과 전달이 중단된다.
- 재입장 후 이전 execution 결과가 전달되지 않는다.

## Current Implementation Gap

현재 Orchestrator는 REST 요청, 메모리 Repository와 프로세스 내부 Scheduler를 사용한다. 최근 메시지 조회, 공통 AiTask, Realtime Service 결과 전달은 아직 연결되지 않았다.

## Open Questions

- Phase 1에서 `triggerId`, `taskId`를 모두 도입할지
- Session Registry와 Result Routing을 어느 수준까지 실제 연동할지

## Related Documents

- [Roadmap](../../../roadmap.md)
- [Server Trigger](../../ai-orchestrator/server-trigger/README.md)
- [Task Execution](../../omni-ai-server/task-execution/README.md)
- [Conversation Start Workflow](../../omni-ai-server/workflows/conversation-start.md)

