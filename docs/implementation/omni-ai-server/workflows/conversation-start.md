# Conversation Start Workflow

Status: In Progress

방 입장 Context를 받아 TRENDING 유형의 대화 시작 추천을 생성한다.

## Input

- 사용자와 방 식별자
- 현재 실행과 session correlation
- 최근 메시지 등 허용된 대화 Context
- 추천 개수, 길이, 언어 옵션

## Output

- 추천 문장 목록
- 추천 유형
- 원 요청과 연결할 execution identifier

## Current Gap

현재 구현은 최근 메시지가 비어 있고 Conversation Start 전용 HTTP 계약을 사용한다. 공통 AiTask와 result delivery 연결이 필요하다.

## Related

- [Conversation Start Use Case](../../use-cases/01-conversation-start/README.md)

