# Tool Runtime

Status: Planned

AI Orchestrator가 소유하는 Tool 관리와 라우팅 구조를 기록한다.

Tool 등록, 권한 확인, lifecycle, Server / Client Tool dispatch, 결과 연결을 실제 구현 기준으로 정리한다.

## Responsibility

- Tool registry와 schema validation
- permission / capability 확인
- Tool lifecycle state
- Server Tool Adapter와 Client Tool Delivery dispatch
- timeout, retry, cancellation
- Tool Result validation과 normalization
- `executionId`, `toolCallId`, `toolAttempt`, `idempotencyKey` correlation
- Omni AI Server execution resume 조정
- audit

## Implementation Sequence

1. 조회 중심 Server Tool
2. Client Tool Delivery
3. Server Tool과 Client Tool을 함께 사용하는 복합 실행

첫 적용과 완료 조건은 [Tool-assisted AI Use Case](../../use-cases/05-tool-assisted-ai/README.md)에서 관리한다.

데이터 변경이나 외부 전송을 수행하는 Tool은 조회 Tool과 분리하고 명시적인 사용자 확인을 요구한다.
