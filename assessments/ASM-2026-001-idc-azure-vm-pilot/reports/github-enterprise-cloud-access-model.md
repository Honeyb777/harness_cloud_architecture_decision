# GitHub Enterprise Cloud 권한·배포 보호 모델

- Assessment: ASM-2026-001
- Date: 2026-09-08
- Status: DRAFT — GitHub Enterprise Cloud의 공식 기능을 기준으로 정리한 권한 설계 초안. 실제 Organization·repository·Environment 설정은 미수행.

## 목적

GitHub Enterprise Cloud에서 사람의 접근 권한, production 배포 승인, workflow가 사용하는 자동 권한을 구분한다. 이 문서는 Azure RBAC를 대신하지 않는다. GitHub는 “누가 코드·배포 절차를 변경하고 승인할 수 있는가”를 통제하고, Azure는 “승인된 자동 작업이 어떤 Azure 리소스를 변경할 수 있는가”를 통제한다.

## 1. 권한 계층

```text
Enterprise
  └─ Organization
       └─ Team
            └─ Repository
                 └─ Environment (dev / prod)
                      └─ Workflow job permissions
                           └─ Azure OIDC / Azure RBAC
```

| 계층 | 관리 대상 | 이 파일럿에서의 역할 |
|---|---|---|
| Enterprise | 여러 Organization의 공통 보안·SSO·정책 | 조직 전반의 계정·보안 기준을 관리 |
| Organization | 구성원, Team, repository, Actions 조직 정책 | 조직의 사람·Team·CI/CD 운영 기준을 관리 |
| Team | 사람 그룹 | 개발, 서비스 오너, 운영 승인, CI/CD 관리자를 묶어 repository 권한을 부여 |
| Repository | 코드, workflow, branch protection, 접근 권한 | source repository와 deployment repository의 사람 접근 경계 |
| Environment | 특정 repository의 배포 대상 보호 | `dev`, `prod` 배포 승인·허용 branch/tag·Environment secret 통제 |
| Workflow job | 자동 작업의 GitHub token·OIDC 요청 범위 | build job과 deploy job의 `permissions:`를 분리 |
| Azure | federated credential, RBAC, Managed Identity | GitHub 승인 이후 VM·Blob·Gateway에 실제로 할 수 있는 작업을 제한 |

## 2. Environment는 branch가 아니다

Branch는 코드의 변경 줄기이고, Environment는 workflow가 배포 대상으로 지정하는 보호 단위다. 하나의 repository에 `main`, `release/*` branch와 `dev`, `prod` Environment가 함께 존재한다.

```text
repository
├─ main branch
├─ release/2026-09 branch
├─ dev Environment
└─ prod Environment
```

workflow job이 `environment: prod`를 참조하면 `prod`에 설정한 보호 규칙을 적용받는다. Environment에서 어떤 branch/tag가 배포 가능한지 제한할 수 있으므로, 예를 들어 `main`과 `release/*`만 prod Environment를 사용할 수 있게 설정한다. Environment가 branch 자체를 만들거나 branch protection을 대체하지는 않는다.

## 3. Repository 기본 역할

Organization repository의 기본 역할은 아래 다섯 가지다.

| 역할 | 주 용도 | 이 파일럿의 권장 대상 |
|---|---|---|
| Read | 코드·실행 결과 조회, PR review | 운영 승인자, 감사자 |
| Triage | 이슈·PR 관리, 코드 push 불가 | 운영 지원 또는 프로젝트 관리 역할 |
| Write | 코드 push, PR merge, 일반 workflow 실행 | 개발자 |
| Maintain | repository 운영 설정 일부 관리, 민감·파괴적 작업 제외 | 서비스 오너 |
| Admin | 접근 권한, repository 설정, branch protection 등 전체 관리 | 최소 인원의 repository 관리자 |

`Write`는 코드와 일반 workflow를 변경·실행할 수 있는 권한이다. production 배포 승인이나 Azure VM 변경 권한을 자동으로 뜻하지 않게 구성한다. `Admin`은 접근 권한과 repository 설정을 변경할 수 있으므로 운영 배포 repository에서는 최소 인원에게만 부여한다. GitHub의 역할별 세부 권한은 [공식 repository role 표](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/repository-roles-for-an-organization)를 따른다.

