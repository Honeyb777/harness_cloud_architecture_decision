# Microsoft 제품 소개·도입 협의 질문지 v2

- 대상: IDC 서비스의 Azure VM 전환 파일럿
- 작성일: 2026-09-08
- 용도: Microsoft 제품 소개와 도입 가능성 협의. 기술 면접이나 정답 검증이 아니라, 참여자가 같은 개념을 이해하고 선택에 필요한 정보를 받기 위한 질문지다.
- 현재 상태: 실제 GitHub, Azure, Azure DevOps 설정 및 배포 검증은 수행하지 않았다.

## 진행 방법

각 주제는 먼저 Microsoft 담당자가 제품 역할과 화면 또는 간단한 예시를 설명하고, 이어서 질문을 확인한다. 답변이 필요한 내용은 표의 `답변 메모`에 기록한다. 이번 자리에서 확정하지 못한 사항은 담당자, 확인 방법, 다음 일정만 정한다.

질문은 현재 검토 중인 운영 모델을 기준으로 한다. 특정 제품 구매나 설계 결정을 전제로 하지 않는다.

## 공통 용어

| 약어·용어 | 전체 이름 | 한국어 뜻 | 이 질문지에서의 의미 |
|---|---|---|---|
| CI | Continuous Integration | 지속적 통합 | 소스 변경을 build·test하여 release 후보를 만드는 과정 |
| CD | Continuous Delivery/Deployment | 지속적 제공·배포 | 검증된 release를 VM에 순서대로 적용하는 과정 |
| OIDC | OpenID Connect | 개방형 인증 연동 표준 | GitHub 또는 Azure DevOps가 비밀번호 없이 Azure 작업 권한을 받는 방식 |
| RBAC | Role-Based Access Control | 역할 기반 접근 제어 | Azure에서 identity가 할 수 있는 작업 범위를 역할로 제한하는 방식 |
| UAMI | User-Assigned Managed Identity | 사용자 할당 관리형 ID | Azure에서 별도 비밀번호 없이 workload에 부여하는 identity |
| VM | Virtual Machine | 가상 머신 | Java/Tomcat, Nginx, Vue 파일이 실행되는 Azure 서버 |
| ACR | Azure Container Registry | Azure 컨테이너 레지스트리 | Azure 안에서 build image를 보관하는 선택지 |
| GHCR | GitHub Container Registry | GitHub 컨테이너 레지스트리 | GitHub 안에서 private build image를 보관하는 선택지 |
| Blob | Azure Blob Storage | Azure 파일·객체 저장소 | CI 결과 release를 보관하고 VM이 내려받는 위치 |
| Probe | Health Probe | 상태 확인 요청 | Application Gateway가 VM이 요청을 받을 수 있는지 확인하는 요청 |
| Readiness | Readiness Check | 준비 상태 확인 | Tomcat과 필요한 의존성이 준비되어 traffic을 받을 수 있는지 확인 |
| Drain | Connection Draining | 기존 연결 비우기 | 새 요청을 멈춘 뒤 진행 중 요청이 끝날 시간을 주는 동작 |

---

## 1. GitHub Enterprise Cloud 권한·배포 보호 모델

**이 주제의 목적:** 코드 변경 권한, 배포 승인 권한, 자동 workflow 권한을 분리해 운영자가 이해할 수 있는 보호 구조를 확인한다.

| 질문 | Microsoft에게 요청할 설명 또는 시연 | 답변 메모 |
|---|---|---|
| GitHub Enterprise Cloud에서 개발자, 서비스 담당자, 배포 승인자, CI/CD 관리자의 권한을 어떻게 나누는지 권장 구성을 보여주실 수 있나요? | Organization, Team, Repository 역할을 간단한 예시로 설명 |  |
| GitHub Environment는 branch 보호와 무엇이 다르며, production 배포 보호에 어떻게 쓰나요? | `prod` Environment의 승인자, 허용 branch/tag, 자기 승인 방지 화면 예시 |  |
| 개발자가 코드를 push할 수 있어도 production 배포를 혼자 승인하거나 실행하지 못하게 하려면 무엇을 설정해야 하나요? | repository 권한, Environment 승인, workflow 권한의 역할 구분 |  |
| GitHub Actions workflow가 필요한 권한만 사용하도록 관리하는 기본 방법은 무엇인가요? | job별 `permissions` 설정과 관리 시 주의점 |  |
| 여러 repository가 있을 때 공통 CI/CD 설정을 안전하게 표준화하는 Microsoft 권장 운영 방식은 무엇인가요? | reusable workflow, 조직 정책, 관리자 역할 중 적용 가능한 방법 |  |

**이 주제에서 확인할 결과:** 사람의 권한과 자동 workflow의 권한을 분리하고, production 배포 승인 흐름을 GitHub에서 어떻게 운영할지 이해한다.

---

## 2. Azure 관리·Identity·권한 모델

**이 주제의 목적:** Azure에서 누가 비용·자원·권한을 관리하고, 자동 배포와 VM이 어떤 범위까지 접근해야 하는지 구분한다.

