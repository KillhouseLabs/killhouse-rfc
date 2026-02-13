# RFC Index

> KILLHOUSE 시스템 설계 결정을 문서화하는 RFC (Request for Comments) 목록

---

## 의존 관계

```
RFC-001 Network Topology
   └─▶ RFC-002 Communication Protocol
   │      └─▶ RFC-003 Queue Architecture
   │      │      └─▶ RFC-008 Project & Analysis Workflow
   │      └─▶ RFC-004 Sandbox Isolation
   │      │      └─▶ RFC-010 Sandbox Infrastructure Architecture
   │      └─▶ RFC-007 API Specification
   └─▶ RFC-005 Authentication
          └─▶ RFC-006 Billing
                 └─▶ RFC-008 Project & Analysis Workflow
                        └─▶ RFC-009 Auto Exploit Agent
```

RFC-001이 확정되어야 나머지를 구체화할 수 있습니다.

---

## RFC 목록

### 인프라 & 통신

| RFC | 제목 | 상태 | 작성일 |
|---|---|---|---|
| [RFC-001](rfc-001-network-topology.md) | Network Topology and Security Boundaries | 📝 Draft | 2025-XX-XX |
| [RFC-002](rfc-002-communication-protocol.md) | Inter-Service Communication Protocol | 📝 Draft | 2025-XX-XX |
| [RFC-003](rfc-003-queue-architecture.md) | Async Pipeline and Queue Architecture | 📝 Draft | 2025-XX-XX |
| [RFC-004](rfc-004-sandbox-isolation.md) | AI Agent Sandbox Isolation | 📝 Draft | 2025-XX-XX |

### 인증 & 결제

| RFC | 제목 | 상태 | 작성일 |
|---|---|---|---|
| [RFC-005](rfc-005-authentication.md) | Authentication and Authorization | 📝 Draft | 2025-XX-XX |
| [RFC-006](rfc-006-billing.md) | Billing and Subscription | 📝 Draft | 2025-XX-XX |

### API 표준

| RFC | 제목 | 상태 | 작성일 |
|---|---|---|---|
| [RFC-007](rfc-007-api-specification.md) | API Specification and Standards | 📝 Draft | 2025-XX-XX |

### 기능 워크플로우

| RFC | 제목 | 상태 | 작성일 |
|---|---|---|---|
| [RFC-008](rfc-008-project-analysis.md) | Project and Analysis Workflow | 📝 Draft | 2025-XX-XX |
| [RFC-009](rfc-009-auto-exploit-agent.md) | Autonomous Exploit Verification Agent | 📝 Draft | 2025-XX-XX |
| [RFC-010](rfc-010-sandbox-infrastructure.md) | Sandbox Infrastructure Architecture | 📝 Draft | 2026-02-13 |

---

## RFC 작성 가이드

각 RFC는 아래 구조를 따릅니다:

| 섹션 | 내용 |
|---|---|
| **Metadata** | 번호, 제목, 작성자, 상태, 날짜 |
| **Summary** | 한 문단 핵심 제안 |
| **Motivation** | 왜 이 결정이 필요한가 |
| **Proposal** | 구체적 설계 (다이어그램, 테이블 포함) |
| **Alternatives** | 검토했지만 채택하지 않은 방안 |
| **Security** | 위협 모델, 완화 방안 |
| **Open Issues** | 미해결 TBD 항목 |
| **References** | 관련 문서, 외부 레퍼런스 |

---

## Lifecycle

| 상태 | 의미 | 다음 단계 |
|---|---|---|
| 📝 Draft | 초안 작성 중 | → Review |
| 🔍 Review | 팀 리뷰 진행 | → Accepted 또는 Draft로 회귀 |
| ✅ Accepted | 채택 확정 | → Implemented |
| 🚀 Implemented | 구현 완료 | → Deprecated (필요 시) |
| ⛔ Deprecated | 새 RFC로 대체됨 | — |