## 4. Production workflow에 추가할 보호 설정

repository 역할만으로 production 배포를 보호하지 않고, `prod` Environment와 workflow를 함께 보호한다.

| 설정 | 하는 일 | 권장 초안 |
|---|---|---|
| Required reviewers | `prod` job 실행 전 지정한 사용자·Team의 승인을 요구 | 운영 승인 Team을 지정 |
| Prevent self-review | workflow를 시작한 사람이 자기 배포를 승인하지 못하게 함 | 사용 |
| Deployment branches/tags | 특정 branch/tag에서만 `prod` Environment 배포 허용 | `main`, 검토된 `release/*` 또는 승인 tag로 제한 |
| Administrator bypass | repository 관리자가 보호 규칙을 건너뛸 수 있는지 결정 | 일반 운영은 금지, 긴급 절차는 별도 기록 |
| Environment secret/variable | 해당 Environment를 참조하고 승인된 job만 값을 읽음 | runtime secret은 Key Vault 우선; GitHub에는 identity 식별자 등 최소값만 |
| Branch protection / ruleset | workflow·배포 스크립트 변경을 PR review·status check로 보호 | deployment workflow, host mapping, 배포 스크립트에 CODEOWNERS 검토 |
| Job `permissions:` | 각 job이 요청할 GitHub API 권한을 최소화 | build는 read 중심, deploy job만 필요한 `id-token: write`·`contents`·`packages` 권한 |

`prod` Environment가 적용되는 job 예시는 다음과 같다. Azure의 실제 권한은 이 YAML에 있지 않고, 뒤의 OIDC와 Azure RBAC에서 결정된다.

```yaml
jobs:
  deploy-prod:
    environment: prod
    permissions:
      contents: read
      id-token: write
    runs-on: ubuntu-latest
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```

`id-token: write`는 GitHub Actions job이 OIDC token을 요청할 수 있게 하는 권한이다. 이것만으로 Azure VM을 변경할 수 있는 것은 아니다. Azure의 federated credential이 해당 job의 claim을 신뢰하고, 그 Azure identity에 필요한 Azure RBAC가 있어야 한다.

