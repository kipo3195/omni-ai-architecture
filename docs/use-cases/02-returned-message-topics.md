# 02. Returned Message Topics

## User Outcome

지원 Client 사용자가 일정 시간 이상 자리비운 뒤 온라인으로 복귀하고, 해당 구간에 읽지 않은 수신 쪽지가 있으면 요약 실행 여부를 확인한다. 사용자가 동의하면 복귀 시점에 선정한 쪽지의 **제목**을 바탕으로 업무 주제별 요약과 메시지별 우선순위·답변 필요 여부·후속 조치 필요 여부를 받는다.

## Trigger

- Presence owner는 인증된 connection context에서 현재 Messenger 계약의 `connectType`이 `1`, `2` 또는 `3`인지 확인하고, 대상 연결의 `AWAY/OFFLINE → ONLINE` 전이와 `offlineSince`, `returnedAt`, `absencePeriodId`를 확정한다.
- AI Orchestrator는 확정된 복귀 사실을 재판정하지 않고 필수 필드, 시간 관계, 기능·사용자 정책과 만료를 검증한다. 현재 `connectionType`은 양수 여부만 검증하며 legacy 숫자별 적용 대상 판단은 Messenger Client Ingress 책임이다.
- 기본 `minAwayDuration`은 **1분**, `maxAbsenceDuration`은 **7일**이다. Trigger가 만료됐거나 부재 기간이 범위를 벗어나면 lifecycle을 생성하지 않는다.
- `maxLookback` 기본값은 24시간이지만, 현재 WS 조회 adapter는 계산된 `lookbackSince` 대신 `offlineSince`를 사용한다. 실제 조회 구간 제한으로 동작하지 않는 점에 유의한다.
- 선정된 unread 쪽지가 없으면 사용자 확인과 AI 실행을 모두 생략한다.

`connectType` 숫자값은 Client와 Messenger 사이의 legacy 계약이다. 후속 `connectionType` 검증은 복귀 사실의 재판정이 아니라 다른 종류의 connection으로 결과를 전달하지 않기 위한 routing constraint다.

이 문서에서 `WS`는 `WebSocket Service`, `NS`는 `TCP Service`를 의미한다. 두 물리 서비스를 포괄하는 논리적 경계는 `Messenger Client Ingress`다. Presence와 Client 관련 항목은 경계 계약이며, 현재 서버 동작은 AI Orchestrator와 Omni AI Server 구현을 기준으로 기술한다.

## End-to-End Flow

```text
지원 Client AWAY/OFFLINE → ONLINE
→ Presence owner가 부재 구간과 USER_RETURNED 확정
→ Messenger Client Ingress가 AI Orchestrator REST API 호출
  - requestId / triggerId / absencePeriodId / trusted routingRef
→ AI Orchestrator
  - 입력·부재 기간·만료·정책·권한 평가
  - DETECTED lifecycle 저장, 중복이면 기존 snapshot 반환
→ Message Query Port → WS 메시지 조회 API
  - MSG_UNREAD 첫 페이지 조회(reqcount=30, reqpage=1)
  - source item key / subject / sequence와 selection 저장
   ├─ 0건: SKIPPED_NO_MESSAGES
   └─ N건: AWAITING_CONFIRMATION 및 확인 metadata를 REST 응답으로 반환
→ Client가 확인 팝업 생성: YES / NO
   ├─ NO 또는 timeout: DECLINED / EXPIRED
   └─ YES: correlation·만료·정책·권한 재검증
→ executionId와 PROCESSING 저장 후 비동기 실행 pool 등록
  - 저장된 selection 사용, unread 재조회 없음
→ Omni AI Server
  - execution과 immutable request를 MariaDB에 저장
  - process-local Queue 등록, 202 Accepted
  - Worker가 제목 기반 구조화 LLM 분석 1회 수행
  - terminal result와 callback outbox event를 함께 저장
→ Callback Worker → AI Orchestrator
   └─ callback 지연·유실: 같은 executionId로 status 조회
→ 공통 result handler가 terminal 결과를 조건부 반영
→ 결과 payload와 stable deliveryId 저장, DELIVERY_PENDING
→ ReturnedMessageResultSender → Result Router
→ Redis에서 요청 session과 동일 connectionType의 현재 owner 확인
→ owner instance REST endpoint → Client Topic Digest
→ Realtime 2xx이면 COMPLETED
→ Client가 로컬 삭제 상태를 반영해 최종 노출
```