| 질문 | Microsoft에게 요청할 설명 또는 시연 | 답변 메모 |
|---|---|---|
| Azure의 Subscription, Resource Group, VM, Storage, Application Gateway는 이 파일럿에서 각각 어떤 관리 단위인가요? | 관리자와 운영자가 이해할 수 있는 계층도 |  |
| 개발·검증 환경과 production 환경의 Azure 권한을 어떻게 분리하는 것이 좋나요? | 환경별 Resource Group, identity, RBAC 범위 예시 |  |
| 자동 배포를 실행하는 identity와 VM이 Blob·Key Vault에 접근하는 identity를 왜 분리해야 하나요? | 배포 identity와 VM Managed Identity의 역할 비교 |  |
| 최소 권한으로 시작할 때 자동 배포 identity에 기본적으로 주지 말아야 할 높은 권한은 무엇인가요? | Subscription Owner/Contributor를 피하는 이유와 대안 |  |
| Blob의 release 파일 보존, 삭제 방지, cleanup은 어떤 Azure 기능과 권한으로 운영하는 것이 좋나요? | lifecycle, soft delete/versioning, publisher와 cleanup 역할 분리 |  |

**이 주제에서 확인할 결과:** 사람·자동 배포·VM runtime의 Azure 접근 범위를 분리하고, 환경별 운영 경계를 정할 수 있다.

---

## 3. GitHub-Azure OIDC 연결·사용 모델

**이 주제의 목적:** Azure client secret이나 SSH 비밀번호를 GitHub에 저장하지 않고 Azure 작업 권한을 받는 방법을 이해한다.

| 질문 | Microsoft에게 요청할 설명 또는 시연 | 답변 메모 |
|---|---|---|
| OIDC 방식이 client secret을 GitHub Secrets에 저장하는 방식과 어떻게 다른지 쉽게 설명해 주실 수 있나요? | token 발급부터 Azure 권한 확인까지의 흐름도 |  |
| GitHub Environment의 `prod` 배포 job만 Azure production identity를 사용할 수 있게 하려면 무엇을 연결해야 하나요? | GitHub Environment, Entra federated credential, Azure RBAC의 연결 예시 |  |
| workflow에 필요한 client ID, tenant ID, subscription ID는 secret인가요? 어디에 보관하는 것이 좋나요? | 식별값과 비밀값의 구분, Environment variable/secret 사용 기준 |  |
| dev와 prod의 OIDC identity를 나누는 권장 방식은 무엇인가요? | 환경별 UAMI 또는 Entra application, federation 조건, RBAC scope 예시 |  |
| 허용되지 않은 repository, branch 또는 Environment의 workflow가 Azure에 접근하려 할 때 어떻게 거절되는지 보여주실 수 있나요? | 실패 로그와 진단 위치 예시 |  |
| GitHub OIDC와 Azure DevOps Pipeline의 인증은 같은 설정을 공유하나요? | GitHub federation과 Azure DevOps service connection federation이 별도임을 확인 |  |

**이 주제에서 확인할 결과:** GitHub Actions가 Azure에 접근하는 조건과 실패 시 확인할 위치를 이해한다.

---

## 4. GitHub Actions CI 빌드·산출물·Azure OIDC 운영 모델

**이 주제의 목적:** build 환경, build 속도 개선용 cache, workflow 증적, 배포 원본을 혼동하지 않고 운영 방법을 정한다.

| 질문 | Microsoft에게 요청할 설명 또는 시연 | 답변 메모 |
|---|---|---|
| GitHub-hosted runner와 JDK·Node·공통 도구를 넣은 build image는 무엇이 다른가요? | runner와 container image의 역할을 그림으로 설명 |  |
| 여러 Java/Vue 서비스가 같은 build 환경을 쓰려면 private GHCR과 ACR 중 어떤 선택 기준을 적용해야 하나요? | 비용, 접근 권한, Azure runtime container 계획, 네트워크 기준 비교 |  |
| private GHCR image를 여러 repository에서 사용할 때 비밀번호 대신 어떤 권한 구성을 권장하나요? | `GITHUB_TOKEN`, package 접근 허용, read/write 분리 예시 |  |
| Actions Cache, GitHub Actions Artifacts, Azure Blob은 각각 무엇을 저장해야 하나요? | cache·증적·release 원본의 차이를 실제 workflow 흐름으로 설명 |  |
| cache가 삭제되거나 비어 있어도 build가 정상 동작하도록 어떤 기준을 적용해야 하나요? | cache key, lockfile, cache miss 시 동작 예시 |  |
| Blob에 release를 게시하는 CI job에는 어떤 Azure 권한만 주는 것이 적절한가요? | upload·checksum 검증과 VM/Gateway/Key Vault 권한 분리 |  |
| release ID, checksum, manifest는 왜 필요하며 CD에서 어떻게 확인하나요? | 이전 release를 재배포하거나 rollback할 때의 예시 |  |

