# GitHub Enterprise Cloud와 Azure OIDC 연결·사용 모델

- Assessment: ASM-2026-001
- Date: 2026-09-08
- Status: DRAFT — GitHub 권한 모델과 Azure identity 모델을 연결한 구현 순서 초안. 실제 federated credential, workflow, RBAC, Azure resource 변경은 미수행.

## 목적

GitHub Enterprise Cloud의 production 배포 보호와 Azure의 workload identity·RBAC를 OIDC로 연결한다. 장기 Azure client secret, 서비스 계정 비밀번호, VM SSH key 없이 외부 CI/CD workload가 필요한 Azure API를 호출하는 구조가 목표다.

관련 기준은 [GitHub 권한·배포 보호 모델](github-enterprise-cloud-access-model.md)과 [Azure 관리·Identity·권한 모델](azure-management-and-identity-model.md)을 따른다.

## 1. 전체 흐름

```text
GitHub repository의 deploy job
  → GitHub Environment 보호 규칙 통과
  → GitHub OIDC token 요청
  → Entra federated credential이 token claim 검증
  → Azure workload identity의 단기 Azure token 발급
  → Azure RBAC 범위에서 Azure API 호출
  → Azure Run Command가 VM Agent에 배포 명령 전달
  → VM Managed Identity가 Blob·Key Vault 접근
```

OIDC는 GitHub job이 Azure 관리 API에 로그인하는 방식이다. VM의 SSH 로그인이나 VM Managed Identity를 대신하지 않는다.

| 주체 | 사용하는 identity | 수행 작업 |
|---|---|---|
| GitHub dev deploy job | `uami-deploy-dev` 또는 동등한 Entra application | dev Azure API 호출 |
| GitHub prod deploy job | `uami-deploy-prod` 또는 동등한 Entra application | prod Azure API 호출 |
| dev VM | `uami-vm-dev` | dev release Blob·Key Vault runtime 접근 |
| prod VM | `uami-vm-prod` | prod release Blob·Key Vault runtime 접근 |

## 2. 연결 전에 정할 경계

OIDC를 만들기 전에 아래 항목을 정한다. 이름을 먼저 정하지 않고 identity·RBAC를 만들면 나중에 prod trust가 넓어질 수 있다.

| 항목 | dev 예시 | prod 예시 |
|---|---|---|
| GitHub deployment repository | `org/deployment` | `org/deployment` |
| GitHub Environment | `dev` | `prod` |
| 실행 가능한 ref | `develop` 또는 허용 dev branch | `main`, `release/*`, 승인 tag |
| Azure workload identity | `uami-deploy-dev` | `uami-deploy-prod` |
| Azure scope | dev Resource Group | prod Resource Group 또는 지정 Resource |
| VM Managed Identity | `uami-vm-dev` | `uami-vm-prod` |

source repository에서 운영 Azure 권한을 직접 얻는지, 중앙 deployment repository만 얻는지는 별도 운영 결정이다. 공유 VM pair와 여러 repository가 있는 현재 초안에서는 중앙 deployment repository가 prod OIDC trust를 갖는 구조를 우선 검토한다.

## 3. GitHub 측 준비

### 3.1 Environment 보호

`dev`, `prod` Environment를 deployment repository에 만든다.

| 설정 | dev | prod |
|---|---|---|
| Required reviewers | 조직 정책에 따라 선택 | 운영 승인 Team 지정 |
| Self-review 방지 | 선택 | 사용 |
| 허용 branch/tag | dev branch 제한 | `main`, release branch 또는 승인 tag 제한 |
| Administrator bypass | 조직 정책에 따라 결정 | 일반 배포 금지, 긴급 절차만 별도 기록 |

Environment는 GitHub의 사람 승인·배포 보호 장치다. Azure RBAC 권한을 주는 기능은 아니다.

### 3.2 Workflow job 최소 권한

Azure OIDC가 필요한 deploy job에만 `id-token: write`를 준다. build/test job에 이를 기본으로 주지 않는다.

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

`id-token: write`는 GitHub가 OIDC token을 발급할 수 있게 하는 job permission이다. Azure에서 어떤 작업을 할 수 있는지는 이 권한이 아니라 federated credential과 Azure RBAC로 결정된다. Azure login에 쓰는 client ID, tenant ID, subscription ID는 식별자이며 client secret이 아니다.

