# 004. AI Execution and Context Identifier Separation

Status: Proposed

## Context

AI 기능이 늘어나면서 방 입장, Client 요청, 파일 처리, 정기 작업, Stateful Conversation처럼 수명주기가 서로 다른 대상을 하나의 ID로 표현하기 어려워졌다.

기능마다 `XXXSessionId`를 추가하면 ID 의미가 불명확해지고, 반대로 `executionId`가 Realtime Session이나 Business State까지 소유하면 AI Orchestrator의 책임이 비대해진다.

## Decision

모든 AI 실행은 `executionId`를 공통 식별자로 사용한다.

`roomSessionId`, `conversationId`, `connectionId` 같은 ID는 AI 실행과 독립된 lifecycle이 실제로 존재할 때만 생성하며, `AI Execution`에는 소유 상태가 아닌 correlation reference로 저장한다.

주요 ID의 역할과 소유권은 다음과 같다.

| ID | Purpose | Owner |
| --- | --- | --- |
| `triggerId` | AI 실행 후보가 된 Business Event 또는 Client Request 식별 | Trigger Source |
| `executionId` | 개별 AI 실행과 상태 전이 식별 | AI Orchestrator |
| `taskId` | Queue 또는 Worker에 전달된 작업 식별 | AI Orchestrator / Task Runtime |
| `eventId` | 결과·진행 이벤트 중복 제거 | Event Producer |
| `idempotencyKey` | 동일 요청의 중복 실행 방지 | Trigger Source / AI Orchestrator |
| `conversationId` | 여러 AI 실행에 걸친 논리적 대화 식별 | AI Orchestrator |
| `clientId` | Client 또는 Device 구분 | Realtime Service |
| `connectionId` | 현재 Realtime Connection 식별 | Realtime Service / Session Registry |
| `roomSessionId` | 특정 Connection의 방 입장 구간 식별 | Realtime Service / Session Registry |

`executionId`는 항상 존재하지만 나머지 ID는 Use Case에 필요한 경우에만 연결한다.

```text
AiExecution
  executionId
  triggerId
  workflowType
  status
  scopeRef
  routingRef
```

예를 들어 Conversation Start는 다음 correlation을 사용한다.

```text
executionId
→ roomSessionId
→ connectionId / clientId
```

`AI Orchestrator`는 이 ID들의 관계와 실행 상태를 관리하지만, WebSocket Connection이나 Room Session 자체를 소유하지 않는다.

## Alternatives

### 기능마다 XXXSessionId 생성

기능별 모델은 명시적이지만, 기능이 늘어날수록 유사한 session 개념과 lifecycle 관리가 반복된다.

### executionId로 모든 Session 상태 통합

실행 추적은 단순해지지만 Connection, Room Entry, Conversation, AI Execution의 서로 다른 lifecycle이 결합되고 AI Orchestrator가 다른 Service의 상태까지 소유하게 된다.

## Consequences

좋아지는 점:

- 모든 AI 기능이 동일한 `executionId` 기반 상태와 correlation을 사용한다.
- 기능별로 불필요한 Session ID를 만들지 않는다.
- Realtime Session 소유권과 AI Execution 소유권이 분리된다.
- Room, File, User, Conversation 기반 Use Case를 같은 실행 모델로 확장할 수 있다.

감수할 점:

- ID별 의미와 lifecycle을 API 및 Event schema에서 명확히 유지해야 한다.
- 결과 전달 시 `executionId` 검증과 `routingRef` 유효성 검증이 모두 필요하다.
- 기존 `sessionId` 필드가 실제 `executionId`를 의미하는 계약은 점진적으로 정리해야 한다.

## Related

- [Architecture](../architecture.md)
- [Session Registry Based AI Result Routing](001-session-registry-result-routing.md)
- [AI Orchestrator as an Application Service](003-ai-orchestrator-application-service.md)
