# Azure 관리·Identity·권한 모델

- Assessment: ASM-2026-001
- Date: 2026-09-08
- Status: DRAFT — Azure 구조와 권한 설계 초안. 실제 Azure tenant·subscription·resource·role assignment는 만들거나 변경하지 않았다.

## 목적

이 문서는 Azure의 비용·리소스·identity 체계를 분리해 설명하고, Azure VM 기반 CI/CD에서 각 자동 작업이 어떤 Azure identity와 권한으로 직접 접근해야 하는지 정리한다. GitHub Enterprise Cloud의 사람 권한·repository·Environment 설정은 [별도 GitHub 리포트](github-enterprise-cloud-access-model.md)에서 다룬다.

## 1. Azure의 세 가지 체계

Azure는 다음 세 가지 체계를 같은 것으로 보지 않는다.

```text
비용 체계       리소스 관리 체계                  Identity 체계
Billing Account  Entra Tenant                      Entra users / groups
  Billing Profile   Management Group                 Applications / service principals
    Invoice Section   Subscription                      Managed Identities
      Subscription      Resource Group                   Workload identity federation
                          Resource
```

### 비용 체계

| 단위 | 의미 |
|---|---|
| Billing Account | 회사의 Azure 계약·청구 최상위 단위 |
| Billing Profile | 청구서, 결제 수단, 청구 주소 단위 |
| Invoice Section | 부서·프로젝트·환경별 비용 분류 단위 |
| Subscription | Azure 사용량이 연결되는 비용·관리 단위 |

### 리소스 관리 체계

| 단위 | 의미 |
|---|---|
| Microsoft Entra Tenant | 사람·그룹·애플리케이션·Managed Identity가 있는 identity 디렉터리 |
| Management Group | 여러 Subscription에 공통 Policy·RBAC·거버넌스를 적용하는 상위 묶음 |
| Subscription | 비용, Azure RBAC, quota, Policy의 큰 경계 |
| Resource Group | 함께 생성·변경·운영·삭제할 Azure 리소스 묶음 |
| Resource | VM, Application Gateway, Storage Account, Key Vault, ACR 등의 실제 Azure 자원 |

Management Group은 리소스 또는 비용을 직접 담는 단위보다 여러 Subscription에 공통 정책을 적용하는 관리 단위에 가깝다. Subscription은 비용뿐 아니라 권한·quota·Policy의 중요한 경계다. Azure는 management group, subscription, resource group, resource의 네 수준 관리 범위를 제공한다. [Azure Resource Manager 관리 범위](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/get-started/how-azure-resource-manager-works)

Subscription은 하나의 Entra Tenant만 신뢰하고, 하나의 Tenant는 여러 Subscription에 identity를 제공할 수 있다. Tenant의 Global Administrator와 Subscription Owner는 서로 자동으로 부여되지 않는다. [Tenant와 Subscription 관계](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-users-assign-role-azure-portal)

## 2. Azure identity 종류

| 종류 | 사용 주체 | 용도 |
|---|---|---|
| Entra user | 사람 | Azure 관리자, 운영자, 개발자의 대화형 접근 |
| Entra group | 사람 그룹 | Azure RBAC를 사람에게 일괄 부여 |
| Application / service principal | 외부 또는 애플리케이션 workload | 기존 앱 통합, 외부 CI/CD workload |
| System-assigned Managed Identity | Azure 자원 하나 | 특정 VM처럼 한 Azure 자원 전용 identity |
| User-assigned Managed Identity, UAMI | 여러 Azure 자원 또는 외부 workload federation | 독립 수명·공유 권한·사전 RBAC 설정이 필요한 identity |
| Workload identity federation | 외부 CI/CD workload | 장기 secret 없이 외부 OIDC token을 Azure token으로 교환 |

Managed Identity는 Azure가 credential을 관리하는 workload identity다. system-assigned identity는 Azure 자원과 수명을 함께하고 하나의 자원에만 연결한다. UAMI는 별도 Azure 자원으로 만들며 여러 Azure 자원에 연결할 수 있고 수명이 독립적이다. 여러 VM이 같은 release storage나 설정을 읽는 경우 UAMI가 후보가 된다. [Managed Identity 개요](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)

## 3. 자동 배포의 identity 흐름

자동 작업은 다른 사람의 Azure 역할을 빌려 쓰는 것이 아니라, 각자에게 부여된 identity와 Azure RBAC로 직접 접근한다.

```text
외부 CI/CD deploy job
  → OIDC token
  → Azure workload identity의 단기 Azure token
  → Azure RBAC 범위에서 Run Command·Gateway 조회/제어

Azure VM
  → VM Managed Identity token
  → Blob release download·Key Vault runtime read
```

외부 CI/CD workload federation identity와 VM Managed Identity는 분리한다.

- 배포 workload identity: VM에 명령을 전달하고 Application Gateway 상태를 조회·필요 시 제어한다.
- VM Managed Identity: release 파일과 필요한 runtime secret만 읽는다.
- 사람 identity: Azure resource·RBAC·Policy의 관리와 긴급 복구에만 사용한다.

