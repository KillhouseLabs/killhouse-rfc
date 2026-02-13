# RFC-007: API Specification and Standards

| Field | Value |
|---|---|
| **RFC** | 007 |
| **Title** | API Specification and Standards |
| **Author** | TBD |
| **Status** | 📝 Draft |
| **Created** | 2025-XX-XX |
| **Depends on** | RFC-002, RFC-005 |
| **Blocks** | — |

---

## Summary

KILLHOUSE API Gateway의 REST API 설계 표준, 버전 관리 전략, 에러 코드 체계, Rate Limiting 정책, 요청/응답 포맷을 정의한다.

---

## Motivation

- API 설계 표준이 없으면 엔드포인트 간 일관성 부족
- 에러 코드가 표준화되지 않으면 클라이언트 에러 처리 복잡
- Rate limiting 없이는 DoS 공격 및 리소스 남용에 취약
- 버전 관리 전략 없이는 하위 호환성 유지 어려움

---

## Proposal

### API 버전 관리

**선택: URL Path 기반**

```
https://api.killhouse.io/v1/projects
https://api.killhouse.io/v1/analyses
```

| 방식 | 장점 | 단점 |
|---|---|---|
| **URL Path** (선택) | 명시적, 캐시 친화적, 디버깅 용이 | URL 변경 필요 |
| Header | URL 깔끔 | 디버깅 어려움 |
| Query Param | 구현 간단 | 비표준 |

!!! note "버전 정책"
    - Major 버전만 URL에 표시 (`v1`, `v2`)
    - Minor/Patch는 하위 호환 유지
    - 이전 버전은 최소 6개월 지원 후 deprecated

### 엔드포인트 명명 규칙

| 규칙 | 예시 |
|---|---|
| 복수형 명사 | `/projects`, `/analyses` |
| 케밥 케이스 | `/subscription-usage` (아님: `subscriptionUsage`) |
| 동사 지양 | `/projects` (아님: `/getProjects`) |
| 중첩 리소스 | `/projects/:id/repositories` |
| 액션은 동사 허용 | `/payment/checkout`, `/auth/logout` |

### HTTP 메서드 사용

| Method | 용도 | 멱등성 | 예시 |
|---|---|---|---|
| GET | 리소스 조회 | O | `GET /projects` |
| POST | 리소스 생성, 액션 실행 | X | `POST /projects` |
| PUT | 리소스 전체 수정 | O | `PUT /projects/:id` |
| PATCH | 리소스 부분 수정 | O | `PATCH /projects/:id` |
| DELETE | 리소스 삭제 | O | `DELETE /projects/:id` |

### 요청 포맷

```http
POST /v1/projects HTTP/1.1
Host: api.killhouse.io
Authorization: Bearer <jwt>
Content-Type: application/json
Accept: application/json

{
  "name": "My Security Project",
  "description": "Vulnerability scanning for my app"
}
```

| 헤더 | 필수 | 설명 |
|---|---|---|
| `Authorization` | 인증 필요 시 | `Bearer <jwt>` |
| `Content-Type` | POST/PUT/PATCH | `application/json` |
| `Accept` | 권장 | `application/json` |
| `X-Request-ID` | 선택 | 클라이언트 생성 추적 ID |

### 응답 포맷

#### 성공 응답

```json
{
  "success": true,
  "data": {
    "id": "proj_abc123",
    "name": "My Security Project",
    "createdAt": "2025-01-15T10:30:00Z"
  }
}
```

#### 목록 응답 (페이지네이션)

```json
{
  "success": true,
  "data": [
    { "id": "proj_1", "name": "Project 1" },
    { "id": "proj_2", "name": "Project 2" }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "totalPages": 3
  }
}
```