AI Orchestrator는 Message Query Port로 WS 메시지 조회 API를 호출하며 쪽지 DB를 직접 조회하지 않는다. Messenger 내부 DB 조회 Queue와 조회 로직은 Message owner 책임이다. Orchestrator는 별도 confirmation prompt를 전송하지 않으며 Client가 REST 응답의 metadata로 팝업을 생성한다.

현재 Trigger 진입점과 결과 전달은 REST를 사용한다. NATS JetStream Trigger Consumer와 Core NATS 결과 전달은 이 흐름에 연결되어 있지 않다.

## Responsibility

| Component | Responsibility |
| --- | --- |
| Presence owner inside Messenger Client Ingress | 대상 connection 판단, 복귀 상태 전이와 부재 구간 확정, trusted routingRef 제공 |
| Message owner / WS 조회 API | unread 후보의 key와 제목 조회. 쪽지 저장소와 내부 조회 Queue 소유 |
| AI Orchestrator | 정책·권한 평가, lifecycle·selection 영속화, 사용자 확인, 실행 correlation, 비동기 start, callback/status 반영, 결과 전달과 복구 |
| Omni AI Server | 실행 접수·멱등성, Queue와 ACCEPTED 복구, 제목 분석·구조화 검증, terminal result·callback outbox 저장, callback 재전송과 status query |
| Result Router | Redis에서 요청 session의 현재 connection owner와 connectionType 확인, 해당 instance에 REST 전달 |
| Messenger Client Ingress | 결과를 선택된 local connection으로 전달 |
| Client | 확인 팝업과 동의 입력, 원본 이동과 로컬 삭제 상태를 반영한 최종 노출 |

## Contracts

### USER_RETURNED REST Request

Presence owner가 복귀를 확정한 뒤 생성하는 source-qualified 요청이다. 기본 API version은 `v1`이며 Orchestrator의 `api.version` 설정을 따른다.

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
  connectionType
```

별도 `eventType`은 없으며 owner 입력은 `routingRef.tenantId`, `routingRef.userId`에서 읽는다. `targetRef`는 현재 구현에서 `clientSessionId`로 해석한다. `connectionType`은 필수 양수이며 Messenger Client Ingress가 인증된 connection context에서 설정해야 한다.

성공 응답은 HTTP 200이며 `requestId`, `triggerId`, `outcome`, `reason`, `status`, `unreadCount`, `confirmationId`, `messageSelectionRef`, `confirmationExpiresAt`, `executionId`, `lifecycleEvent`를 포함한다. 해당하지 않는 값은 null일 수 있다. Client는 `AWAITING_CONFIRMATION`과 확인 metadata를 사용해 팝업을 만든다.

### Message Selection Result

```text
absencePeriodId
unreadCount
messageSelectionRef
selectedAt
selectionExpiresAt
```

현재 WS adapter는 `MSG_UNREAD`, `reqcount=30`, `reqpage=1`로 조회한다. **unreadCount는 선정·저장한 항목 수이며 전체 unread 쪽지 수가 아니다.** 조회 시작·종료 시각은 `offlineSince`와 `returnedAt`을 Asia/Seoul의 분 단위 형식으로 변환한다.

WS 응답의 key와 nullable subject를 받아 Orchestrator가 `returned_message_selection`과 `returned_message_selection_item`에 selection metadata, key, 제목, 순서를 저장한다. `messageSelectionRef`와 유효기간은 이 저장소에서 생성하며 동일 요청의 유효 selection은 재사용한다. 원시 unread sentinel과 쪽지 DB schema는 Message owner 내부 계약이다.

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
  connectionType
```

