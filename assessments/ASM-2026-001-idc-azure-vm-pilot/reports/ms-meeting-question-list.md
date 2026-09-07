# Microsoft 서비스 소개 미팅 질문지

- Date: 2026-09-08
- 목적: Microsoft의 제품·권장 구조를 확인하고, 내부에서 선택할 수 있도록 질문별 답변과 비교 기준을 남긴다.
- 전제: GitHub Enterprise Cloud, GitHub-hosted runner, 이중화 Azure VM, Application Gateway, 서비스별 Tomcat/JVM과 Vue/Nginx를 사용한다.

## 사용 방법

각 질문을 하나씩 설명하고, Microsoft의 답을 `답변 메모`에 기록한다. `기대하는 답`은 정답을 유도하기 위한 것이 아니라 현재 내부 초안과 비교할 기준이다. `내부 결정`은 Microsoft 답변을 들은 뒤 내부에서 정한다.

파일럿 범위·성공 기준·release manifest 형식은 내부 결정 사항이다. Microsoft에게 결정받지 않는다.

## 1. GitHub와 Azure 권한·identity

| ID | Microsoft에 할 쉬운 질문 | 왜 묻는가 | 기대하는 답 또는 비교 기준 | 답변 메모 | 내부 결정 |
|---|---|---|---|---|---|
| ID-01 | GitHub에서는 개발자, 운영 승인자, CI/CD 관리자의 권한을 어떻게 나누는 것이 좋나요? | 개발자가 운영 배포 정책이나 Azure 권한까지 바꾸지 않게 하기 위해 | 개발자는 source repo의 코드·CI, 운영 승인자는 production Environment 승인, CI/CD 관리자는 workflow·Environment 정책을 관리하는 분리 모델. 사람 권한은 Team 기반 최소 권한인지 확인 |  | 실제 Team·repository role 매핑 |
| ID-02 | GitHub Environment의 운영 승인 기능은 언제 사용하고, Azure RBAC는 어디에 사용하나요? | GitHub 승인과 Azure 권한을 같은 것으로 오해하지 않기 위해 | Environment는 “배포를 시작해도 되는가”의 사람 승인, Azure RBAC는 “승인된 자동 작업이 어떤 Azure 작업을 할 수 있는가”의 기술 권한. 둘 다 필요한지 확인 |  | prod 승인자·self-review·bypass 정책 |
| ID-03 | GitHub Actions가 Azure에 로그인할 때, 비밀번호 대신 OIDC를 사용하는 Microsoft 권장 방식은 무엇인가요? | Azure client secret·서비스 계정 비밀번호를 만들지 않기 위해 | GitHub OIDC token과 Entra federated credential을 사용하고, deploy job에만 OIDC token 요청 권한을 주는지 확인 |  | OIDC 사용 확정 여부와 신뢰 조건 |
| ID-04 | dev와 prod는 Azure 접근 권한을 어떻게 나누는 것이 좋나요? | dev workflow가 prod VM을 변경하지 않게 하기 위해 | dev/prod마다 별도 Azure identity·federated credential·RBAC scope를 두는지, prod는 production Environment job만 신뢰하는지 확인 |  | dev/prod identity·scope 분리 |
| ID-05 | GitHub Actions용 Azure identity는 환경별 UAMI와 Entra application 중 어떤 기준으로 선택하나요? | Azure에서 GitHub OIDC를 받을 identity 유형을 정하기 위해 | UAMI는 Azure 리소스로 수명·RBAC를 관리하기 쉽고, Entra application은 기존 조직 표준이 있으면 적합. 둘 다 OIDC federation이 가능한지와 조직 표준 확인 |  | UAMI 또는 Entra application |
| ID-06 | VM이 Blob과 Key Vault에 접근할 때, GitHub Actions용 identity와 별도로 Managed Identity를 두는 것을 권장하나요? | GitHub Actions가 runtime secret·VM 파일 권한을 모두 갖지 않게 하기 위해 | GitHub deployer는 Run Command 같은 Azure 제어 작업만, VM MI는 Blob release read와 필요한 Key Vault read만 갖는 분리 모델인지 확인 |  | deployer identity·VM MI RBAC |
| ID-07 | GitHub-hosted runner만 사용하고 SSH를 외부에 열지 않을 때, Azure Run Command 배포가 적합한가요? | SSH key·공인 관리 포트 없이 VM에 배포하기 위해 | GitHub OIDC → Azure Run Command → VM Agent → VM MI의 흐름이 가능한지, Agent·timeout·취소·감사 제약 확인 |  | Run Command 채택 여부·운영 기준 |

