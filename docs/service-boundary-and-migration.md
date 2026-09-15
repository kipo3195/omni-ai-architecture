# Service Boundary and Migration

> Role: 현재 WS service의 책임 집중 상태와 Target Service Boundary / Migration 방향을 구분한다.
> Status: Planned
> 이 문서는 Omni AI 1차 구축 범위와 Messenger service split migration topic을 분리하기 위한 문서다.

---

## 1. Current State

현재 Messenger Backend는 `WS service`가 여러 책임을 함께 처리한다.

```text
WS service
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
```

즉 현재 기준으로 `realtime-message-service`, `user-service`, `auth-service`, `file-service`가 물리적으로 분리되어 있다고 가정하지 않는다.

Status: Current

---

## 2. Omni AI Immediate Scope

Omni AI Architecture의 우선 범위는 기존 WS service를 즉시 분리하는 것이 아니다.

우선 범위:

```text
WS service integration
ai-orchestrator
omni-ai-server
NATS JetStream trigger
Core NATS streaming / result routing
Conversation metadata / runtime history boundary
Server Tool Relay / Client Tool Relay boundary
```

즉 1차 구축에서는 현재 WS service와 연동하면서 `ai-orchestrator`와 `omni-ai-server`의 책임 경계를 먼저 검증한다.

Status: Planned

---

## 3. Target Service Boundary

Target Architecture / Evolution Direction에서는 WS service의 책임을 다음 Service Boundary로 점진 분리하는 것을 고려한다.

```text
realtime-message-service
→ WebSocket Connection, Chat, Note, Alert, History REST, Realtime Push

user-service
→ User Profile, Organization / Class, Rule / Cache, Presence, Label / Address Book

auth-service
→ Authentication, Token Policy, JWT / Cookie Policy, User / Tenant Authentication Context

file-service
→ File Upload / Download, Metadata, Permission, Attachment
```

이 구조는 현재 구현 완료 상태가 아니라, AI 기능 추가로 기존 WS service가 더 비대해지는 것을 피하기 위한 evolution direction이다.

Status: Target

---

## 4. Migration Principle

서비스 분리는 Omni AI 1차 구축 범위와 분리해서 다룬다.

원칙:

- 현재 WS service와 연동 가능한 `ai-orchestrator` / `omni-ai-server` 경계를 먼저 만든다.
- `ai-orchestrator`가 특정 future service split에 과도하게 의존하지 않도록 한다.
- Target service name은 문서에서 경계 설명용으로 사용하되, 현재 배포 구조와 혼동하지 않는다.
- WS service 내부 기능이 분리되더라도 `ai-orchestrator`와 `omni-ai-server`의 계약이 크게 흔들리지 않게 한다.

Status: Planned

---

## 5. Remaining Decisions

- `realtime-message-service` 분리 시 unread count 계산 주체
- `user-service`에서 Label / Address Book을 유지할지 별도 분리할지
- `file-service`의 AI Context 연계 범위
- `auth-service`와 tenant policy 연계 방식
- 현재 WS service에서 NATS event를 발행하는 범위
- service split 전후 Server Tool Relay contract 호환성
