# 05. Tool-assisted AI

Status: Planned

---

## User Outcome

사용자가 자연어로 업무를 요청하면 AI가 필요한 정보를 Tool로 조회하고, 결과를 바탕으로 쪽지 초안과 같은 실행 가능한 결과를 만든다.

첫 적용 후보는 지난주 주간보고에서 결재가 필요한 항목을 찾고 결재 요청 쪽지 초안을 생성하는 기능이다.

## Trigger

- Client가 Tool 사용이 가능한 AI 요청을 보낸다.

## End-to-End Flow

```text
사용자 요청
→ AI Orchestrator → Omni AI Server
→ LLM Tool Decision
→ AI Orchestrator Tool Runtime
→ Server 또는 Client Tool
→ Tool Result
→ Omni AI Server resume
→ 결과 초안
→ 사용자 확인
```

## Responsibility

| Component | Responsibility |
| --- | --- |
| AI Orchestrator | Tool registry, 권한, lifecycle, dispatch, 결과 정규화 |
| Omni AI Server | Tool 선택, argument 구성, 대기, 결과 기반 실행 재개 |
| Server Tool Adapter | Messenger Service API 호출 |
| Client Tool Delivery | 현재 Client session으로 Tool 요청·결과 전달 |
| Client | 결과 확인과 side effect 승인 |

## Contracts

최소 correlation은 `executionId`, `toolCallId`, `toolAttempt`, `idempotencyKey`다. Tool schema와 normalized result 계약을 Tool Registry에서 관리한다.

## State and Idempotency

AI 실행은 Tool 요청 시 `WAITING_TOOL_RESULT`로 전환한다. 동일 `toolCallId + toolAttempt` 결과는 한 번만 반영한다.

## Failure and Deferred Execution

Tool timeout, retry, cancellation과 Client disconnect를 구분한다. 데이터 변경이나 외부 전송은 사용자 확인 없이는 실행하지 않는다.

## Scope

1. `5-A`: 조회 중심 Server Tool
2. `5-B`: Client Tool Delivery
3. `5-C`: Server/Client 복합 Tool 실행

## Non-scope

- 모든 Messenger 기능의 Tool 등록
- 무제한 autonomous execution
- 사용자 확인 없는 side effect

## Acceptance Criteria

- LLM이 요청에 따라 Tool 사용 여부와 종류를 결정한다.
- Tool Result가 원래 execution에 정확히 한 번 반영된다.
- Tool 결과 이후 AI 실행이 재개되어 사용자 결과를 만든다.
- side effect는 명시적인 사용자 확인 후 수행된다.

## Current Implementation Gap

Tool Runtime, Tool schema, execution suspend/resume과 Server/Client Tool Adapter가 아직 구현되지 않았다.

## Open Questions

- Phase 5-A에서 사용할 첫 Server Tool과 read-only 범위
- 사용자 확인 token의 유효기간과 재사용 방지 방식

## Related Documents

- [Roadmap](../../../roadmap.md)
- [Tool Runtime](../../ai-orchestrator/tool-runtime/README.md)
- [Tool Calling](../../omni-ai-server/tool-calling/README.md)
- [ADR 002](../../../decisions/002-tool-runtime-ownership-client-tool-dispatch.md)
