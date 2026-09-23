# Returned Message Topics Workflow

Status: Planned

자리비움 구간에 수신한 쪽지를 입력받아 발신자와 업무 주제 중심의 Topic Digest를 생성한다.

## Initial Contract

- 입력: 자리비움 구간, 대상 메시지, 발신자와 원본 식별자
- 출력: `senderName`, `topic`, `messageCount`, 원본 이동 식별자

## Quality Focus

- 원문에 없는 업무나 긴급도를 만들지 않는다.
- 짧고 구분 가능한 Topic 제목을 생성한다.
- 원본 메시지와의 추적 가능성을 유지한다.

## Related

- [Returned Message Topics Use Case](../../use-cases/02-returned-message-topics/README.md)