## 4. Azure 측 준비

### 4.1 Workload identity 생성

환경별 UAMI 또는 Entra application/service principal을 만든다.

| 선택 | 장점 | 적용 기준 |
|---|---|---|
| 환경별 UAMI | Azure resource로 수명·RBAC·감사를 관리하기 쉬움 | GitHub Actions OIDC workload의 기본 후보 |
| 환경별 Entra application | 기존 조직의 application/service principal 운영 표준과 통합 가능 | 조직 표준이 이미 있는 경우 |

어느 선택이든 dev와 prod는 별도 identity를 사용한다. identity 하나에 dev와 prod Azure 권한을 함께 넣지 않는다.

### 4.2 Federated credential 생성

각 Azure workload identity에 GitHub OIDC issuer, audience, subject 조건을 가진 federated credential을 만든다.

| 항목 | 값의 의미 | 설정 원칙 |
|---|---|---|
| Issuer | OIDC token을 발급한 주체 | GitHub Enterprise Cloud의 Actions issuer와 정확히 일치 |
| Audience | Azure token 교환 대상 | Azure public cloud 기본값 또는 실제 cloud 값 확인 |
| Subject | 어떤 GitHub workflow 실행을 신뢰할지 | repository·Environment·실제 claim 형식에 정확히 제한 |

Environment를 job에 지정하면 subject는 Environment 기준으로 구성될 수 있다. branch/tag 제한은 GitHub Environment의 deployment branch/tag rule로 별도 적용한다. GitHub는 repository 생성·rename·transfer 시 ID 기반 subject 형식도 지원하므로, 이름 기반 예시를 그대로 복사하지 않고 실제 token claim을 확인한 뒤 federated credential을 만든다.

권장 신뢰 범위는 아래처럼 최소화한다.

```text
uami-deploy-dev
  ← deployment repository + environment:dev

uami-deploy-prod
  ← deployment repository + environment:prod
```

필요하면 reusable workflow의 경로 같은 추가 claim을 포함해 신뢰 범위를 더 좁히는 방안을 검토한다. Azure는 federated credential에 설정한 조건만 검증하므로, 존재하는 claim을 실제로 설정에 반영해야 한다.

