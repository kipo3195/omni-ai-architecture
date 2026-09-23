# Roadmap

> Role: 사용자에게 전달되는 End-to-End AI 기능을 기준으로 구현 순서와 진행 상태를 관리한다.
> Status: In Progress
> 실제 흐름과 완료 조건은 [Use Cases](implementation/use-cases/README.md), 서비스 내부 구조는 [Implementation](implementation/README.md)에 기록한다.

---

## Roadmap Principle

각 Phase는 공통 기술 컴포넌트의 완성이 아니라 하나의 사용자 기능이 입력부터 결과 전달까지 동작하는 것을 목표로 한다. 공통 Runtime과 Application 구조는 Use Case를 구현하면서 필요한 범위부터 만들고, 재사용이 확인된 내용을 서비스별 구현 문서로 승격한다.

Phase 상태는 해당 Use Case의 완료 여부를 나타낸다. 이후 새로운 Workflow나 Tool이 추가되더라도 이미 완료된 Phase의 상태를 되돌리지 않는다.

---

## Phase 1. Conversation Start End-to-End

Status: In Progress

사용자가 방에 입장했을 때 AI Orchestrator가 Conversation Start 실행 여부를 판단하고, Omni AI Server가 대화 시작 데이터를 생성해 현재 Client에 전달한다.

```text
ROOM_ENTERED / ROOM_LEFT
→ Messenger Business Logic
→ AI Orchestrator
→ Omni AI Server: TRENDING Conversation Start 생성
→ Realtime Service
→ Client
```

주요 범위:

- Enter Room 요청과 실행 예약
- Leave Room 발생 시 예약 취소 또는 오래된 결과 폐기
- Conversation Start 실행 판단과 최소 Context 구성
- Omni AI Server의 TRENDING Workflow 실행
- Structured Result를 현재 room session의 Client에 전달
- 재입장과 중복 요청에 대한 execution correlation

완료 기준:

- Enter Room부터 Client 결과 전달까지 하나의 경로가 동작한다.
- Leave Room 이후 취소된 실행이나 오래된 결과가 전달되지 않는다.
- Business Rule과 AI Runtime 책임이 분리되어 있다.

상세: [Conversation Start](implementation/use-cases/01-conversation-start/README.md)

---

## Phase 2. User Returned Message Topic Digest

Status: Planned

사용자의 상태가 자리비움에서 온라인으로 변경되면 자리비움 동안 수신한 쪽지를 조회하고, Omni AI Server가 발신자와 업무 주제 중심의 Topic Digest를 생성해 전달한다.

```text
AWAY → ONLINE
→ AI Orchestrator
→ 자리비움 구간의 수신 쪽지 조회
→ Omni AI Server: Topic Extraction
→ Topic Digest 알림
→ Client
```

주요 범위:

- 상태 변경 이벤트와 자리비움 구간 식별
- 대상 쪽지가 없을 때 실행하지 않는 정책
- 발신자별 또는 대화별 Topic Extraction
- 동일 복귀 구간에 대한 중복 실행 방지
- 원본 쪽지로 이동할 수 있는 결과 계약

완료 기준:

- 동일 복귀 이벤트는 한 번만 처리된다.
- 쪽지가 없는 경우 LLM을 호출하지 않는다.
- 생성된 Topic Digest가 Client에 전달되고 원본 메시지와 연결된다.

상세: [Returned Message Topics](implementation/use-cases/02-returned-message-topics/README.md)

---

## Phase 3. Client LLM 기반 Scheduled Weekly Report Summary

Status: Planned

Client LLM이 사용자의 자연어 요청을 `ScheduleSpec`으로 변환하고, 서버가 이를 검증·등록하여 정해진 시각에 주간보고 쪽지를 요약해 전달한다.

```text
사용자 자연어 요청
→ Client LLM: ScheduleSpec 생성
→ AI Orchestrator: 검증 및 Schedule 등록
→ Scheduler Trigger
→ 온라인 상태와 대상 메시지 확인
→ Omni AI Server: Weekly Report Summary
→ 별도 쪽지로 결과 전달
```

구현 순서:

1. `3-A`: Client LLM → ScheduleSpec → 서버 검증 및 등록
2. `3-B`: Scheduler Trigger → 메시지 조회 → 요약 → 쪽지 전달
3. `3-C`: 오프라인 실행 보류 → 온라인 복귀 → 사용자 확인 → 지연 실행 또는 skip

완료 기준:

- Client가 생성한 ScheduleSpec을 서버가 신뢰하지 않고 schema, permission, timezone을 검증한다.
- 온라인 상태에서는 정해진 회차가 중복 없이 실행된다.
- 오프라인 상태에서는 AI를 실행하지 않고 회차를 보류한다.
- 복귀 후 사용자 확인 결과에 따라 보류 회차를 실행하거나 종료한다.

상세: [Scheduled Weekly Report](implementation/use-cases/03-scheduled-weekly-report/README.md)

---

## Phase 4. Server LLM 기반 Schedule Intent Parsing

Status: Planned

Phase 3의 Schedule 실행 구조를 유지하면서, 자연어를 ScheduleSpec으로 변환하는 역할을 Client LLM에서 Omni AI Server로 옮긴다.

```text
사용자 자연어 요청
→ AI Orchestrator
→ Omni AI Server: Schedule Intent Parsing
→ AI Orchestrator: 검증 및 사용자 확인
→ Scheduler 등록
```

주요 범위:

- Phase 3과 동일한 ScheduleSpec 계약 재사용
- 자연어 요청을 구조화된 Schedule Intent로 변환
- 불명확하거나 지원하지 않는 요청의 오류 계약
- Orchestrator의 schema, permission, policy 검증
- 사용자 확인 이후 Schedule 등록

완료 기준:

- Client LLM 없이 동일한 ScheduleSpec을 생성할 수 있다.
- LLM 결과가 직접 Scheduler를 변경하지 않는다.
- Phase 3의 실행·보류·전달 구조를 수정하지 않고 재사용한다.

상세: [Server Schedule Intent](implementation/use-cases/04-server-schedule-intent/README.md)

---

## Phase 5. Tool Calling 기반 AI 기능

Status: Planned

LLM이 실행 중 필요한 Tool을 선택하는 구체적인 AI 기능을 구현하고, 그 과정에서 공통 Tool Calling Runtime을 검증한다.

첫 적용 후보:

```text
"지난주 주간보고에서 결재가 필요한 내용을 찾아서
결재 요청 쪽지 초안을 만들어 줘"

→ searchMessages
→ getUserInfo
→ LLM 결과 생성
→ createMessageDraft
→ 사용자 확인
```

구현 순서:

1. `5-A`: 조회 중심 Server Tool Calling
2. `5-B`: Client Tool Delivery
3. `5-C`: Server Tool과 Client Tool을 함께 사용하는 복합 실행

완료 기준:

- LLM이 요청에 따라 Tool 필요 여부와 종류를 선택한다.
- Tool Request와 Result가 `executionId`, `toolCallId`로 연결된다.
- Tool 결과 수신 후 중단된 AI 실행이 재개된다.
- 데이터 변경이나 외부 전송은 사용자 확인을 거친다.

상세: [Tool-assisted AI](implementation/use-cases/05-tool-assisted-ai/README.md)

---

## Later Work

다음 항목은 위 Use Case에서 필요성이 확인된 이후 별도 Phase로 승격한다.

- Stateful Chatbot / Agent Runtime 확장
- 여러 차례 Tool Call과 장기 실행
- AI Context Projection / Cache
- Observability / Evaluation / Execution Budget
- WebSocket Service responsibility split

---

## Status Summary

| Phase | User Outcome | Status |
| --- | --- | --- |
| Phase 1 | 방 입장 시 Conversation Start 추천 수신 | In Progress |
| Phase 2 | 복귀 시 자리비움 동안의 쪽지 Topic Digest 수신 | Planned |
| Phase 3 | Client LLM으로 등록한 주간보고 요약을 예약 수신 | Planned |
| Phase 4 | Server LLM으로 자연어 Schedule 요청 처리 | Planned |
| Phase 5 | Tool Calling 기반 검색·판단·초안 생성 기능 | Planned |
