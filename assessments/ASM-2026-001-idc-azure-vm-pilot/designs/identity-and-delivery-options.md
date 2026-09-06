# OIDC·권한·승인·runner 대안

상태: 조사 및 권장안. 기준일: 2026-09-07. 실제 설정/권한 부여/배포는 미수행.
최신 push/pull·Key Vault·기존 도구 제외 조건은 [후속 비교](hosted-only-push-pull-key-vault-static.md)를 따른다.
사용자 확인: GitHub Enterprise Cloud, Azure VM, 자체 Actions runner 가능하면 미사용.

## 1. 권장 흐름

```text
서비스 repo: 빌드·검사 → 검증된 release/digest 게시
중앙 배포 repo: release manifest → dev 검증 → prod Environment 승인
  → GitHub OIDC → Entra의 배포 Identity → Azure ARM
  → 서버 쌍 공통 잠금/상태 확인 → Managed Run Command
  → Azure VM Agent → 로컬 배포 절차
  → VM Managed Identity → Blob에서 승인된 release 다운로드
```

중앙 repo와 Managed Run Command는 아직 권장안이다. [결정 후보](../decisions/DEC-2026-002-deployment-trust-boundary.md)에서 비교한다.
GitHub Enterprise Cloud는 SaaS이며 사용자 Azure VM에 GitHub 서버를 설치한다는 뜻이 아니다. Azure 자원과 GitHub 연결·과금·인증은 각각 설정한다.

## 2. 개인 권한과 워크로드 권한

| 계층 | 주체 | 결정하는 내용 |
|---|---|---|
| GitHub 로그인·조직 권한 | 사람·팀, 조직의 SSO 정책 | 누가 repo를 수정하고 workflow를 실행·관리하는가 |
| GitHub Environment | 지정 reviewer·팀 | 해당 release 배포 job을 실행하도록 승인하는가 |
| OIDC federated credential | repo/environment 등의 claim을 만족하는 job | 어느 Azure Identity로 token을 교환할 수 있는가 |
| Azure RBAC | Entra service principal 또는 UAMI | 어떤 VM·Blob·Gateway에 어떤 작업을 할 수 있는가 |
| VM 런타임 인증 | VM에 연결된 Managed Identity | 실행 서버가 어떤 Blob/Key Vault 등에 접근하는가 |

