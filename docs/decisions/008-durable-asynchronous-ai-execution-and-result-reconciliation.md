# 008. Durable Asynchronous AI Execution and Result Reconciliation

Status: Proposed

## Context

AI Orchestrator는 사용자 확인과 실행 정책을 통과한 작업에 stable `executionId`를
부여하고 Omni AI Server에 실제 Workflow와 LLM 실행을 위임한다. LLM 실행 시간은
일반적인 서비스 간 HTTP timeout보다 길거나 편차가 클 수 있다. 실행이 끝날 때까지
동기 HTTP 연결을 유지하면 다음 불확실성이 생긴다.

```text
Omni AI Server가 실행을 시작하거나 완료함
→ 응답을 보내기 전 또는 보내는 중 HTTP timeout
→ AI Orchestrator는 실행 존재 여부와 결과를 알 수 없음
```

Timeout을 실패로 간주해 새 실행을 만들면 동일 LLM 작업이 중복 실행될 수 있다.
반대로 재시도하지 않으면 Orchestrator lifecycle이 영구적으로 처리 중에 남을 수 있다.
Callback만 사용하면 callback 전송 실패와 Orchestrator 재시작을 복구할 수 없고,
polling만 사용하면 정상 완료 전달 지연과 불필요한 조회 부하가 커진다.

두 서비스 사이에 distributed transaction을 도입하지 않으면서 실행 접수, 결과 전달과
복구가 각각 process 재시작과 network failure를 견딜 수 있는 공통 실행 계약이 필요하다.

## Decision

장시간 AI 작업은 stable `executionId`를 사용하는 durable asynchronous execution으로
처리한다. Callback을 정상 결과 전달 경로로 사용하고 execution status query를 최종
reconciliation 경로로 사용한다.

### Execution identity and ownership

- AI Orchestrator가 외부 호출 전에 `executionId`를 생성하고 feature lifecycle과 함께
  영속화한다.
- 최초 start, 중복 start, callback, status query와 결과 반영은 모두 같은
  `tenantId + executionId`를 사용한다.
- Omni AI Server는 실제 Workflow 실행 상태와 terminal result의 authoritative owner다.
- AI Orchestrator는 feature lifecycle, 실행 correlation, 마지막으로 관찰한 외부 상태와
  Client delivery 상태를 소유한다.

### Durable asynchronous start

Omni AI Server의 start command는 다음 순서를 따른다.

```text
start request validation
→ execution record와 immutable request fingerprint 저장
→ durable execution queue 등록
→ HTTP 202 Accepted
```

`202 Accepted`는 AI 결과 성공을 의미하지 않는다. 실행이 내구성 있게 접수되어 이후
Worker 또는 recovery process가 처리할 수 있다는 뜻이다. Execution record와 queue
등록 사이의 crash window는 같은 local transaction, transactional queue 또는 동등한
내구성 보장으로 제거한다.

Start command는 `tenantId + executionId`에 대해 멱등하다.

- 처음 보는 ID와 유효한 요청은 실행을 생성한다.
- 같은 ID와 같은 immutable request는 기존 실행 상태를 반환하며 LLM을 다시 실행하지 않는다.
- 같은 ID에 다른 immutable request가 오면 conflict로 거절한다.
- Transport retry를 이유로 새 `executionId`를 만들지 않는다.

Queue 제품과 Worker 배치 방식은 이 ADR에서 결정하지 않는다.

### Execution lifecycle

Omni AI Server의 외부 관찰 상태는 최소한 다음을 지원한다.

```text
ACCEPTED → RUNNING → SUCCEEDED
                   | FAILED
                   | CANCELLED
```

Workflow 내부 node와 agent state는 이 공통 상태에 노출하지 않는다. Terminal 상태와 결과는
retention 기간 동안 `tenantId + executionId`로 다시 조회할 수 있어야 한다.

### Terminal result and callback outbox

Workflow가 terminal 상태에 도달하면 Omni AI Server는 하나의 local transaction에서
다음을 저장한다.

```text
execution terminal status와 result 또는 error
+ callback outbox event(PENDING)
```

Callback은 결과를 저장한 뒤 전송한다. Callback 성공 이후에 처음 결과를 저장하는 순서는
허용하지 않는다. 별도 callback delivery worker가 PENDING event를 전송하고 Orchestrator의
2xx acknowledgement를 받은 뒤 DELIVERED로 변경한다.

Timeout, connection failure와 retryable 5xx에서는 같은 `eventId`와 `executionId`로
backoff 후 재전송한다. 재전송 과정에서 callback은 중복 도착할 수 있으므로 전달 보장은
at-least-once이고 AI Orchestrator는 callback을 멱등 처리한다. 영구 실패와 retry budget
소진 event는 운영 확인이 가능한 terminal delivery 상태로 이동한다.

### Callback acknowledgement

AI Orchestrator는 callback의 tenant, feature request와 `executionId` correlation을 검증하고
결과를 자신의 durable lifecycle에 반영한 뒤에만 2xx를 반환한다. 처리 중 예외가 발생하거나
결과를 영속화하지 못하면 성공 acknowledgement를 반환하지 않는다.