저장된 owner, 부재 구간, confirmation, selection, session과 connectionType이 일치해야 한다. 유효기간은 Client의 `decidedAt` 대신 DB 현재 시각으로 판정하며, `min(confirmation timeout, selection expiresAt)`을 사용한다. 기본 confirmation timeout은 2분, selection TTL은 5분이다.

`NO`는 `DECLINED`, 응답 만료는 `EXPIRED`로 종료한다. `YES`는 정책·권한을 다시 평가하고 저장된 selection 항목 수를 확인한 뒤 조건부로 `PROCESSING`에 진입한다. 이 전이에 성공한 처리자만 AI start를 등록한다. `YES` 결과가 `PROCESSING` 또는 `DELIVERY_PENDING`이면 HTTP 202, 그 외 성공 응답은 200이다.

### Returned Message AI Task

Orchestrator는 stable `executionId`를 저장한 뒤 `AiExecutionGateway`를 호출한다. 내부 command에는 request/trigger/execution 식별자, owner, 부재 구간, selection 항목과 routingRef가 포함된다. Omni AI wire contract에는 분석에 필요한 입력만 snake_case로 전달한다.

```text
POST /ai/v1/returned-message-topics/generate
X-Tenant-Id: <tenantId>

execution_id
tenant_id
task_type: RETURNED_MESSAGE_TOPICS
contract_version: 2
input
  request_id
  user_id
  absence_period_id
  message_selection_ref
  selected_items[]                 # 1~30건
    source_item_key
    subject                       # nullable
    sequence                      # 0 이상
```

`source_item_key`와 `sequence`는 각각 중복될 수 없다. 입력에 본문·발신자·수신 시각·첨부 파일명은 포함하지 않는다.

Omni AI의 성공 결과는 다음과 같다. Callback에서는 camelCase, start/status 응답에서는 snake_case를 사용하며 Orchestrator adapter가 같은 모델로 변환한다.

```text
result
  topics[]
    topic
    summary
    messages[]
      messageRef
      subject
      priority: LOW | NORMAL | HIGH | URGENT | UNKNOWN
      priorityReason
      replyRequired
      replyReason
      actionRequired
      actionReason
  generatedAt
```

`subject`는 LLM이 생성하지 않고 서버가 입력의 원래 제목을 붙인다. 내부 Orchestrator domain은 topic의 source reference 목록과 `messageAssessments`로 나누어 표현하지만 외부 v2 결과는 위 중첩 구조를 사용한다.

```text
ACCEPTED → RUNNING → SUCCEEDED | FAILED | CANCELLED

GET /ai/v1/returned-message-topics/executions/{executionId}
X-Tenant-Id: <tenantId>

POST /api/v1/returned-messages/{requestId}/execution-results
X-Tenant-Id: <tenantId>

eventId / tenantId / userId / executionId
taskType: RETURNED_MESSAGE_TOPICS
contractVersion: 2
status / result / failureReason
```

Start의 HTTP 202는 내구성 있는 접수이며 생성 성공을 뜻하지 않는다. 동일 실행이 이미 terminal이면 start는 HTTP 200과 기존 결과를 반환한다. Callback은 header/body tenant 일치와 task type/version을 확인하며, status 조회와 같은 result handler로 반영한다. Orchestrator의 REST validation·충돌·접근 거절·저장소 장애 응답은 성공 DTO와 분리된 RFC 9457 Problem Details를 사용한다.

## Message Selection and Read/Delete Semantics

복귀 구간에서 최초 조회 당시 unread였던 쪽지의 key·제목·순서를 고정한다. **YES 시 MSG_UNREAD를 다시 조회하지 않는다.** 그 사이 사용자가 쪽지를 읽어도 저장된 selection을 AI 입력으로 사용한다. TTL은 선택과 확인의 유효기간이며 DB row 자동 삭제를 의미하지 않는다.

