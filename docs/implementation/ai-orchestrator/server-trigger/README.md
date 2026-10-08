# Server Trigger

Status: In Progress

Business Event가 AI 실행으로 이어지는 AI Orchestrator의 공통 구조를 기록한다.

Event 수신, Trigger Policy, Context Assembly, AiTask 생성, 실행 연계의 책임과 흐름을 실제 구현 기준으로 정리한다. Conversation Start와 같은 기능은 이 구조의 적용 사례로 다루고, 기능 고유의 정책과 Context만 구분해 기록한다.

## 현재 적용 사례: Conversation Start

`ai-orchestrator`의 `conversationstart` 패키지에 구현된 흐름이다. 방 입장 비즈니스 이벤트는 Realtime Service가 먼저 처리하며, Realtime Service가 AI Orchestrator의 REST API를 호출해 Conversation Start 실행을 요청한다. 따라서 전체 시스템 관점에서는 방 입장 이벤트로 시작하지만, AI Orchestrator 내부의 직접 진입점은 Business Event Consumer가 아니라 REST API다.

```text
Realtime Service가 방 입장 비즈니스 이벤트 처리
  → AI Orchestrator의 POST /api/v1/conversation-starts 호출
  (roomSessionId, clientSessionId, userID, roomKey, chatType)
  → ConversationStartService.start()
  → roomSessionId로 활성 실행 조회
      ├─ 같은 사용자·방의 실행이 존재: 중복 입장 요청으로 판단하고 기존 실행 반환
      ├─ 다른 사용자 또는 방의 실행이 존재: roomSessionId 충돌로 판단하고 요청 거부
      └─ 활성 실행 없음: 신규 입장 요청 처리 계속
  → 신규 입장 요청과 동일한 사용자·방의 이전 활성 실행 조회
      ├─ 이전 실행 존재: 이전 실행과 예약 작업 취소
      └─ 이전 실행 없음: 그대로 진행
  → RoutingRef와 executionId 생성
  → AiExecution을 SCHEDULED 상태로 메모리에 저장
  → 설정된 시간 뒤 스케줄러가 실행
  → 예약 실행과 현재 roomSession 실행의 일치 여부 확인
  → TRENDING 추천을 Omni AI Server에 HTTP 요청
  → 응답의 session_id와 executionId 일치 확인
  → 결과를 실행 객체에 저장하고 COMPLETED로 전이
  → AiResultRouter를 통해 현재 Realtime 연결로 결과 전달 시도
```

- `controller`: 방 입장 요청을 시작 Use Case에 전달하고 `201 Created`와 `AiExecutionResponse`를 반환한다. 취소 요청은 `DELETE /api/v1/conversation-starts/{roomSessionId}?userID=...&roomKey=...` 형식으로 취소 Use Case에 전달한다.
- `application`: 동일 `roomSessionId` 요청의 멱등 처리, 기존 실행 취소, 지연 실행 예약, 현재 실행 여부 확인, AI 호출, 상태 전이와 결과 라우팅을 조정한다.
- `domain`: `ConversationStart`가 기능 데이터와 결과를 보관하고, 공통 `AiExecution`에 `CREATED → SCHEDULED → EXECUTING → COMPLETED` 상태 전이를 위임한다. `FAILED`, `CANCELLED` 상태도 공통 실행 모델에서 관리한다.
- `infrastructure`: `@Primary`인 메모리 Repository와 프로세스 내부 Scheduler를 사용한다. Redis Repository도 등록되어 있지만 현재 메서드는 구현하지 않았다. `OmniAiConversationStartClient`가 `/ai/v1/chat/suggestions/generate`를 호출한다.

현재 설정에서 기능은 활성화되어 있고 지연 시간은 10초다. 실행 시 추천 유형은 `TRENDING`, 최근 메시지는 빈 목록이며, AI 요청 옵션은 최대 5개·100자·한국어로 고정되어 있다. `chatType`은 대소문자와 관계없이 `open`일 때만 `open`으로 저장하고 그 외 값이나 미입력은 `chat`으로 정규화한다.

