# Module Reference

> 모든 문서와 다이어그램에서 사용하는 통일 명칭

---

## 명칭 테이블

| ID | 정규 명칭 | 다이어그램 ID | 설명 | 런타임 |
|---|---|---|---|---|
| M1 | **web-client** | `webcli` | Next.js 프론트엔드 (Auth, Dashboard, Billing, Report) | EC2 Docker |
| M2 | **api-gateway** | `apigw` | 메인 백엔드 — 인증, 프로젝트, 결제, 오케스트레이션 | EC2 Docker |
| M3 | **message-queue** | `mq` | 비동기 작업 큐 (Redis / RabbitMQ) | EC2 Docker |
| M4 | **vuln-scanner** | `scan` | SAST, SCA, 컨테이너, IaC 스캔 워커 | EC2 Docker |
| M5 | **runtime-agent** | `agent` | LLM AI 에이전트 — 코드리뷰, 익스플로잇, 수정, 리포트 | EC2 Docker |
| M6 | **sandbox** | `sbox` | Docker-in-Docker 격리 실행 환경 (RAM 4GB, STOR 10GB) | EC2 Docker |
| M7 | **knowledge-base** | `kb` | RAG 파이프라인 — 크롤링, 청크, 임베딩, 검색 | EC2 Docker |
| M8 | **cve-crawler** | `crawl` | NVD, GHSA, exploit-db 크롤러 (CronJob 7일 주기) | EC2 Docker |
| M9 | **resource-monitor** | `resmon` | 컨테이너 메트릭 수집, 임계치 경고, 구독 업그레이드 유도 | EC2 Docker |

---

## 외부 서비스

| ID | 명칭 | 다이어그램 ID | 설명 |
|---|---|---|---|
| N1 | **nginx-proxy** | `nginx` | Reverse Proxy + TLS 종단 (Let's Encrypt) |
| D1 | **supabase-pg** | `pg` | Supabase PostgreSQL |
| D2 | **supabase-pgvector** | `vec` | Supabase pgvector 벡터 검색 |
| X1 | **OpenAI API** | `openai` | GPT-4o (판단, 계획 전용) |
| X2 | **CVE Sources** | `cvesrc` | NVD, GHSA, exploit-db |
| X3 | **DDNS Provider** | `ddns` | Duck DNS / Cloudflare |

---

## 명명 규칙

!!! info "규칙"
    - **문서 본문**: 정규 명칭 사용 (예: `web-client`, `api-gateway`)
    - **Mermaid 다이어그램**: 다이어그램 ID 사용 (예: `webcli`, `apigw`)
    - **코드 저장소**: 정규 명칭을 디렉토리명으로 사용 (예: `web-client/`, `api-gateway/`)