Client 로컬 삭제 처리는 다음 경계 계약을 따른다. 서버는 Client 로컬 DB의 삭제 여부를 판정하지 않는다.

- Client는 `topics[].messages[].messageRef`를 로컬 DB와 비교해 삭제된 원본을 노출에서 제외한다.
- 남은 원본이 없는 Topic은 노출하지 않는다.
- 일부 원본만 삭제됐다면 남은 원본 기준으로 이동 대상과 표시 개수를 조정한다.
- Topic Digest 수신 자체는 원본 쪽지를 읽음 처리하지 않는다.

## Policy

| 정책 | 현재 동작 |
| --- | --- |
| Source contract | 양수 connectionType, `offlineSince < returnedAt <= expiresAt` 검증. Presence 상태 전이는 재판정하지 않음 |
| 부재·만료 | 기능 enabled, DB 현재 시각 기준 Trigger 유효성, 1분 이상·7일 이하 부재 확인 |
| 조회 범위 | maxLookback 24시간을 계산하지만 WS adapter에는 미반영 |
| Global / Feature / User | 최초 선정과 YES에서 순서대로 평가. YES에서 거절되면 BLOCKED_BY_POLICY |
| Permission | 선정/AI context 목적별 Port 호출. 별도 adapter가 없을 때 기본 구현은 ALLOWED |
| Unread / Confirmation | 유효 selection 1건 이상, confirmation correlation과 DB 만료 조건 확인 |
| Duplicate period | tenant·user·absencePeriodId 및 request/trigger 식별자 기반 중복·충돌 구분 |

정책·권한 조회 장애는 정상적인 정책 거절과 구분해 오류로 반환한다. 별도 cooldown은 현재 이 Use Case의 필수 정책으로 두지 않는다.

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

- lifecycle은 JDBC에 저장한다. primary key는 `(tenantId, requestId)`, unique key는 `(tenantId, triggerId)`와 `(tenantId, userId, absencePeriodId)`다. 정상 중복은 기존 snapshot을 반환하며 최초 조회를 다시 시작하지 않는다.
- 확인·실행·결과 반영은 현재 상태와 correlation을 포함한 SQL 조건부 update로 경쟁을 제어한다. 외부 HTTP 호출 중 DB transaction·row lock을 유지하지 않는다.
- Omni AI start는 `tenantId + executionId`와 immutable request fingerprint로 멱등 처리한다. 같은 ID의 다른 입력은 409로 거절한다.
- Callback의 `eventId`는 필수지만 Orchestrator가 별도 event 이력으로 중복 제거하지 않는다. terminal 결과는 `PROCESSING + executionId` 조건으로 한 번만 반영한다.
- 성공 결과와 stable `deliveryId`를 먼저 저장하고 전달한다. 전달 재시도는 같은 결과·deliveryId를 사용하며 LLM을 다시 실행하지 않는다.
- AI 생성 성공과 Client 전달 성공은 별도 상태다. Client 공개 상태는 `DELIVERY_PENDING`과 `COMPLETED`를 모두 `TOPICS_READY`로 표현하며, 공개 상태가 같으면 lifecycle event ID와 revision을 유지한다.

## AI Workflow

### Execution Shape

Omni AI Server는 FastAPI 실행 API와 단일 노드 LangGraph를 사용한다.

```text
START
→ analyze_message
  - sequence 순서로 입력 정렬, subject 공백 정규화
  - 제목 묶음 전체에 대해 JSON Schema 기반 LLM 호출 1회
  - Pydantic schema와 source reference 검증
→ END
→ 입력의 원래 subject와 generatedAt을 붙여 결과 생성
```

모든 입력 메시지는 `topics[].messages[]`에 정확히 한 번 포함되어야 한다. 중복·누락·알 수 없는 reference, 필수 필드 또는 enum 오류는 실행 실패로 처리한다. 현재 실패는 `AI_GENERATION_FAILED`, `retryable=false`로 저장하며 자동 LLM 재시도는 수행하지 않는다.