Callback과 status query 결과는 같은 application result handling 경로로 수렴한다. 다음과
같은 conditional update를 사용해 callback 중복 및 callback/polling 경쟁을 처리한다.

```text
current feature status = PROCESSING
and stored executionId = received executionId
```

한 경로가 먼저 terminal 결과를 반영하면 다른 경로는 기존 결과를 유지한다.

### Status query and reconciliation

Omni AI Server는 `tenantId + executionId`로 현재 상태와 terminal result를 조회하는 API를
제공한다. AI Orchestrator는 callback이 정해진 시간 안에 도착하지 않았거나 dispatch 결과가
불확실한 PROCESSING 실행을 durable reconciliation 대상으로 관리한다.

```text
ACCEPTED / RUNNING
→ backoff와 jitter를 적용해 다음 조회 예약

SUCCEEDED / FAILED / CANCELLED
→ callback과 같은 결과 처리 경로로 반영

timeout / retryable server failure
→ terminal로 추정하지 않고 다음 조회 예약

404
→ visibility grace 동안 재조회
→ 한 번의 404로 start를 재호출하지 않음
```

Reconciliation deadline을 넘긴 실행은 운영상 unresolved 상태로 분류할 수 있지만, deadline
초과만으로 Omni AI 실행이 없었다고 단정하거나 새 LLM 실행을 자동 허용하지 않는다.

### Separation from Client result delivery

Omni AI execution completion과 Messenger Client result delivery는 별도 상태다.

```text
Omni AI execution SUCCEEDED
→ Orchestrator result 저장
→ Client delivery pending
→ delivered 또는 delivery failed
```

Client delivery 실패는 Omni AI start 재호출로 이어지지 않는다. Client delivery의 outbox,
acknowledgement와 retry는 Result Routing 계약에서 별도로 정의한다.

### Transaction boundaries

- 서비스 간 distributed transaction을 사용하지 않는다.
- 외부 HTTP 호출 중 local DB transaction이나 row lock을 유지하지 않는다.
- Omni AI Server는 execution/result와 callback outbox를 자신의 local transaction으로 관리한다.
- AI Orchestrator는 feature state claim과 결과 반영을 각각 짧은 conditional transaction으로 처리한다.
- Crash window는 stable identifier, idempotent command, outbox와 reconciliation으로 복구한다.

## Alternatives

### Keep the synchronous HTTP response open until LLM completion

구현은 단순하지만 실행 시간 편차에 맞춰 긴 timeout과 connection 자원을 유지해야 한다.
응답 전송 구간의 timeout이 실제 실행 결과를 알려주지 못하며 process 재시작 후 결과를
복구할 authoritative query가 없다.

### Callback only

정상 완료 latency는 낮지만 callback 전송이 장기간 실패하거나 Orchestrator가 결과 반영 전에
재시작한 경우 수신자가 실행 상태를 능동적으로 복구할 수 없다.

### Polling only

구현 경계는 단순하지만 모든 실행에 반복 조회가 필요하고 정상 결과 전달 latency와 조회
부하가 증가한다.

### Retry start with a new execution identifier

Network uncertainty가 새로운 LLM 실행으로 바뀌므로 동일 business operation의 중복 실행을
막을 수 없다.

## Consequences

### Positive

- 긴 LLM 실행 시간을 HTTP request timeout과 분리할 수 있다.
- Timeout과 process restart 이후에도 동일 execution을 조회하고 복구할 수 있다.
- Callback으로 정상 완료를 빠르게 전달하면서 polling으로 유실을 보완할 수 있다.
- Outbox로 terminal result와 callback delivery 사이의 유실 구간을 제거할 수 있다.
- 중복 start와 callback이 중복 LLM 실행 또는 중복 결과 반영으로 이어지지 않는다.
- AI 실행 재시도와 Client delivery 재시도를 분리할 수 있다.

### Negative

- Omni AI Server에 durable execution repository, queue worker, status API와 callback outbox가 필요하다.
- AI Orchestrator에 callback endpoint와 durable reconciliation scheduler가 필요하다.
- Callback, polling과 늦은 응답이 경쟁하므로 모든 결과 반영이 멱등해야 한다.
- Execution과 outbox retention, retry budget 및 unresolved 운영 절차가 필요하다.
- 두 서비스에서 execution 상태와 delivery 상태를 관찰할 운영 지표가 늘어난다.

## Deferred Decisions

- Execution queue 기술과 Worker concurrency
- Callback transport가 HTTP인지 durable messaging인지 여부
- Callback retry 횟수, backoff, jitter와 dead-letter 운영값
- Reconciliation 주기, visibility grace와 deadline 기본값
- Execution/result/outbox retention 기간
- Cancellation을 지원하는 Task 범위와 cancel command schema

## Related

- [Common AI Task Contract](007-ai-task-contract.md)
- [AI Execution and Context Identifier Separation](004-ai-execution-and-context-identifiers.md)
- [Returned Message Topics](../use-cases/02-returned-message-topics.md)
- [Omni AI Server Task Execution](../components/omni-ai-server/task-execution.md)
