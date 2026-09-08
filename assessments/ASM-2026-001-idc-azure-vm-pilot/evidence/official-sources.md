# 공식 출처 및 증거 범위

확인일: 2026-09-07. 공식 문서의 기능 설명을 확인했으며 실제 tenant/repo/VM에 접속해 설정·호환성·배포를 검증한 결과는 아니다.
가격과 리전별 비용은 견적 입력 부족으로 산정하지 않았다. 아래 근거를 조합한 구조와 절차는 이 Assessment의 설계 제안이다.

| ID | 공식 문서 | 확인한 내용 |
|---|---|---|
| S01 | [GitHub OIDC reference](https://docs.github.com/en/actions/reference/security/oidc) | repo/environment/branch subject, immutable ID 형식, subject customization |
| S02 | [Azure Login with OIDC](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect) | Entra app 또는 UAMI의 federated credential과 azure/login |
| S03 | [GitHub environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments) | 승인·branch 제한·self-review·bypass 및 custom rules |
| S04 | [Azure DevOps WIF service connection](https://learn.microsoft.com/en-us/azure/devops/pipelines/release/configure-workload-identity?view=azure-devops) | 개인 접근 권한과 연결의 workload identity·pipeline 사용 허가 구분 |
| S05 | [Azure private networking for hosted runners](https://docs.github.com/en/organizations/managing-organization-settings/about-azure-private-networking-for-github-hosted-runners-in-your-organization) | larger runner의 VNet 및 IDC 접근, standard runner 제한 |
| S06 | [Managed Run Command Linux](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command-managed) | VM Agent 실행, timeout, 병렬 가능, instanceView/exitCode |
| S07 | [Action Run Command Linux](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command) | SSH 없이 실행, Agent outbound, 기본 elevated 실행, 시간·출력 제한 |
| S08 | [Azure Compute permissions](https://learn.microsoft.com/en-us/azure/role-based-access-control/permissions/compute) | runCommand/action과 runCommands/read/write/delete 구분 |
| S09 | [Azure Networking permissions](https://learn.microsoft.com/en-us/azure/role-based-access-control/permissions/networking) | Application Gateway 조회·backendhealth/action·write |
| S10 | [AzCopy managed identity](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-authorize-managed-identity) | VM MI와 Blob Data Reader 다운로드 |
| S11 | [Blob Entra authorization](https://learn.microsoft.com/en-us/azure/storage/blobs/authorize-access-azure-active-directory) | 데이터 접근 RBAC와 token 인증 |
| S12 | [Application Gateway probes](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-probe-overview) | unhealthy 대상 신규 트래픽 제외·복귀, probe 주기/threshold |
| S13 | [Application Gateway backend HTTP settings](https://learn.microsoft.com/en-us/azure/application-gateway/configuration-http-settings) | 명시적 backend 제거의 connection drain, timeout·affinity 예외 |
| S14 | [Application Gateway backend health](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-backend-health) | pool/settings별 상태, 조회와 실제 probe 주기의 관계 |
| S15 | [GitHub concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency) | repo 범위, cancel-in-progress, queue: max, 대기 순서 |
| S16 | [Reusable workflow configurations](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations) | caller의 context·runner·권한으로 실행 |
| S17 | [GITHUB_TOKEN](https://docs.github.com/en/actions/concepts/security/github_token) | job token의 저장소 범위 및 수명 |
| S18 | [Lease Blob](https://learn.microsoft.com/en-us/rest/api/storageservices/lease-blob) | Blob 쓰기·삭제 잠금, 유한·무기한 lease |
| S19 | [Actions artifacts retention](https://docs.github.com/en/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization) | 기간 기반 보존, private 상한 및 상위 정책 |
| S20 | [Workflow artifacts](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts) | 산출물 전달·보관과 cache의 용도 차이 |
| S21 | [Blob lifecycle policy structure](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-structure) | 시간 기반 lifecycle 조건 |
| S22 | [ACR artifact cache](https://learn.microsoft.com/en-us/azure/container-registry/artifact-cache-overview) | 컨테이너 upstream별 캐시 및 인증 조건 |
| S23 | [Tomcat parallel deployment](https://tomcat.apache.org/tomcat-10.1-doc/config/context.html) | 같은 context의 여러 버전과 세션별 라우팅; 현재 버전 적용 여부 미확인 |
| S24 | [NGINX control](https://nginx.org/en/docs/control.html) | reload 시 새 worker 및 구 worker graceful 종료 |
| S25 | [Vite build](https://vite.dev/guide/build) | 이전 chunk 삭제 시 기존 브라우저 오류, HTML 캐시 |
| S26 | [Vite troubleshooting](https://vite.dev/guide/troubleshooting) | version skew와 이전 chunk 보존; 실제 Vite 채택은 미확인 |
| S27 | [VM managed identity token](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-to-use-vm-token) | VM 리소스가 MI 보안 경계 |
| S28 | [Key Vault와 기존 자격증명 접근](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/tutorial-windows-managed-identities-vm-access) | Entra 미지원 대상은 비밀값이 남을 수 있음; Windows 튜토리얼의 인증 개념만 참조 |
| S29 | [Copilot code review configuration](https://docs.github.com/en/enterprise-cloud@latest/copilot/how-tos/copilot-on-github/set-up-copilot/configure-code-review) | Copilot 조직 정책과 승인 설정; 배포 승인과 별도 |

## 시점에 따른 주의사항

- S01은 2026-07-15 이후 신규 repo와 rename/transfer에서 immutable owner/repo ID가 포함되는 subject를 설명한다. 과거 이름 기반 예시를 그대로 붙여 넣지 않고 실제 issuer/aud/sub를 확인한다.
- S15는 `queue: max`를 제공한다. 과거의 'pending 한 개만 가능' 설명은 현재 기본값에 해당한다.
- S06의 설명 중 권한 단수 표기가 있으므로 실제 custom role 정의는 S08의 provider operation `runCommands/*`와 대상 API를 기준으로 확인한다.
- Copilot 승인 기능의 문서 간 갱신 시점 차이가 있어 '항상 comment만 가능'으로 단정하지 않는다. 이 파일럿은 사람의 운영 배포 승인을 별도로 두는 정책을 제안한다.

## Hosted 전환 후속 조사

확인일: 2026-09-07. 적용 시험과 구분한다.

| ID | 공식 문서 | 확인한 내용 |
|---|---|---|
| S30 | [Key Vault security](https://learn.microsoft.com/en-us/azure/key-vault/general/secure-key-vault) | 비밀·키·인증서, 환경 분리 및 복구·감사 |
| S31 | [Key Vault network security](https://learn.microsoft.com/en-us/azure/key-vault/general/network-security) | trusted services는 모든 hosted 실행을 포괄하지 않음 |
| S32 | [Key Vault RBAC](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide) | 사용자·워크로드별 데이터 권한 |
| S33 | [Application Gateway Key Vault certificates](https://learn.microsoft.com/en-us/azure/application-gateway/key-vault-certs) | UAMI, 인증서 secret URI 및 갱신 |
| S34 | [GHCR](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) | GITHUB_TOKEN 및 패키지 접근 권한 |
| S35 | [Container jobs](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/run-jobs-in-a-container) | runner와 job container 구분 |
| S36 | [Dependency caching](https://docs.github.com/en/actions/concepts/workflows-and-actions/dependency-caching) | 캐시와 산출물 구분, 원본 필요 |
| S37 | [setup-node](https://github.com/actions/setup-node/blob/main/README.md) | 패키지 매니저 캐시와 node_modules 구분 |
| S38 | [ACR Managed Identity](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication-managed-identity) | Azure 워크로드의 이미지 pull 인증 |
| S39 | [Storage private endpoints](https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints) | Private Endpoint와 공개 접근 차단 별도 |
| S40 | [NGINX core module](https://nginx.org/en/docs/http/ngx_http_core_module.html) | root, open_file_cache, 심볼릭 링크 제약 |
| S41 | [GitHub repository roles](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/repository-roles-for-an-organization) | Read/Triage/Write/Maintain/Admin 역할과 repository별 team·collaborator 권한 |
| S42 | [GitHub custom repository roles](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/about-custom-repository-roles) | Enterprise Cloud custom repository role, 상속 역할 및 base/team 권한의 누적 |
| S43 | [GitHub predefined organization roles](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-peoples-access-to-your-organization-with-roles/permissions-of-predefined-organization-roles) | CI/CD admin 등 Organization 역할의 범위 |
| S44 | [GitHub Packages billing](https://docs.github.com/en/enterprise-cloud@latest/billing/concepts/product-billing/github-packages) | Enterprise Cloud Packages 50GB/월 100GB와 Container registry의 현행 이미지 저장·전송 무료 정책 |
| S45 | [GitHub Actions billing](https://docs.github.com/en/enterprise-cloud@latest/billing/concepts/product-billing/github-actions) | Actions artifacts와 Packages의 shared storage, repository별 Actions cache 10GB, Enterprise Cloud custom image storage 150GB |
| S46 | [GitHub Container registry](https://docs.github.com/en/enterprise-cloud@latest/packages/working-with-a-github-packages-registry/working-with-the-container-registry) | OCI image registry, enterprise data residency endpoint 및 layer당 10GB 제한 |
| S47 | [GitHub dependency caching](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching) | repository별 기본 10GB, 최근 7일 미접근 cache 정리, size/retention·budget 정책 |
| S48 | [GitHub artifact retention](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization) | private/internal repository artifact·log 기본 90일, 1~400일 설정과 organization/enterprise 상한 |
| S49 | [GitHub package deletion/restoration](https://docs.github.com/en/enterprise-cloud@latest/packages/learn-github-packages/deleting-and-restoring-a-package) | package 삭제 권한과 30일 내 복원 조건 |
| S50 | [Blob lifecycle delete](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-delete) | 시간 기반 lifecycle 삭제와 soft-delete 상태의 관계 |
| S51 | [Blob soft delete/versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-blob-overview) | versioning·soft delete의 복구·비용 영향 |
| S52 | [GitHub workflow artifacts](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts) | workflow 내 job 간 파일 전달과 artifact의 역할 |
| S53 | [GitHub Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) | private package 기본 visibility, package 접근과 digest pull |
| S54 | [GitHub package access control](https://docs.github.com/en/packages/learn-github-packages/configuring-a-packages-access-control-and-visibility) | repository 상속 또는 package granular access, private package read/write/admin 권한 |
| S55 | [GitHub Actions concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency) | group별 running/pending, `queue: single`/`max`, 취소 동작 |
| S56 | [GitHub Environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments) | wait timer 비과금, approval 전 environment secret 비공개 |
| S57 | [GitHub job execution time](https://docs.github.com/en/actions/how-tos/monitor-workflows/view-job-execution-time) | private repository GitHub-hosted runner job execution의 billable minutes 확인 |
| S58 | [Azure Login with GitHub OIDC](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect) | Entra application/UAMI federated credential, `id-token: write`, Azure Login 입력 식별자 |
| S59 | [GitHub-hosted runner Azure private networking](https://docs.github.com/en/organizations/managing-organization-settings/about-azure-private-networking-for-github-hosted-runners-in-your-organization) | Azure VNet에서 hosted runner private resource 접근과 network 정책 |
| S60 | [Azure Managed Run Command](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command-managed) | VM Agent script 실행, instance view execution state·exit code |
| S61 | [Application Gateway custom probe](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-create-probe-portal) | probe path/host/interval/timeout/unhealthy threshold와 backend HTTP setting 연결 |
| S62 | [Tomcat context / parallel deployment](https://tomcat.apache.org/tomcat-10.1-doc/config/context) | 같은 context의 version 병행과 session 기반 route 동작 |
| S63 | [Azure Pipelines approvals and checks](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops) | Environment approval, protected resource check, exclusive lock와 `sequential`/`runLatest` |
| S64 | [Azure Pipelines environments](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/environments?view=azure-devops) | deployment job target, approval, deployment history |
| S65 | [Azure Pipelines parallel jobs](https://learn.microsoft.com/en-us/azure/devops/pipelines/licensing/concurrent-jobs?view=azure-devops) | Azure subscription 연결, private project의 1 Microsoft-hosted job·월 1,800분, self-hosted capacity |
| S66 | [Azure Pipelines GitHub repositories](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/github?view=azure-devops) | GitHub source repository trigger |
| S67 | [Azure Pipelines manual validation](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/manual-validation-v0?view=azure-pipelines) | YAML agentless job에서의 수동 검증 대기 |
| S68 | [Application Gateway features](https://learn.microsoft.com/en-us/azure/application-gateway/features) | backend traffic에 대한 connection draining 역할 |
| S69 | [GitHub Actions OIDC](https://docs.github.com/en/actions/reference/security/oidc) | job별 `id-token: write`, repository·branch·Environment claim 조건 |
| S70 | [Azure Login with OpenID Connect](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect) | Entra application/UAMI federated credential와 GitHub Actions Azure login |
| S71 | [GitHub dependency caching](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching) | repository별 기본 10 GB, 7일 미접근 cache 정리와 cache 보안 주의 |
| S72 | [GitHub Actions artifact retention](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization) | private/internal repository artifact·log 기본 90일 및 1~400일 설정 범위 |