GitHub OIDC와 Azure의 연결 방식은 [Azure Login with OIDC](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect), claim 구성은 [GitHub OIDC reference](https://docs.github.com/en/actions/reference/security/oidc)를 기준으로 실제 구현 직전에 재확인한다.

### 4.3 Azure RBAC 부여

federated credential은 “어느 GitHub job을 신뢰할지”를 정하고, Azure RBAC는 “신뢰된 job이 Azure에서 무엇을 할 수 있는지”를 정한다.

| identity | 필요한 작업 | Azure RBAC 방향 |
|---|---|---|
| `uami-deploy-dev` | dev VM 배포·Gateway 상태 확인 | dev VM Run Command, dev Gateway read·backend health, dev control storage |
| `uami-deploy-prod` | prod VM 배포·Gateway 상태 확인 | prod VM Run Command, prod Gateway read·backend health, prod control storage |
| `uami-deploy-prod` | Gateway backend 직접 제외 방식 선택 시 | 지정 Gateway write를 별도 추가 |
| `uami-vm-dev` | dev release download·runtime 설정 | dev Blob Data Reader, 필요한 dev Key Vault read |
| `uami-vm-prod` | prod release download·runtime 설정 | prod Blob Data Reader, 필요한 prod Key Vault read |

`uami-deploy-prod`에 Subscription Contributor/Owner를 기본으로 주지 않는다. Run Command, Application Gateway, Blob data plane 권한은 서로 분리해 실제 API와 custom role을 dev에서 허용·거부 시험한 뒤 확정한다.

## 5. 실제 사용 단계

### 5.1 dev 배포

1. 허용된 dev ref의 workflow가 `environment: dev` job에 진입한다.
2. job이 GitHub OIDC token을 요청한다.
3. Entra가 `uami-deploy-dev`의 federated credential 조건과 token claim을 비교한다.
4. 일치하면 job이 짧은 시간 Azure access token을 받고 dev Azure API를 호출한다.
5. job은 Run Command로 dev VM 배포를 시작하고 Gateway backend health를 확인한다.
6. VM은 `uami-vm-dev`로 Blob에서 고정 release ID를 내려받고 필요한 Key Vault 값을 읽는다.

### 5.2 prod 배포

1. 허용된 prod ref의 workflow가 `environment: prod` job에 진입한다.
2. GitHub Environment의 운영 승인·self-review·branch/tag 규칙을 통과한다.
3. job이 GitHub OIDC token을 요청한다.
4. Entra가 `uami-deploy-prod`의 federated credential 조건과 token claim을 비교한다.
5. 일치하면 job이 prod Azure API를 호출한다.
6. job은 pair lock·VM 1 drain·Run Command·검증·재편입·VM 2 반복을 수행한다.
7. prod VM은 `uami-vm-prod`로 Blob과 Key Vault에 접근한다.

## 6. OIDC가 허용하지 않는 경우

아래는 의도적으로 Azure token 교환 또는 Azure API 호출이 실패해야 하는 경우다.

| 상황 | 기대 결과 |
|---|---|
| 허용되지 않은 repository의 workflow | federated credential 불일치로 Azure token 교환 거부 |
| prod가 아닌 Environment job | prod federated credential 불일치로 거부 |
| prod 허용 branch/tag가 아닌 실행 | GitHub Environment 진입 또는 신뢰 조건에서 거부 |
| build/test job의 OIDC token 요청 | `id-token: write`가 없으면 token 요청 불가 |
| dev deployer가 prod VM Run Command 호출 | prod scope에 RBAC가 없으므로 거부 |
| prod VM identity가 dev Key Vault read | dev vault RBAC가 없으므로 거부 |

정상 login만 확인하지 않고 위의 거부 시험을 dev에서 수행한다. OIDC 설정 오류와 Azure RBAC 오류는 원인이 다르므로 workflow log, Entra sign-in log, Azure Activity Log를 함께 확인한다.

## 7. 감사와 운영 증적

| 증적 | 확인 내용 |
|---|---|
| GitHub Actions run | 요청자, commit/release ID, Environment 승인, workflow 실행 결과 |
| GitHub Environment deployment | 승인자, 승인 시각, 보호 규칙 적용 여부 |
| Entra sign-in log | workload identity token 교환 성공·실패 |
| Azure Activity Log | Run Command, Application Gateway, RBAC 등 control plane 작업 |
| Blob deployment state | pair lock, 대상 VM, release digest, 원격 command ID, 단계 상태 |
| VM application log | 실제 기동·readiness·release 적용 결과 |

OIDC token 자체, Key Vault secret, 사용자 JWT, 원본 민감 로그는 증적에 저장하지 않는다.

## 8. 적용 순서

1. GitHub deployment repository와 `dev`·`prod` Environment 보호 규칙을 확정한다.
2. Azure에 `uami-deploy-dev`, `uami-deploy-prod` 또는 동등한 identity를 만든다.
3. 각 identity에 실제 GitHub OIDC claim을 기준으로 federated credential을 만든다.
4. dev Azure resource에 최소 RBAC를 부여한다.
5. dev job에서 OIDC login과 read-only Azure 조회를 먼저 시험한다.
6. dev에서 Blob upload/download, Run Command, Gateway backend health 조회의 허용·거부 시험을 수행한다.
7. dev 배포·drain·rollback·lock 복구를 검증한다.
8. 동일한 분리 원칙으로 prod identity·federated credential·RBAC·Environment 보호를 설정한다.

## 확인 필요 항목

- 실제 GitHub Organization, deployment repository, Environment, branch/tag 정책
- 실제 OIDC issuer, audience, subject 및 추가 claim
- UAMI와 Entra application 중 조직의 workload identity 표준
- 실제 Azure Tenant, Subscription, Resource Group, VM, Gateway, Storage, Key Vault 식별자
- Run Command의 custom role operation과 최소 scope
- Application Gateway backend 직접 제외 여부와 Gateway write 필요성
- 감사 로그 보존 기간과 긴급 복구의 break-glass 절차