Azure DevOps도 일반적인 WIF 서비스 연결에서는 개인의 Azure 권한이 배포 시 전달되는 방식이 아니다. 사람이 서비스 연결을 만들거나 사용할 권한과, 파이프라인이 사용하는 서비스 연결의 Identity/RBAC가 별개다. GitHub에서는 repo·environment·workflow 신뢰와 Azure Identity/RBAC를 연결한다. [Azure DevOps WIF](https://learn.microsoft.com/en-us/azure/devops/pipelines/release/configure-workload-identity?view=azure-devops)

GitHub repo와 Azure Identity를 반드시 1:1로 만들 필요는 없다. 다만 공용 Identity에 모든 repo를 신뢰시키면 권한 범위가 합쳐진다. 파일럿은 build publisher, dev deployer, prod deployer, VM reader를 분리하고 prod 실행 주체는 중앙 repo로 좁히는 안을 제안한다.

## 3. OIDC 설정 기준

Azure 측 Entra application/service principal 또는 User-assigned Managed Identity(UAMI)에 federated credential을 추가하고 필요한 RBAC를 할당할 수 있다. GitHub job은 `id-token: write`로 OIDC token 발급을 요청하고 `azure/login`을 통해 Azure access token으로 교환한다. [Azure OIDC](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect)

- github.com 기준 issuer 예시: `https://token.actions.githubusercontent.com`.
- Azure public cloud audience 예시: `api://AzureADTokenExchange`.
- 이름 기반 subject 예시: `repo:ORG/deployment-repo:environment:prod`.
- 신규 immutable 형식 예시: `repo:ORG@OWNER_ID/deployment-repo@REPO_ID:environment:prod`.
- 실제 GHE 도메인·repo OIDC 설정의 issuer/audience/subject를 확인해 정확히 일치시킨다. token 자체를 로그·문서에 남기지 않는다.
- `environment`를 쓰면 기본 subject에는 branch가 함께 들어가지 않는다. prod Environment의 허용 branch/tag 제한을 별도로 구성한다.
- 필요하면 `job_workflow_ref` 등을 subject에 포함해 승인된 reusable workflow로 좁힌다. claim이 존재하는 것만으로 Azure가 자동 검사한다고 가정하지 않는다.

GitHub는 2026-07-15 이후 생성된 repo 등에 ID 기반 subject를 적용한다고 명시한다. 과거 예시 복사보다 실제 claim 확인이 우선이다. [OIDC reference](https://docs.github.com/en/actions/reference/security/oidc)

UAMI를 선택하더라도 GitHub-hosted runner에서는 OIDC로 로그인한다. runner가 Azure VM의 IMDS를 쓰는 `auth-type: IDENTITY` 방식과 혼동하지 않는다. VM 내부에서는 VM 자신의 MI를 사용한다.
Client ID/Tenant ID/Subscription ID는 장기 비밀번호가 아닌 식별자다. 조직 정책에 따라 Environment variables 또는 secrets로 관리하고 client secret은 만들지 않는다.

파일럿의 권장 Identity 후보는 환경별 UAMI다. Azure 자원으로 수명과 RBAC를 관리하기 쉽고 runner에 연결하지 않아도 federated credential로 사용할 수 있다. 기존 조직 표준이 Entra app/service principal이면 같은 OIDC 구조로 적용할 수 있다. 두 방식 모두 권한과 신뢰는 별도로 설정한다.

아래는 보호 Environment와 federated credential을 구성한 뒤 사용하는 인증 확인 job 예시다. 파일럿에서 우선 dev로 시험하며, 실제 배포 단계는 포함하지 않는다. 표시한 major tag는 문서용이며 적용할 때 검토한 action commit SHA로 고정한다.

```yaml
jobs:
  verify-identity:
    environment: dev
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - name: Verify subscription context
        run: az account show --query '{subscription:id,tenant:tenantId}' --output json
```

위 예시는 token을 출력하지 않는다. login 성공은 Blob data 접근이나 VM 배포 권한까지 검증했다는 뜻이 아니므로 기능별 허용/거부 시험을 이어서 수행한다.

## 4. 권한 초안

아래는 기능별 최소 권한을 구성하기 위한 초안이며 완성된 custom role JSON이 아니다. 선택한 API·scope·SDK 조회 동작과 실제 거부 시험 후 확정한다.

| 주체 | 범위 | 권한 후보 | 기본 제외 |
|---|---|---|---|
| CI publisher | 해당 서비스의 Blob container | 업로드·조회용 data action, 필요 시 Blob Data Contributor로 시험 후 축소 | VM 실행·prod Gateway 변경·RBAC 수정 |
| prod deployer | 지정 VM 두 대 | VM read, `Microsoft.Compute/virtualMachines/runCommands/read`, `.../write`; 삭제는 정리/취소 책임에 따라 별도 | 임의 VM 생성·네트워크/Identity 관리·roleAssignments/write |
| prod deployer | 지정 App Gateway | `Microsoft.Network/applicationGateways/read`, `.../backendhealth/action` | probe 파일 방식에서는 gateway write 불필요 |
| 명시적 pool 제거 실행자 | 지정 App Gateway | 필요 시 `Microsoft.Network/applicationGateways/write` 및 실제 API의 관련 권한 검증 | 다른 Gateway·VNet 수정 |
| 잠금/상태 관리 주체 | 별도 deployment-control container | lease·상태 기록용 blob data 권한 | release 정리 권한과 분리 |
| VM MI | 필요한 release container | Storage Blob Data Reader | release 업로드·삭제·prod 배포 Identity 사용 |
| VM MI | 필요한 Key Vault 등 | 실제 런타임에 필요한 read만 | 관리/신뢰 변경 |
| 보존 정리 Identity | 확정된 release 범위 | 정책에 따른 조회·삭제 | 배포·승인·Identity 관리 |

Action Run Command의 `.../runCommand/action`과 Managed Run Command의 `.../runCommands/write`는 다르다. Backend Health 조회도 단순 Reader의 `*/read`로 충분하다고 단정하지 않는다. [Compute 권한](https://learn.microsoft.com/en-us/azure/role-based-access-control/permissions/compute), [Network 권한](https://learn.microsoft.com/en-us/azure/role-based-access-control/permissions/networking)

Run Command 권한은 VM에서 강한 권한으로 코드를 실행할 수 있게 한다. RBAC만으로 'A 서비스 배포 스크립트만 실행'을 강제할 수 없다. prod deployer는 신뢰된 중앙 workflow만 사용하고 변경 PR에 운영/보안 검토를 붙인다. UAMI 추가만으로 공유 VM 내부의 서비스별 보안 경계가 생기지도 않는다. VM의 MI는 그 VM 안 코드가 접근할 수 있는 리소스 경계이므로 엄격한 격리가 필요하면 VM 분리 또는 검증된 broker를 별도로 검토한다. [VM MI 경계](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-to-use-vm-token)

## 5. 누가 승인·설정하는가

| 작업 | 권한/주체 기준 | 파일럿 제안 |
|---|---|---|
| repo/workflow/Environment 관리 | GitHub의 해당 관리 권한 보유자 | 플랫폼 담당자가 정책 설정, 일반 개발자와 구분 |
| 운영 배포 승인 | Environment에 등록된 user/team, repo read 이상 | 기존 SE/운영팀을 reviewer로 지정하고 self-review 금지 |
| OIDC 신뢰 생성 | app 소유/적절한 Entra 권한 또는 UAMI federated credential 관리 권한 | Identity 담당자 |
| Azure 역할 부여 | 대상 scope의 role assignment 관리 권한 | Azure RBAC 관리자; deployer에 재부여 권한 주지 않음 |
| 승인된 job의 Azure 실행 | federated Identity에 부여된 RBAC | 승인자의 개인 Azure 권한 불필요 |
| 장애·잠금 해제 | 별도 운영 runbook과 권한 | 미종료 명령 확인·정지 및 최소 1대 확보 후 수행 |

prod Environment에 required reviewers, prevent self-review, 명시적 허용 branch/tag, administrator bypass 불허를 설정하는 안을 제안한다. 표준 required reviewers는 최대 6개의 사용자/팀을 등록하지만 그중 1명의 승인으로 진행한다. SE와 서비스 오너 모두의 승인이 필요하면 두 Environment의 순차 Gate 또는 custom rule을 설계해야 한다. custom deployment protection rules는 현재 public preview로 문서화되어 있다. [Environment 규칙](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)

승인 시 확인할 대상은 release ID·digest, 테스트 결과, 영향 서비스와 서버 쌍, rollback release, 가용 용량, 변경 티켓이다. 승인 뒤 실행할 manifest가 바뀌면 재승인한다. `.github/workflows`, 배포 스크립트, manifest·host mapping은 PR/ruleset/CODEOWNERS 검토 대상으로 둔다.
승인 job 뒤에 실제 prod 권한 job을 따로 무보호 상태로 두지 않는다. 모든 prod Azure 인증/변경 job은 승인된 Environment 신뢰에 묶는다.

## 6. self-hosted runner 없이 VM에 배포

| 대안 | 동작 | 장점 | 제약 및 운영/비용 |
|---|---|---|---|
| A. standard hosted runner + Managed Run Command | runner→공용 ARM API→Azure VM Agent→로컬 배포, VM이 Blob pull | SSH ingress·상주 Actions runner 불필요, 파일럿 구성이 단순 | VM Agent/egress 필요. runner의 사설 Blob/Key Vault 업로드는 별도 해결. 실행상태 polling·복구 필요 |
| B. Azure VNet 연결 GitHub-hosted larger runner | 관리형 runner가 VNet/private endpoint/연결된 IDC에 접근 | 사설 Blob·Key Vault·내부 smoke test 가능, runner OS 직접 운영 불필요 | larger runner 과금, subnet·NSG·DNS·egress 설정·지원 리전 검증 |
| C. self-hosted runner | 초기 검토의 내부망 실행 대안 | 내부망 실행 가능 | hosted 기반 선호에 따라 현재 추천에서 제외; Ansible도 목표 구성에서 제외 |

A 또는 B에서도 VM 배포 실행은 Run Command로 통일할 수 있다. VNet runner를 쓰는 것이 SSH를 쓰겠다는 결정은 아니다.
B는 standard runner의 옵션이 아니라 larger runner 기능이다. 사설망에서 GitHub와 Entra/ARM에 나갈 경로, 남아 있는 업무 IDC 의존성이 있다면 그에 대한 연결·DNS를 확인해야 한다. OIDC는 네트워크 연결을 만들어 주지 않는다. [VNet hosted runner](https://docs.github.com/en/organizations/managing-organization-settings/about-azure-private-networking-for-github-hosted-runners-in-your-organization)

Managed Run Command는 GitHub self-hosted runner와 별개인 Azure VM Agent 채널이다. `ProvisioningState=Succeeded`만 보지 않고 `instanceView.executionState`와 exit code 및 서비스 readiness를 확인한다. 복수 Managed Run Command는 병렬 실행이 가능하므로 자동 잠금으로 취급하지 않는다. [Managed Run Command](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command-managed)
기존 Action Run Command는 90분 제한, 한 번에 하나, 취소 제약 등이 있으므로 긴 배포에는 차이를 고려한다. VM Agent 통신·결과 반환 egress도 확인한다. [Action Run Command](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command)
Ansible은 사용자 결정으로 목표 구성에서 제외한다. 기존 playbook은 필요한 동작을 파악하는 입력 자료로만 쓰고, 검증 가능한 Bash/PowerShell 등 VM 로컬 배포 스크립트로 전환하는 안을 검토한다.

## 7. 'No password'의 실제 범위

- GitHub→Azure: OIDC federated workload identity.
- VM→Blob: VM MI와 Blob Data Reader. [AzCopy MI](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-authorize-managed-identity)
- VM→Entra 지원 서비스: 지원하는 MI/token 방식 검토.
- 남는 IDC 업무 DB·LDAP 등이 federation을 지원하지 않으면 별도 비밀값이 남을 수 있다. Key Vault로 옮기는 것은 비밀값 보관 개선이며 passwordless 전환 완료가 아니다. [기존 자격증명 접근](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/tutorial-windows-managed-identities-vm-access)
- GitHub 저장소 간 자동 호출: 일반 `GITHUB_TOKEN`은 호출 repo 범위다. 중앙 repo로 자동 dispatch하려면 별도 권한 설계가 필요하다. 파일럿은 중앙 repo의 사람이 작성한 manifest PR 또는 수동 workflow_dispatch부터 시작할 수 있다. 이후 GitHub App 설치 token/broker를 검토하되 App private key 관리가 남으면 완전한 장기 비밀값 제거라고 부르지 않는다. [GITHUB_TOKEN](https://docs.github.com/en/actions/concepts/security/github_token)
- dev 자동 배포를 유지하려면 중앙 repo가 Azure에 게시된 승인 가능한 release manifest를 OIDC로 주기 조회하는 방법도 비교할 수 있다. 추가 비밀값은 줄지만 지연·중복 처리·publisher 신뢰·출처 검증이 필요하다. 자동 dispatch 경로 확정 전 이 요구를 완료로 처리하지 않는다.

## 8. OIDC부터 설정하는 실행 순서 제안

1. org/repo 및 dev/prod 실제 Azure scope와 관리자·승인팀을 확정한다.
2. prod 권한을 중앙 배포 repo에 둘지 결정한다. build와 배포 Identity를 분리한다.
3. Environment와 workflow 변경 통제를 먼저 구성하고 실제 claim 형식을 확인한다.
4. Azure 관리자가 환경별 UAMI 또는 app 및 federated credential을 구성한다.
5. 필요한 리소스 scope에만 역할을 부여한다. 구독 Contributor/Owner를 기본값으로 사용하지 않는다.
6. dev에서 OIDC 로그인과 read-only 조회를 시험하고 잘못된 repo/branch/environment 인증이 실패하는지 확인한다.
7. 네트워크·VM Agent·Blob MI 다운로드를 시험한 뒤 drain 없는 파일 전달부터 검증한다.
8. 서버 쌍 잠금·실패 복구·무중단 시험을 통과한 뒤 prod 승인과 연계한다.

현재는 1~2를 위한 사실 확인과 대안 검토 단계다. 실제 변경에 필요한 식별자와 권한은 아직 제공되지 않았다.
