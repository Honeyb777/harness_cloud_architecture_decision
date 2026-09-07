# GitHub 접근 권한·Azure Identity 거버넌스 선택안

- Assessment: ASM-2026-001
- Step / Phase: STEP-01 / DISCOVERY
- Status: PROPOSED — 선택을 위한 분류이며 GitHub·Entra·Azure 설정은 미수행
- 확인일: 2026-09-07

## 목적과 분리 원칙

이 문서는 권한을 다음 세 층으로 나눈다. 같은 사람이 세 층에 권한을 갖거나 GitHub의 `Admin` 역할이 Azure 권한을 자동으로 갖는 것으로 해석하지 않는다.

| 층 | 통제 대상 | 질문 | 설정 위치 |
|---|---|---|---|
| GitHub 사람 권한 | 누가 코드·workflow·Environment 설정을 바꾸는가 | 어느 team이 어느 repo에서 Read/Write/Maintain/Admin인가 | GitHub Organization / repository |
| GitHub job 권한 | 어느 workflow job이 OIDC token·GitHub API를 요청하는가 | job에 필요한 `permissions:`는 무엇인가 | workflow YAML / Environment |
| Azure workload identity | 어떤 검증된 job과 VM이 어느 Azure 리소스에 접근하는가 | 어떤 OIDC claim이 어떤 Entra identity로 교환되고, 그 identity의 RBAC scope는 어디인가 | Entra federated credential / Azure RBAC |

Azure 배포 권한은 승인자의 개인 Azure 권한이 아니라, 승인된 job이 교환한 workload identity의 Azure RBAC에서 나온다. VM 런타임의 Managed Identity도 GitHub 배포 identity와 별도다.

## 권한 배치표 — GitHub와 Azure를 분리해서 보기

사람의 GitHub 로그인과 Azure 로그인은 별도 계정·권한 체계다. 회사 Entra ID를 GitHub Enterprise Cloud SSO에 연결할 수는 있지만, SSO는 로그인 연계일 뿐 repository role을 Azure RBAC에 복사하거나 Azure RBAC를 GitHub role에 복사하지 않는다.

| 주체 | GitHub에서 갖는 권한 | Azure에서 갖는 권한 | 배포 시 하는 일 |
|---|---|---|---|
| 개발자 | source repository `Write` | 기본값 없음 | 코드·CI 변경 PR 작성. 운영 VM을 직접 배포하지 않음 |
| 서비스 오너 | source repository `Maintain`, deployment repository `Read` | 기본값 없음 | 릴리스 후보와 변경 내용을 검토 |
| 운영 승인자 | deployment repository `Read`, `prod` Environment reviewer | 기본값 없음 | 운영 배포를 승인 또는 거절. Azure 리소스를 직접 변경하지 않음 |
| CI/CD 관리자 | deployment repository의 workflow/Environment 관리 권한 | 기본값 없음 | Actions·Environment 정책을 관리. Azure RBAC는 별도 담당자가 부여 |
| Azure RBAC 관리자 | 필요 시 GitHub `Read` | Entra federated credential·Azure RBAC 관리 권한 | OIDC 신뢰와 최소 RBAC를 구성. 배포 승인자가 아님 |
| GitHub Actions의 prod deploy job | job별 `id-token: write`, 필요한 최소 `GITHUB_TOKEN` | **prod deployer workload identity**가 가진 지정 VM·Gateway 조회·control storage 권한 | 승인된 manifest만 사용해 Azure Run Command를 호출 |
| Azure VM의 배포/런타임 프로세스 | 없음 | **VM Managed Identity**의 Blob release read·Key Vault runtime read 권한 | 승인된 release를 내려받고 필요한 런타임 설정을 읽음 |

`prod deployer workload identity`와 `VM Managed Identity`는 Azure에 존재하지만, 사람의 Azure 계정과도 다르다. 전자는 GitHub Actions job이 OIDC로 잠시 사용하는 identity이고, 후자는 Azure VM 안에서만 사용하는 identity다.

