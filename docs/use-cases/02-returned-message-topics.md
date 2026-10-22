# 02. Returned Message Topics

## User Outcome

Desktop Client 사용자가 일정 시간 이상 자리비운 뒤 온라인으로 복귀하고, 해당 구간에 읽지 않은 수신 쪽지가 있으면 요약 실행 여부를 확인한다. 사용자가 동의하면 복귀 시점에 선정한 쪽지를 발신자·업무 주제·우선순위 중심의 짧은 목록으로 받는다.

## Trigger

- Presence owner는 인증된 connection context에서 `clientPlatform = DESKTOP`이고 현재 Messenger 계약의 `connectType`이 `1` 또는 `2`인지 확인한다. 다른 연결은 이 Use Case의 `USER_RETURNED` 요청을 생성하지 않는다.
- Presence owner는 대상 연결의 `AWAY/OFFLINE → ONLINE` 상태 전이를 감지하고 `offlineSince`, `returnedAt`, `absencePeriodId`를 확정한다. 이 조건이 성립한 경우에만 `USER_RETURNED`를 생성한다.
- AI Orchestrator는 확정된 복귀 사실을 재판정하지 않고, Returned Message Topics의 기능별 정책을 적용한다. 자리비움 시간은 `minAwayDuration` 이상이어야 하며 초기 정책값 후보는 15분이다.
- `maxAbsenceDuration`, `maxLookback`, Trigger expiration을 넘긴 장기 부재는 AI Orchestrator가 실행하지 않거나 제한된 조회 구간만 사용한다.
- 복귀 구간에 읽지 않은 수신 쪽지가 없으면 사용자 확인과 AI 실행을 모두 생략한다.
- 다른 `connectType`의 단순 연결 시도 시각은 메시지를 확인했다는 근거로 사용하지 않는다. 향후 명시적인 메시지함 열람 또는 동기화 이벤트가 생기면 별도 정책으로 검토한다.

`connectType` 숫자값은 현재 Desktop Client와 Messenger 사이의 legacy 계약이다. AI Orchestrator와 Omni AI Server가 이 숫자의 의미를 해석하지 않으며, Messenger Client Ingress가 이 Use Case의 적용 대상인지 판단한 뒤 검증한 값을 Trigger와 `routingRef`에 제공한다. Orchestrator의 후속 `connectionType` 확인은 복귀 사실을 재판정하는 정책이 아니라 다른 종류의 connection으로 잘못 전달하지 않기 위한 routing constraint 검증이다.

이 문서에서 `WS`는 `WebSocket Service`, `NS`는 `TCP Service`를 의미한다. 두 물리 서비스를 포괄하는 논리적 경계는 `Messenger Client Ingress`다.

## End-to-End Flow

```text
Desktop Client(connectType 1 또는 2) AWAY/OFFLINE → ONLINE
→ Presence owner
  - 인증된 Desktop / connectType 자격 확인
  - 상태 전이 감지
  - offlineSince / returnedAt / absencePeriodId 확정
  - USER_RETURNED 확정
→ WebSocket Service (WS) 또는 TCP Service (NS)가 AI Orchestrator REST API 호출
  - USER_RETURNED request
  - absence period
  - trusted routingRef
→ AI Orchestrator
  - source event contract 검증(복귀 사실은 재판정하지 않음)
  - 기능·사용자·부재 시간·만료·중복 정책 확인
→ Message Query Port
→ 현재 메시지 조회 책임 Service의 내부 DB 조회 Queue
→ 복귀 구간의 unread count와 messageSelectionRef 반환
   ├─ 0건: SKIPPED_NO_MESSAGES
   └─ N건: 현재 요청 connection으로 요약 확인 팝업 전달
→ Client: YES / NO
   ├─ NO 또는 timeout: DECLINED / EXPIRED
   └─ YES: stable executionId 생성 후 RETURNED_MESSAGE_TOPICS 실행 dispatch
→ 저장된 messageSelectionRef와 source item key로 AI 실행 입력 구성
→ Omni AI Server: Topic Extraction + Priority
→ AI Orchestrator Result Router
→ routingRef와 동일한 connectionType의 현재 connection owner resolve
→ Messenger Client Ingress의 현재 connection owner
→ Client Topic Digest
→ Client가 로컬 삭제 상태를 반영해 최종 노출
```

