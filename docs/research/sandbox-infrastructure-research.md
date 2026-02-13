# 샌드박스 인프라 구축 자료조사

> **문서 유형**: 기술 리서치
> **작성일**: 2026-02-13
> **관련 RFC**: RFC-004 (AI Agent Sandbox Isolation)

---

## 1. Executive Summary

KILLHOUSE 플랫폼의 AI 기반 취약점 스캐너는 **신뢰할 수 없는 AI 생성 코드**를 실행해야 합니다. 이를 위해 강력한 샌드박스 인프라가 필수적입니다. 본 문서는 샌드박스 격리 기술의 비교 분석과 KILLHOUSE에 적합한 아키텍처를 제안합니다.

### 핵심 결론

| 구분 | 현재 설계 (RFC-004) | 권장 개선안 |
|------|---------------------|-------------|
| **격리 기술** | Docker-in-Docker | Firecracker microVM 또는 gVisor |
| **네트워크** | `--network none` | ✅ 유지 (DENY ALL) |
| **리소스 제한** | cgroup + PID limit | ✅ 유지 + io_uring 차단 |
| **런타임** | Docker | Podman (rootless) 또는 Firecracker |

---

## 2. 컨테이너 샌드박스 격리 기술 비교

### 2.1 기술 개요

```
┌─────────────────────────────────────────────────────────────────┐
│                      격리 수준 스펙트럼                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  약함 ◄────────────────────────────────────────────────► 강함   │
│                                                                 │
│  Container    gVisor      Kata         Firecracker    Full VM   │
│  (Docker)   (App Kernel) (Light VM)   (microVM)      (QEMU)    │
│                                                                 │
│  ~10ms       50-100ms    150-300ms    100-200ms      10+ sec   │
│  공유 커널   유저스페이스  VM 격리      microVM 격리   완전 VM   │
│              커널 에뮬                               격리       │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 상세 비교

| 기술 | 아키텍처 | 시작 시간 | 보안 수준 | 오버헤드 | 적합 사례 |
|------|----------|-----------|-----------|----------|-----------|
| **Docker/Podman** | 공유 커널 + namespace | ~10ms | 중간 | 최소 | 신뢰할 수 있는 코드 |
| **gVisor** | 유저스페이스 커널 | 50-100ms | 높음 | 중간 (56% 성능 저하) | 신뢰할 수 없는 코드 |
| **Kata Containers** | 경량 VM + OCI 호환 | 150-300ms | 매우 높음 | 높음 | 멀티테넌트 SaaS |
| **Firecracker** | microVM (Rust) | 100-200ms | 매우 높음 | 낮음 (5MB) | 서버리스, FaaS |

### 2.3 각 기술 상세

#### gVisor
```
┌─────────────────────────────────────────┐
│              Application                │
├─────────────────────────────────────────┤
│         Sentry (User-space Kernel)      │
│      - Go로 작성된 syscall 에뮬레이터      │
│      - ~70-80% syscall 지원              │
├─────────────────────────────────────────┤
│             Host Kernel                 │
└─────────────────────────────────────────┘
```

- **장점**:
  - 하드웨어 가상화 불필요 (nested virtualization 지원 안 되는 환경에서 사용 가능)
  - 기존 컨테이너 워크플로우 유지
  - 커널 공격 표면 대폭 감소

- **단점**:
  - 95% (ptrace) ~ 56% (KVM) 성능 저하
  - eBPF, 고급 ioctl 미지원
  - 일부 애플리케이션 호환성 문제

- **사용 사례**: Google Cloud Run, GKE Sandbox

#### Firecracker
```
┌─────────────────────────────────────────┐
│            Guest Application            │
├─────────────────────────────────────────┤
│            Guest Linux Kernel           │
├─────────────────────────────────────────┤
│    Firecracker VMM (50K lines Rust)     │
│       - 최소화된 디바이스 에뮬레이션        │
│       - 24 syscall whitelist            │
├─────────────────────────────────────────┤
│   Jailer (cgroups + namespaces + seccomp)│
├─────────────────────────────────────────┤
│               Host Kernel               │
└─────────────────────────────────────────┘
```

- **장점**:
  - AWS Lambda, Fargate에서 검증된 기술
  - 125ms 부팅, 초당 150개 microVM 생성 가능
  - 5MB 메모리 오버헤드
  - 하드웨어 격리 (각 워크로드별 전용 커널)

- **단점**:
  - KVM 지원 필수 (VT-x/AMD-V)
  - 오케스트레이션 인프라 직접 구축 필요
  - EC2 nested virtualization 필요

- **사용 사례**: AWS Lambda, AWS Fargate, E2B, Fly.io

#### Kata Containers
```
┌─────────────────────────────────────────┐
│        Kubernetes / containerd          │
├─────────────────────────────────────────┤
│           Kata Runtime Shim             │
├─────────────────────────────────────────┤
│    VMM (Firecracker/Cloud Hypervisor)   │
├─────────────────────────────────────────┤
│              Guest VM                   │
│  ┌───────────────────────────────────┐  │
│  │         Guest Kernel              │  │
│  ├───────────────────────────────────┤  │
│  │         kata-agent                │  │
│  ├───────────────────────────────────┤  │
│  │     Container Runtime (runc)      │  │
│  ├───────────────────────────────────┤  │
│  │        Application                │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