```text
[GitHub] 개발자·승인자·CI/CD 관리자
     │ repository role / Environment 승인
     ▼
GitHub Actions prod deploy job
     │ GitHub OIDC (승인된 repo·Environment·workflow claim만)
     ▼
[Azure] prod deployer workload identity
     │ Azure RBAC (지정 VM·Gateway·control storage만)
     ▼
Azure VM ── VM Managed Identity ──> Blob release / Key Vault runtime 항목
```

이 표의 기본값은 사람이 Azure subscription의 `Contributor`나 `Owner`를 받아 직접 배포하지 않는 것이다. 예외적인 수동 복구가 필요하면 별도 break-glass 절차·감사 기록·만료 조건을 정의한다.

## 1. GitHub repository 단위 사람 권한

실제 Organization, repo, team 이름은 `UNK-01`이 해소될 때 채운다. 아래 역할명은 조직의 사람 또는 team 이름이 아니라 책임을 구분하기 위한 논리 역할이다.

| 논리 역할 | source repository | 중앙 deployment repository | GitHub Organization 권한 | 허용 업무 | 기본 제외 |
|---|---|---|---|---|---|
| 개발자 | Write | Read 또는 없음 | Member | 애플리케이션 코드·일반 CI 변경 PR | production Environment 관리, 배포 workflow 직접 수정, Azure RBAC |
| 서비스 오너 | Maintain | Read | Member | branch 보호·릴리스 후보 검토 | repo 접근자/Environment/Actions secret 관리, Azure RBAC |
| 배포 운영자 | Read | Maintain | Member | release manifest·배포 run 확인, 장애 runbook 수행 | source 코드 임의 변경, Organization/Entra 관리자 권한 |
| 운영 승인자 | Read | Read | Member | production Environment 승인 | workflow·secret·Azure RBAC 변경, 본인 승인 |
| CI/CD 관리자 | 필요 시 Maintain | Admin 또는 custom role | CI/CD admin 또는 최소 custom organization role | Actions 정책·runner/network·workflow/Environment 관리 | Azure RBAC 및 앱 런타임 secret 읽기 |
| GitHub 조직 관리자 | 필요 시 Admin | 필요 시 Admin | Owner를 최소 2인으로 유지 | 조직·SSO·정책·복구 관리 | 일상 배포 승인·Azure resource 변경 |

GitHub의 기본 repository role은 Read, Triage, Write, Maintain, Admin이다. GitHub Enterprise Cloud의 custom repository role을 쓰는 경우에도 기본 권한과 team 권한은 누적될 수 있으므로, 실제 부여 전 repository access export로 합산 결과를 확인한다.

### repository 구조 선택

| 선택지 | source와 deployment repository | 장점 | 위험·운영 영향 |
|---|---|---|---|
| A. 중앙 deployment repository 분리 | 서비스 repo는 build·release 후보만 만들고, 중앙 repo만 운영 배포 job을 실행 | prod workflow·Environment·OIDC subject·감사 경로를 한곳에 고정; 공유 VM pair 잠금에 적합 | manifest 반입과 중앙 repo 운영 책임을 정해야 함 |
| B. 서비스 repository별 배포 | 각 서비스 repo가 build와 deploy를 모두 실행 | 서비스별 흐름이 단순 | repo마다 Environment·OIDC·RBAC·공통 잠금 통제를 반복; 공유 VM 경합 위험 증가 |

파일럿 권장안은 A다. 이는 [DEC-2026-002](../decisions/DEC-2026-002-deployment-trust-boundary.md)의 Option A와 같은 방향이며, 아직 사용자 선택은 아니다.

### workflow와 Environment의 최소 정책 초안