VM에 연결한 Managed Identity의 권한은 VM 안에서 코드를 실행할 수 있는 주체가 사용할 수 있다. 같은 VM에 여러 서비스가 있으면 VM identity 하나만으로 서비스별 비밀값 경계가 완전히 분리되지는 않는다. 따라서 최소 권한과 VM·secret 경계를 함께 검토한다. [Managed Identity 최소 권한 권장사항](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/managed-identity-best-practice-recommendations)

## 4. 환경별 identity 분리 초안

dev와 prod는 별도 identity, federation 신뢰, Azure RBAC scope를 사용한다. dev workflow가 prod VM을, prod VM이 dev Key Vault를 접근하지 못하게 하는 것이 목적이다.

| 논리 identity | 환경 | 사용 주체 | Azure 권한 범위 |
|---|---|---|---|
| `uami-ci-publisher` | 공통 또는 CI | release publisher | release container upload만 |
| `uami-deploy-dev` | dev | dev deploy workload | dev VM Run Command, dev Gateway 상태 조회, dev deployment-control storage |
| `uami-deploy-prod` | prod | prod deploy workload | 지정 prod VM Run Command, prod Gateway 상태 조회, prod deployment-control storage |
| `uami-vm-dev` | dev | dev VM pair | dev release Blob read, 필요한 dev Key Vault read |
| `uami-vm-prod` | prod | prod VM pair | prod release Blob read, 필요한 prod Key Vault read |
| `uami-acr-image-publisher` | 공통 build image | build image publisher | 공통 ACR image repository write |
| `uami-acr-image-reader` | CI consumer | service CI workload | 공통 ACR image repository read |

각 identity는 다음 세 가지가 모두 좁게 설정되어야 한다.

1. **신뢰 경계**: 외부 workload identity는 승인된 CI/CD repository·Environment·workflow OIDC claim만 신뢰한다.
2. **권한 경계**: Azure RBAC는 필요한 Azure resource 또는 Resource Group까지만 부여한다.
3. **환경 경계**: dev와 prod identity·Storage·Key Vault·VM·Gateway Resource Group을 분리한다.

UAMI가 유일한 선택은 아니다. 기존 조직 표준이 Entra application/service principal이면 federated credential을 붙여 같은 passwordless 구조를 만들 수 있다. 이 리포트의 UAMI 이름은 설계를 설명하기 위한 논리 이름이며 실제 Azure 자원 생성 결정은 아니다.

## 5. Azure RBAC 권한 배치 초안

| 작업 | 수행 identity | 권한 방향 | 권장 scope |
|---|---|---|---|
| release Blob upload | CI publisher | `Storage Blob Data Contributor` | release upload container |
| release Blob download | VM MI | `Storage Blob Data Reader` | release download container |
| Blob lifecycle 정책 변경 | infrastructure administrator | Storage management policy 변경 권한 | Storage Account |
| ACR common image push | ACR publisher | repository writer 또는 기존 RBAC의 push 권한 | 공통 image repository 또는 registry |
| ACR common image pull | CI consumer 또는 runtime VM MI | repository reader 또는 기존 RBAC의 pull 권한 | 공통 image repository 또는 registry |
| VM 배포 명령 | dev/prod deployer | Managed Run Command 실행·상태 조회 | 지정 VM 또는 해당 Resource Group |
| Gateway backend health 조회 | dev/prod deployer | Application Gateway read·backend health action | 지정 Application Gateway |
| Gateway backend 명시적 제외·복귀 | dev/prod deployer | Application Gateway write, 필요 API 권한 | 지정 Application Gateway |
| Key Vault runtime secret read | VM MI | 필요한 secret read 또는 key crypto operation만 | 해당 Key Vault·필요 항목 |