현재 Messenger 구조에서는 쪽지 DB 조회 Queue와 조회 로직을 해당 구현을 보유한 Messenger Service가 소유한다. AI Orchestrator는 조회를 조정하는 Message Query Port만 사용하고 쪽지 DB를 직접 조회하지 않는다.

현재는 NATS JetStream Trigger Consumer가 구현되어 있지 않으므로 `WebSocket Service (WS)` 또는 `TCP Service (NS)`가 공통 AI Orchestrator REST API를 호출한다. 서비스별 AI Trigger Queue는 만들지 않는다. Trigger 재처리와 durable delivery를 이 프로젝트에서 도입할 때는 동일한 request 계약을 NATS JetStream adapter로 확장한다.

## Responsibility

| Component | Responsibility |
| --- | --- |
| Presence owner inside Messenger Client Ingress | 인증된 Desktop `connectType 1, 2` 적용 대상 판단, `AWAY/OFFLINE → ONLINE` 전이 감지, 자리비움 구간과 `USER_RETURNED` 확정, trusted connection context 제공 |
| 현재 Message owner | 내부 DB 조회 Queue를 통한 unread count, 후보 메시지 선정과 AI Context 제공 |
| AI Orchestrator | source event contract 검증, 기능별 실행 정책 평가, 사용자 확인 lifecycle, Context Assembly, 중복 방지, AiTask와 실행 이력 관리. Presence 상태 전이는 재판정하지 않음 |
| Omni AI Server | 선정된 메시지의 발신자·업무 주제·우선순위 추출 |
| Result Router | trusted `routingRef` 검증, 동일 connectionType의 현재 connection owner 선택 |
| Messenger Client Ingress | 확인 팝업과 Topic Digest를 선택된 WebSocket 또는 TCP local connection에 전달 |
| Client | 사용자 동의 입력, 로컬에서 삭제된 원본을 제외한 최종 결과 노출 |

## Contracts

### USER_RETURNED REST Request

이 요청은 Presence owner가 적용 대상 connection과 복귀 상태 전이를 확정한 뒤 생성하는 source-qualified Event다. AI Orchestrator는 Client가 보낸 원시 상태나 legacy `connectType`으로 복귀 여부를 다시 계산하지 않는다.

```text
POST /api/v1/returned-messages

requestId
triggerId
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

REST endpoint 자체가 `USER_RETURNED`를 나타내므로 현재 HTTP body에 별도 `eventType`을 두지 않는다. `routingRef.tenantId`, `routingRef.userId`가 owner의 단일 입력 출처이며 최상위에 같은 값을 중복하지 않는다. 현재 단계에서는 Messenger Client Ingress가 source-qualified service request를 구성하고, 향후 인증 계약에서는 owner를 검증된 인증 context에서 복원한다.

`routingRef.connectionType`은 Returned Message Topics에서 필수다. Client request payload를 그대로 신뢰하지 않고 Messenger Client Ingress가 인증된 connection context에서 설정한다. Result Router는 target reference로 현재 connection을 찾은 뒤 Connection Registry의 `connectionType`이 이 값과 일치하는지 검증한다.

### Message Selection Result

```text
absencePeriodId
unreadCount
messageSelectionRef
selectedAt
selectionExpiresAt
```

`readDate == '00000000000000'` 같은 저장소 표현은 Message owner 내부 adapter가 unread 조건으로 해석한다. AI Use Case 계약에는 원시 sentinel 값을 노출하지 않는다.

현재 WebSocket Service query는 unread 메시지 key 목록을 반환한다. AI Orchestrator adapter가 해당 목록을 저장하면서 `messageSelectionRef`, `selectedAt`, `selectionExpiresAt`을 생성하므로 위 구조는 Orchestrator의 `MessageSelection` port 결과다. `messageSelectionRef`는 복귀 확인 시점에 unread였던 메시지 key 집합을 가리키는 짧은 수명의 reference다. 대량의 원문을 Trigger나 사용자 확인 요청에 직접 싣지 않는다.

### User Confirmation

```text
POST /api/v1/returned-messages/{requestId}/confirmations

absencePeriodId
confirmationId
messageSelectionRef
decision: YES | NO
decidedAt
routingRef
  tenantId
  userId
  targetRef
  connectionType: 1 | 2
