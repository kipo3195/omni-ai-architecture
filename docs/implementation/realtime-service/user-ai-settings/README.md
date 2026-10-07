# Realtime Service User AI Settings

Status: Designed

---

## Purpose

Realtime Service가 인증된 사용자의 AI ON/OFF 설정을 변경하고 DB와 Redis에 반영하는 구현 책임을 정의한다.

AI 실행 여부의 최종 판단과 Runtime 조회 방식은 [ADR 006](../../../decisions/006-ai-feature-enablement-policy-and-user-preference.md)을 따른다.

---

## Responsibility

Realtime Service는 다음 책임을 가진다.

- 사용자 AI 설정 조회 및 변경 API 제공
- 인증 context의 tenant와 user 기준으로 요청 검증
- User Policy DB 저장
- DB commit 이후 Redis cache 갱신

Client가 전달한 tenantId 또는 userId를 저장 대상의 authoritative 값으로 사용하지 않는다. 인증된 요청 context에서 대상 사용자를 식별한다.

---

## Update Flow

```text
Client
→ Realtime Service
→ authentication / authorization
→ User Policy DB update and commit
→ Redis cache update
```

DB가 User Policy의 Source of Truth다. Redis 갱신은 DB transaction commit 이후에 실행하며, commit되지 않은 값을 cache에 먼저 노출하지 않는다.

Cache key는 AI Orchestrator와 동일한 계약을 사용한다.

```text
ai:user:{tenantId}:{userId}:enabled
```

Cache value에는 ON/OFF 상태를 저장하고 설정된 TTL을 적용한다. DB commit 이후 Redis 갱신에 실패하더라도 이미 commit된 DB 변경을 rollback하지 않는다. 실패를 기록하고 cache 갱신을 retry하거나 invalidation한다.

---

## Read Flow

Client에 현재 설정을 제공하는 조회 API는 인증된 tenant와 user를 기준으로 DB의 값을 반환한다. Redis는 AI 실행 시 Runtime 조회를 위한 cache이며 사용자 설정의 Source of Truth로 사용하지 않는다.

AI Orchestrator의 정책 조회 흐름은 Realtime Service API 처리와 분리한다.

```text
Client setting read / write
→ Realtime Service

AI execution policy read
→ AI Orchestrator
→ Redis
→ miss/error: DB
```

---

## Failure Handling

| Failure | Handling |
| --- | --- |
| DB update failure | 설정 변경 실패로 응답하고 Redis를 갱신하지 않음 |
| DB commit 후 Redis update failure | DB 변경은 유지하고 retry 또는 invalidation 수행 |
| Redis unavailable | 설정 API의 DB 처리는 유지하고 cache 동기화 실패를 기록 |

Redis 갱신 실패 시 기존 cache 값이 TTL까지 남을 수 있다. OFF 변경의 반영 지연을 관측할 수 있도록 cache update failure metric을 남긴다.

---

## Non-responsibilities

- Global / Feature Policy의 Runtime snapshot 관리
- AI 실행 여부의 최종 판단
- Trigger, Cooldown 또는 Dedup 판단
- Omni AI 호출과 Workflow 실행

---

## Related

- [ADR 006. AI Feature Enablement Policy and User Preference Strategy](../../../decisions/006-ai-feature-enablement-policy-and-user-preference.md)
- [Realtime Service Implementation](../README.md)
- [AI Orchestrator Policy](../../ai-orchestrator/policy/README.md)