ID-02는 간단히 말하면 “사람이 GitHub에서 배포를 승인한 뒤, 자동 작업이 Azure에서 어디까지 할 수 있게 할 것인가”를 묻는 질문이다.

ID-06은 identity를 따로 만든다는 뜻만으로 충분하지 않다. GitHub에서 온 identity가 신뢰할 repository·Environment·workflow와 Azure에서 부여할 RBAC를 별도로 제한해야 한다. VM에 연결한 Managed Identity 권한은 VM 안에서 코드를 실행할 수 있는 주체가 사용할 수 있으므로 최소 권한이 중요하다. [Managed Identity 권장사항](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/managed-identity-best-practice-recommendations), [GitHub Environments](https://docs.github.com/en/enterprise-cloud@latest/actions/reference/workflows-and-actions/deployments-and-environments)

## 2. CI: 빌드·private registry·릴리스 파일

| ID | Microsoft에 할 질문 | 왜 묻는가 | 기대하는 답 또는 비교 기준 | 답변 메모 | 내부 결정 |
|---|---|---|---|---|---|
| CI-01 | GitHub-hosted runner에서 Java·Node·추가 빌드 도구를 운영할 때, 설치 방식과 공통 build image 방식의 선택 기준은 무엇인가요? | 추가 모듈이 build runner용으로 필요하기 때문 | 설치가 짧고 repo별로 다르면 setup/install, 반복 설치가 길고 여러 repo가 공유하면 고정 build image를 비교 |  | 설치 방식 또는 공통 image |
| CI-02 | private build image를 쓸 때 GHCR과 ACR은 각각 언제 권장하나요? | private registry를 도입할 필요와 위치를 정하기 위해 | GitHub Actions build 환경은 GHCR과 `GITHUB_TOKEN` package access가 자연스러운지, Azure runtime이 image를 직접 pull하면 ACR·Managed Identity가 자연스러운지 비교 |  | GHCR만 사용 또는 ACR 추가 |
| CI-03 | 여러 repository가 private GHCR image를 함께 쓸 때 권한은 어떻게 관리하나요? | PAT 같은 장기 비밀번호를 피하기 위해 | package별 Actions access를 부여하고 `GITHUB_TOKEN`·`packages: read`로 접근하는지, 조직 package 운영 기준 확인 |  | package owner·접근 repo 목록 |
| CI-04 | 기존 Nexus에만 있는 Maven/npm 패키지는 어떤 영구 package registry로 옮기는 것이 적합한가요? | Actions Cache가 Nexus의 원본을 대체하지 않기 때문 | GitHub Packages와 Azure Artifacts를 private package, 비용, 기존 도구 호환성, 권한 모델로 비교 |  | package registry 선택 |
| CI-05 | CI가 만든 release를 CD가 같은 파일로 배포하게 하는 Microsoft 권장 관리 방법이 있나요? | release 파일이 바뀌거나 `latest`를 잘못 배포하지 않기 위해 | release ID·commit SHA·파일 digest를 release manifest에 기록하는 내부 기본안을 확인받되, 실제 형식은 내부 표준으로 결정 |  | manifest 최소 항목 |

Actions Cache는 빌드 속도를 높이는 재생성 가능한 파일용이다. 배포 파일과 private package의 유일한 원본으로 사용하지 않는다. private GHCR package는 접근이 허용된 repository의 `GITHUB_TOKEN`으로 workflow에서 사용할 수 있다. [GitHub Packages와 Actions](https://docs.github.com/en/packages/managing-github-packages-using-github-actions-workflows/publishing-and-installing-a-package-with-github-actions)

## 3. CD: 여러 repository·공유 VM pair·push/pull

| ID | Microsoft에 할 질문 | 왜 묻는가 | 기대하는 답 또는 비교 기준 | 답변 메모 | 내부 결정 |
|---|---|---|---|---|---|
| CD-01 | 여러 service repository가 같은 VM 두 대를 공유할 때, GitHub Actions만으로 배포 순서를 관리하는 reference architecture나 blueprint가 있나요? | A 서비스와 B 서비스가 서로 다른 VM을 동시에 배포하는 사고를 막기 위해 | GitHub Actions `concurrency`가 repository 단위라는 전제에서 중앙 deployment repo·공통 pair lock·상태 저장소의 권장 구조 확인 |  | GitHub Actions 단독 운영 가능성 |
| CD-02 | GitHub Actions만 쓴다면 중앙 deployment repository의 최소 구성은 무엇인가요? | 중앙 repo 운영 부담이 실제로 어느 정도인지 보기 위해 | release 요청 접수, prod Environment 승인, pair별 queue/lock, Run Command 실행, 결과·HOLD 기록이 최소 구성인지 비교 |  | 중앙 repo 필요 여부·소유팀 |
| CD-03 | Azure Pipelines Environment의 approval·exclusive lock을 CD 조정에만 쓰는 방식이 적합한가요? | Azure Pipelines 추가 비용과 중앙 repo 운영 부담을 비교하기 위해 | `sequential` lock이 Azure Pipelines CD 요청을 하나씩 처리하는 범위, GitHub Actions CI와의 연동 방식, 관리 대상 증가 확인 |  | Azure Pipelines CD 도입 여부 |
| CD-04 | Azure Pipelines의 lock만으로 shared VM pair를 안전하게 보호할 수 있나요? | lock의 적용 범위를 과대평가하지 않기 위해 | Azure Pipelines 밖의 수동 작업·이미 시작된 Run Command·Gateway 변경은 별도 pair 상태/HOLD가 필요한지 확인 |  | 공통 pair lock 필요 여부 |
| CD-05 | GitHub Actions나 Azure Pipelines에서 VM 배포 명령은 push하고, VM이 Blob에서 release를 pull하는 방식이 권장되나요? | SSH 없이 passwordless 배포를 만들기 위해 | pipeline은 OIDC로 Azure Run Command 호출, VM은 MI로 Blob download. 직접 SSH push와 비교한 제약·운영성 확인 |  | 명령 push + 파일 pull 채택 여부 |
| CD-06 | Azure Pipelines를 추가할 때 파일럿·운영의 Microsoft-hosted 사용량과 예상 비용은 얼마인가요? | 비용 거부감을 숫자로 비교하기 위해 | 공개 기준 free 1 job/1,800분, 첫 유료 job의 효과, 한국 통화·EA 할인·승인/대기 사용량을 견적으로 확인 |  | 비용 상한·도입 판단 |

GitHub Actions concurrency는 repository 안에서만 동작한다. Azure Pipelines exclusive lock은 동일 protected resource를 쓰는 Azure Pipelines stage를 순차화한다. 둘 다 별도 수동 작업이나 VM에서 이미 실행 중인 작업을 자동으로 통제하지는 않는다. [GitHub Actions concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency), [Azure Pipelines exclusive lock](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops)

## 4. 무중단 전환: Application Gateway·Nginx·connection draining

| ID | Microsoft에 할 질문 | 왜 묻는가 | 기대하는 답 또는 비교 기준 | 답변 메모 | 내부 결정 |
|---|---|---|---|---|---|
| HA-01 | VM 두 대에서 Java/Tomcat을 한 대씩 배포할 때, Application Gateway의 권장 rolling deployment 구성은 무엇인가요? | 이번 구조의 전체 무중단 전환 패턴을 확인하기 위해 | VM 1 제외 → 배포·warm-up·검증 → 재편입 → VM 2 반복의 표준 절차 확인 |  | 배포 단계와 운영 runbook |
| HA-02 | Application Gateway backend member를 직접 제외하는 방식과 Nginx/Tomcat custom readiness probe 방식 중 어느 쪽을 권장하나요? | 트래픽 제외 방식을 선택하기 위해 | 직접 제외는 Gateway 구성 변경과 draining을 활용, probe는 Gateway 구성을 덜 바꾸지만 probe 전파·서비스별 readiness 설계가 필요하다는 비교 |  | 제외 방식 또는 조합 |
| HA-03 | custom probe로 VM을 unhealthy로 만들면 기존 요청도 안전하게 끝나는가요? | 신규 요청 제외와 기존 연결 처리를 구분하기 위해 | probe는 신규 요청 제어, 기존 연결은 별도 drain 기준이 필요한지 확인. probe unhealthy만으로 connection draining timeout이 적용되는지 명시적으로 확인 |  | drain 기준·앱 지표 |
| HA-04 | backend member를 명시적으로 제외할 때 connection draining은 어떻게 설정하나요? | 다운로드·WebSocket 등 오래 가는 연결의 오류를 줄이기 위해 | timeout을 최대 예상 연결 시간보다 길게 설정하는지, affinity 예외와 rollback 재편입 순서 확인 |  | timeout·affinity 정책 |
| HA-05 | Nginx custom probe를 쓴다면 어떤 readiness 응답을 권장하나요? | marker 파일만 보고 너무 빨리 재편입되는 것을 막기 위해 | 배포 허용 상태, Tomcat 실제 readiness, 예상 release ID 확인을 함께 보는지. 일반 사용자 URL과 probe URL을 분리하는지 확인 |  | readiness endpoint 계약 |
| HA-06 | VM 1 배포 실패 또는 workflow 취소 시 VM 2 배포를 막고 복구하는 권장 절차는 무엇인가요? | 반대편 VM까지 배포해 전체 서비스를 잃지 않기 위해 | VM 2 배포 금지, VM 1 상태 확인, rollback 또는 HOLD, pair lock 해제 조건 확인 |  | 실패·복구 runbook |

pipeline은 배포 순서를 자동화한다. Application Gateway probe는 신규 요청을 정상 backend에 보내는 데 사용한다. connection draining은 backend를 명시적으로 제외할 때 기존 연결을 timeout 동안 유지하도록 돕고 affinity 요청은 예외일 수 있다. 세 기능을 같은 것으로 보지 않고 현재 Gateway 설정으로 조합을 확인한다. [Application Gateway probe](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-probe-overview), [connection draining](https://learn.microsoft.com/en-us/azure/application-gateway/configuration-http-settings)

## 5. Blob·Key Vault와 네트워크

| ID | Microsoft에 할 질문 | 왜 묻는가 | 기대하는 답 또는 비교 기준 | 답변 메모 | 내부 결정 |
|---|---|---|---|---|---|
| NET-01 | standard GitHub-hosted runner가 Blob에 release를 업로드할 때, public endpoint의 Entra 인증 접근과 private endpoint 전용 구성은 각각 어떤 조건에서 권장하나요? | Blob network 경계와 runner 선택을 정하기 위해 | public endpoint의 인증 접근은 anonymous 공개와 다르며, private-only는 VNet 경로가 필요하다는 점 확인 |  | Blob network 정책 |
| NET-02 | Blob을 private endpoint 전용으로 둘 때 GitHub-hosted runner에서 upload하는 권장 방식은 무엇인가요? | upload identity만으로 private network 접근이 되는지 확인하기 위해 | standard runner는 고객 VNet 밖이므로 VNet 연결 larger runner 또는 Azure 내부 publish 경로가 필요한지 확인 |  | larger runner 또는 다른 publish 경로 |
| NET-03 | Key Vault는 GitHub Actions가 아닌 VM과 Application Gateway만 접근하게 구성할 수 있나요? | runtime secret을 workflow에 노출하지 않기 위해 | VM MI와 Gateway identity만 최소 read, GitHub deployer는 기본적으로 secret read 없음, private endpoint·DNS 권장 구조 확인 |  | Key Vault network·RBAC 경계 |

identity는 “접근 권한이 있는가”를 해결하고, private endpoint는 “해당 endpoint까지 네트워크로 도달할 수 있는가”를 해결한다. Storage public network access를 Disabled로 설정하면 private endpoint 요청만 허용된다. 따라서 identity가 있어도 standard GitHub-hosted runner가 private Blob endpoint에 직접 업로드할 수는 없다. [Azure Storage 보안](https://learn.microsoft.com/en-us/azure/storage/common/secure-storage), [public network access 설정](https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security-set-default-access)

## 회의 후 내부 결정 목록

- GitHub Actions 단독 CD 또는 Azure Pipelines CD 조정기
- 중앙 deployment repository와 공통 pair lock의 책임자
- backend 직접 제외, custom probe, 또는 두 방식의 조합
- 환경별 UAMI 또는 조직 표준 Entra application
- GHCR, ACR, GitHub Packages, Azure Artifacts의 역할
- Blob public authenticated access 또는 private-only publish 경로
- dev 파일럿 범위·성공 기준·release manifest·보존·rollback 기준
