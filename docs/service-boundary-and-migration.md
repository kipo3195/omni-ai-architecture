# Service Boundary and Migration

> Role: 현재 WebSocket Service의 책임 집중 상태와 Target Service Boundary / Migration 방향을 구분한다.
> Status: Planned
> 이 문서는 Omni AI 1차 구축 범위와 Messenger service split migration topic을 분리하기 위한 문서다.

---

## 1. Current State

현재 Messenger Backend는 `WebSocket Service`가 여러 책임을 함께 처리한다. TCP 기반 Client는 `TCP Realtime Service`를 통해 realtime connection과 delivery를 처리한다.

```text
WebSocket Service
├─ WebSocket Connection / Session
├─ 실시간 채팅
├─ 쪽지
├─ 알림
├─ 사용자 상태
├─ 사용자 정보
├─ 인증
├─ 파일
├─ REST API
└─ 기타 Business Logic

TCP Realtime Service
├─ TCP Connection / Session
├─ TCP Client Delivery
└─ Realtime Push
```

즉 현재 기준으로 WebSocket Service의 내부 책임이 `user-service`, `auth-service`, `file-service`로 물리적으로 분리되어 있다고 가정하지 않는다.

Status: Current

---

## 2. Omni AI Immediate Scope

Omni AI Architecture의 우선 범위는 기존 WebSocket Service를 즉시 분리하는 것이 아니다.

우선 범위:

```text
WebSocket Service integration
TCP Realtime Service integration
AI Orchestrator
Omni AI Server
NATS JetStream trigger
Core NATS streaming / result routing
Conversation metadata / runtime history boundary
Tool Runtime / Server Tool Adapter / Client Tool Delivery boundary
```

즉 1차 구축에서는 현재 WebSocket Service, TCP Realtime Service와 연동하면서 `AI Orchestrator`와 `Omni AI Server`의 책임 경계를 먼저 검증한다.

Status: Planned

---

## 3. Target Service Boundary

Target Architecture / Evolution Direction에서는 WebSocket Service의 책임을 다음 Service Boundary로 점진 분리하는 것을 고려한다.

```text
WebSocket Service
→ WebSocket Connection, Chat, Note, Alert, History REST, Realtime Push

TCP Realtime Service
→ TCP Connection, TCP Session, Realtime Push

user-service
→ User Profile, Organization / Class, Rule / Cache, Presence, Label / Address Book

auth-service
→ Authentication, Token Policy, JWT / Cookie Policy, User / Tenant Authentication Context

file-service
→ File Upload / Download, Metadata, Permission, Attachment
```

이 구조는 현재 구현 완료 상태가 아니라, AI 기능 추가로 기존 WebSocket Service가 더 비대해지는 것을 피하기 위한 evolution direction이다.

Status: Target

---

## 4. Migration Principle

서비스 분리는 Omni AI 1차 구축 범위와 분리해서 다룬다.

원칙:

- 현재 WebSocket Service, TCP Realtime Service와 연동 가능한 `AI Orchestrator` / `Omni AI Server` 경계를 먼저 만든다.
- WebSocket Client와 TCP Client가 동일한 AI 기능, 동일한 요청 / 응답 규격, 동일한 실행 정책을 사용하게 한다.
- `AI Orchestrator`가 특정 future service split에 과도하게 의존하지 않도록 한다.
- Target service name은 문서에서 경계 설명용으로 사용하되, 현재 배포 구조와 혼동하지 않는다.
- WebSocket Service 내부 기능이 분리되더라도 `AI Orchestrator`와 `Omni AI Server`의 계약이 크게 흔들리지 않게 한다.

Status: Planned

---

## 5. Remaining Decisions

- `WebSocket Service` 분리 시 unread count 계산 주체
- `user-service`에서 Label / Address Book을 유지할지 별도 분리할지
- `file-service`의 AI Context 연계 범위
- `auth-service`와 tenant policy 연계 방식
- 현재 WebSocket Service에서 NATS event를 발행하는 범위
- TCP Realtime Service의 AI Request / Result Routing contract
- service split 전후 Server Tool Adapter contract 호환성