**이 주제에서 확인할 결과:** build image, cache, artifacts, Blob release의 lifecycle과 권한 경계를 선택할 수 있다.

---

## 5. GitHub Actions CD와 Azure Pipelines CD: 공유 이중화 VM 배포 운영 모델

**이 주제의 목적:** 여러 서비스가 두 VM을 공유할 때 두 VM이 동시에 traffic에서 제외되지 않도록 CD 순서와 무중단 배포 절차를 이해한다.

### 먼저 함께 볼 흐름

```text
Blob release
   │
   ▼
CD 대기열 또는 잠금
   │
   ▼
VM2 Healthy 확인 → VM1 drain·배포·readiness → VM1 Healthy 확인 → VM2 반복
```

여기서 대기열·잠금은 동시 배포를 막는 기능이고, drain·readiness는 Application Gateway와 Nginx/Tomcat을 이용해 traffic을 안전하게 전환하는 절차다.

| 질문 | Microsoft에게 요청할 설명 또는 시연 | 답변 메모 |
|---|---|---|
| 여러 서비스가 같은 VM1/VM2를 사용할 때, 서비스 A와 B가 서로 다른 VM을 동시에 배포하지 못하게 하는 가장 이해하기 쉬운 운영 방법은 무엇인가요? | shared VM pair를 하나의 배포 대상과 잠금 단위로 보는 예시 |  |
| GitHub Actions `concurrency`와 Azure Pipelines Environment exclusive lock은 각각 무엇을 막고, 무엇을 자동으로 하지 않나요? | 순차 실행 제어와 traffic 전환의 차이 |  |
| Resource Group 또는 VM pair를 GitHub concurrency group이나 Azure DevOps Environment에 어떻게 이름으로 매핑하는 것이 좋나요? | `prod-rg-a-vm-pair-01` 같은 논리적 식별자 예시 |  |
| GitHub Actions만 CD에 쓰는 경우와 Azure Pipelines를 CD에 쓰는 경우의 운영·승인·비용 차이를 비교해 주실 수 있나요? | 각 선택지의 최소 구성과 관리 책임 설명 |  |
| Azure Pipelines exclusive lock을 쓰면 Application Gateway traffic 전환도 자동으로 되나요? | **아니오.** lock은 순서 제어이고, drain/readiness는 pipeline 단계로 구현해야 함을 확인 |  |
| Application Gateway와 Nginx를 이용해 VM1을 배포할 때 권장 절차를 보여주실 수 있나요? | Nginx marker off, probe Unhealthy, 기존 요청 drain, Blob release 적용, Tomcat readiness, marker on, Healthy 확인 |  |
| VM1을 제외하기 전에 VM2가 Healthy인지 왜 확인해야 하나요? | VM2가 유일한 backend가 되는 상황과 HOLD 기준 설명 |  |
| Application Gateway probe가 Unhealthy가 된 뒤 기존 연결은 어떻게 처리되나요? | connection draining, HTTP/WebSocket/SSE, timeout 확인 방법 |  |
| Blob 파일을 VM에 적용할 때 SSH 직접 push와 Azure Run Command + VM Blob pull 중 어떤 방법이 적합한가요? | GitHub-hosted runner, private network, SSH credential, VM Managed Identity 조건 비교 |  |
| VM1 배포 실패·pipeline 취소·timeout이 발생하면 VM2와 다음 서비스 배포는 어떻게 막고 상태를 복구하나요? | HOLD, 실제 VM/Run Command 상태 확인, rollback 절차 예시 |  |
| Vue 정적 파일을 바꾼 뒤 이전 브라우저 탭이 이전 JS/CSS를 요청하면 어떻게 처리해야 하나요? | hash/version asset 2버전 공존, asset 보존기간과 cleanup 예시 |  |

**이 주제에서 확인할 결과:** CD 조정 도구의 선택과 별개로, shared VM pair·Application Gateway·Nginx·Tomcat·Vue를 포함한 안전한 순차 배포 runbook을 정할 수 있다.

---

## 회의 마무리 확인표

| 확인할 내용 | 기록 |
|---|---|
| Microsoft가 추천한 GitHub Enterprise Cloud 권한·Environment 보호 구조 |  |
| Azure identity와 RBAC의 환경별 분리 방식 |  |
| GitHub OIDC 및 Azure DevOps service connection의 도입 경로 |  |
| private GHCR/ACR, cache, artifacts, Blob release의 권장 조합 |  |
| GitHub Actions CD와 Azure Pipelines CD 중 추가 검토 대상 |  |
| shared VM pair의 queue/lock과 운영 책임 |  |
| Application Gateway probe·drain·Tomcat readiness의 Pilot 범위 |  |
| Vue 이전 asset 보존·cleanup 정책의 확인 주체 |  |
| Microsoft 후속 자료, 담당자, 확인 일정 |  |

이 질문지는 도입 검토를 위한 안내 자료다. 실제 제품 설정이나 파일럿 결과를 나타내지 않으며, 회의에서 확인된 내용은 별도의 결정 기록과 Pilot 증적으로 갱신한다.
