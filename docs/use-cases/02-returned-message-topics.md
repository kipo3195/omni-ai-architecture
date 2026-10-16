# 02. Returned Message Topics

## User Outcome

Desktop Client 사용자가 일정 시간 이상 자리비운 뒤 온라인으로 복귀하고, 해당 구간에 읽지 않은 수신 쪽지가 있으면 요약 실행 여부를 확인한다. 사용자가 동의하면 복귀 시점에 선정한 쪽지를 발신자·업무 주제·우선순위 중심의 짧은 목록으로 받는다.

## Trigger

- 적용 대상은 `clientPlatform = DESKTOP`이면서 현재 Messenger 계약의 `connectType`이 `1` 또는 `2`인 연결이다.
- 대상 연결의 사용자 상태가 `AWAY/OFFLINE → ONLINE`으로 변경되면 복귀 후보가 된다.
- 자리비움 시간이 `minAwayDuration` 이상이어야 한다. 초기 정책값 후보는 15분이다.
- `maxAbsenceDuration`, `maxLookback`, Trigger expiration을 넘긴 장기 부재는 실행하지 않거나 제한된 조회 구간만 사용한다.
- 복귀 구간에 읽지 않은 수신 쪽지가 없으면 사용자 확인과 AI 실행을 모두 생략한다.
- 다른 `connectType`의 단순 연결 시도 시각은 메시지를 확인했다는 근거로 사용하지 않는다. 향후 명시적인 메시지함 열람 또는 동기화 이벤트가 생기면 별도 정책으로 검토한다.

`connectType` 숫자값은 현재 Desktop Client와 Messenger 사이의 legacy 계약이다. AI Orchestrator와 Omni AI Server가 이 숫자의 의미를 해석하지 않으며, Realtime/Messenger 경계가 검증한 값을 Trigger와 `routingRef`에 제공한다.

이 문서의 `WS/NS`는 현재 Desktop connection과 상태 변경을 처리하는 배포 컴포넌트를 가리킨다. 공통 아키텍처에서는 두 컴포넌트를 `Realtime/Messenger Service` 경계로 표현하며, 물리적 서비스 분리는 이 Use Case의 범위에 포함하지 않는다.

## End-to-End Flow

```text
Desktop Client(connectType 1 또는 2) AWAY/OFFLINE → ONLINE
→ Presence owner: offlineSince / returnedAt / absencePeriodId 확정
→ WS/NS가 AI Orchestrator REST API 호출
  - USER_RETURNED request
  - absence period
  - trusted routingRef
→ AI Orchestrator: 기능·사용자·부재 시간·만료·중복 정책 확인
→ Message Query Port
→ 현재 메시지 조회 책임 Service의 내부 DB 조회 Queue
→ 복귀 구간의 unread count와 messageSelectionRef 반환
   ├─ 0건: SKIPPED_NO_MESSAGES
   └─ N건: 현재 요청 connection으로 요약 확인 팝업 전달
→ Client: YES / NO
   ├─ NO 또는 timeout: DECLINED / EXPIRED
   └─ YES: RETURNED_MESSAGE_TOPICS AiTask 생성
→ Message owner를 통해 messageSelectionRef의 AI Context 조회
→ Omni AI Server: Topic Extraction + Priority
→ AI Orchestrator Result Router
→ routingRef와 동일한 connectionType의 현재 connection owner resolve
→ Realtime Service
→ Client Topic Digest
→ Client가 로컬 삭제 상태를 반영해 최종 노출
```

현재 Messenger 구조에서는 쪽지 DB 조회 Queue와 조회 로직을 해당 Service가 소유한다. AI Orchestrator는 조회를 조정하는 Port만 사용하고 쪽지 DB를 직접 조회하지 않는다. 향후 Message Service가 분리되면 이 Port의 adapter를 Service API, gRPC 또는 명시적으로 계약된 Projection으로 교체한다.

현재는 NATS JetStream Trigger Consumer가 구현되어 있지 않으므로 WS/NS가 공통 AI Orchestrator REST API를 호출한다. WS/NS별 AI Trigger Queue는 만들지 않는다. 향후 Trigger 재처리와 durable delivery가 필요해지면 동일한 request 계약을 NATS JetStream adapter로 확장할 수 있지만, 이는 현재 흐름에 포함하지 않는다.

