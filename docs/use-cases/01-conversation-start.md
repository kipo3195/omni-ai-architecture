# 01. Conversation Start

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
→ Messenger Client Ingress의 현재 connection owner: room session 확인
→ Client Suggestion
```

## Responsibility

| Component | Responsibility |
| --- | --- |
| Messenger Business Logic | 방 입장·퇴장 사실과 인증된 Client context에서 구성한 `routingRef` 제공 |
| AI Orchestrator | 실행 예약, 취소, Context 구성, AI 호출, 유효한 결과 판정 |
| Omni AI Server | Conversation Start 데이터 생성 |
| Messenger Client Ingress | `WebSocket Service (WS)` 또는 `TCP Service (NS)`에서 현재 session 조회와 Client 전달 |
| Client | 추천 표시 |

## Contracts

현재 방 입장 REST 계약은 다음과 같다.

```text
POST /api/v1/conversation-starts

roomSessionId
roomKey
chatType (선택)
routingRef
  tenantId
  userId
  targetRef
  connectionType (선택)
```

`routingRef`는 owner와 delivery target의 단일 입력 출처다. 현재 HTTP adapter는 문자열 `targetRef`를 `CLIENT_SESSION_CURRENT`의 `clientSessionId`로 변환하고, `roomSessionId`로 `ROOM_SESSION` scope를 구성한다. Conversation Start는 특정 Desktop connection type으로 전달을 제한하지 않으므로 `connectionType`을 요구하지 않는다.

현재 단계에서는 Messenger Client Ingress가 인증된 Client context로 구성한 `routingRef`를 service-to-service 요청 body에 전달한다. 향후 AI Orchestrator의 인증 계약이 확정되면 `tenantId`, `userId`는 검증된 인증 context에서 얻고 `routingRef`에는 순수 delivery target만 남긴다. Client가 제출한 임의의 owner 또는 routing identifier를 신뢰한다는 의미가 아니다.

방 퇴장 계약은 아직 `DELETE /api/v1/conversation-starts/{roomSessionId}?userID=...&roomKey=...` 형식이며, 인증 context 기반 owner 추출로 전환하는 작업은 별도 범위다. `executionId`는 AI Orchestrator가 방 입장 요청을 수락할 때 생성한다.

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

## AI Workflow

방 입장 Context를 받아 `TRENDING` 유형의 대화 시작 추천을 생성한다.

### Input

- 사용자와 방 식별자
- 현재 실행과 session correlation
- 최근 메시지 등 허용된 대화 Context
- 추천 개수, 길이, 언어 옵션

### Output

- 추천 문장 목록
- 추천 유형
- 원 요청과 연결할 execution identifier

현재 구현은 최근 메시지가 비어 있고 Conversation Start 전용 HTTP 계약을 사용한다. 결과는 공통 Result Router가 Redis에서 현재 realtime connection owner를 조회한 뒤 해당 instance의 REST endpoint로 전달한다. 공통 AiTask 계약과 계획된 Core NATS delivery는 아직 연결되지 않았다.

## Current Implementation Gap

현재 Orchestrator는 REST ingress, 메모리 Repository, 프로세스 내부 Scheduler를 사용한다. 동일 `roomSessionId` 멱등 처리, 재입장 시 이전 실행 취소, 실행 직전 Global → Feature → User Policy 확인, Omni AI Server 호출과 REST 기반 Result Delivery까지 구현되어 있다. 방 입장 요청의 owner와 Client session은 최상위 개별 필드가 아니라 공통 `routingRef`에서 읽는다.

남은 범위는 최근 메시지·Presence Context 조회, cooldown과 권한 판단, 검증된 인증 context에서 `tenantId`·`userId`를 복원하는 계약, 공통 AiTask 및 전체 correlation, durable 실행 저장소와 Scheduler, 계획된 Core NATS delivery, 실패 retry와 최종 Client Suggestion E2E 검증이다. 등록되어 있는 Redis Conversation Start Repository는 아직 구현되지 않았고 `@Primary` 메모리 Repository가 사용된다.

## Open Questions

- Phase 1에서 `triggerId`, `taskId`를 모두 도입할지
- Phase 1 완료 기준에서 현재 REST Delivery를 유지할지, Core NATS 전환까지 포함할지
- `Messenger Client Ingress`의 현재 connection owner가 event를 수신한 뒤 local Client Suggestion을 전달하는 구간을 어떤 환경에서 E2E 검증할지

## Related Documents

- [Roadmap](../roadmap.md)
- [Server Trigger](../components/ai-orchestrator/server-trigger.md)
- [Task Execution](../components/omni-ai-server/task-execution.md)
- [Session Registry Based AI Result Routing](../decisions/001-session-registry-result-routing.md)
- [Messenger Client Ingress Result Delivery](../components/messenger-client-ingress/result-delivery.md)
- [AI Orchestrator Result Router](../components/ai-orchestrator/result-router.md)
- [Omni AI Server Result Event](../components/omni-ai-server/result-event.md)
