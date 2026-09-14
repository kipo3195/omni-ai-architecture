# Design Decisions

> Role: 주요 설계 결정의 요약 인덱스이며, 자세한 개별 결정은 `docs/decisions/`에 ADR로 기록한다.  
> Status: Designed  
> 이 문서는 README의 Core Design을 기준으로 주요 설계 판단과 Trade-off를 정리한다.

---

## Business Policy는 Java Messenger Backend에서 처리한다

결정  
→ AI 실행 여부는 Java Messenger Backend에서 판단한다.

배경  
→ Messenger Backend는 User State, Permission, Feature Enable, Room State, Label, Unread, Cooldown, Business Data를 이미 관리한다.

이유  
→ AI Runtime이 Business Rule을 소유하면 인증, 권한, 상태 판단이 중복된다. Java Backend가 실행 여부를 판단하고 Omni AI는 실행이 확정된 Task 처리에 집중하는 편이 책임 경계가 명확하다.

Trade-off  
→ Omni AI 단독으로 기능을 시작하기 어렵다. 대신 AiTask 계약을 명확히 관리해야 한다.

Status: Designed

---

## Business Event와 AiTask를 분리한다

결정  
→ Business Event와 AiTask를 다른 모델로 둔다.

배경  
→ `ROOM_ENTERED`는 발생한 일을 나타내고, `CONVERSATION_START`는 AI가 수행할 작업을 나타낸다.

이유  
→ 하나의 Event에서 여러 Task 후보를 만들 수 있고, 같은 Task를 여러 Event나 Client Request 경로에서 재사용할 수 있다.

Trade-off  
→ Event-to-Task mapping과 Policy 조합을 별도로 관리해야 한다.

Status: Designed

---

## 모든 Event를 Queue에 넣지 않는다

결정  
→ `EXECUTE`가 확정된 AiTask만 Queue에 전달한다.

배경  
→ Messenger Event는 많고, 모든 Event가 AI 실행으로 이어져야 하는 것은 아니다.

이유  
→ Queue 이전에 불필요한 AI 실행 후보를 제거하면 비용, latency, Worker 점유, Queue traffic을 줄일 수 있다.

Trade-off  
→ Queue Worker가 원본 Event를 보고 뒤늦게 판단하는 유연성은 줄어든다.

Status: Designed

---

## Client Integration은 Execution Mode가 아니라 Provider이다

결정  
→ Workflow / Agent는 실행 방식이고, Client Integration은 Context / Tool Provider로 분리한다.

배경  
→ Client Tool을 사용한다고 해서 모두 Agent Execution인 것은 아니다. 사전 정의 Workflow도 Client Context가 필요할 수 있다.

이유  
→ 실행 방식과 Context 공급 방식을 분리해야 기능 조합이 자연스럽다.

Trade-off  
→ 문서와 구현에서 두 축을 계속 명확히 구분해야 한다.

Status: Designed

---

## Client는 Omni AI와 직접 연결하지 않는다

결정  
→ Client는 Java Messenger Server와 WebSocket으로 통신하고, Omni AI는 Java Tool Gateway를 통해 Client Tool을 사용한다.

배경  
→ Java Messenger Server는 Authentication, Session, WebSocket, Permission, Device State를 이미 소유한다.

이유  
→ Omni AI Server가 Client 연결을 직접 소유하면 세션과 권한 책임이 중복된다.

Trade-off  
→ Client Tool Calling에는 Gateway hop이 추가된다.

Status: Designed

---

## Agent Runtime은 단계적으로 확장한다

결정  
→ 예측 가능한 기능은 Workflow Execution으로 먼저 다루고, Runtime Tool Calling이 필요한 기능은 Agent Runtime으로 확장한다.

배경  
→ Conversation Start나 Urgent Message Summary는 Workflow로 시작하기 적합하다. 반면 여러 Tool을 순차 호출하고 다음 단계를 동적으로 결정하는 기능은 Agent Lifecycle이 필요하다.

이유  
→ 초기 복잡도를 낮추면서 공통 AiTask / Queue / Context 구조를 먼저 검증할 수 있다.

Trade-off  
→ Agent Runtime, correlation, timeout, resume 상태 관리는 별도 설계가 필요하다.

Status: Planned

---

## Queue는 Workload 기준으로 확장한다

결정  
→ 초기에는 Queue 하나로 시작하고, 필요하면 workload 기준으로 분리한다.

배경  
→ Event Type은 Business 확장 단위이고 Queue / Worker Pool은 자원 격리 단위이다.

이유  
→ Event Type별 Queue는 기능이 늘어날수록 운영 복잡도를 키운다. 자원 격리는 latency, 처리 시간, 비용, 부하 특성에 맞춰야 한다.

Trade-off  
→ 초기 Queue 하나에서는 서로 다른 workload가 같은 실행 자원을 공유한다.

Status: Designed
