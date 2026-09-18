# Server Trigger

Business Event가 AI 실행으로 이어지는 AI Orchestrator의 공통 구조를 기록한다.

Event 수신, Trigger Policy, Context Assembly, AiTask 생성, 실행 연계의 책임과 흐름을 실제 구현 기준으로 정리한다. Conversation Start와 같은 기능은 이 구조의 적용 사례로 다루고, 기능 고유의 정책과 Context만 구분해 기록한다.