같은 `roomSessionId`로 동일 사용자·방이 다시 요청하면 새 실행을 만들지 않고 기존 활성 실행을 반환한다. 같은 사용자·방에 새로운 `roomSessionId`로 요청하면 이전 활성 실행을 자동 취소하고 새 실행을 예약한다.

방 퇴장에 따른 명시적 취소는 `roomSessionId`, `userID`, `roomKey`가 모두 일치하는 활성 실행만 대상으로 한다. 성공하면 실행을 `CANCELLED`로 변경하고 예약 작업을 취소한 뒤 `204 No Content`를 반환하며, 대상이 없거나 식별자가 일치하지 않으면 `404 Not Found`를 반환한다. 이미 AI 요청이 실행 중이면 요청 자체를 중단하지는 않지만, 이후 응답을 완료 처리하거나 Realtime Service로 전달하지 않는다. 이미 `COMPLETED` 또는 `FAILED`인 실행은 취소할 수 없다.

### 현재 구현 경계

- 기능 활성화 설정과 지연 실행은 사용하지만 `cooldown` 설정, 권한 판단, 공통 Trigger Policy는 아직 실행 흐름에 연결되지 않았다. Policy Provider와 관련 모델은 선언 단계다. 다만 `roomSessionId` 기반 멱등 처리와 동일 사용자·방의 이전 실행 취소는 `ConversationStartService`에 직접 구현되어 있다.
- 최근 메시지 조회 Port와 방 Presence 확인 Port는 호출되지 않는다. 현재 Context Assembly는 `executionId`, `roomSessionId`, 사용자·방 식별자, 입장 시각, 추천 유형과 빈 메시지 목록으로 제한된다. 이 중 AI HTTP 요청에는 `executionId`, 사용자·방 식별자, 추천 유형과 빈 메시지 목록만 전달된다.
- 공통 `AiTask`와 `triggerId / taskId / executionId` 상관관계 모델은 아직 없다. 대신 공통 `AiExecution`과 `RoutingRef`를 사용해 실행 상태와 결과 전달 대상을 연결한다. `executionId`는 Conversation Start 전용 AI 요청의 `session_id`로 전달된다.
- AI 결과는 메모리의 실행 객체에 저장된 후 공통 `AiResultRouter`로 전달된다. 라우터는 Redis에서 `clientSessionId`의 현재 Realtime 연결과 소유 인스턴스를 확인하고, 설정된 REST 엔드포인트로 `CONVERSATION_START_COMPLETED` 이벤트를 전달한다. `ConversationSuggestionSender`와 `RealtimeMessageSuggestionClient`는 이 경로에서 사용되지 않는다.
- 결과 전달 실패는 로그로 기록하지만 이미 완료된 실행을 실패 상태로 되돌리거나 재시도하지 않는다. 실행 결과 조회 API도 없으며, POST 응답의 `Location`에 대응하는 GET API도 아직 없다.
- 실행 Repository와 Scheduler가 프로세스 내부에 있으므로 재시작·다중 인스턴스에서 실행 상태와 예약 작업을 공유하지 않는다. 결과 라우팅에 사용하는 Redis 연결 정보는 이 실행 저장소와 별개다.

상세 E2E 흐름과 완료 조건은 [Conversation Start Use Case](../../use-cases/01-conversation-start/README.md)에서 관리한다.

## 다음 적용 사례: Returned Message Topics

사용자 상태가 `AWAY → ONLINE`으로 변경되면 자리비움 구간과 수신 쪽지를 확인하고, Topic Extraction이 필요한 경우 AiTask를 생성한다.

```text
USER_RETURNED
  → absence period 확인
  → 해당 기간의 수신 쪽지 조회
  → no message / duplicate / expired 판단
  → RETURNED_MESSAGE_TOPICS AiTask
  → Topic Digest 결과 전달
```

이 적용 사례의 상태는 `Planned`다. 상세 범위는 [Returned Message Topics Use Case](../../use-cases/02-returned-message-topics/README.md)에서 관리한다.
