# Client Request

Client의 명시적 AI 요청을 처리하는 AI Orchestrator의 공통 구조를 기록한다.

요청 수신, 실행 판단, Context 구성, AiTask 생성과 결과 계약을 실제 구현 기준으로 정리한다. 채널별 연결과 전달 책임은 각 Realtime Service의 경계로 둔다.
