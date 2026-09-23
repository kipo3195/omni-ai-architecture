# 02. Returned Message Topics

Status: Planned

---

## User Outcome

사용자가 자리비움에서 온라인으로 복귀하면 자리비움 동안 수신한 쪽지에서 확인해야 할 업무 주제를 발신자와 함께 짧은 목록으로 받는다.

## Trigger

- 사용자 상태가 `AWAY → ONLINE`으로 변경된다.
- 자리비움 구간에 수신 쪽지가 없으면 실행하지 않는다.

## End-to-End Flow

```text
AWAY → ONLINE
→ AI Orchestrator: 구간·정책·중복 확인
→ Message Service: 수신 쪽지 조회
→ Omni AI Server: Topic Extraction
→ Realtime Service
→ Client Topic Digest
```

## Responsibility

| Component | Responsibility |
| --- | --- |
| Presence owner | 상태 변경과 자리비움 구간 제공 |
| Message owner | 해당 구간의 수신 쪽지 조회 |
| AI Orchestrator | 실행 판단, Context Assembly, 중복 방지 |
| Omni AI Server | 발신자·업무 주제 중심 Topic 추출 |
| Realtime Service | 알림과 원본 쪽지 이동 정보 전달 |

## Contracts

입력에는 사용자, 자리비움 구간, 대상 메시지와 원본 식별자가 필요하다. 결과는 `senderName`, `topic`, `messageCount`, 원본 이동 식별자를 포함한다.

## State and Idempotency

동일 자리비움 구간은 `userId + absencePeriodId`를 기준으로 한 번만 처리한다.

## Failure and Deferred Execution

메시지 조회 실패와 AI 생성 실패를 구분한다. 실패한 복귀 이벤트의 retry와 expiration 정책은 구현 전에 확정한다.

## Scope

- 자리비움 구간 메시지 조회
- 발신자/업무 주제 추출
- Topic Digest 전달

## Non-scope

- 전체 대화 요약
- 사용자가 정의한 반복 Schedule
- Tool Calling

## Acceptance Criteria

- 대상 메시지가 없으면 LLM을 호출하지 않는다.
- 동일 복귀 구간이 중복 처리되지 않는다.
- Topic Digest에서 원본 메시지로 이동할 수 있다.

## Current Implementation Gap

상태 변경 Trigger, Cross-domain 메시지 조회, Topic Extraction Workflow와 결과 전달이 아직 구현되지 않았다.

## Open Questions

- 발신자별, 대화방별 또는 업무 주제별 그룹화 기준
- 자리비움 구간과 Trigger expiration의 Source of Truth

## Related Documents

- [Roadmap](../../../roadmap.md)
- [Server Trigger](../../ai-orchestrator/server-trigger/README.md)
- [Returned Message Topics Workflow](../../omni-ai-server/workflows/returned-message-topics.md)

