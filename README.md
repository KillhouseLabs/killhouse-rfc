# killhouse-docs

KILLHOUSE 시스템 아키텍처 및 RFC 문서.

MkDocs Material + Mermaid 기반, GitHub Pages로 자동 배포.

## Local Preview

```bash
pip install -r requirements.txt
mkdocs serve
# http://127.0.0.1:8000
```

## Deploy

`main` 브랜치에 push하면 GitHub Actions가 자동으로 GitHub Pages에 배포합니다.

### 최초 설정

1. GitHub에 `killhouse-docs` 레포 생성
2. Settings > Pages > Source를 `gh-pages` 브랜치로 설정
3. `mkdocs.yml`에서 `site_url`과 `repo_url`의 username 수정
4. push

```bash
git init
git remote add origin git@github.com:YOUR_USERNAME/killhouse-docs.git
git add .
git commit -m "init: mkdocs project"
git push -u origin main
```

## Structure

```
killhouse-docs/
├── mkdocs.yml              # MkDocs 설정
├── requirements.txt        # Python 의존성
├── .github/workflows/
│   └── deploy.yml          # GitHub Pages 자동 배포
└── docs/
    ├── index.md            # Home
    ├── architecture/
    │   ├── overview.md     # 논리 아키텍처
    │   ├── aws-infrastructure.md  # 인프라 다이어그램 + OSI
    │   └── module-reference.md    # 모듈 명칭 테이블
    └── rfc/
        ├── index.md        # RFC 목록
        ├── rfc-001-network-topology.md
        ├── rfc-002-communication-protocol.md
        ├── rfc-003-queue-architecture.md
        └── rfc-004-sandbox-isolation.md
```