### Boundary

- Spring AI Orchestrator는 선정·사용자 확인·정책 평가, 실행 요청과 correlation, callback/status reconciliation, 결과 저장과 Client routing을 책임진다.
- Omni AI Server는 durable 실행 접수, Queue/worker, AI 분석·검증, terminal result와 callback outbox를 책임진다.
- Omni AI Server는 메시지 DB 조회, 사용자 확인, Client connection 선택을 수행하지 않는다. 분석 결과는 Orchestrator로 반환한다.

### Extension Rule

현재 Graph는 `analyze_message` 하나다. 추가 전처리, Tool, review 또는 조건부 재처리가 실제 요구사항이 될 때 기존 입력·출력 계약을 기준으로 확장한다.

### Context Scope

- 현재 분석 입력: `sourceItemKey`, nullable `subject`, `sequence`.
- 실행 correlation: request/user/absence/selection 식별자.
- 현재 미사용: 본문, 발신자, 수신 시각, 첨부 파일명과 파일 내용.

제목 외 Context 확장은 현재 구현 범위에 포함하지 않는다.

### Quality Focus

- 제목에 근거한 업무 주제별 그룹화와 메시지별 독립 판정을 수행한다.
- 원문에 없는 업무·마감·긴급도·답변 또는 후속 조치를 만들지 않는다.
- 정보가 부족하면 우선순위는 `UNKNOWN`, 근거 없는 reply/action 판단은 false를 사용한다.
- 같은 고객명·제품명만으로 묶지 않으며 같은 구체적 업무라는 근거가 부족하면 Topic을 분리한다.
- 요약과 판정 근거는 한국어로 생성하며 각 reason은 50자 이하다. 원본 key로 추적 가능성을 유지한다.

## Result Routing

`RealtimeReturnedMessageResultSender`가 공통 `AiResultRouter`에 연결되어 있으며, 결과 event type은 `RETURNED_MESSAGE_TOPICS_COMPLETED`다.

- Redis의 현재 connection pointer와 owner record를 읽고 pointer를 재확인한다. 저장된 user/session 및 connectionType이 일치하는 owner instance에 REST로 전달한다.
- Trigger source instance를 최종 target으로 고정하지 않는다. 요청 session에 연결된 현재 connection을 조회하며 다른 session이나 모든 사용자 연결에 broadcast하지 않는다.
- Realtime HTTP 2xx를 전달 완료로 취급한다. 이는 Client 표시 완료 acknowledgement를 뜻하지 않는다.
- stable deliveryId를 사용하지만 전달 worker와 recovery가 겹칠 수 있으므로 외부 중복 억제 없이 exactly-once를 보장하지 않는다.

Client payload는 `topics`, `counts`, `generatedAt`이다. Topic 안의 메시지에는 `messageRef`, `subject`, `priority`, `replyRequired`, `actionRequired`를 포함하고 내부 reason 필드는 전달하지 않는다.

`counts`는 전달 메시지를 기준으로 Orchestrator가 계산한다. `total`은 전체 수, `urgent`는 URGENT 수, `replyRequired`와 `actionRequired`는 각각 true인 수, `other`는 UNKNOWN 수다. 각 항목은 독립 집계하므로 중복될 수 있다.

## Failure and Deferred Execution

