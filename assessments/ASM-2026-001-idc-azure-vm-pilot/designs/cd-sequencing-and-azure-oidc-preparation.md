# CD 순차 제어와 Azure OIDC 준비

- Assessment: ASM-2026-001
- Step / Phase: STEP-01 / DISCOVERY; STEP-03/05 준비 조사
- Status: PROPOSED — GitHub Environment·OIDC·RBAC·lock·workflow는 미적용
- 확인일: 2026-09-07

## 1. 여러 Actions가 같은 서버 그룹을 배포할 때

GitHub Actions는 기본적으로 여러 workflow/job을 병렬 실행한다. 같은 VM 쌍을 여러 서비스가 공유하면 각 서비스 repository의 배포 job이 서로 다른 VM을 동시에 drain·restart할 수 있으므로, 서비스별 workflow만으로는 안전하지 않다.

### GitHub concurrency가 보장하는 범위

| 방식 | 동작 | 공유 VM 쌍에 충분한가 |
|---|---|---|
| 서비스 repository별 `concurrency` | 같은 repository 안에서 같은 group의 job을 직렬화 | 아니오. 다른 repository의 group과는 독립 |
| 중앙 deployment repository의 `concurrency` | 실제 dev/prod deploy job을 한 repository에서 실행하고 `pair-<environment>-<pair-id>` group을 공유 | 같은 중앙 repository를 거치는 자동 배포에는 유효 |
| 중앙 repository + 영속 pair lock/state | concurrency 외에 Blob 등의 control store에 pair 상태·lease·현재 실행·fencing 정보를 기록 | 수동 작업, Run Command 잔존, 재시작·취소·교차 경로까지 통제하기 위한 권장 조합 |

GitHub concurrency group은 한 번에 하나만 실행한다. 기본 `queue: single`은 실행 중인 job 하나와 대기 job 하나만 남기며, 새 요청이 오면 기존 대기 job을 취소한다. 모든 배포 요청을 순서대로 처리해야 하면 `queue: max`를 명시해 최대 100개까지 대기시킨다. `queue: max`와 `cancel-in-progress: true`는 함께 쓸 수 없다.

파일럿 권장안은 중앙 deployment repository에서 실제 배포 job을 실행하고 아래처럼 서버 쌍별 group을 지정하는 것이다. source repository의 build/publish job은 VM 배포를 직접 시작하지 않는다.

```yaml
concurrency:
  group: deploy-prod-pair-${{ inputs.pair_id }}
  queue: max

jobs:
  deploy:
    environment: prod
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
```

이것은 workflow 구조 예시이며 실제 repository, pair ID, branch/tag 조건, action SHA는 미확정이다. GitHub concurrency만으로 VM side effect를 fencing하지는 못한다. deploy job은 시작 후에도 control store의 pair lock을 획득하고, 현재 실행·peer health·이전 Run Command 종료 상태를 확인해야 한다. lock을 기다리는 shell loop를 runner step 안에 두지 않는다.

### source CI와 중앙 CD의 역할

| 단계 | source repository | 중앙 deployment repository |
|---|---|---|
| CI | source commit으로 build/test, release manifest·파일 digest 생성 | 실행하지 않음 |
| release publish | 승인된 publisher 경로가 Blob에 immutable release ID를 게시 | publish 결과를 검증할 수 있음 |
| CD 요청 | release ID·source repo·commit·digest·대상 pair를 전달 | 요청을 검증하고 concurrency queue에 넣음 |
| dev/prod 배포 | VM/Gateway 권한 없음 | Environment 승인, OIDC Azure login, pair lock, Run Command·검증·rollback 실행 |
| 수동 재배포 | release를 새로 build하지 않음 | `workflow_dispatch`로 기존의 승인된 release ID와 pair를 선택해 재배포 가능 |

즉, source repository의 push는 CI와 release publish를 시작하고, CD는 중앙 repository가 맡는다. 중앙 CD가 release를 받는 기본 후보는 source CI가 publisher identity로 Blob에 release를 게시한 뒤 metadata를 전달하거나, 중앙 repository가 승인된 manifest를 읽어 배포 대상으로 등록하는 방식이다. source repository의 Actions Artifact를 중앙 repository가 매번 직접 download하려면 교차 repository token·만료·권한 경로가 추가되므로 기본안으로 두지 않는다.

## 2. 대기와 runner 실행 시간

| 대기 위치 | runner 실행 시간 | 판단 |
|---|---|---|
| GitHub concurrency의 `pending` 상태 | job이 실행되지 않은 대기 상태. GitHub billable time은 job execution time으로 표시되므로 pilot에서 Usage로 확인 | runner step에서 sleep/lock polling하지 않음 |
| GitHub Environment wait timer | GitHub가 명시적으로 billable time에 포함하지 않음 | 정해진 변경 대기 시간에 사용 가능 |
| required reviewer 승인 대기 | 승인 전 Environment secret에 접근하지 못하며 job은 대기 상태 | 실제 private repository Usage로 runner allocation·청구를 확인 |
| runner step에서 Blob lease/lock 획득을 기다림 | **예. runner가 이미 실행 중** | 금지. lock 실패면 job을 종료하고 중앙 queue/retry/state 절차로 재조정 |
| Run Command 결과 polling·VM drain·배포·검증 | **예. runner가 실행 중** | 필요한 CD 실행 시간. timeout·HOLD 기준을 둠 |

