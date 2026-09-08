# GitHub Actions CD와 Azure Pipelines CD: 공유 이중화 VM 배포 운영 모델

- 대상: IDC 서비스의 Azure VM 전환 파일럿
- 작성일: 2026-09-08
- 상태: DRAFT — 운영 선택지를 설명하는 리포트다. GitHub, Azure DevOps, Azure Application Gateway, VM에 실제 설정·권한 부여·배포 실행은 하지 않았다.

## 이 리포트의 범위

여러 Java/Tomcat 서비스와 Vue 정적 파일이 같은 두 Azure VM에 있고, Azure Application Gateway가 두 VM으로 traffic을 보낸다는 전제로 CD를 설명한다. CI가 이미 Blob에 고정 release ID, WAR/JAR, frontend 파일, checksum, manifest를 게시한 뒤의 단계다.

비교 대상은 다음 두 가지다.

1. **GitHub Actions만 사용**: 중앙 deployment repository의 GitHub Actions가 CD 요청을 순서대로 처리한다.
2. **GitHub Actions CI + Azure Pipelines CD**: GitHub Actions는 CI만 처리하고, Azure Pipelines가 CD 승인과 요청 순서를 처리한다.

## 먼저 구분할 것: 잠금과 무중단 전환은 다른 기능이다

pair lock, GitHub Actions `concurrency`, Azure Pipelines Environment exclusive lock은 모두 **동시에 실행되는 배포를 제한하는 기능**이다. VM traffic을 자동 전환하거나 Tomcat을 자동 교체하지 않는다.

```text
순차 실행 제어                         무중단 배포 절차
────────────────                       ────────────────────────
pair lock / concurrency                Nginx marker off
Azure Pipelines exclusive lock     →   Application Gateway probe 감지
Environment approval                   기존 요청 drain 확인
                                        Blob release 적용
                                        Tomcat restart + readiness
                                        marker on + Gateway Healthy 확인
                                        VM1 성공 뒤 VM2 반복
```

잠금은 “한 VM 쌍에서 지금 누가 배포를 실행할 수 있는가”를 정한다. 무중단 배포 절차는 “그 배포가 VM1과 VM2의 traffic을 어떻게 안전하게 전환하는가”를 정한다. 둘 다 필요하다.

## 왜 공유 VM 쌍에 전역 순서가 필요한가

서비스 A와 서비스 B가 같은 VM1/VM2에 배포된다고 가정한다. 서비스별 workflow가 동시에 실행되어 A가 VM1을, B가 VM2를 동시에 drain하면 Application Gateway에 남은 Healthy backend가 없을 수 있다. 한 서비스만 배포했더라도 다른 서비스 요청까지 함께 실패할 수 있다.

따라서 배포 잠금 단위는 개별 service repository가 아니라 **공유 VM pair**다.

```text
service A CD ─┐
service B CD ─┼─► prod-rg-a-vm-pair-01 배포 대기열 ─► 한 요청만 실행
service C CD ─┘
                                                   │
                                             VM1 → 검증 → VM2
```

여기서 `prod-rg-a-vm-pair-01`은 Azure가 자동으로 만드는 값이 아니다. Resource Group, VM pair, 환경을 기준으로 조직이 정한 **논리적 배포 대상 식별자**다.

## CD 조정 방식 비교

| 항목 | GitHub Actions만 사용 | GitHub Actions CI + Azure Pipelines CD |
|---|---|---|
| CI | GitHub Actions | GitHub Actions |
| CD 실행 위치 | 중앙 deployment repository의 GitHub Actions | Azure DevOps Pipeline |
| 순차 실행 단위 | 중앙 repository의 `concurrency.group`과 pair lock | Azure DevOps Environment exclusive lock과 pair lock |
| 승인 | GitHub Environment reviewer·branch/tag 규칙 | Azure DevOps Environment approval/check |
| Azure 인증 | GitHub OIDC와 Entra federated credential | Azure DevOps service connection workload identity federation |
| 배포 이력 | GitHub Actions run·Environment deployment | Azure DevOps Environment·Pipeline deployment history |
| 자동으로 하지 않는 일 | Gateway drain, Tomcat 교체, readiness, rollback | 동일 |

### GitHub Actions만 사용하는 경우

중앙 deployment repository가 모든 서비스의 release ID를 받아 CD를 실행한다. `concurrency.group: deploy-prod-rg-a-vm-pair-01`처럼 pair를 group에 넣어 같은 pair에 대한 workflow를 순차 처리한다. GitHub Actions의 concurrency는 repository 범위이므로, 모든 shared-pair CD가 이 중앙 repository를 거치게 해야 한다.