## Responsibility

| Component | Responsibility |
| --- | --- |
| WS/NS 또는 Presence owner | Desktop `connectType 1, 2`의 상태 변경 감지, 자리비움 구간 확정, 검증된 connection context 제공 |
| 현재 Message owner | 내부 DB 조회 Queue를 통한 unread count, 후보 메시지 선정과 AI Context 제공 |
| AI Orchestrator | 정책 평가, 사용자 확인 lifecycle, Context Assembly, 중복 방지, AiTask와 실행 이력 관리 |
| Omni AI Server | 선정된 메시지의 발신자·업무 주제·우선순위 추출 |
| Result Router | trusted `routingRef` 검증, 동일 connectionType의 현재 connection owner 선택 |
| Realtime Service | 확인 팝업과 Topic Digest를 선택된 local connection에 전달 |
| Client | 사용자 동의 입력, 로컬에서 삭제된 원본을 제외한 최종 결과 노출 |

## Contracts

### USER_RETURNED REST Request

```text
requestId
triggerId
eventType: USER_RETURNED
tenantId
userId
absencePeriodId
offlineSince
returnedAt
occurredAt
expiresAt
routingRef
  tenantId
  userId
  targetRef
  connectionType: 1 | 2
```

`routingRef.connectionType`은 Client request payload를 그대로 신뢰하지 않고 WS/NS 또는 Realtime Service가 인증된 connection context에서 설정한다. Result Router는 target reference로 현재 connection을 찾은 뒤 Connection Registry의 `connectionType`이 이 값과 일치하는지 검증한다.

### Message Query Result

```text
absencePeriodId
unreadCount
messageSelectionRef
selectedAt
selectionExpiresAt
```

`readDate == '00000000000000'` 같은 저장소 표현은 Message owner 내부 adapter가 unread 조건으로 해석한다. AI Use Case 계약에는 원시 sentinel 값을 노출하지 않는다.

`messageSelectionRef`는 복귀 확인 시점에 unread였던 메시지 key 집합 또는 그 집합을 가리키는 짧은 수명의 reference다. 대량의 원문을 Trigger나 사용자 확인 요청에 직접 싣지 않는다.

### User Confirmation

```text
requestId
absencePeriodId
messageSelectionRef
unreadCount
decision: YES | NO
routingRef
decidedAt
```

확인 응답은 인증된 현재 connection에서 받아야 하며 `requestId + absencePeriodId`로 중복 응답을 차단한다. `NO`는 정상 종료이고, 제한 시간 내 응답이 없으면 `EXPIRED`로 종료한다.

### AiTask and Result

```text
AiTask
  taskType: RETURNED_MESSAGE_TOPICS
  triggerId
  taskId
  executionId
  tenantId
  userId
  absencePeriodId
  messageSelectionRef 또는 contextRef
  routingRef

TopicDigestResult
  executionId
  topics[]
    senderName
    topic
    summary
    priority
    priorityReason
    messageCount
    sourceMessageRefs[]
  generatedAt
```

## Message Selection and Read/Delete Semantics

사용자 확인 팝업을 만들 때는 복귀 구간에 수신했고 당시 unread였던 쪽지를 선정한다. 사용자가 `YES`를 선택해 AI 실행이 시작된 뒤에는 해당 후보의 현재 `readDate`를 다시 필터 조건으로 사용하지 않는다. 그 사이 사용자가 쪽지를 읽었더라도 최초 `messageSelectionRef`에 포함된 메시지는 AI 처리 대상에 유지한다.

삭제 여부는 Client 로컬 DB가 최종 노출 시 판단한다.

- Client는 `sourceMessageRefs`를 로컬 DB와 비교해 삭제된 원본을 결과에서 제외한다.
- 하나의 Topic에 남은 원본이 하나도 없으면 해당 Topic을 노출하지 않는다.
- 일부 원본만 삭제되었다면 남은 원본 기준으로 이동 대상과 표시 `messageCount`를 조정한다.
- Topic Digest 수신 자체가 원본 쪽지의 `readDate`를 변경한다는 의미는 아니다.

