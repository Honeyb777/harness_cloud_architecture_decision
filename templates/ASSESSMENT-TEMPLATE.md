# Assessment — <검토 목적 / 범위>

## Document Control

- Assessment ID: ASM-YYYY-NNN
- Status: PROPOSED
- Current Step: STEP-01
- Phase: INTAKE
- Created:
- Updated:
- Owner:
- Source requirements: requirements.md
- Context: context.md
- Environments / Workloads:
- Related assessments:
- Artifact links: decisions/, designs/, evidence/, evaluations/, reports/, history/

상태와 전환 기준은 `sot/WORKFLOW.md`를 따른다. 각 Step에 Phase, Decision links, Validation evidence, Completion Gate 결과를 기록한다.
대안 비교는 해당 Step에서 수행하며 STEP-09에서 통합한다.

## 1. Intake

- Business owner:
- Technical owner:
- Department:
- Services / Systems:
- Environments (Cloud / IDC / Hybrid, context.md 참조):
- Expected users:
- Expected regions:
- Data classification:
- Compliance:
- Budget:
- Target date:

## 2. Requirements

## 3. Current State

## 4. Constraints

## 5. Unknowns

| ID | Unknown | Owner | 확인 방법 | 차단 Step / Gate |
|---|---|---|---|---|

## 6. 역할 선택 및 평가

| Step / 작업 | 역할 | 선택 / 제외 근거 | 담당 산출물 | 사용 후 평가 링크 | 개선 상태 |
|---|---|---|---|---|---|

모든 역할 작업 종료 시 평가한다. DevOps, 네트워크/IDC 연동의 적용 여부를 Discovery에서 확인하고 해당 없음은 근거를 기록한다.

## 7. Decision Steps

### STEP-01 — Organization / Scope / Account / Subscription

Status: PROPOSED

Decisions:
- 포함 환경·워크로드·IDC 사이트 및 책임 경계
- Tenant / Organization 구조 (해당 시)
- Account / Subscription 분리 기준
- Department / Service ownership
- Billing / Cost center

### STEP-02 — Region / Global Strategy

Status: BLOCKED_BY_STEP_01

Decisions:
- Primary Region
- Secondary Region
- Data residency
- Multi-region 필요성
- Global routing

### STEP-03 — Identity / OIDC / Access

Status: BLOCKED_BY_STEP_02

Decisions:
- Human identity
- Workload identity
- Federation
- OIDC
- Kubernetes workload access
- Privileged access

### STEP-04 — Landing Zone / Network / IDC Connectivity

Status: BLOCKED_BY_STEP_03

Decisions:
- Hub/Spoke 또는 대안
- VNet/VPC 구조
- Private connectivity
- Shared services
- DNS
- Egress/Ingress
- Policy boundary
- IDC/Cloud 통신 매트릭스, CIDR 중복, DNS·라우팅·NAT·TLS
- 기존 DB·인증·파일·배치·API 의존성과 연결 책임
- 회선/VPN 대안, 대역폭·지연·비용, 이중화·장애 전환
- 연결 시험 및 cutover/rollback 계획

### STEP-05 — Compute / Platform / Application Runtime / DevOps

Status: BLOCKED_BY_STEP_04

Decisions:
- Kubernetes 필요성
- Managed Kubernetes vs 대안
- Cluster 분리
- Namespace/tenant 분리
- Autoscaling
- Upgrade strategy
- Java·Kotlin/JDK/Tomcat, Vue/Node.js/npm 등 실제 스택 호환성
- WAR/JAR/정적 파일/SSR, VM·컨테이너 등 실행 방식
- CI/CD 도구·runner 위치·망 접근·artifact 저장소
- 빌드·테스트·공급망 검사·버전 고정·환경별 artifact 승격
- 배포 승인·OIDC/비밀값·DB migration·rollback 및 책임 경계

### STEP-06 — Data / Storage / Database

Status: BLOCKED_BY_STEP_05

Decisions:
- Database
- Storage
- Encryption
- Backup
- Retention
- Lifecycle tier
- Restore validation
- RPO/RTO

### STEP-07 — Security / Audit / Observability

Status: BLOCKED_BY_STEP_06

Decisions:
- Logging
- Audit
- SIEM integration
- Retention
- Security posture
- Alerting
- Evidence

### STEP-08 — FinOps / Tagging

Status: BLOCKED_BY_STEP_07

Decisions:
- Required tags
- Cost allocation
- Budget alert
- Commitment model
- Non-prod scheduling
- Storage lifecycle savings
- Egress controls

### STEP-09 — Alternatives / Final Architecture

Status: BLOCKED_BY_STEP_08

- 최종 대안 비교
- 선택안
- 보류안
- 제외안과 제외 이유

### STEP-10 — Validation / Report

Status: BLOCKED_BY_STEP_09

- Architecture validation
- 환경별 배포·복구 및 IDC 연동 검증 증적
- 역할 평가 및 필수 누락 해소 확인
- Decision records
- Executive one-page
- Full report