| 대상 | 정책 초안 | 목적 |
|---|---|---|
| source repo | default `GITHUB_TOKEN`을 read로 두고, job마다 필요한 `permissions:`만 선언 | job token 권한 확대 방지 |
| deployment repo workflow | `.github/workflows/`, 배포 스크립트, VM host mapping, manifest 검증 코드를 branch protection/ruleset과 CODEOWNERS 검토 대상으로 둠 | prod identity를 얻는 코드 변경 통제 |
| `dev` Environment | 실제 dev scope만 신뢰·RBAC 부여; 배포 기록을 남김 | 최소 권한 OIDC·배포 경로 시험 |
| `prod` Environment | required reviewers, prevent self-review, 허용 branch/tag를 적용하고 administrator bypass 정책을 조직 기준으로 결정 | 사람 승인과 job 신뢰를 함께 통제 |
| OIDC가 필요한 deploy job | `id-token: write`를 해당 job에만 부여; build/test job에는 부여하지 않음 | OIDC token 발급 범위 축소 |

Environment 승인만으로 Azure 신뢰가 자동 제한되지는 않는다. prod Azure 인증을 하는 모든 job은 `environment: prod`를 선언하고, Azure federated credential도 그 Environment를 포함한 실제 claim에 한정한다.

## 2. GitHub job과 Azure identity 연동

OIDC 연동은 GitHub repository의 사람 권한을 Entra에 동기화하는 기능이 아니다. GitHub Actions job이 발급받은 OIDC token의 issuer, audience, subject를 Entra application/service principal 또는 User-assigned Managed Identity(UAMI)의 federated credential과 일치시켜 Azure access token으로 교환하는 방식이다.

| 구성 요소 | 권장 분리 | 신뢰 조건 | Azure 권한 |
|---|---|---|---|
| CI publisher identity | 서비스 또는 release publishing 경계별 | source repo의 승인된 branch/tag 및 publisher workflow | 해당 release container에 upload/read만; VM·Gateway·RBAC 권한 없음 |
| dev deployer identity | dev Environment | 중앙 deployment repo + `environment: dev` + 승인된 workflow | dev VM·control storage read/write와 필요한 조회만 |
| prod deployer identity | prod Environment | 중앙 deployment repo + `environment: prod` + 허용 branch/tag + 승인된 workflow | 지정 prod VM Run Command·Gateway 상태 조회·control storage만 |
| VM runtime identity | VM 또는 신뢰 경계별 Managed Identity | Azure VM에서 IMDS token 사용; GitHub OIDC 없음 | 필요한 Blob release read·Key Vault runtime read만 |
| retention identity | 정리 작업 전용 | 승인된 cleanup workflow 또는 별도 운영 경로 | 확정된 retention 범위의 목록·삭제만 |

OIDC token의 실제 `iss`, `aud`, `sub`와 필요한 claim은 설정 직전 검증한다. 이름 기반 예시를 복사하지 않고 GitHub의 현재 ID 기반 subject 형식과 대상 repo/Environment를 사용한다. token 본문은 로그나 증적에 기록하지 않는다.

### deployment identity 모델 선택

| 선택지 | 모델 | 장점 | 위험·운영·감사 영향 |
|---|---|---|---|
| A. 환경별 UAMI | `uami-deploy-dev`, `uami-deploy-prod`처럼 환경별 UAMI에 각각 federated credential과 최소 RBAC 부여 | dev/prod 권한·감사·폐기를 분리하고 Azure 리소스 수명으로 관리하기 쉬움 | identity와 credential 수가 증가; 실제 Azure scope 매핑 필요 |
| B. 환경별 Entra app/service principal | 환경별 app에 federated credential과 최소 RBAC 부여 | 기존 Entra app 운영 표준이 있으면 통합 가능 | app ownership·수명·감사 책임을 별도로 관리해야 함 |
| C. 하나의 공유 identity | source/build/deploy와 여러 환경이 하나의 identity를 공유 | 초기 객체 수가 적음 | trust와 RBAC가 합쳐져 prod 오용·감사 분석·폐기 범위가 커짐 |

파일럿 권장안은 A이며, 조직의 app/service principal 표준이 이미 확정되어 있다면 B를 동등한 구조로 검토한다. C는 파일럿 기본안에서 제외한다. UAMI를 선택해도 GitHub-hosted runner는 VM Managed Identity로 로그인하지 않으며 OIDC로 UAMI의 federated credential을 사용한다.

## 3. Identity 정책 선택안