- **장점**:
  - OCI/CRI 호환 (Kubernetes 네이티브)
  - VMM 선택 가능 (Firecracker, Cloud Hypervisor, QEMU)
  - 프로덕션 레디 오케스트레이션

- **단점**:
  - 복잡한 설정
  - 높은 리소스 오버헤드

- **사용 사례**: OpenShift Sandboxed Containers, 멀티테넌트 Kubernetes

---

## 3. AI 코드 실행 샌드박스 플랫폼 분석

### 3.1 주요 플랫폼 비교

| 플랫폼 | 격리 기술 | 시작 시간 | 특징 | 가격 |
|--------|-----------|-----------|------|------|
| **E2B** | Firecracker | ~150ms | AI 에이전트 특화, 오픈소스 | 사용량 기반 |
| **Daytona** | Docker/Kata/Sysbox | 27-90ms | 상태 유지, 빠른 시작 | 엔터프라이즈 |
| **Modal** | gVisor | <1s | ML 워크로드 최적화, GPU 지원 | 사용량 기반 |
| **Docker Sandbox** | Firecracker microVM | - | Claude Code 등 AI 에이전트용 | 새로운 제품 |

### 3.2 E2B 아키텍처 (참고용)

```
┌──────────────────────────────────────────────────────┐
│                    E2B Architecture                   │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌─────────────┐     ┌─────────────────────────────┐ │
│  │   AI App    │     │      E2B Control Plane      │ │
│  │  (SDK)      │────▶│   - Kubernetes 오케스트레이션  │ │
│  └─────────────┘     │   - 샌드박스 라이프사이클 관리  │ │
│                      └─────────────────────────────┘ │
│                                 │                    │
│                                 ▼                    │
│  ┌──────────────────────────────────────────────────┐│
│  │              Firecracker microVMs                ││
│  │  ┌────────┐  ┌────────┐  ┌────────┐            ││
│  │  │ VM #1  │  │ VM #2  │  │ VM #N  │            ││
│  │  │ Agent  │  │ Agent  │  │ Agent  │            ││
│  │  │ Code   │  │ Code   │  │ Code   │            ││
│  │  └────────┘  └────────┘  └────────┘            ││
│  └──────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────┘
```

**E2B 선택 이유**:
- Firecracker microVM으로 하드웨어 수준 격리
- 각 실행마다 전용 Linux 커널 제공
- 컨테이너 탈출 공격 차단
- AWS Lambda와 동일한 기술 기반

---

## 4. 보안 위협 및 완화 전략

### 4.1 주요 위협 분석

| 위협 | 설명 | 완화 방법 |
|------|------|-----------|
| **컨테이너 탈출** | 커널 취약점을 통한 호스트 접근 | microVM, gVisor, seccomp |
| **네트워크 공격** | C2 서버 연결, 데이터 유출 | `--network none`, 방화벽 |
| **리소스 고갈 (DoS)** | Fork bomb, 메모리 폭주 | cgroup, PID limit, timeout |
| **Prompt Injection** | AI 조작으로 악성 코드 생성 | 기술적 격리 (프롬프트로 방어 불가) |
| **Supply Chain 공격** | 악성 패키지 설치 | 네트워크 차단, 화이트리스트 |
| **CVE-2025-9074** | Docker Desktop SSRF | 샌드박스 네트워크 격리 |

