# Realtime Service Implementation

Status: Designed

---

## Implementation Areas

| Area | 기록 범위 |
| --- | --- |
| [Result Delivery](result-delivery/README.md) | Connection Registry, reconnect fencing, REST target resolution, local Client delivery |
| [User AI Settings](user-ai-settings/README.md) | 사용자 AI 설정 API, DB 저장, Redis cache 갱신 |

---

## Scope

Realtime Service는 WebSocket 또는 TCP connection을 소유하고 Client protocol의 ingress와 delivery를 담당한다.

주요 책임:

- Realtime connection lifecycle 관리
- Realtime Connection Registry 등록과 갱신
- 인증된 Client와 connection correlation
- Browser JWT `sid`와 현재 connection correlation
- REST / WebSocket 요청의 delivery target resolution
- AI Result의 local connection lookup과 최종 전송
- 사용자 AI 설정 조회·변경과 영속화

---

## Non-responsibilities

- AI 실행 정책과 상태 전이
- `executionId` 생성과 execution correlation 소유
- AI Workflow와 LLM 실행
- 전역 Result routing decision

공통 routing 계약과 전체 흐름은 [Session Registry Based AI Result Routing](../../decisions/001-session-registry-result-routing.md)에서 관리한다.
