# Implementation

> Role: 설계 문서에서 정의한 책임 경계를 실제 runtime과 application structure로 연결한다.
> Status: Planned

---

## Purpose

이 디렉터리는 Omni AI Platform의 구현 예정 구조와 구현 관점을 정리한다.

상위 Architecture 문서는 책임 경계와 실행 흐름을 설명하고, 이 디렉터리는 각 runtime이 어떤 application structure로 구현되었는지 기록한다.

---

## Implementation Documents

| Document | Role | Status |
| --- | --- | --- |
| [AI Orchestrator](ai-orchestrator.md) | Spring Boot 기반 AI Orchestrator 내부 구조 | Planned |
| [Omni AI Server](omni-ai-server.md) | Python 기반 AI Runtime 역할과 책임 | Planned |

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