### 4.2 방어 심층 (Defense in Depth)

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 5: 시간 제한                        │
│                    - 300초 실행 timeout                      │
│                    - 무한 루프 방지                          │
├─────────────────────────────────────────────────────────────┤
│                    Layer 4: 파일시스템                       │
│                    - read-only root                         │
│                    - tmpfs /tmp (10GB)                      │
│                    - 볼륨 마운트 금지                         │
├─────────────────────────────────────────────────────────────┤
│                    Layer 3: 리소스 제한                      │
│                    - 4GB RAM                                │
│                    - 2 vCPU                                 │
│                    - 256 PID limit                          │
├─────────────────────────────────────────────────────────────┤
│                    Layer 2: 컨테이너 격리                    │
│                    - seccomp profile                        │
│                    - no-new-privileges                      │
│                    - dropped capabilities                   │
├─────────────────────────────────────────────────────────────┤
│                    Layer 1: 네트워크 격리                    │
│                    - --network none                         │
│                    - Security Group DENY ALL                │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 seccomp 프로파일 권장 사항

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "read", "write", "open", "close", "stat", "fstat",
        "mmap", "mprotect", "munmap", "brk",
        "rt_sigaction", "rt_sigprocmask",
        "ioctl", "access", "pipe", "select",
        "sched_yield", "mremap", "msync",
        "mincore", "madvise", "shmget", "shmat",
        "clone", "fork", "vfork", "execve",
        "exit", "wait4", "kill", "uname",
        "fcntl", "flock", "fsync", "fdatasync",
        "getcwd", "chdir", "mkdir", "rmdir",
        "link", "unlink", "chmod", "chown",
        "getpid", "getuid", "getgid",
        "gettimeofday", "clock_gettime"
      ],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "comment": "io_uring 차단 (CVE 다수)",
      "names": [
        "io_uring_setup",
        "io_uring_enter",
        "io_uring_register"
      ],
      "action": "SCMP_ACT_ERRNO",
      "errnoRet": 1
    },
    {
      "comment": "위험한 syscall 명시적 차단",
      "names": [
        "ptrace", "kexec_load", "kexec_file_load",
        "init_module", "finit_module", "delete_module",
        "acct", "settimeofday", "stime",
        "mount", "umount", "umount2", "pivot_root",
        "swapon", "swapoff", "reboot"
      ],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```

**핵심 권장사항**:
1. **io_uring 차단 필수** - 최근 커널 취약점의 주요 원인
2. **기본 거부 정책** - 필요한 syscall만 화이트리스트
3. **대부분의 컨테이너는 40-70개 syscall만 필요** (기본 Docker는 300+개 허용)

---

## 5. KILLHOUSE 권장 아키텍처

### 5.1 아키텍처 옵션 비교

| 옵션 | 격리 수준 | 복잡도 | 비용 | 권장 |
|------|-----------|--------|------|------|
| **Option A: Docker + seccomp 강화** | 중간 | 낮음 | $ | 초기 MVP |
| **Option B: Podman + gVisor** | 높음 | 중간 | $$ | ✅ 추천 |
| **Option C: Firecracker 자체 구축** | 매우 높음 | 높음 | $$$ | 장기 목표 |
| **Option D: E2B/Daytona 사용** | 매우 높음 | 낮음 | $$$$ | 빠른 출시 |

### 5.2 권장 아키텍처 (Option B: Podman + gVisor)

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS VPC (Private Subnet)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   EC2 (Exploit Agent Host)                │   │
│  │                   Amazon Linux 2023                       │   │
│  │                                                           │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │              Podman (Rootless Mode)                  │ │   │
│  │  │                                                      │ │   │
│  │  │  ┌─────────────────────────────────────────────┐    │ │   │
│  │  │  │           gVisor Runtime (runsc)             │    │ │   │
│  │  │  │                                              │    │ │   │
│  │  │  │  ┌───────────────────────────────────────┐  │    │ │   │
│  │  │  │  │         Sandbox Container             │  │    │ │   │
│  │  │  │  │  - AI 생성 exploit 코드 실행            │  │    │ │   │
│  │  │  │  │  - --network none                     │  │    │ │   │
│  │  │  │  │  - read-only rootfs                   │  │    │ │   │
│  │  │  │  │  - tmpfs /tmp                         │  │    │ │   │
│  │  │  │  │  - 300s timeout                       │  │    │ │   │
│  │  │  │  └───────────────────────────────────────┘  │    │ │   │
│  │  │  │                                              │    │ │   │
│  │  │  │  Sentry (User-space Kernel)                 │    │ │   │
│  │  │  │  - syscall 인터셉션 및 에뮬레이션            │    │ │   │
│  │  │  └─────────────────────────────────────────────┘    │ │   │
│  │  │                                                      │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  │                                                           │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │              Exploit Agent Container                 │ │   │
│  │  │  - 샌드박스 오케스트레이션                            │ │   │
│  │  │  - OpenAI API 통신                                  │ │   │
│  │  │  - 결과 수집 및 보고                                 │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Security Group: sg-agent                                        │
│  - Inbound: 8081 from ECS only                                  │
│  - Outbound: 443 (OpenAI), 5432 (Supabase)                      │
│                                                                  │
│  Security Group: sg-sandbox (implicit via --network none)        │
│  - All traffic: DENY                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 Podman + gVisor 설정

```bash
# gVisor 설치 (EC2 user_data.sh)
wget https://storage.googleapis.com/gvisor/releases/release/latest/x86_64/runsc
chmod +x runsc
sudo mv runsc /usr/local/bin/

# gVisor OCI 런타임 등록
sudo tee /usr/share/containers/containers.conf.d/gvisor.conf << 'EOF'
[engine]
runtime = "runsc"

[engine.runtimes]
runsc = ["/usr/local/bin/runsc"]
EOF

# rootless Podman 설정
loginctl enable-linger ec2-user
```

```bash
# 샌드박스 컨테이너 실행 예시
podman run \
  --runtime runsc \
  --rm \
  --network none \
  --memory 4g \
  --cpus 2 \
  --pids-limit 256 \
  --read-only \
  --tmpfs /tmp:size=10g \
  --security-opt no-new-privileges \
  --security-opt seccomp=/etc/killhouse/seccomp.json \
  --user 65534:65534 \
  killhouse/exploit-sandbox:latest \
  timeout 300 /entrypoint.sh
```

### 5.4 단계별 구현 로드맵

```
Phase 1: MVP (현재 설계 기반)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Docker-in-Docker on EC2
✅ --network none
✅ cgroup 리소스 제한
✅ 기본 seccomp 프로파일
⏱️ 예상 기간: 구현 완료

Phase 2: 보안 강화
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
□ Docker → Podman (rootless) 마이그레이션
□ 커스텀 seccomp 프로파일 (io_uring 차단)
□ 상세 syscall 감사 로깅
□ Resource Monitor 고도화
⏱️ 예상 기간: TBD

Phase 3: gVisor 도입
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
□ gVisor (runsc) 런타임 설치
□ 호환성 테스트 (exploit 코드 실행)
□ 성능 벤치마크
□ 프로덕션 롤아웃
⏱️ 예상 기간: TBD

Phase 4: Firecracker 마이그레이션 (Optional)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
□ Firecracker VMM 구축
□ Jailer 설정
□ 커스텀 게스트 커널 빌드
□ 오케스트레이션 레이어 개발
⏱️ 예상 기간: TBD (보안 감사 후 결정)
```

---

## 6. AWS Fargate 샌드박스 가능성 검토

### 6.1 Fargate의 보안 모델

AWS Fargate는 이미 Firecracker microVM을 사용합니다:

```
┌─────────────────────────────────────────┐
│          Customer Task/Pod              │
│  ┌─────────────────────────────────────┐│
│  │         Container(s)                ││
│  └─────────────────────────────────────┘│
├─────────────────────────────────────────┤
│       Dedicated Guest Kernel            │
├─────────────────────────────────────────┤
│         Firecracker microVM             │
├─────────────────────────────────────────┤
│    EC2 Instance (AWS 관리)               │
└─────────────────────────────────────────┘
```

**Fargate 보안 특성**:
- 태스크당 전용 microVM (다른 고객과 하드웨어 공유 없음)
- 호스트 파일시스템, 디바이스, 네트워킹 접근 차단
- privileged 모드 불가
- CAP_SYS_ADMIN, CAP_NET_ADMIN 제한

### 6.2 Fargate를 샌드박스로 사용하지 않는 이유

| 문제점 | 설명 |
|--------|------|
| **네트워크 차단 불가** | Fargate는 `--network none` 지원 안 함 |
| **컨트롤 부족** | seccomp 프로파일 커스터마이징 제한 |
| **비용** | 짧은 실행에 과금 비효율 |
| **Cold Start** | 샌드박스 생성에 수 초 소요 |

**결론**: Fargate는 scanner-api, web-client 같은 장기 실행 서비스에 적합하지만, exploit 코드 실행용 샌드박스로는 부적합합니다.

---

## 7. 비용 분석

### 7.1 인프라 비용 (월간)

| 구성 요소 | 현재 (Docker) | gVisor 추가 | Firecracker |
|-----------|---------------|-------------|-------------|
| EC2 (t3.medium) | $30 | $30 | $45 (larger) |
| EBS 스토리지 | $10 | $10 | $15 |
| NAT Gateway | $35 | $35 | $35 |
| 데이터 전송 | $5 | $5 | $5 |
| **합계** | **$80** | **$80** | **$100** |

### 7.2 외부 서비스 비용 (vs 자체 구축)

| 서비스 | 비용 | 장점 | 단점 |
|--------|------|------|------|
| **E2B** | $0.0001/초 | 즉시 사용, 검증된 보안 | 의존성, 장기 비용 |
| **Daytona** | 엔터프라이즈 | 빠른 시작, 상태 유지 | 비용 불투명 |
| **자체 구축** | $80-100/월 | 완전한 통제, 낮은 비용 | 개발/운영 비용 |

**권장**: 초기에는 자체 구축 (Option B), 트래픽 증가 시 E2B 하이브리드 검토

---

## 8. 참고 자료

### 격리 기술 비교
- [Kata Containers vs Firecracker vs gVisor](https://northflank.com/blog/kata-containers-vs-firecracker-vs-gvisor)
- [Firecracker vs gVisor](https://northflank.com/blog/firecracker-vs-gvisor)
- [Making Containers More Isolated](https://unit42.paloaltonetworks.com/making-containers-more-isolated-an-overview-of-sandboxed-container-technologies/)
- [Choosing a Workspace for AI Agents](https://medium.com/@iSoftStone/choosing-a-workspace-for-ai-agents-the-ultimate-showdown-between-gvisor-kata-and-firecracker-46a8528ae37c)

### Docker 보안
- [Docker Security Announcements](https://docs.docker.com/security/security-announcements/)
- [CVE-2025-9074 Explained](https://pvotal.tech/breaking-dockers-isolation-using-docker-cve-2025-9074/)
- [Docker Sandboxes for AI](https://www.docker.com/blog/docker-sandboxes-a-new-approach-for-coding-agent-safety/)
- [Docker Security Best Practices for AWS](https://medium.com/@Nelsonalfonso/docker-security-best-practices-for-using-aws-2025-15dbc8c64395)

### AWS Fargate
- [AWS Fargate Security Overview](https://d1.awsstatic.com/whitepapers/AWS_Fargate_Security_Overview_Whitepaper.pdf)
- [Fargate Security Considerations](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-security-considerations.html)
- [ECS Fargate Threat Modeling](https://www.sysdig.com/blog/ecs-fargate-threat-modeling/)

### AI 샌드박스 플랫폼
- [E2B Documentation](https://e2b.dev/docs)
- [Top AI Code Sandbox Products](https://modal.com/blog/top-code-agent-sandbox-products)
- [Daytona vs E2B Comparison](https://northflank.com/blog/daytona-vs-e2b-ai-code-execution-sandboxes)
- [Awesome Sandbox for AI](https://github.com/restyler/awesome-sandbox)

### seccomp
- [Docker seccomp Profiles](https://docs.docker.com/engine/security/seccomp/)
- [Container Security Fundamentals: seccomp](https://securitylabs.datadoghq.com/articles/container-security-fundamentals-part-6/)
- [Kubernetes seccomp Tutorial](https://kubernetes.io/docs/tutorials/security/seccomp/)
- [Securing Containers with Seccomp](https://blog.gitguardian.com/securing-containers-with-seccomp/)

---

## 9. 결론 및 다음 단계

### 핵심 권장사항

1. **즉시 적용**: 커스텀 seccomp 프로파일 작성 (io_uring 차단)
2. **단기**: Docker → Podman rootless 마이그레이션
3. **중기**: gVisor 런타임 도입으로 커널 공격 표면 감소
4. **장기**: 보안 감사 후 Firecracker 마이그레이션 검토

### 다음 단계

- [ ] RFC-004 업데이트: seccomp 프로파일 상세 정의
- [ ] Podman + gVisor PoC 구현
- [ ] 성능 벤치마크 수행
- [ ] 보안 감사 일정 수립

---

*이 문서는 KILLHOUSE 샌드박스 인프라 설계를 위한 기술 자료조사입니다. RFC-004와 함께 검토하시기 바랍니다.*