- 메시지 조회 오류·잘못된 selection은 `FAILED`로 기록한다. 사용자가 거절하거나 확인이 만료되면 AI를 호출하지 않는다.
- Omni start의 400/422는 입력 오류, 409는 실행 충돌로 실패 처리한다. timeout·연결 실패·기타 불확실한 오류는 `PROCESSING`을 유지한다.
- Callback은 terminal result와 같은 transaction으로 outbox에 저장한다. 전송 실패 시 같은 eventId로 지수 backoff 재전송하며 최대 시도 후 `DEAD_LETTER`로 남긴다. callback worker는 callback base URL이 설정된 경우 동작한다.
- Orchestrator recovery는 기본 30초 간격으로 claim/lease를 사용해 비종료 lifecycle을 조회한다. `DETECTED`는 선정 재개, `AWAITING_CONFIRMATION`은 대기, `PROCESSING`은 status 조회, `DELIVERY_PENDING`은 저장 결과 전달만 재시도한다.
- Status 404/NOT_FOUND는 `PROCESSING`을 유지하고 **start를 자동 재호출하지 않는다.** 실행 pool 등록 실패나 start 전 process 종료로 Omni에 접수되지 않은 요청도 이 상태에 남을 수 있다. 현재 reconciliation deadline과 backoff/jitter metadata는 없다.
- 확인 만료는 sweeper 실행 여부와 관계없이 DB 조건으로 결정한다. sweeper는 기본 30초 간격으로 만료 상태를 반영한다.
- Omni의 ACCEPTED는 durable 저장소에서 다시 Queue에 올린다. lease가 만료된 RUNNING은 내부 `LEASE_EXPIRED`로 격리하며 자동 재실행하지 않는다.
- 결과 전달의 retryable 실패는 기본 최초 시도 후 1회 추가 재시도하고 소진 시 `DELIVERY_FAILED`로 기록한다. 일반 RuntimeException 경로는 pending을 유지하며 retry count를 소비하지 않을 수 있다.

## Scope

- Presence owner가 확정한 복귀 요청과 trusted routingRef 수신
- 부재 기간·만료·기능/사용자 정책과 확인 lifecycle
- WS API를 통한 최대 30건 unread key·제목 선정과 저장
- 사용자 동의 후 제목 기반 주제·요약·메시지별 판정
- durable AI 실행, callback/status reconciliation
- 요청 session·connectionType 기반 결과 전달과 재시도
- Client 로컬 삭제 상태를 반영하는 노출 계약

## Non-scope

- Presence 상태 전이의 Orchestrator 내부 재판정
- 지원 대상 외 connection과 모든 사용자 연결 broadcast
- 쪽지 본문·첨부 파일 또는 전체 대화 요약
- 사용자가 정의한 반복 Schedule, Tool Calling
- 현재 Messenger 서비스의 물리적 분리·통합

## Acceptance Criteria

- Presence owner가 복귀와 대상 connection을 확정하며 Orchestrator는 source contract와 기능 정책을 검증한다.
- 부재 기간·Trigger 만료·정책상 비대상 요청은 lifecycle을 생성하지 않는다.
- 대상 메시지가 없으면 팝업과 LLM 호출을 생략하며, Client는 `AWAITING_CONFIRMATION` 응답으로 팝업을 만든다.
- 유효한 YES 이전에는 LLM을 호출하지 않으며 YES에서도 저장된 selection을 사용한다.
- 동일 복귀 구간·확인 응답·execution start와 경쟁하는 terminal 결과가 중복 실행·상태 전이를 만들지 않는다.
- AI 결과의 모든 입력 reference가 누락·중복 없이 한 번씩 포함된다.
- Callback 유실 시 status 조회로 결과를 복구하며 404가 start 재호출로 이어지지 않는다.
- 전달 실패는 저장된 결과만 재전송하고 LLM을 재실행하지 않는다.
- 결과는 요청 session과 동일 connectionType의 현재 owner로 전달되며 Realtime 2xx와 Client 표시 완료를 구분한다.
- Client는 원본 key를 이용해 로컬 삭제 상태를 반영하고 원본 쪽지로 이동할 수 있어야 한다.

## Presence Trigger Verification Contract

Messenger Client Ingress의 Presence owner가 보장할 경계 계약이다. 이번 AI Orchestrator·Omni AI 구현 분석으로 Presence 구현 완료 여부를 판정하지 않는다.

