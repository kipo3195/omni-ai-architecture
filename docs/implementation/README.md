# Implementation

> Role: 설계 문서에서 정의한 책임 경계를 실제 runtime과 application structure로 연결한다.
> Status: Planned

---

## Purpose

이 디렉터리는 Omni AI Platform의 서비스별 구현 구조와 진행 상태를 정리한다.

상위 Architecture 문서는 책임 경계와 목표 실행 흐름을 설명한다. 이 디렉터리는 각 runtime이 어떤 application structure와 공통 실행 방식으로 구현되었는지 기록한다. 개별 AI 기능은 해당 실행 방식의 적용 사례로 다루며, 여러 기능이 공유하는 구조를 반복해서 작성하지 않는다.

---

## Implementation Documents

| Document | Role | Status |
| --- | --- | --- |
| [AI Orchestrator](ai-orchestrator/README.md) | Spring Boot 기반 AI Orchestrator 내부 구조 | Planned |
| [Omni AI Server](omni-ai-server/README.md) | Python 기반 AI Runtime 역할과 책임 | Planned |

---

## Boundary

```text
WebSocket Service / TCP Realtime Service
→ connection / session / channel delivery

AI Orchestrator
→ policy / context assembly / AiTask / execution correlation

Omni AI Server
→ prompt / workflow / agent / LLM runtime
```

Realtime channel이 WebSocket인지 TCP인지와 무관하게 AI 실행 판단과 요청 / 응답 규격은 AI Orchestrator에서 공통화한다.
