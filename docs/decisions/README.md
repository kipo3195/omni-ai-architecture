# Architecture Decision Records

> Role: 되돌리기 어렵거나 이후 구현 방향에 영향을 주는 개별 설계 결정을 기록하는 디렉터리
> Status: Planned

---

## Purpose

`docs/design-decisions.md`는 주요 설계 결정의 요약 인덱스이고, 이 디렉터리는 개별 결정의 상세 기록을 보관한다.

ADR은 "무엇을 선택했는가"뿐 아니라 "왜 그렇게 선택했는가"와 "어떤 Trade-off를 감수했는가"를 남기기 위한 문서이다.

---

## When to Add an ADR

다음과 같은 결정은 별도 ADR로 남긴다.

- Queue 기술 또는 운영 방식 선택
- Current WebSocket Service split / migration 순서
- AI Orchestrator runtime 선택
- AI Orchestrator boundary와 HA 전략
- NATS JetStream / Core NATS 사용 범위
- AiTask schema의 큰 변경
- Conversation Metadata Store 선택
- AI History Store 선택
- Trigger expiration / dedup 정책 변경
- Session Registry와 Result Routing 방식 선택
- Stream reconnect / resume 정책
- Chatbot history retention 정책
- Tool Runtime / Server Tool Adapter 통신 방식과 권한 경계
- Label / Address Book 분리 여부
- unread count 계산 주체
- file-service의 AI Context 연계 범위
- Context Store 도입 여부
- Client Tool Delivery 통신 방식 선택
- Agent Runtime lifecycle 변경
- retry / timeout / duplicate 방지 정책 변경
- 비용이나 latency에 큰 영향을 주는 구조 변경

사소한 구현 메모나 일시적인 버그는 `docs/notes/`에 기록한다.

---

## File Naming

```text
001-ai-task-model.md
002-queue-selection.md
003-client-tool-relay.md
```

번호는 작성 순서를 나타낸다. 제목은 결정의 주제를 짧게 표현한다.

---

## ADR Template

```md
# 001. 결정 제목

Status: Proposed | Accepted | Superseded

## Context

어떤 문제가 있었는가.

## Decision

무엇을 선택했는가.

## Alternatives

검토한 대안은 무엇이었는가.

## Consequences

좋아진 점과 감수해야 할 점은 무엇인가.

## Related

관련 문서나 이슈가 있다면 연결한다.
```

---

## Index

- [001. Session Registry Based AI Result Routing](001-session-registry-result-routing.md)
- [002. Tool Runtime Ownership and Client Tool Dispatch](002-tool-runtime-ownership-client-tool-dispatch.md)
- [003. AI Orchestrator as a Spring Boot Application Service](003-ai-orchestrator-spring-boot-service.md)
