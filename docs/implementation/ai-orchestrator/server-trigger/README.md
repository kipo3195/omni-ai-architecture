# Server Trigger

Business Event가 AI 실행으로 이어지는 AI Orchestrator의 공통 구조를 기록한다.

Event 수신, Trigger Policy, Context Assembly, AiTask 생성, 실행 연계의 책임과 흐름을 실제 구현 기준으로 정리한다. Conversation Start와 같은 기능은 이 구조의 적용 사례로 다루고, 기능 고유의 정책과 Context만 구분해 기록한다.

## 현재 적용 사례: Conversation Start

`ai-orchestrator`의 `conversationstart` 패키지에 구현된 흐름이다. 방 입장에 해당하는 요청을 받지만, 현재 진입점은 Business Event Consumer가 아닌 REST API다.

```text
POST /api/v1/conversation-starts (userID, roomKey, chatType)
  → ConversationStartService.start()
  → 동일 사용자·방의 활성 실행 취소
  → executionId 생성, SCHEDULED 상태로 메모리에 저장
  → 설정된 시간 뒤 스케줄러가 실행
  → TRENDING 추천을 Omni AI Server에 HTTP 요청
  → sessionId 일치 확인 후 결과를 실행 객체에 저장, COMPLETED
```

- `controller`: 방 입장 요청과 실행 취소 요청(`DELETE /api/v1/conversation-starts/{sessionId}`)을 Use Case에 전달한다.
- `application`: 기존 실행 취소, 지연 실행 예약, 현재 실행 여부 확인, AI 호출과 상태 전이를 조정한다.
- `domain`: `ConversationStart`가 `CREATED → SCHEDULED → EXECUTING → COMPLETED` 상태를 관리하며, 취소·실패 상태도 가진다.
- `infrastructure`: 메모리 Repository와 프로세스 내부 Scheduler를 사용한다. `OmniAiConversationStartClient`가 `/ai/v1/chat/suggestions/generate`를 호출한다.

현재 설정의 지연 시간은 10초다. 실행 시 추천 유형은 `TRENDING`, 최근 메시지는 빈 목록이며, AI 요청 옵션은 최대 5개·100자·한국어로 고정되어 있다. 같은 사용자·방에 새 요청이 오면 이전 활성 실행을 취소하고 새 실행을 예약한다. 취소된 실행이나 더 이상 현재 실행이 아닌 경우에는 결과를 완료 처리하지 않는다.

### 현재 구현 경계

- 기능 활성화 설정과 지연 실행은 사용하지만, cooldown·권한·중복 이벤트 판단을 수행하는 Trigger Policy는 아직 연결되지 않았다. Policy Provider와 관련 모델은 선언 단계다.
- 최근 메시지 조회 Port는 있으나 호출되지 않는다. 따라서 현재 Context Assembly는 식별자·입장 시각과 빈 메시지 목록으로 제한된다.
- 공통 `AiTask` 생성이나 `triggerId / taskId / executionId` 상관관계 저장 없이 Conversation Start 전용 HTTP 계약으로 AI Server를 호출한다. 현재 `executionId`는 AI 요청의 `session_id`로 전달된다.
- AI 결과는 메모리의 실행 객체에 저장된다. `ConversationSuggestionSender` 구현과 Realtime Service 전달은 연결되지 않았고, 결과 조회 API도 없다.
- Repository와 Scheduler가 프로세스 내부에 있으므로 재시작·다중 인스턴스에서 실행 상태와 예약 작업을 공유하지 않는다.