Environment는 최대 여섯 명 또는 Team을 required reviewer로 둘 수 있으며, 그 중 한 명의 승인이 기본적으로 job 진행에 충분하다. 다중 독립 승인이 필요하면 Environment를 나누어 순차 gate로 만들거나 조직의 별도 절차를 설계한다. [GitHub Environments](https://docs.github.com/en/enterprise-cloud@latest/actions/reference/workflows-and-actions/deployments-and-environments)

## 5. 기본 repository 역할 외의 역할

기본 repository 역할만으로 운영하지 않고, 아래 Organization·Team·custom 역할을 필요에 따라 사용한다.

| 구분 | 역할 또는 기능 | 용도 | 파일럿 적용 방향 |
|---|---|---|---|
| Organization | Member | 일반 조직 구성원 | 개발·운영 인원의 기본 역할 |
| Organization | Owner | Organization 전체 관리 | 최소 2명, 일상 배포·승인과 분리 |
| Organization | CI/CD admin | Actions 정책, runner, hosted network, 조직 secrets·variables·사용량 관리 | CI/CD 플랫폼 담당자에게만 부여 검토 |
| Organization | Security manager | 조직·모든 repository 보안 정책과 경고 관리 | 보안 담당자에게만 부여 검토 |
| Organization | Billing manager | 구독·비용 조회·관리 | 비용 담당자에게만 부여 검토 |
| Organization | App manager | GitHub App 생성·관리 | GitHub App 연동이 있을 때만 부여 |
| Organization | All-repository role | 모든 repository에 Read/Write/Maintain/Admin 일괄 부여 | 넓은 Write/Admin은 지양; 감사·보안 Read에 제한적으로 검토 |
| Team | Team maintainer | Team 구성과 일부 Team 설정 관리 | 역할별 Team 운영자에게만 부여 |
| Repository | Custom repository role | 기본 역할보다 세밀한 권한 조합 | 기본 역할로 부족할 때만 도입 |
| Organization | Custom organization role | 조직 기능에 대한 세밀한 권한 조합 | CI/CD·보안 운영을 Owner와 분리할 때 검토 |
| Environment | Required reviewer | 배포 자체를 승인하는 사람·Team | repository role과 별도 배포 보호 계층 |
| Workflow | `GITHUB_TOKEN` / OIDC permission | 사람 역할과 독립된 자동 작업 권한 | job별 최소 권한 선언 |

CI/CD admin은 Azure 권한이 아니다. GitHub Actions 정책·runner·조직 secrets 등을 관리하는 GitHub Organization 역할이다. Security manager와 Billing manager도 Azure 구독 권한을 자동으로 주지 않는다. [GitHub Organization 사전 정의 역할](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-peoples-access-to-your-organization-with-roles/permissions-of-predefined-organization-roles)

Custom repository role은 기본 Read/Triage/Write/Maintain/Admin으로 충분하지 않을 때 사용한다. 예를 들어 특정 보안 기능 관리나 repository 설정 일부만 위임하는 경우에 검토한다. 먼저 기본 역할과 Environment 보호로 운영 가능한지 확인하고, 역할이 늘어나서 권한 검토가 복잡해지는 경우에는 도입하지 않는다. [Custom repository role](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/about-custom-repository-roles)

## 6. 권장 역할 배치 초안

| 논리 역할 | GitHub 권한 | Environment | Azure 권한 |
|---|---|---|---|
| 개발자 | source repository `Write` | 없음 | 기본값 없음 |
| 서비스 오너 | source repository `Maintain`, deployment repository `Read` | 필요 시 dev reviewer | 기본값 없음 |
| 운영 승인자 | deployment repository `Read` | `prod` required reviewer | 기본값 없음 |
| CI/CD 관리자 | deployment repository 관리 권한, 필요 시 Organization CI/CD admin | Environment 정책 관리 | 기본값 없음 |
| GitHub 조직 관리자 | Organization Owner | 일반 승인자와 분리 | 기본값 없음 |
| Actions deploy job | 사람 repository role과 별도 job permission | `prod`를 참조 | OIDC로 얻는 최소 Azure RBAC |
| VM 런타임 | GitHub 권한 없음 | 없음 | VM Managed Identity의 Blob·Key Vault 최소 권한 |

이 표에서 운영 승인자는 GitHub에서 배포를 승인하지만 Azure Portal에서 VM을 직접 변경하지 않는다. Actions deploy job은 사람의 Azure 계정이 아니라 OIDC로 받은 단기 Azure token을 사용한다. VM은 별도 Managed Identity로 release 파일과 필요한 runtime secret만 읽는다.

## 7. 설정 순서

1. Organization의 Owner, CI/CD admin, Security manager, Billing manager 책임자를 정한다.
2. 개발·서비스 오너·운영 승인·CI/CD 관리 Team을 만든다.
3. source repository와 deployment repository의 Team별 기본 역할을 부여한다.
4. deployment repository에 `dev`, `prod` Environment를 만든다.
5. `prod` Environment에 reviewer, self-review 차단, 허용 branch/tag, bypass 정책을 설정한다.
6. deployment workflow·스크립트·host mapping을 branch protection/ruleset과 CODEOWNERS로 보호한다.
7. build/deploy job의 GitHub `permissions:`를 최소화한다.
8. 이 GitHub 보호 조건을 실제 OIDC federated credential과 Azure RBAC의 repository·Environment·scope 제한에 연결한다.

## 확인 필요 항목

- Enterprise와 Organization의 실제 관리자·SSO·SCIM·감사 정책
- source/deployment repository 분리 여부와 Team 구성
- `prod` reviewer가 한 명의 승인으로 충분한지, 서비스 오너와 운영의 이중 승인 절차가 필요한지
- administrator bypass의 허용 범위와 긴급 배포 기록 방식
- custom role이 기본 역할보다 실제로 필요한지
- GitHub OIDC claim과 Azure federated credential·RBAC의 실제 값