```

확인 응답은 인증된 현재 connection에서 받아야 하며 `requestId + absencePeriodId`로 중복 응답을 차단한다. `NO`는 정상 종료이고, 제한 시간 내 응답이 없으면 `EXPIRED`로 종료한다.

### Current Execution Command and Planned AiTask

현재 구현은 공통 `AiTask`를 생성하지 않고 stable `executionId`를 저장한 뒤 다음 실행 command를 `AiExecutionGateway`로 전달한다.

```text
ExecutionCommand
  requestId
  triggerId
  executionId
  tenantId
  userId
  absencePeriodId
  messageSelectionRef
  selectedItems[]
  routingRef
```

공통 실행 계약이 연결된 이후의 목표 `AiTask`는 다음과 같다.

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

- `SourceEventContractValidation`: trusted Presence owner가 생성한 Event인지, 필수 필드와 시간 구간이 정합적인지 확인. Desktop/connectType 의미나 상태 전이를 재판정하지 않음
- `FeatureAwayDurationPolicy`: Returned Message Topics의 `minAwayDuration` 이상인지 확인
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
→ SKIPPED_NO_MESSAGES
| EXPIRED
| FAILED
| AWAITING_CONFIRMATION
    → BLOCKED_BY_POLICY
    | DECLINED
    | EXPIRED
    | PROCESSING
        → DELIVERY_PENDING
            → COMPLETED
            | DELIVERY_FAILED
        | FAILED
        | CANCELLED
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

- 공통 구현: Result Router가 Redis에서 현재 connection owner를 조회하고 해당 instance REST endpoint로 전달
- Returned Message 현재 연결 상태: `ReturnedMessageResultSender`의 실제 adapter가 없어 위 공통 라우팅 경로와 아직 연결되지 않음
- 계획된 경로: 같은 resolution 이후 Core NATS owner-instance subject로 전달
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

- Presence owner의 Desktop `connectType 1, 2` 적용 대상 판단과 복귀 Event 확정
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
- 현재 Messenger 서비스의 물리적 분리·통합

## Acceptance Criteria

- Presence owner는 Desktop `connectType 1, 2` 이외의 연결에서 이 Use Case의 `USER_RETURNED` 요청을 생성하지 않는다.
- Orchestrator는 source-confirmed `USER_RETURNED`를 상태 원시값으로 재판정하지 않는다.
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

## Presence Trigger Verification Contract

Messenger Client Ingress의 Presence owner는 다음 계약을 검증해야 한다.

| 구분 | 입력/조건 | 기대 결과 |
|---|---|---|
| 적용 대상 | 인증된 `connectionType` 1 또는 2 | Presence Trigger 판단 수행 |
| 비대상 | `connectionType` 1, 2 외의 연결 | `USER_RETURNED` 미생성 |
| OFFLINE | state 0 | 부재 구간 시작 |
| ONLINE | state 1 | 기존 부재 구간이 있을 때만 `USER_RETURNED` 생성 |
| AWAY | state 2~9 | 부재 구간 시작 |
| 반복 부재 상태 | AWAY/OFFLINE 간 전이 | 최초 `offlineSince`와 `absencePeriodId` 유지 |
| 비정상 종료 | 명시적 state 요청 없이 disconnect | current connection일 때 부재 구간 시작 |
| stale 종료 | current가 아닌 과거 connection disconnect | 부재 구간 미생성 |
| AT 재발급 | Refresh Token의 SID로 Access Token 재발급 | SID 유지 |
| SID 변경 | 재인증 후 새 SID, 동일 user/type 복귀 | 기존 부재 종료, 새 SID로 routing |
| connectionType 변경 | type 1 부재 후 type 2 ONLINE | type 1 부재를 종료하지 않음 |
| 중복 ONLINE | 동일 부재 구간에 대한 복수 복귀 | Trigger 최대 1회 |

부재 상태의 식별 단위는 `tenantId + userId + connectionType`이다. `clientSessionId`는 인증된 current connection 조회와 `routingRef.targetRef`에 사용하며, 부재 구간의 식별자로 사용하지 않는다.

Redis의 부재 정보 TTL은 AI Orchestrator의 `maxAbsenceDuration`보다 길어야 한다. TTL이 더 짧으면 Orchestrator가 장기 부재 정책을 판단하기 전에 Presence owner가 Trigger 근거를 소실한다.

### Verification Ownership

| 검증 범위 | 소유 Component/Repository |
|---|---|
| 상태 계약, SID, connectionType, disconnect/reconnect, Redis 부재 구간, REST retry | WebSocket Service Presence Trigger Verification |
| source contract, min/max absence, Trigger expiration, lifecycle 멱등성 | AI Orchestrator Returned Message Lifecycle Verification |
| 실제 Client 복귀에서 확인, AI 실행과 결과 routing까지 | Cross-component E2E Verification |

## System Integration Verification

Returned Message Topics의 System Integration Verification은 개별 Component의 내부 동작이 아니라, 실제 서비스 경계를 거쳐 사용자 확인부터 Topic Digest 전달까지 전체 Use Case가 완성되는지를 검증한다.

핵심 서비스 간 호출 경로에 Mock 또는 Stub을 사용하지 않으며, 각 단계에서 생성·전달되는 `triggerId`, `absencePeriodId`, `executionId` 등의 Correlation 정보가 일관되게 연결되어 전체 흐름을 추적할 수 있어야 한다.

검증은 해당 Use Case의 Acceptance Criteria를 기준으로 수행하며, 실패하거나 검증하지 못한 경계는 별도로 기록한다.

| 항목 | 값 |
|---|---|
| Status | Planned |
| Scope | WS → AI Orchestrator ↔ Message Owner → Omni AI Server → Realtime → Client |
| Environment | - |
| Component Versions / Commits | - |
| Last Verified At | - |
| Evidence | - |
| Known Gaps | - |

### Integration Test Summary

| ID | 통합 검증 시나리오 | 기대 결과 | Status |
|---|---|---|---|
| INT-001 | 사용자 복귀 Trigger 전달 | WS에서 생성한 `USER_RETURNED`가 AI Orchestrator에 전달되고 lifecycle이 생성된다. | Planned |
| INT-002 | 메시지 선정 및 사용자 확인 | 실제 Message owner 조회 결과로 확인 요청이 올바른 Client에 전달된다. | Planned |
| INT-003 | 사용자 `YES` 후 AI 실행 | 유효한 확인 응답에 대해 Omni AI Server 실행이 한 번만 시작된다. | Planned |
| INT-004 | AI 결과 전달 | Topic Digest가 동일 `routingRef`의 현재 Client에 전달되고 lifecycle이 완료된다. | Planned |
| INT-005 | 중복 Trigger | 같은 `absencePeriodId`의 재전송이 lifecycle 또는 AI 실행을 중복 생성하지 않는다. | Planned |
| INT-006 | 사용자 `NO` 또는 만료 | lifecycle이 `DECLINED` 또는 `EXPIRED`로 종료되고 AI 실행은 시작되지 않는다. | Planned |

### Execution Evidence

| 항목 | 값 |
|---|---|
| Environment | - |
| Component Versions / Commits | - |
| Executed At | - |
| Correlation IDs | `triggerId`, `absencePeriodId`, `executionId` |
| Result | - |
| Evidence Link | - |

## Current Implementation Gap

WebSocket Service에는 Desktop `connectionType 1, 2` 적용 대상 판단, 명시적 상태 변경과 disconnect/reconnect 기반 부재 구간 관리, Redis current connection 검증과 `USER_RETURNED` REST 제출이 구현되어 있다. 자동화 검증과 durable Trigger Queue는 아직 완료되지 않았다.

AI Orchestrator에는 `USER_RETURNED` REST endpoint, JDBC lifecycle 및 selection 저장소, 기능·사용자 정책과 permission 평가, WebSocket Service DB 조회 API를 연결한 Message Query adapter, confirmation 상태 전이, 실행 correlation과 recovery/maintenance 흐름이 구현되어 있다. 공통 Result Router의 Redis connection resolution과 `connectionType` 검증도 구현되어 있다.

아직 연결되지 않은 범위는 TCP Service(NS) Presence Trigger, 실제 confirmation delivery adapter, Omni AI Server 실행·reconciliation adapter, Returned Message result delivery adapter, Topic Extraction/우선순위 Workflow와 전체 Client E2E다. 해당 outbound port가 설정되지 않으면 현재 fallback 구현은 전송 실패 또는 `UNKNOWN` 실행 상태를 반환한다. NATS JetStream Trigger Consumer도 현재 구현 범위가 아니다.

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
- [Messenger Client Ingress Result Delivery](../components/messenger-client-ingress/result-delivery.md)