GitHub의 concurrency pending과 required-reviewer 대기의 과금 표시는 실제 enterprise 계약·workflow 상태에서 확인해야 한다. 이번 파일럿은 `VAL-35`에서 pending/approval/runner step wait를 각각 실행해 billable minutes와 job 실행 시간을 비교한다.

## 3. Azure 권한을 받는 방식: OIDC, Key Vault, GitHub Secrets의 역할

Azure CD 권한은 Key Vault에서 비밀번호를 꺼내거나 Azure client secret을 GitHub Secrets에 저장하는 방식으로 받지 않는다. 권장 흐름은 GitHub Actions OIDC이다.

```text
GitHub prod deployment job
  ├─ `id-token: write`로 짧은 GitHub OIDC token 요청
  ├─ Azure Login이 token의 issuer/audience/subject를 Entra와 교환
  ▼
Entra UAMI 또는 application/service principal의 federated credential
  └─ Azure RBAC: 지정 VM Run Command·Gateway health·control storage만
```

| 구성 요소 | 어디에 준비하는가 | 보관하는 것 | 하지 않는 일 |
|---|---|---|---|
| GitHub Environment `dev` / `prod` | 중앙 deployment repository | required reviewers, branch/tag 규칙, self-review/bypass 정책 | Azure RBAC를 직접 부여하지 않음 |
| GitHub workflow | 중앙 deployment repository | `id-token: write`, 최소 `contents`/`packages` 권한, Environment 참조 | Azure client secret 보관·사용하지 않음 |
| GitHub Environment secrets 또는 variables | `dev`/`prod` Environment | `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` 식별자. Azure 문서는 secrets 사용 예시 제공 | client secret을 만들 필요 없음 |
| Entra federated credential | 환경별 UAMI 또는 Entra application | GitHub issuer, audience, 실제 repo/Environment/workflow claim subject | 사람의 GitHub repository role을 Azure RBAC로 변환하지 않음 |
| Azure RBAC | 실제 VM·App Gateway·Storage scope | deployer identity의 최소 control-plane/data-plane 권한 | 구독 Owner/Contributor를 기본 부여하지 않음 |
| Key Vault | VM/Gateway 런타임 secret 경계 | DB/API secret, TLS certificate, JWT key 등 | GitHub→Azure OIDC 로그인용 client secret 저장소가 아님 |

Azure 문서는 OIDC에서도 client ID, tenant ID, subscription ID를 GitHub Secrets에 두는 예시를 제공한다. 세 값은 비밀번호가 아닌 식별자지만, Environment secret으로 두면 approval 전 job에서 접근할 수 없어 environment별 경계를 명확히 할 수 있다. 조직 정책이 식별자를 일반 변수로 취급하면 Environment variables로 둘 수 있으나, client secret은 생성하지 않는다.

중앙 deployment repository를 쓰면 이 세 값을 모든 source repository에 복사하지 않는다. GitHub UI에서 **중앙 repository → Settings → Environments → `dev` 또는 `prod` → Environment secrets**에 각각 등록한다.

```text
prod Environment secrets
  AZURE_CLIENT_ID       = uami-deploy-prod 또는 prod Entra application의 client ID
  AZURE_TENANT_ID       = 대상 Entra tenant ID
  AZURE_SUBSCRIPTION_ID = 대상 Azure subscription ID
```

실제 인증 비밀값은 GitHub에 저장하지 않는다. GitHub job은 실행 시점에만 OIDC token을 요청하고, Entra의 federated credential이 repository·Environment claim을 검증한 뒤 해당 workload identity로 Azure token을 발급한다. Key Vault는 이 세 값을 가져오거나 OIDC 교환을 수행하지 않는다.

### 준비 순서

1. 실제 중앙 deployment repository, `dev`/`prod` Environment, pair ID와 Azure tenant/subscription/resource group을 식별한다.
2. Azure 관리자가 `uami-deploy-dev`, `uami-deploy-prod` 또는 조직 표준의 환경별 Entra application을 만든다.
3. 각 identity에 실제 GitHub OIDC issuer/audience/subject를 신뢰하는 federated credential을 하나씩 만든다. prod는 중앙 repository와 `environment: prod` claim으로 좁힌다.
4. Azure RBAC 관리자가 dev/prod scope에 최소 권한을 부여한다. deployer에는 role assignment·network·identity 관리 권한을 주지 않는다.
5. GitHub 관리자가 Environment protection과 `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`를 환경별로 등록한다.
6. dev에서 올바른 repo/Environment OIDC login과 잘못된 repo/branch/Environment 거부를 먼저 시험한다.
7. 그 다음 Run Command, pair lock, VM MI Blob pull, drain·rollback을 순서대로 검증한다.

## 4. 최소 workflow login 형태

```yaml
jobs:
  deploy:
    environment: prod
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: azure/login@v2 # 실제 적용 시 검증한 commit SHA로 고정
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - run: az account show --query '{subscription:id,tenant:tenantId}' --output json
```

위 job은 Azure login 확인 예시일 뿐 VM 배포 권한을 부여하거나 secret 값을 출력하지 않는다. Azure Login 뒤에 Run Command·Gateway 조회를 추가하기 전에 scope별 허용/거부 시험과 pair lock 절차가 필요하다.

## 근거

- [GitHub Actions concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency)
- [GitHub Environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
- [GitHub job execution time](https://docs.github.com/en/actions/how-tos/monitor-workflows/view-job-execution-time)
- [Azure Login with GitHub OIDC](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect)
- [운영 배포 trust boundary 결정](../decisions/DEC-2026-002-deployment-trust-boundary.md)
- [배포 절차 초안](../reports/deployment-draft.md)