이 정책은 AI 실행 도중 발생한 읽음 상태 변경 때문에 결과가 흔들리는 것을 막고, Client에만 존재하는 삭제 상태는 실제 노출 직전에 반영하기 위한 것이다.

## Policy

AI Orchestrator는 Omni AI Server 호출 전에 다음 정책을 평가한다.

- `DesktopConnectionPolicy`: Desktop이며 `connectType in {1, 2}`인지 확인
- `AwayDurationPolicy`: `minAwayDuration` 이상인지 확인
- `LongAbsencePolicy`: `maxAbsenceDuration` 또는 `maxLookback` 적용
- `Feature/User/PermissionPolicy`: 기능과 사용자 설정 확인
- `UnreadMessagePolicy`: 초기 선정 시 unread 메시지가 하나 이상인지 확인
- `DuplicatePeriodPolicy`: 동일 자리비움 구간의 중복 확인과 실행 방지
- `ConfirmationPolicy`: 유효한 `YES` 응답인지 확인
- `TriggerExpirationPolicy`: 오래된 Trigger, 확인 요청과 selection 만료 처리

별도의 cooldown은 초기 필수 정책으로 두지 않는다. 동일 구간 멱등성, 확인 요청 만료와 장기 부재 정책으로 과도한 실행을 우선 통제하고 운영 지표를 본 뒤 추가한다.

## State and Idempotency

```text
DETECTED
→ SKIPPED_POLICY
| SKIPPED_NO_MESSAGES
| AWAITING_CONFIRMATION
    → DECLINED
    | EXPIRED
    | PROCESSING
        → COMPLETED
        | FAILED
        | DELIVERY_FAILED
```

- 복귀 구간은 `tenantId + userId + absencePeriodId`를 idempotency key로 사용한다.
- 사용자 확인은 `requestId + absencePeriodId`로 한 번만 상태를 전이한다.
- AI 실행은 `executionId`, 결과는 `eventId`로 중복 처리한다.
- 이벤트 이력 저장은 공통 비동기 처리 경로를 사용할 수 있지만 graceful shutdown만으로 내구성을 보장하지 않는다. 이력이 복구 대상이면 Outbox, durable Queue 또는 durable execution repository를 사용한다.
- AI 생성 성공과 Client 전달 성공을 별도 상태로 기록한다.

## AI Workflow

복귀 시점에 선정된 쪽지를 발신자와 업무 주제 중심으로 그룹화하고 요약과 우선순위를 생성한다. LangGraph 적용 여부는 구현 선택이며, 공통 입력·출력 계약과 품질 조건을 우선한다.

### Context Scope

- 필수: 메시지 key, 발신자, 수신 시각, subject, 원본 이동 식별자
- 선택: 요약에 필요한 최소 본문, 권한이 확인된 첨부 파일명
- 제외: 원본 파일 내용, 무제한 과거 메시지, Trigger payload에 실은 전체 원문

subject만 사용할지 본문과 파일명까지 사용할지는 품질·개인정보·token cost를 측정해 확정한다. 파일명은 권한 확인 후 필요한 경우에만 포함한다.

### Quality Focus

- 원문에 없는 업무나 긴급도를 만들지 않는다.
- `priority`에는 원문 근거를 반영하고 `priorityReason`과 `sourceMessageRefs`로 추적 가능하게 한다.
- 근거가 부족하면 기본 우선순위 또는 `UNKNOWN`을 사용한다.
- 짧고 구분 가능한 Topic 제목을 생성한다.
- 원본 메시지와의 추적 가능성을 유지한다.

## Result Routing

확인 팝업과 AI 결과는 모든 사용자 연결에 broadcast하지 않는다. `routingRef`에 담긴 논리 target과 `connectionType`을 사용해 요청한 Desktop Client와 같은 타입의 현재 connection으로 전달한다.

- 현재 구현: Result Router가 Redis에서 현재 connection owner를 조회하고 해당 instance REST endpoint로 전달
- Target Architecture: 같은 resolution 이후 Core NATS owner-instance subject로 전달
- Result 처리 시점에 현재 owner를 다시 조회하며 Trigger source instance를 최종 target으로 사용하지 않는다.
- targetRef는 인증 session 또는 device 같은 논리 target을 식별하고, `connectionType`은 선택된 connection에 대한 추가 검증 조건이다.

