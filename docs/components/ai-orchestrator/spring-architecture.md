# Spring Architecture

Status: Planned

---

## Purpose

Spring Boot 기반 AI Orchestrator의 공통 서비스 구조와 이관 과정을 기록한다.

기능 중심 package 구성, application boundary, 공통 설정과 외부 시스템 adapter 구조를 실제 구현과 검증 결과에 맞춰 정리한다.

## Scope

- 기능 중심 package와 module dependency 규칙
- application use case와 domain model 경계
- REST / Event ingress adapter
- DB / Redis / NATS / Omni AI Server outbound adapter
- transaction, configuration과 observability 공통 규칙

## Current Implementation Gap

현재 공통 package 규칙과 dependency 방향이 문서로 확정되지 않았다. 실제 `ai-orchestrator` 구조를 검증한 뒤 package 예시, 허용 dependency와 migration 순서를 기록한다.

## Related

- [AI Orchestrator Implementation](README.md)
- [ADR 003. AI Orchestrator as an Application Service](../../decisions/003-ai-orchestrator-application-service.md)