#### 에러 응답

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "요청 데이터가 올바르지 않습니다",
    "details": [
      { "field": "name", "message": "이름은 필수입니다" },
      { "field": "email", "message": "올바른 이메일 형식이 아닙니다" }
    ]
  },
  "requestId": "req_xyz789"
}
```

### 표준 에러 코드

#### HTTP 상태 코드 매핑

| HTTP | 코드 | 설명 |
|---|---|---|
| 400 | `BAD_REQUEST` | 잘못된 요청 |
| 400 | `VALIDATION_ERROR` | 입력 검증 실패 |
| 401 | `UNAUTHORIZED` | 인증 필요 |
| 401 | `TOKEN_EXPIRED` | 토큰 만료 |
| 401 | `INVALID_TOKEN` | 유효하지 않은 토큰 |
| 403 | `FORBIDDEN` | 권한 없음 |
| 403 | `LIMIT_EXCEEDED` | 구독 제한 초과 |
| 404 | `NOT_FOUND` | 리소스 없음 |
| 404 | `PROJECT_NOT_FOUND` | 프로젝트 없음 |
| 404 | `ANALYSIS_NOT_FOUND` | 분석 결과 없음 |
| 409 | `CONFLICT` | 리소스 충돌 |
| 409 | `DUPLICATE_EMAIL` | 이메일 중복 |
| 422 | `UNPROCESSABLE_ENTITY` | 처리 불가 |
| 429 | `RATE_LIMITED` | 요청 제한 초과 |
| 500 | `INTERNAL_ERROR` | 서버 내부 오류 |
| 502 | `UPSTREAM_ERROR` | 외부 서비스 오류 |
| 503 | `SERVICE_UNAVAILABLE` | 서비스 일시 중단 |

#### 도메인별 에러 코드

| 도메인 | 코드 | HTTP | 설명 |
|---|---|---|---|
| Auth | `INVALID_CREDENTIALS` | 401 | 이메일/비밀번호 불일치 |
| Auth | `EMAIL_NOT_VERIFIED` | 403 | 이메일 미인증 |
| Auth | `OAUTH_FAILED` | 400 | OAuth 인증 실패 |
| Project | `MAX_PROJECTS_REACHED` | 403 | 프로젝트 수 제한 |
| Analysis | `MAX_ANALYSES_REACHED` | 403 | 월간 분석 제한 |
| Analysis | `ANALYSIS_IN_PROGRESS` | 409 | 이미 분석 진행 중 |
| Payment | `PAYMENT_FAILED` | 400 | 결제 실패 |
| Payment | `ALREADY_SUBSCRIBED` | 409 | 이미 구독 중 |
| Repository | `REPO_ACCESS_DENIED` | 403 | 저장소 접근 권한 없음 |
| Repository | `REPO_NOT_FOUND` | 404 | 저장소 없음 |

### Rate Limiting

#### 제한 정책

| 플랜 | 요청/분 | 요청/시간 | 버스트 |
|---|---|---|---|
| **FREE** | 60 | 1,000 | 10 |
| **PRO** | 300 | 10,000 | 50 |
| **ENTERPRISE** | 1,000 | 100,000 | 200 |

| 엔드포인트 | 추가 제한 | 사유 |
|---|---|---|
| `POST /auth/login` | 10/분 | 브루트포스 방지 |
| `POST /auth/signup` | 5/분 | 스팸 계정 방지 |
| `POST /analyses` | 플랜별 월간 제한 | 리소스 보호 |
| `POST /payment/*` | 10/분 | 결제 어뷰징 방지 |

#### Rate Limit 헤더

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1704067200
Retry-After: 30
```

| 헤더 | 설명 |
|---|---|
| `X-RateLimit-Limit` | 제한 요청 수 |
| `X-RateLimit-Remaining` | 남은 요청 수 |
| `X-RateLimit-Reset` | 제한 리셋 시간 (Unix timestamp) |
| `Retry-After` | 재시도 가능 시간 (초) - 429 응답 시 |

#### 429 응답

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "요청 제한을 초과했습니다. 30초 후 다시 시도하세요.",
    "retryAfter": 30
  }
}
```

### 페이지네이션

#### 요청

```http
GET /v1/projects?page=2&limit=20&sort=createdAt&order=desc
```

| 파라미터 | 기본값 | 최대값 | 설명 |
|---|---|---|---|
| `page` | 1 | — | 페이지 번호 |
| `limit` | 20 | 100 | 페이지당 항목 수 |
| `sort` | `createdAt` | — | 정렬 필드 |
| `order` | `desc` | — | 정렬 순서 (asc/desc) |

#### 응답

```json
{
  "success": true,
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 45,
    "totalPages": 3,
    "hasNext": true,
    "hasPrev": true
  }
}
```

### 필터링 및 검색

```http
GET /v1/projects?status=active&search=security&createdAfter=2025-01-01
```

| 패턴 | 예시 | 설명 |
|---|---|---|
| 정확히 일치 | `status=active` | status가 active인 항목 |
| 검색 | `search=keyword` | 이름/설명에 키워드 포함 |
| 날짜 범위 | `createdAfter=2025-01-01` | 생성일 필터 |
| 다중 값 | `status=active,pending` | OR 조건 |

### 날짜/시간 포맷

| 항목 | 포맷 | 예시 |
|---|---|---|
| **요청** | ISO 8601 | `2025-01-15T10:30:00Z` |
| **응답** | ISO 8601, UTC | `2025-01-15T10:30:00.000Z` |
| **날짜만** | ISO 8601 | `2025-01-15` |

### API 엔드포인트 목록

#### 인증 (Auth)

| Method | Path | 설명 |
|---|---|---|
| POST | `/v1/auth/signup` | 회원가입 |
| POST | `/v1/auth/login` | 로그인 |
| POST | `/v1/auth/logout` | 로그아웃 |
| GET | `/v1/auth/me` | 현재 사용자 정보 |
| DELETE | `/v1/auth/account` | 계정 삭제 |

#### 프로젝트 (Projects)

| Method | Path | 설명 |
|---|---|---|
| GET | `/v1/projects` | 프로젝트 목록 |
| POST | `/v1/projects` | 프로젝트 생성 |
| GET | `/v1/projects/:id` | 프로젝트 상세 |
| PUT | `/v1/projects/:id` | 프로젝트 수정 |
| DELETE | `/v1/projects/:id` | 프로젝트 삭제 |

#### 저장소 (Repositories)

| Method | Path | 설명 |
|---|---|---|
| GET | `/v1/projects/:id/repositories` | 저장소 목록 |
| POST | `/v1/projects/:id/repositories` | 저장소 추가 |
| PUT | `/v1/projects/:id/repositories/:repoId` | 저장소 수정 |
| DELETE | `/v1/projects/:id/repositories/:repoId` | 저장소 삭제 |

#### 분석 (Analyses)

| Method | Path | 설명 |
|---|---|---|
| GET | `/v1/projects/:id/analyses` | 분석 목록 |
| POST | `/v1/projects/:id/analyses` | 분석 시작 |
| GET | `/v1/analyses/:id` | 분석 상세 |
| GET | `/v1/analyses/:id/report` | 분석 리포트 |

#### 구독 (Subscription)

| Method | Path | 설명 |
|---|---|---|
| GET | `/v1/subscription` | 현재 구독 정보 |
| GET | `/v1/subscription/usage` | 사용량 통계 |
| POST | `/v1/subscription/cancel` | 구독 취소 |

#### 결제 (Payment)

| Method | Path | 설명 |
|---|---|---|
| POST | `/v1/payment/checkout` | 결제 준비 |
| POST | `/v1/payment/verify` | 결제 검증 |
| GET | `/v1/payment/history` | 결제 내역 |
| POST | `/v1/payment/refund` | 환불 요청 |

#### 외부 연동 (Integrations)

| Method | Path | 설명 |
|---|---|---|
| GET | `/v1/github/repositories` | GitHub 저장소 목록 |
| GET | `/v1/github/repositories/:owner/:repo/branches` | GitHub 브랜치 목록 |
| GET | `/v1/gitlab/repositories` | GitLab 저장소 목록 |
| GET | `/v1/gitlab/repositories/:id/branches` | GitLab 브랜치 목록 |

---

## Alternatives

### Alt-1: GraphQL

단일 엔드포인트로 유연한 쿼리. 클라이언트 주도 데이터 fetching. **기각 사유**: 초기 단계에서 REST의 단순함 선호, 캐싱 용이.

### Alt-2: gRPC-Web

브라우저에서 gRPC 사용. 타입 안전성, 성능 우수. **기각 사유**: 프론트엔드 생태계 호환성, 디버깅 복잡도.

### Alt-3: Header 기반 버전 관리

`Accept: application/vnd.killhouse.v1+json`. URL 깔끔. **기각 사유**: 개발/디버깅 시 버전 확인 어려움.

---

## Security Considerations

!!! warning "고려사항"
    - **인젝션 방지**: 모든 입력 Zod 스키마 검증
    - **에러 정보 노출**: 프로덕션에서 상세 에러 숨김 (requestId로 로그 추적)
    - **Rate Limiting 우회**: IP + User ID 조합으로 제한
    - **열거 공격**: 404 응답 통일 (리소스 존재 여부 노출 방지)
    - **CORS**: 허용 오리진 명시 (`https://killhouse.io`)

---

## Open Issues

| # | 이슈 | 결정 필요 시점 |
|---|---|---|
| 1 | OpenAPI/Swagger 문서 자동 생성 도구 | API 개발 착수 시 |
| 2 | API 키 기반 인증 (CI/CD, 자동화 용) | 공개 API 제공 시 |
| 3 | Webhook 발송 기능 (분석 완료 알림) | 고객 요구 시 |
| 4 | 벌크 API (다중 리소스 처리) | 성능 요구사항 발생 시 |
| 5 | GraphQL 지원 여부 | 클라이언트 요구 증가 시 |

---

## References

- [REST API Design Best Practices](https://restfulapi.net/)
- [HTTP Status Codes](https://httpstatuses.com/)
- [JSON:API Specification](https://jsonapi.org/)
- [RFC-002 Communication Protocol](rfc-002-communication-protocol.md)
- [RFC-005 Authentication](rfc-005-authentication.md)