| 구분 | 기대 계약 |
| --- | --- |
| 적용 대상 | 인증된 connectType 1, 2, 3에서만 이 Use Case의 복귀 요청 생성 |
| 상태 전이 | AWAY/OFFLINE → ONLINE에서 기존 부재 구간을 종료하고 복귀 확정 |
| 반복·중복 | 반복 부재 상태에서 최초 offlineSince/absencePeriodId 유지, 중복 ONLINE으로 Trigger 중복 생성 방지 |
| disconnect / reconnect | current connection 기준으로 부재를 관리하고 stale disconnect는 제외 |
| routing | 복귀한 인증 session을 targetRef로 제공하고 connectionType 일치 유지 |
| 보존·재전송 | 부재 정보는 maxAbsenceDuration보다 길게 보존하고 REST 재전송 시 동일 correlation 유지 |

부재 구간의 식별 단위는 `tenantId + userId + connectionType`이며 `clientSessionId`는 현재 connection 조회와 routing에 사용한다.

### Verification Ownership

| 검증 범위 | 소유 Component |
| --- | --- |
| 상태·session·connectionType, disconnect/reconnect, 부재 구간, Trigger 재전송 | Messenger Client Ingress / Presence owner |
| source contract, 정책, 선정·확인, 멱등성, callback/status, 결과 routing | AI Orchestrator |
| durable 접수, 분석 schema/reference, Queue 복구, callback outbox | Omni AI Server |
| 실제 Client 복귀부터 최종 표시·원본 이동까지 | Cross-component E2E |

## System Integration Verification

실제 서비스 경계를 거쳐 복귀 요청 → 선정 → Client 확인 → AI 실행 → callback/status → 결과 전달이 연결되는지 검증한다. `triggerId`, `absencePeriodId`, `executionId`, `deliveryId`의 correlation을 추적하고, 개별 구현 존재를 전체 E2E 검증 완료로 간주하지 않는다. 이 문서는 코드 분석 기준이며 실제 통합 실행 결과를 기록하지 않는다.

## Current Implementation Gap

AI Orchestrator의 JDBC lifecycle/selection, 비동기 start, callback과 status reconciliation, Redis 기반 Result Router 및 실제 Returned Message 전달 adapter는 구현되어 있다. Omni AI Server의 MariaDB 실행 저장소, process-local Queue와 ACCEPTED 복구, 단일 노드 분석, 결과·callback outbox와 재전송도 구현되어 있다.

현재 핵심 제약은 다음과 같다.

- `maxLookback`의 WS 조회 미반영, 첫 페이지 최대 30건 조회.
- 권한 Port의 기본 ALLOWED 구현과 callback의 tenant 대조를 넘어선 발신 서비스 인증 보완.
- Orchestrator에서 Omni 접수 전 실행 유실 시 자동 복구 부재, reconciliation deadline 미구현.
- Omni RUNNING lease heartbeat, 격리 실행·dead-letter 운영 도구, retention/cleanup 보완.
- Orchestrator의 결과 reference와 최초 selection 간 전체 대조 미구현. Omni 분석 단계에서는 reference 완전성을 검증한다.
- NATS 기반 Trigger/결과 전달과 실제 Presence·Client를 포함한 전체 E2E 확인은 별도 범위.

## Related Documents

- [AI Orchestrator Returned Message 구현 문서](../../../../ai-orchestrator/docs/features/returned-message.md)
- [AI Orchestrator Returned Message 소스](../../../../ai-orchestrator/src/main/java/com/ucware/ai_orchestrator/returnedmessage)
- [Omni AI Returned Message Topics 소스](../../../omni-ai-platform/app/features/returned_message_topics)
- [Roadmap](../roadmap.md)
- [Server Trigger](../components/ai-orchestrator/server-trigger.md)
- [Server-driven AI](../concepts/server-driven-ai.md)
- [Session Registry Based AI Result Routing](../decisions/001-session-registry-result-routing.md)
- [Durable Asynchronous AI Execution and Result Reconciliation](../decisions/008-durable-asynchronous-ai-execution-and-result-reconciliation.md)
- [Messenger Client Ingress Result Delivery](../components/messenger-client-ingress/result-delivery.md)
