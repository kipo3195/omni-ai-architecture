# 006. AI Feature Enablement Policy and User Preference Strategy

Status: Accepted

## Context

AI 실행 전에는 모든 사용자에게 적용되는 Global / Feature ON/OFF와 사용자별 AI ON/OFF를 함께 확인해야 한다.

Global / Feature Policy는 데이터가 작고 변경 빈도가 낮지만 거의 모든 AI 요청에서 참조된다. 반면 User Policy는 사용자 수에 비례해 증가하고 사용자마다 변경될 수 있다.

모든 정책을 요청마다 DB에서 조회하면 AI Request Hot Path가 DB latency와 availability에 종속된다. 반대로 모든 사용자 설정을 각 AI Orchestrator instance의 Memory에 적재하면 데이터가 중복되고 변경 동기화가 복잡해진다.

AI는 Messenger의 핵심 기능이 아닌 부가기능이므로 정책을 확인할 수 없는 상황에서는 실행하지 않는 방향을 우선한다.

## Decision

DB를 Global, Feature, User Policy의 Source of Truth로 사용한다. Runtime 조회 방식은 정책 특성에 따라 구분한다.

| Policy | Runtime Read | Consistency |
| --- | --- | --- |
| Global AI ON/OFF | Local In-Memory | Scheduled Refresh 기반 Eventual Consistency |
| Feature ON/OFF | Local In-Memory | Scheduled Refresh 기반 Eventual Consistency |
| User AI ON/OFF | Redis, miss 시 DB | Cache-Aside 기반 Eventual Consistency |

Global / Feature Policy는 각 AI Orchestrator instance가 DB에서 주기적으로 갱신하고 AI 요청 처리 시에는 Local In-Memory 값만 확인한다. 따라서 DB 조회를 AI Request Hot Path에 포함하지 않는다.

```text
DB
→ Scheduled Refresh
→ Orchestrator Instance Local In-Memory
```

각 instance가 독립적으로 갱신하므로 설정은 refresh interval 내에 순차적으로 반영될 수 있다. 최초 DB 로딩에 실패하면 기본값을 OFF로 두고 AI 실행을 차단한다. 정상 로딩 이후 refresh가 실패하면 실패한 값으로 덮어쓰지 않고 마지막으로 정상 로딩한 값을 유지한다.

User Policy는 Realtime Service가 설정 변경 API와 DB 저장을 담당하고, DB commit 이후 Redis cache를 갱신한다. AI Orchestrator는 사용자별 설정이 필요할 때 Redis를 조회하고, cache miss 시 DB에서 읽어 Redis에 적재한다. Cache에는 TTL을 적용한다.

```text
Realtime Service
→ User Policy DB update
→ Redis update

AI Orchestrator
→ Redis
  ├─ HIT: cached value
  └─ MISS: DB read → Redis put → value
```

사용자 전체 설정을 AI Orchestrator Local Memory에 적재하지 않는다.

Redis 장애 시 DB로 fallback한다. DB 조회에 성공하면 해당 값으로 판단하고 cache 복구는 best effort로 처리한다. Redis와 DB를 모두 조회할 수 없으면 User Policy를 확인할 수 없으므로 AI 실행을 Fail-Closed 처리한다.

AI 실행 여부의 최종 판단은 AI Orchestrator가 담당한다.

```text
Global Policy
→ User Policy
→ Feature Policy
→ Trigger / Cooldown / Dedup
→ Omni AI
```

서비스별 책임 경계는 다음과 같다.

- Realtime Service는 사용자 AI 설정의 변경과 저장을 담당한다.
- AI Orchestrator는 모든 AI Policy를 적용하고 실행 여부를 최종 결정한다.
- Omni AI는 허용된 요청을 실행하며 ON/OFF Policy 판단에 관여하지 않는다.

## Alternatives

### 모든 정책을 요청마다 DB에서 조회

항상 DB의 최신 값을 확인할 수 있지만 DB I/O가 모든 AI 요청의 latency와 availability에 직접 영향을 준다. 변경 빈도가 낮은 Global / Feature Policy까지 요청마다 조회해야 하므로 선택하지 않는다.

### 모든 사용자 설정을 Orchestrator Memory에 적재

조회는 빠르지만 사용자 수와 instance 수에 비례해 데이터가 중복된다. 사용자 설정 변경 시 모든 instance의 Memory를 동기화해야 하므로 선택하지 않는다.

### Redis를 User Policy의 Source of Truth로 사용

조회 구조는 단순해지지만 cache eviction이나 장애가 사용자 설정 유실로 이어질 수 있다. Redis는 Runtime Cache로만 사용하고 DB를 Source of Truth로 유지한다.

### 정책 저장소 장애 시 Fail-Open

AI 가용성은 높아지지만 관리자가 차단했거나 사용자가 비활성화한 기능을 실행할 수 있다. AI는 부가기능이므로 안전과 사용자 설정을 우선해 선택하지 않는다.

### 변경 이벤트만으로 Global / Feature Policy 전파

설정을 빠르게 반영할 수 있지만 event loss와 신규 instance 초기화를 별도로 처리해야 한다. 현재는 Scheduled Refresh를 사용하고, 즉시성이 필요해지면 변경 이벤트를 추가하되 Scheduled Refresh는 복구 수단으로 유지한다.

## Consequences

좋아지는 점:

- Global / Feature Policy의 DB 조회가 AI Request Hot Path에서 제거된다.
- 사용자 설정을 모든 Orchestrator instance에 중복 적재하지 않는다.
- DB를 Source of Truth로 유지하면서 Local Memory와 Redis로 조회 부하를 줄인다.
- AI 실행 판단이 AI Orchestrator에 모이고 Omni AI는 실행에 집중한다.
- 최초 정책 로딩 실패나 User Policy 저장소 장애 시 의도하지 않은 AI 실행을 막는다.

감수해야 할 점:

- Global / Feature Policy 변경은 refresh interval만큼 늦게 반영될 수 있다.
- Multi-instance 환경에서 짧은 시간 동안 instance별 판단이 다를 수 있다.
- Redis 갱신 실패 시 User Policy가 TTL 만료까지 stale할 수 있다.
- Redis 장애 시 DB fallback 트래픽이 증가할 수 있다.
- 마지막 Global / Feature Policy 값을 유지하는 동안 새로운 Kill Switch가 반영되지 않을 수 있으므로 refresh 실패를 모니터링해야 한다.

## Related

- [ADR 003. AI Orchestrator as an Application Service](003-ai-orchestrator-application-service.md)
- [Architecture](../architecture.md)
- [Service Boundary and Migration](../service-boundary-and-migration.md)