## Failure and Deferred Execution

- 메시지 조회 실패, 사용자 확인 전달 실패, AI 생성 실패와 결과 전달 실패를 구분한다.
- USER_RETURNED REST 호출의 timeout과 retry에도 동일한 `triggerId`를 사용해 중복 실행을 방지한다.
- DB 조회 Queue의 retry와 timeout은 현재 Message owner가 책임지고, Orchestrator는 Port 호출 결과를 execution 상태로 반영한다.
- 만료된 Trigger, 확인 요청 또는 `messageSelectionRef`는 재실행하지 않는다.
- AI 실행 중 Client가 reconnect하면 trusted `routingRef`의 target과 `connectionType`으로 현재 owner를 다시 resolve한다.
- 유효한 target이 없으면 요청 당시 instance로 fallback하지 않고 제한된 retry 후 `DELIVERY_FAILED`로 기록한다.

## Scope

- Desktop `connectType 1, 2`의 복귀 감지
- 자리비움 구간과 장기 부재 정책
- 현재 Service의 DB 조회 Queue를 이용한 unread 후보 선정
- 사용자 확인 후 AI 실행
- 발신자·업무 주제·우선순위 생성
- 요청 connectionType 기반 Topic Digest 전달
- Client 로컬 삭제 상태 기반 최종 노출

## Non-scope

- Desktop 이외 Client와 `connectType 1, 2` 이외 연결
- 모든 사용자 connection에 대한 broadcast
- 전체 대화 요약
- 사용자가 정의한 반복 Schedule
- Tool Calling
- Message Service 물리 분리

## Acceptance Criteria

- Desktop `connectType 1, 2` 이외의 복귀는 이 기능을 시작하지 않는다.
- `minAwayDuration` 미만이거나 장기 부재/만료 정책에 걸리면 실행하지 않는다.
- 대상 unread 메시지가 없으면 확인 팝업과 LLM 호출을 수행하지 않는다.
- 사용자가 `YES`를 선택하기 전에는 LLM을 호출하지 않는다.
- AI 실행 시작 후에는 후보 메시지의 현재 읽음 여부와 관계없이 최초 selection을 처리한다.
- 동일 복귀 구간과 동일 확인 응답이 중복 처리되지 않는다.
- Orchestrator는 쪽지 DB를 직접 조회하지 않는다.
- 팝업과 결과는 trusted `routingRef`와 같은 `connectionType`의 현재 connection으로 전달된다.
- Client는 로컬에서 삭제된 원본만 결과 노출에서 제외한다.
- Topic Digest에서 노출된 원본 메시지로 이동할 수 있다.
- 우선순위와 요약을 원본 메시지까지 추적할 수 있다.

## Current Implementation Gap

Desktop 복귀 감지와 자리비움 구간 관리, USER_RETURNED REST endpoint, DB 조회 Queue adapter를 연결한 Message Query Port, 사용자 확인 lifecycle, Topic Extraction/우선순위 Workflow와 connectionType 검증을 포함한 결과 전달이 아직 구현되지 않았다. NATS JetStream Trigger Consumer는 현재 구현 범위가 아니다.

## Open Questions

- `minAwayDuration`, `maxAbsenceDuration`, `maxLookback`과 확인 timeout의 운영값
- `absencePeriodId` 생성 주체와 Presence 상태의 저장소
- 발신자별, 대화방별 또는 업무 주제별 그룹화 기준
- subject만 사용할지 최소 본문과 첨부 파일명까지 사용할지
- 일부 원본 삭제 후 Client가 표시할 Topic summary와 `messageCount` 보정 규칙의 상세 protocol

## Related Documents

- [Roadmap](../roadmap.md)
- [Server Trigger](../components/ai-orchestrator/server-trigger.md)
- [Server-driven AI](../concepts/server-driven-ai.md)
- [Session Registry Based AI Result Routing](../decisions/001-session-registry-result-routing.md)
- [Realtime Service Result Delivery](../components/realtime-service/result-delivery.md)
