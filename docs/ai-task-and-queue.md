# AiTask and Queue

> Status: Designed  
> 이 문서는 AiTask와 AI Task Queue의 실행 경계를 설명한다. Queue 운영 구현 완료를 의미하지 않는다.

---

## 1. AiTask

AiTask는 AI 실행이 확정된 작업을 표현하는 공통 실행 단위이다.

Business Event나 Client Request가 "무슨 일이 들어왔는가"를 나타낸다면, AiTask는 "AI가 무엇을 실행해야 하는가"를 나타낸다.

Status: Designed

---

## 2. Candidate Fields

초기 필드 후보:

```text
taskId
taskType
triggerType
userId
roomId
requestedAt
priority
workloadType
contextRef
metadata
```

필수 필드는 구현 과정에서 축소하거나 확장할 수 있다.

Status: Designed

---

## 3. Not a Raw Context Carrier

AiTask를 대화 원문 운반 객체로 만들지 않는다.

권장:

```text
AiTask
  - task metadata
  - contextRef
  - small metadata
```

지양:

```text
AiTask
  - full chat history
  - large client local data
  - raw file content
```

대용량 Context는 Context Store 또는 원본 저장소에서 조회하는 방향으로 둔다.

Status: Designed

---

## 4. Queue Entry Condition

AI Task Queue에는 실행이 확정된 Task만 들어간다.

```text
Server-triggered path                  Client-explicit path
Business Event                         Client Request
        ↓                                      ↓
Business Policy                        Permission / Context Scope / Policy
        ↓                                      ↓
SKIP / EXECUTE                         SKIP / EXECUTE
        ↓                                      ↓
        └──── EXECUTE인 경우에만 AiTask 생성 ──┘
        ↓
AI Task Queue
```

Queue는 모든 Business Event를 받는 Event Bus가 아니다.

Status: Designed

---

## 5. Queue Role

AI Task Queue는 Messenger 핵심 흐름과 AI 실행을 분리한다.

- Messenger 핵심 요청 흐름과 AI latency 분리
- AI 장애가 Messenger 핵심 기능에 전파되는 것을 방지
- 순간적인 AI workload 흡수
- retry / backpressure 기반 제공
- 향후 workload별 Worker 분리 가능

Status: Designed

---

## 6. After Queue

Queue 이후에는 Omni AI Runtime이 실행 책임을 가진다.

```text
AiTask
  ↓
Task Router
  ↓
Context Resolution
  ↓
Workflow Execution 또는 Agent Execution
  ↓
LLM / Model / Tool
  ↓
Validation
  ↓
Structured Result
```

Business Rule 판단은 Queue 이전에 끝나야 한다.

Status: Designed

---

## 7. Initial Queue and Future Split

초기에는 Queue 하나로 시작한다.

향후 실제 부하 특성이 확인되면 workload 기준으로 분리할 수 있다.

```text
REALTIME
NORMAL
BACKGROUND
HEAVY
```

Event Type은 Business 확장 단위이고, Queue / Worker Pool은 자원 격리 단위이다.

Status: Designed

---

## 8. Execution Policy

Execution Policy는 이미 승인된 Task의 실행 안정성을 다룬다.

- retry
- timeout
- rate limit
- duplicate execution 방지
- backpressure
- failure handling

Status: Designed

