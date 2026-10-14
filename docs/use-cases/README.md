# End-to-End Use Cases

> Role: 사용자 기능이 Business Trigger부터 Client 결과 전달까지 동작하는 전체 구현 흐름과 완료 조건을 관리한다.

---

## Why Use Case Documents

서비스별 구현 문서는 각 서비스 내부 구조를 설명한다. Use Case 문서는 여러 서비스에 걸친 하나의 사용자 결과를 기준으로 전체 흐름, 책임, 계약, 실패 처리와 완료 조건을 연결한다.

```text
Use Case
→ 사용자 관점의 End-to-End 구현과 상태

Component Documentation
→ 해당 Use Case를 지원하는 서비스 내부 구조
```

---

## Implementation Sequence

문서 번호는 구현 순서를 나타낸다. 뒤 단계는 앞 단계에서 검증한 계약과 실행 구조를 재사용한다.

| 순서 | Use Case | 핵심 확장 |
| --- | --- | --- |
| 01 | [Conversation Start](01-conversation-start.md) | 최초 Server-driven E2E |
| 02 | [Returned Message Topics](02-returned-message-topics.md) | 상태 변경 Trigger와 Cross-domain Context |
| 03 | [Scheduled Weekly Report](03-scheduled-weekly-report.md) | Client LLM, Schedule, deferred execution |
| 04 | [Server Schedule Intent](04-server-schedule-intent.md) | Server LLM 기반 intent parsing |
| 05 | [Tool-assisted AI](05-tool-assisted-ai.md) | Runtime Tool Calling |

Use Case의 우선순위와 상태는 [Roadmap](../roadmap.md)에서만 관리한다.

## Writing a New Use Case

새 Use Case는 구현 순서에 맞는 두 자리 번호와 짧은 kebab-case 이름을 사용한다.

```text
06-example-use-case.md
```

[Use Case Template](TEMPLATE.md)을 복사한 뒤 모든 섹션을 검토한다. 아직 결정되지 않은 내용은 생략하지 않고 `Open`으로 표시하며, 되돌리기 어려운 설계 결정은 ADR로 분리한다.