Blob data 권한은 control plane의 Storage Account Contributor와 별개다. Blob upload에는 `Storage Blob Data Contributor`, download에는 `Storage Blob Data Reader`를 사용한다. [Blob Entra 권한](https://learn.microsoft.com/en-us/azure/storage/blobs/authorize-access-azure-active-directory)

Application Gateway 상태 조회와 변경은 다른 권한이다. backend health 조회에는 `Microsoft.Network/applicationGateways/backendhealth/action`이 필요하고, Gateway 구성 변경은 `Microsoft.Network/applicationGateways/write`에 해당한다. Nginx/app readiness probe 방식만 사용하면 deployer에 Gateway write 권한을 기본으로 줄 필요가 없다. [Azure Networking 권한](https://learn.microsoft.com/en-us/azure/role-based-access-control/permissions/networking)

Managed Run Command 권한은 VM에서 강한 코드 실행 권한이므로 deployer identity의 trust와 RBAC scope를 특히 좁힌다. 실제 API에 따라 필요한 Run Command action/write permission을 custom role로 검증하고, VM Agent·timeout·exit code·취소 동작을 dev에서 시험한다. [Managed Run Command](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command-managed)

## 6. Blob release 보존

오래된 release 삭제는 별도 cleanup workload identity보다 Blob Lifecycle Management를 우선 사용한다. 배포 workload와 VM에 release 삭제 권한을 주지 않아도 된다.

```text
release container
├─ immutable/<service>/<release-id>/...
├─ protected/<service>/current/...
├─ protected/<service>/rollback/...
└─ deployment-control/<environment>/<pair-id>/...
```

| 대상 | 보존 방식 |
|---|---|
| 일반 immutable release | prefix 또는 Blob index tag를 대상으로 age 기반 lifecycle tier/delete |
| current·rollback release | lifecycle 대상과 분리하거나 정책 filter에서 제외 |
| deployment-control 상태·lock | release와 분리하고 장애 복구 정책을 별도 정의 |

Lifecycle policy는 시간·prefix·Blob index tag 기준으로 tier 변경·삭제한다. “최근 N개 유지”처럼 다른 blob의 상태나 개수를 비교하는 보존 규칙은 직접 처리하지 않는다. policy는 보통 하루 한 번 실행되므로 즉시 삭제 수단도 아니다. [Blob Lifecycle Management](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-structure)

## 7. ACR 공통 build image

공통 build image가 필요하면 image를 만드는 workload와 이를 사용하는 service CI workload를 분리한다.

```text
platform-build-images workload
  → ACR common image repository write

service CI workload
  → ACR common image repository read
  → application build
```

ACR가 ABAC repository 권한 모드이면 `Container Registry Repository Writer`와 `Container Registry Repository Reader`를 repository 조건으로 제한한다. 기존 RBAC 모드이면 `AcrPush`와 `AcrPull`을 사용한다. ABAC 모드에서는 기존 `AcrPush`·`AcrPull`이 적용되지 않으므로 registry mode를 먼저 확인한다. [ACR ABAC repository 권한](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-rbac-abac-repository-permissions)

Azure VM이 runtime container image를 직접 pull하게 되면 VM Managed Identity에 ACR reader/pull 권한을 준다. 현재처럼 WAR·정적 파일을 VM에 직접 배포하는 구조에서는 ACR은 build image 용도이며 runtime 배포의 필수 구성은 아니다.

private ACR image를 CI의 native job container로 사용하려면 image pull이 workflow step보다 먼저 일어나므로 별도 registry credential이 필요하다. passwordless 원칙을 유지하려면 runner에서 Azure workload identity로 로그인한 뒤 ACR image를 pull·실행하는 방식, 또는 GHCR의 GitHub package token 사용을 비교한다. [GitHub job container registry credentials](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/run-jobs-in-a-container)

## 8. 네트워크와 identity는 별개

identity는 Azure resource에 접근할 **권한**을 제공하고, network는 Azure endpoint까지 **도달할 경로**를 제공한다.

| 상황 | identity | network |
|---|---|---|
| CI workload가 Blob upload | Blob Contributor | Blob public authenticated endpoint 또는 private endpoint까지의 네트워크 경로 |
| VM이 Blob download | VM MI Blob Reader | VM VNet에서 Blob endpoint까지의 경로 |
| VM이 Key Vault read | VM MI secret read | VM VNet에서 Key Vault endpoint까지의 경로 |

Storage Account의 public network access를 Disabled로 설정하면 private endpoint를 통한 요청만 허용된다. 외부 hosted runner에 identity가 있어도 고객 VNet의 private endpoint에 네트워크로 도달하지 못하면 upload할 수 없다. public endpoint에서 Entra 인증을 사용하는 것은 anonymous public access와 다르다. [Azure Storage 보안](https://learn.microsoft.com/en-us/azure/storage/common/secure-storage)

## 9. 적용 순서

1. Tenant, Subscription, Resource Group과 dev/prod 환경 경계를 확정한다.
2. 사람 관리자용 Entra group과 Azure RBAC를 정리한다.
3. 환경별 deployer workload identity와 VM Managed Identity를 만든다.
4. external CI/CD OIDC federation trust를 각 환경 identity에 연결한다.
5. 최소 Azure RBAC를 Blob, VM, Gateway, Key Vault, ACR에 부여한다.
6. dev에서 OIDC login, Blob upload/download, Run Command, Gateway backend health 조회의 허용·거부 시험을 수행한다.
7. drain·rollback·lock 복구 시험을 통과한 뒤 prod identity와 권한을 적용한다.

## 확인 필요 항목

- 실제 Entra Tenant, Subscription, Resource Group, Application Gateway, VM pair, Storage, Key Vault, ACR 식별자
- 기존 Azure Landing Zone·Management Group·Policy·Private Endpoint 표준
- UAMI와 Entra application/service principal 중 조직의 workload identity 표준
- ACR RBAC-only 또는 RBAC+ABAC mode
- Blob lifecycle 보존 기간과 current·rollback 보호 방식
- Application Gateway에서 backend 직접 제외를 사용할지, custom readiness probe를 사용할지
- 사람이 직접 Azure에서 수행할 긴급 복구의 break-glass 절차와 감사 기준
