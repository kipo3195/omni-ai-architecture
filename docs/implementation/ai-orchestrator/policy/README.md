# AI Orchestrator Policy

Status: Designed

---

## Purpose

AI Orchestrator가 Global, Feature, User Policy를 조회하고 AI 실행 여부를 최종 판단하는 구현 구조를 정의한다.

정책의 Source of Truth와 cache 전략은 [ADR 006](../../../decisions/006-ai-feature-enablement-policy-and-user-preference.md)을 따른다.

---

## Internal Structure

```text
policy/
├── application
│   └── AiExecutionPolicyService
├── domain
│   ├── GlobalAiPolicyProvider
│   ├── FeaturePolicyProvider
│   └── UserAiPolicyProvider
└── infrastructure
    ├── PolicySnapshotRefresher
    ├── PolicySnapshotRepository
    ├── UserAiPolicyCache
    └── UserAiPolicyRepository
```

| Component | Responsibility |
| --- | --- |
| `AiExecutionPolicyService` | 요청에 필요한 정책을 순서대로 평가하고 실행 허용 여부 반환 |
| `GlobalAiPolicyProvider` | Local In-Memory에서 전체 AI 활성화 상태 조회 |
| `FeaturePolicyProvider` | Local In-Memory에서 기능별 활성화 상태 조회 |
| `UserAiPolicyProvider` | Redis와 DB를 통해 사용자별 활성화 상태 조회 |
| `PolicySnapshotRefresher` | DB의 Global / Feature Policy를 주기적으로 읽어 Memory snapshot 교체 |
| `PolicySnapshotRepository` | Global / Feature Policy DB 조회 |
| `UserAiPolicyCache` | 사용자별 Policy의 Redis 조회와 저장 |
| `UserAiPolicyRepository` | Cache miss 또는 Redis 장애 시 User Policy DB 조회 |

Application과 Domain은 DB 또는 Redis client에 직접 의존하지 않고 provider와 repository interface를 사용한다.

---

## Runtime Decision Flow

```text
AI Request
→ GlobalAiPolicyProvider
→ UserAiPolicyProvider
→ FeaturePolicyProvider
→ Trigger / Cooldown / Dedup
→ Omni AI
```

어느 한 정책이라도 OFF이거나 확인할 수 없으면 Omni AI를 호출하지 않는다. 정책에 의한 정상 거부와 저장소 장애에 의한 거부는 내부 reason code로 구분한다.

```text
GLOBAL_AI_DISABLED
FEATURE_DISABLED
USER_AI_DISABLED
USER_POLICY_UNAVAILABLE
```

---

## Global and Feature Policy

`PolicySnapshotRefresher`는 설정된 주기마다 DB에서 Global / Feature Policy 전체를 읽고 검증된 immutable snapshot으로 교체한다.

```text
DB
→ PolicySnapshotRepository
→ validate
→ atomic snapshot replace
→ GlobalAiPolicyProvider / FeaturePolicyProvider
```

Process 시작 시 snapshot 기본값은 OFF다. 최초 refresh에 성공하기 전에는 AI 요청을 Fail-Closed 처리한다.

정상 snapshot을 한 번 이상 로딩한 후 refresh가 실패하면 기존 snapshot을 유지한다. 부분 조회 결과나 검증에 실패한 값으로 현재 snapshot을 덮어쓰지 않는다.

각 instance는 독립적으로 refresh하며 다른 instance의 Memory를 동기화하지 않는다. 따라서 변경은 refresh interval 안에서 Eventual Consistency로 반영된다.

---

## User Policy

`UserAiPolicyProvider`는 요청 대상 사용자 한 명의 설정만 조회한다.

```text
Redis GET
├─ HIT
│   └─ cached value 반환
└─ MISS
    └─ DB read
        ├─ Redis PUT with TTL
        └─ DB value 반환
```

Redis timeout 또는 connection failure는 cache miss와 구분해 기록하되 DB fallback을 수행한다. DB 조회에 성공하면 Redis 복구 여부와 관계없이 해당 값으로 현재 요청을 판단한다.

Redis와 DB를 모두 조회할 수 없으면 `USER_POLICY_UNAVAILABLE`로 실행을 차단한다. 저장소 장애 결과는 Redis에 OFF 값으로 cache하지 않는다.

Cache key에는 tenant와 user 식별자를 포함한다.

```text
ai:user:{tenantId}:{userId}:enabled
```

TTL은 application configuration으로 관리한다. 사용자 전체 설정을 Orchestrator Local Memory에 적재하지 않는다.

---

## Configuration and Observability

최소 운영 설정은 다음을 포함한다.

- Global / Feature Policy refresh interval
- User Policy cache TTL
- Redis와 DB timeout
- DB fallback concurrency limit

각 instance는 마지막 정상 snapshot 갱신 시각과 refresh 성공 여부를 노출한다. Policy별 허용·거부 횟수와 `USER_POLICY_UNAVAILABLE`을 별도 metric으로 기록한다.

Redis 장애 시 DB fallback 요청이 급증하지 않도록 timeout, connection pool과 circuit breaker를 적용한다.

---

## Non-responsibilities

- 사용자 AI 설정 변경 API와 영속화
- Admin Policy 변경 API
- LLM, Agent, Prompt 또는 Workflow 실행
- Realtime connection과 Client result delivery

---

## Related

- [ADR 006. AI Feature Enablement Policy and User Preference Strategy](../../../decisions/006-ai-feature-enablement-policy-and-user-preference.md)
- [AI Orchestrator Implementation](../README.md)
- [Realtime Service User AI Settings](../../realtime-service/user-ai-settings/README.md)