GitHub Environment는 승인과 branch/tag 제한을 담당한다. Azure 권한은 deploy job의 GitHub OIDC token, Entra federated credential, Azure RBAC 조합으로 얻는다.

### Azure Pipelines를 CD에 사용하는 경우

GitHub Actions CI가 Blob release를 게시하고 Azure Pipelines가 고정 release ID를 입력받아 배포한다. `prod-rg-a-vm-pair-01`에 대응하는 Azure DevOps Environment에 approval과 exclusive lock을 설정하면, 그 Environment를 쓰는 deployment stage가 순차 실행된다.

Azure Pipelines exclusive lock도 Resource Group이나 VM을 자동 인식하지 않는다. 해당 Environment를 쓰도록 pipeline을 구성해야 하며, Azure Portal 수동 작업이나 그 Environment를 거치지 않는 다른 pipeline까지 막아주지 않는다.

Azure Pipelines CD는 GitHub OIDC를 그대로 쓰지 않는다. Azure DevOps service connection의 workload identity federation과 Azure RBAC을 별도로 구성한다. [Azure Pipelines approvals and checks](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops), [workload identity federation](https://learn.microsoft.com/en-us/azure/devops/pipelines/release/configure-workload-identity?view=azure-devops)

## 권장 배포 흐름

```text
Blob release: release ID + manifest digest + checksums
                     │
                     ▼
             pair lock / CD queue 획득
                     │
          peer VM이 Gateway Healthy인가?
               ┌─────┴─────┐
             아니오       예
               │           │
             HOLD          ▼
                    VM1 Nginx marker off
                           │
                    Gateway probe가 Unhealthy 감지
                           │
                    기존 요청 drain 기준 확인
                           │
                    VM1이 Blob에서 release pull
                           │
                    Tomcat 교체 · Vue 전환 · readiness
                           │
                    marker on → Gateway Healthy 확인
                           │
                    안정 관찰 성공 시 VM2 반복
                           │
                    결과 기록 후 pair lock 해제
```

배포 전 VM2가 Healthy인지 확인하는 이유는 VM1을 제외하는 동안 VM2가 유일한 backend가 되기 때문이다. VM2가 Unhealthy, 용량 부족, maintenance 중이거나 이전 배포의 상태가 Unknown이면 VM1의 marker를 내리지 않고 HOLD한다.

## Application Gateway, Nginx, Tomcat의 역할

| 구성 요소 | 역할 | 하지 않는 일 |
|---|---|---|
| Application Gateway | custom probe 결과로 VM을 Healthy/Unhealthy로 분류하고, backend traffic을 전송 | 배포 순서 조정, Tomcat 교체, release 파일 download |
| Nginx | `/_deploy/ready` 같은 readiness endpoint 제공, marker on/off로 신규 요청 수신 가능 상태 노출, Vue 파일 제공 | Azure Pipeline/GitHub queue 관리 |
| Tomcat | 새 WAR/JAR를 같은 port에서 기동하고 application readiness를 반환 | Gateway에서 자기 자신을 제거 |
| CD workflow/pipeline | pair lock, marker 제어, Azure 명령 실행, 검증과 HOLD/rollback 흐름 관리 | Application Gateway probe 없이 traffic을 안전하게 전환 |

“수동 probe 제외”는 배포 script가 Nginx marker를 off로 바꾸어 readiness endpoint가 실패하게 만드는 동작이다. “자동 감지”는 Application Gateway가 해당 endpoint를 probe하여 backend 상태를 Unhealthy로 바꾸는 동작이다. probe가 Unhealthy가 되었다고 기존 연결이 즉시 끝나는 것은 아니므로 connection draining과 실제 HTTP·WebSocket·SSE 요청 종료 기준을 별도로 검증해야 한다. [Application Gateway probes](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-probe-overview), [connection draining](https://learn.microsoft.com/en-us/azure/application-gateway/configuration-http-settings)

## Blob download와 VM 전송 방식

| 선택 | 방식 | 장점 | 추가 조건 |
|---|---|---|---|
| A. 직접 push | runner/agent가 SSH 또는 deployment endpoint로 파일과 명령을 VM에 전달 | pipeline이 파일 전송을 직접 제어 | VM inbound 경로, private runner network 또는 공개 endpoint, SSH key/certificate·host key 관리 |
| B. 명령 push + 파일 pull | CD job이 Azure Managed Run Command로 VM script를 시작하고, VM Managed Identity가 Blob에서 release를 받음 | VM inbound 배포 port와 SSH credential을 줄이고 CI/deployer·VM runtime identity를 분리 | VM Agent, Blob outbound/private network, Run Command timeout·잔류 실행 관리 |

파일럿의 Blob release와 passwordless identity 전제에서는 **B**가 권장안이다. GitHub Actions 또는 Azure Pipelines가 Azure control plane에서 Run Command를 요청하고, VM은 자신의 Managed Identity로 고정 release ID와 checksum을 Blob에서 읽는다. [Azure Managed Run Command](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command-managed)

## Vue 이전 버전 호출을 보호하는 방법

새 HTML을 받은 브라우저 탭과 오래 열린 이전 HTML 탭이 잠시 공존할 수 있다. 이전 HTML이 참조한 JS/CSS chunk를 제거하면 404가 발생할 수 있다.

| 선택 | 방식 | 사용자 영향 |
|---|---|---|
| 단일 active slot | `current` 파일만 유지하고 이전 asset 삭제 | 이전 탭의 asset 요청 실패 가능성을 수용, 새로고침이 필요할 수 있음 |
| hash/version asset 2버전 공존 | 이전·새 release asset URL을 함께 제공하고 새 HTML만 새 경로를 참조 | 이전 탭의 asset 요청을 보호, asset 보존과 cleanup 정책이 필요 |

권장안은 **hash/version asset 2버전 공존**이다. 먼저 VM1과 VM2 모두에 이전·새 asset을 준비하고 두 URL이 모두 200인지 확인한다. 그 다음 각 VM의 `current` HTML을 새 release로 전환한다.

```text
/srv/<service>/frontend/
  releases/r101/index.html       → /assets/r101/app.a1b2.js
  releases/r102/index.html       → /assets/r102/app.c3d4.js
  assets/r101/...
  assets/r102/...
  current → releases/r102
```

`/assets/app.js` 같은 공용 경로를 새 파일로 덮어쓰면 이전 HTML도 새 파일을 요청한다. 따라서 2버전 공존 효과를 내려면 HTML이 release별 versioned asset URL을 참조해야 한다. cleanup은 양 VM의 `current`, `previous`, in-flight, rollback-pinned release를 확인한 뒤 수행한다. 보존 시간은 사용자 세션과 browser cache 정책을 확인해 정한다.

## 실패 시 중단 기준

| 상황 | 처리 |
|---|---|
| peer VM이 Healthy가 아님 | 배포 시작 전 HOLD, 대상 VM drain 금지 |
| marker off 뒤 Gateway 상태 확인 실패 | probe/backend health를 확인하고 HOLD |
| Blob checksum 또는 manifest 불일치 | Tomcat·Vue 전환 전 중단, VM2 미실행 |
| VM1 Tomcat readiness 실패 | marker off 유지, VM2 및 다음 서비스 CD 차단 |
| pipeline/job 취소·timeout | Run Command와 VM 실제 상태를 확인할 때까지 다음 CD 차단 |
| VM2 실패 | VM1을 즉시 되돌린다고 가정하지 않고, 두 VM의 실제 release와 traffic 상태를 기록한 뒤 정해진 rollback 절차 수행 |

## 선택이 필요한 내용

1. CD 조정기를 GitHub Actions 중앙 deployment repository와 Azure Pipelines 중 어디로 둘지
2. pair 식별자와 queue/Environment 이름을 Resource Group·VM pair·환경에 어떻게 매핑할지
3. pair lock/control store와 수동 Azure 작업을 관리하는 운영 절차
4. marker, probe interval/threshold, connection drain, Tomcat warm-up, 남은 VM의 최소 용량 기준
5. 직접 push와 명령 push + VM Blob pull 중 실제 네트워크·인증 조건에 맞는 방식
6. Vue 이전 asset 보존 기간, cleanup 주체, cache header 정책

## Pilot 검증

- 다른 서비스의 동시 CD 요청이 같은 VM pair에서 하나씩만 실행되는지
- peer VM이 비정상이면 대상 VM drain이 시작되지 않는지
- marker off → Gateway Unhealthy → drain → release 적용 → readiness → Healthy 순서가 기록되는지
- VM1 실패가 VM2와 다음 서비스 CD를 차단하는지
- GitHub Actions와 Azure Pipelines의 승인·대기·실행 시간과 비용 표시가 어떻게 다른지
- 이전·새 Vue asset URL이 VM1/VM2 순차 전환 동안 모두 제공되는지

이 리포트는 권장 운영 모델이다. 실제 Application Gateway probe, Nginx 설정, VM Managed Identity, GitHub/Azure DevOps 권한, 배포·복구 검증 결과가 확보되기 전까지 실제 적용으로 간주하지 않는다.
