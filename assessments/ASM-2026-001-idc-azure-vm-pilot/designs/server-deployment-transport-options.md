# VM 배포 전송 모델 선택 — 직접 push 또는 명령 push·파일 pull

- Assessment: ASM-2026-001
- Step / Phase: STEP-01 / DISCOVERY; STEP-04/05 준비 조사
- Status: PROPOSED — SSH·Run Command·VM MI·Storage network 설정은 미적용
- 확인일: 2026-09-07

## 선택 범위

이번 파일럿에서 비교하는 VM 배포 방식은 두 가지다. 두 방식 모두 중앙 또는 개별 CD workflow가 release ID·digest·대상 pair를 고정하고, VM1 배포·검증 뒤 VM2로 진행한다.

| 선택지 | 명령 전달 | 파일 전달 | 주된 identity·경로 |
|---|---|---|---|
| A. 직접 push | GitHub Actions runner가 SSH 또는 동등한 VM 관리 endpoint로 직접 실행 | runner가 VM에 직접 복사 | runner→VM private/public network + SSH/endpoint 인증 |
| B. 명령 push + 파일 pull | GitHub Actions가 Azure ARM Managed Run Command를 호출 | VM이 Managed Identity로 Blob에서 fixed release를 download | runner→Entra/ARM OIDC, VM Agent/MI→Blob |

완전히 자율적인 VM timer/poller가 desired manifest를 읽는 방식도 가능하지만, 이번 선택은 사용자가 제시한 두 방식에 한정한다. 자율 pull은 별도 배포 실행 코드·지연·조정 책임이 생기므로 현재 기본안이 아니다.

## Option A — GitHub Actions가 SSH 등으로 직접 push

```text
GitHub Actions runner
  → SSH/SCP 또는 deployment endpoint
  → VM1: 파일 복사·Tomcat/Nginx 배포 스크립트 실행·검증
  → VM2: 같은 절차
```

| 관점 | 내용 |
|---|---|
| 네트워크 | runner가 VM 관리 port에 도달해야 한다. standard hosted runner에는 private VM 접속 경로가 없으므로 공용 endpoint/allowlist 또는 Azure VNet 연결 larger runner 등의 별도 설계가 필요 |
| 인증 | GitHub OIDC는 Azure API token이지 SSH login token이 아니다. SSH key·SSH certificate broker·별도 deployment API identity 중 하나를 설계해야 함 |
| 장점 | 파일 전송·명령·상태를 runner workflow에서 직접 제어하기 쉽고 Azure VM Agent 의존성이 없음 |
| 단점 | VM 관리 ingress·host key·SSH/endpoint credential·runner network·접근 회수·감사를 별도로 운영. 장기 SSH private key는 passwordless 요구와 맞지 않음 |
| Azure 권한 | Azure ARM Run Command 권한은 필수가 아니지만, Blob/Gateway 조회·잠금 등 선택 기능에 OIDC/RBAC가 필요할 수 있음 |
| 실패 처리 | runner 종료·network 단절·복사 중단·원격 script 잔존을 확인하고 VM2 진행을 막아야 함 |

직접 push를 선택하더라도 VM SSH를 인터넷에 공개하는 것이 유일한 방법은 아니다. VNet 연결 GitHub-hosted larger runner 또는 조직의 사설 연결을 쓸 수 있다. 다만 이 경우 larger runner, subnet·DNS·NSG·egress와 SSH credential lifecycle이 추가된다.

## Option B — Azure Managed Run Command로 명령 push, VM MI로 파일 pull

```text
GitHub Actions deploy job
  → GitHub OIDC → Azure deployer workload identity → ARM Managed Run Command
  → VM Agent가 local deploy script 실행
  → VM Managed Identity가 Blob에서 fixed release ID/digest pull
```

| 관점 | 내용 |
|---|---|
| 네트워크 | runner는 Entra/ARM HTTPS에만 접근. VM에 SSH/배포 endpoint ingress를 열지 않아도 됨. VM Agent와 Blob/Key Vault의 outbound·private DNS/route를 설계 |
| 인증 | GitHub job은 Entra federated credential으로 Azure deployer identity를 사용. VM은 별도의 Managed Identity로 Blob release·Key Vault runtime 항목을 읽음 |
| 장점 | GitHub-hosted runner에서 VM inbound 관리 port를 줄이고, 명령 실행과 artifact data path를 분리. 장기 SSH key 없이 구현 가능 |
| 단점 | VM Agent·Run Command 상태/timeout·egress에 의존. Run Command는 VM에서 강한 code execution 권한이며, workflow 취소 뒤 원격 실행이 남을 수 있음 |
| Azure 권한 | deployer identity에는 지정 VM의 Managed Run Command와 필요한 Gateway health/control store만. VM MI에는 Blob read와 필요한 Key Vault runtime read만 |
| 실패 처리 | ARM operation과 Run Command `instanceView` execution state·exit code, service readiness를 모두 확인. 상태 unknown이면 HOLD하고 VM2를 진행하지 않음 |

## 권장안과 선택 기준

| 기준 | 직접 push (A) | 명령 push·파일 pull (B) |
|---|---|---|
| self-hosted runner 회피 | VNet larger runner 또는 public/private SSH 경로가 필요 | standard hosted runner로도 ARM 경로 가능 |
| 장기 비밀값 제거 | SSH certificate/broker 등 별도 설계 없이는 어려움 | GitHub OIDC + VM MI로 기본 경로를 구성 가능 |
| VM inbound 최소화 | 관리 ingress가 필요 | SSH/배포 endpoint ingress 불필요 |
| artifact 접근 | runner가 release를 직접 보유·전송 | VM MI가 Blob canonical release를 직접 읽음 |
| 운영 복잡도 | SSH 키·host·network·endpoint 관리 | Azure Agent·RBAC·Storage network·remote execution 상태 관리 |

현재 요구인 GitHub-hosted runner 우선, 장기 비밀번호·서비스 계정 비밀값 제거, Blob release, Azure VM에 따라 **Option B**를 권장한다. Option A는 조직에 이미 검증된 SSH certificate/broker와 private runner network가 있고, Azure Agent/Blob pull을 쓰지 않기로 할 때 선택한다.

어느 방식을 선택해도 shared pair concurrency, persistent pair lock, drain/readiness, rollback, release digest 검증은 별도 공통 제어다. 선택은 [DEC-2026-007](../decisions/DEC-2026-007-server-deployment-transport.md)에 기록한다.

## 근거

- [Azure Managed Run Command](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command-managed)
- [Azure VM Run Command](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command)
- [GitHub-hosted runner Azure private networking](https://docs.github.com/en/organizations/managing-organization-settings/about-azure-private-networking-for-github-hosted-runners-in-your-organization)
- [Azure Login with GitHub OIDC](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect)
- [배포 절차 초안](../reports/deployment-draft.md)
