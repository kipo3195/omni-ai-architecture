# Current Service Boundary

> Role: 현재 배포된 Messenger Client 접점과 Omni AI 서비스의 물리·논리 경계를 정의한다.
> 서비스 분리·통합을 가정한 미래 배포 구조는 이 프로젝트의 범위에 포함하지 않는다.

---

## 1. Physical Services

```text
Web Client
→ WebSocket
→ WebSocket Service (WS)

Native Client
→ TCP
→ TCP Service (NS)
```

- `WebSocket Service (WS)`는 Web Client의 connection, session, request ingress와 delivery를 소유하며, 현재 여기에 구현된 User / Authentication / File을 포함한 Messenger Business Logic을 유지한다.
- `TCP Service (NS)`는 Native Client의 TCP connection, session, legacy protocol ingress와 delivery를 소유한다.
- 현재 Messenger Business Logic과 DB / Queue 접근 책임은 실제 구현을 보유한 `WebSocket Service (WS)` 또는 `TCP Service (NS)`에 남는다.

---

## 2. Messenger Client Ingress

`Messenger Client Ingress`는 `WebSocket Service (WS)`와 `TCP Service (NS)`를 포괄하는 논리적 경계이다. 별도 배포 서비스나 추가 network hop을 의미하지 않는다.

공통 책임:

- Client protocol ingress
- 인증된 Client / tenant / session / connection context 구성
- 소유한 connection과 Client 상태 전이 확정
- Business Event / Client AI Request를 AI Orchestrator에 전달
- Connection Registry 등록과 local connection lookup
- AI Result / Client Tool Request의 최종 Client delivery

`Messenger Client Ingress`는 AI 기능별 정책, Cross-domain Context Assembly, AiTask, execution lifecycle을 소유하지 않는다.

---

## 3. Role Names

Use Case와 Component 문서에서는 미래 서비스 이름 대신 현재 책임을 나타내는 역할 이름을 사용한다.

| Role | Meaning |
| --- | --- |
| Presence owner | 현재 Presence 상태와 전이를 구현·저장하는 Messenger Service |
| Message owner | 현재 메시지 DB, 조회 Queue와 read/delete 규칙을 구현한 Messenger Service |
| Authentication owner | 현재 Client / tenant / session 인증 규칙을 구현한 Messenger Service |
| File owner | 현재 파일 metadata, content, permission을 구현한 Messenger Service |
| Connection owner | 현재 WebSocket 또는 TCP local connection을 소유한 service instance |

역할 이름은 별도 microservice의 존재를 의미하지 않는다. 현재 물리 구현을 기록할 때는 `WebSocket Service (WS)` 또는 `TCP Service (NS)`와 역할을 함께 표시한다.

---

## 4. Omni AI Boundary

```text
Messenger Client Ingress
→ Business Event / Client AI Request
→ AI Orchestrator
→ AiTask
→ Omni AI Server
→ Result Router
→ Messenger Client Ingress
→ Client
```

- `AI Orchestrator`는 AI 기능 정책, Context Assembly, execution correlation, Tool lifecycle과 Result Routing을 조정한다.
- `Omni AI Server`는 Prompt, Workflow / Agent, LLM 실행과 Runtime State를 소유한다.
- 사용자·메시지·파일·Presence 원본 상태는 AI Orchestrator가 소유하지 않는다.

---

## 5. Identifier Naming

문서의 공통 식별자는 lower camel case를 사용한다.

| Canonical name | 현재 또는 legacy 표현 | 규칙 |
| --- | --- | --- |
| `tenantId` | - | 문서와 신규 계약에서 `tenantId` 사용 |
| `userId` | `userID` | `userID`는 현재 API를 정확히 인용할 때만 사용하고 신규 계약은 `userId` 사용 |
| `roomId` | `roomKey` | 동일 의미라고 자동 가정하지 않고 Service adapter에서 의미와 변환을 명시 |
| `clientSessionId` | JWT `sid` | WebSocket 인증 세션의 논리 target이며 `connectionId`와 구분 |
| `deviceId` | - | 신뢰 가능한 장치 식별자가 있는 Client에서만 사용 |
| `clientId` | - | Proposed ADR 004의 일반 명칭이므로 현재 routing 계약의 `clientSessionId`나 `deviceId`를 대체하지 않음 |

현재 API 또는 코드의 필드명이 canonical name과 다르면 Component 문서에 mapping을 기록한다. 서로 lifecycle이 다른 식별자를 이름이 비슷하다는 이유로 통합하지 않는다.

---

## 6. Documentation Rules

- 현재 배포 단위는 `WebSocket Service (WS)`와 `TCP Service (NS)`로 기록한다.
- 두 Client 접점을 포괄할 때는 `Messenger Client Ingress`를 사용한다.
- 물리 서비스가 아닌 책임은 `Presence owner`, `Message owner` 같은 역할로 표현한다.
- 현재 존재하지 않는 서비스를 현재 Architecture, Use Case, Component flow에 사용하지 않는다.
- 서비스 분리·통합 설계는 이 Repository의 범위 밖으로 둔다.
