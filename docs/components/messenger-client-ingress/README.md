# Messenger Client Ingress Implementation

Status: In Progress

---

## Implementation Areas

| Area | 기록 범위 |
| --- | --- |
| [Result Delivery](result-delivery.md) | Connection Registry, reconnect fencing, REST target resolution, local Client delivery |
| [User AI Settings](user-ai-settings.md) | 사용자 AI 설정 API, DB 저장, Redis cache 갱신 |

---

## Scope

`Messenger Client Ingress`는 `WebSocket Service (WS)`와 `TCP Service (NS)`를 포괄하는 논리적 경계다. 각 물리 서비스는 자신의 connection을 소유하고 Client protocol ingress와 delivery를 담당한다.

주요 책임:

- Realtime connection lifecycle 관리
- Realtime Connection Registry 등록과 갱신
- 인증된 Client와 connection correlation
- 해당 Messenger Client Ingress 구현이 Presence를 소유하는 경우 connection / presence 상태 전이 감지
- 인증된 connection context와 상태 이력을 기준으로 Business Event와 식별자 확정
- Browser JWT `sid`와 현재 connection correlation
- REST / WebSocket 요청의 delivery target resolution
- AI Result의 local connection lookup과 최종 전송
- 사용자 AI 설정 조회·변경과 영속화

---

## Non-responsibilities

- source-confirmed Business Event를 특정 AI 기능으로 전환하는 Cross-domain 실행 정책과 AI execution 상태 전이
- `executionId` 생성과 execution correlation 소유
- AI Workflow와 LLM 실행
- 전역 Result routing decision

공통 routing 계약과 전체 흐름은 [Session Registry Based AI Result Routing](../../decisions/001-session-registry-result-routing.md)에서 관리한다.

## Source Event Boundary

`WebSocket Service (WS)` 또는 `TCP Service (NS)`가 connection 또는 Presence의 Source of Truth인 경우, `AWAY/OFFLINE → ONLINE` 같은 상태 전이와 그 전이에서 파생되는 `offlineSince`, `returnedAt`, `absencePeriodId`를 확정한다. Legacy `connectType`의 의미도 Messenger Client Ingress 경계에서 해석한다.

AI Orchestrator에는 원시 Presence 변경 요청이 아니라 확정된 Business Event와 trusted routing context를 제공한다. Messenger Client Ingress는 Global / Feature / User Policy, 기능별 부재 시간, Cross-domain Context, cooldown, AI execution dedup을 조합해 AiTask 생성 여부를 최종 판단하지 않는다.