| 정책 영역 | 권장 정책 | 선택 또는 확인 필요 |
|---|---|---|
| 사람 identity | Enterprise SSO/조직 정책에 따르고, team으로 repo 권한을 부여하며 개인별 직접 Admin 부여를 예외로 관리 | SSO·MFA·SCIM·break-glass의 현행 정책과 담당자 |
| GitHub 관리자 | GitHub Organization Owner는 최소 2인, CI/CD 설정은 별도 CI/CD admin 또는 custom role로 위임 | enterprise/organization 관리자와 긴급 복구 책임자 |
| 배포 승인 | 개발자·운영 승인자·Azure RBAC 관리자를 분리; prod self-review 차단 | 승인자가 한 명이면 충분한지, 서비스 오너와 운영자의 이중 승인 필요 여부 |
| GitHub job identity | build, publish, dev deploy, prod deploy, cleanup을 분리; prod identity는 중앙 deployment repo의 protected Environment job만 신뢰 | 중앙 deployment repo 채택 여부와 실제 workflow 경로 |
| Azure RBAC | resource group·VM·storage container 등 최소 scope부터 부여; subscription Contributor/Owner를 기본값으로 쓰지 않음 | tenant/subscription/RG·VM·Gateway·Storage의 실제 ID와 custom role 필요성 |
| VM identity | GitHub deployer와 분리; VM MI에는 runtime Blob/Key Vault data-plane 권한만 부여 | VM 공유가 서비스 간 secret/data 경계에 허용되는지 |
| 비밀값 | client secret·PAT를 Azure 로그인 수단으로 만들지 않음. Entra 미지원 기존 시스템의 secret은 Key Vault에 별도 관리 | IDC DB/API/패키지 registry의 passwordless 지원 여부 |
| 권한 검토·감사 | 정기 access review, GitHub audit log, Azure Activity Log와 deployment manifest/run URL/operation ID를 연결 | 검토 주기·감사 보존 기간·증적 보관 책임 |

## 4. 선택 순서와 Decision Gate

이 순서는 GitHub repository의 사람 권한부터 확정해 Azure identity를 연결할 수 있게 한다. 현재는 STEP-01 Discovery이므로, 아래 선택을 모두 완료했다고 표시하지 않는다.

1. Organization, source repo, 중앙 deployment repo 후보와 dev/prod Environment를 식별한다.
2. 논리 역할을 실제 GitHub team 및 repository role에 매핑하고, 직접·상속·custom role의 합산 권한을 검토한다.
3. 중앙 deployment repository(A) 또는 서비스별 배포(B)를 사용자 결정으로 기록한다.
4. dev/prod deployer에 UAMI(A) 또는 Entra app(B)를 사용자 결정으로 기록한다.
5. 실제 OIDC claim을 확인한 뒤 federated credential의 issuer/audience/subject를 고정한다.
6. 실제 Azure scope에 RBAC를 설계하고, 허용·거부 시험을 수행한다.

다음 User Decision은 두 건으로 분리한다.

- **D-01: 운영 배포 repository 경계** — 중앙 deployment repository(A) 또는 서비스 repository별 배포(B).
- **D-02: 배포 workload identity 유형** — 환경별 UAMI(A) 또는 조직 표준의 환경별 Entra app/service principal(B).

두 선택은 Azure resource 생성이나 권한 부여 승인이 아니다. 실제 federated credential, RBAC, Environment protection, VM/Storage 설정 변경은 대상 식별·최소 권한 설계와 별도의 적용 승인이 갖춰진 뒤 수행한다.

## 근거

- [GitHub repository roles](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/repository-roles-for-an-organization)
- [GitHub custom repository roles](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/about-custom-repository-roles)
- [GitHub organization CI/CD admin](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-peoples-access-to-your-organization-with-roles/permissions-of-predefined-organization-roles)
- [GitHub Actions OIDC claims](https://docs.github.com/en/actions/reference/security/oidc)
- [Azure Login with GitHub OIDC](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect)
- [기존 권한·배포 상세](identity-and-delivery-options.md)
