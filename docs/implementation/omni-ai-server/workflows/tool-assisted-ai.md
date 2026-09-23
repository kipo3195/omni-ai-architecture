# Tool-assisted AI Workflow

Status: Planned

사용자 요청에 따라 LLM이 필요한 Tool을 선택하고 Tool Result를 사용해 실행 가능한 결과 초안을 생성한다.

## First Candidate

지난주 주간보고에서 결재가 필요한 항목을 찾고 결재 요청 쪽지 초안을 생성한다.

## Boundary

Tool 선택과 결과 기반 추론은 Omni AI Server가 담당한다. Tool permission, dispatch, lifecycle과 side effect 승인은 AI Orchestrator가 담당한다.

## Related

- [Tool-assisted AI Use Case](../../use-cases/05-tool-assisted-ai/README.md)
- [Tool Calling](../tool-calling/README.md)
