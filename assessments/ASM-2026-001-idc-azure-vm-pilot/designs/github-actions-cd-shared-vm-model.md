# GitHub Actions CD: 공유 이중화 VM·Application Gateway 배포 모델

- Assessment: ASM-2026-001
- Step / Phase: STEP-01 / DISCOVERY
- Status: PROPOSED — CD 도구, Azure 권한, Gateway probe, VM script, Blob container는 아직 만들거나 실행하지 않았다.
- 확인일: 2026-09-08

## 범위와 전제

이 초안은 여러 Java/Tomcat 서비스와 Vue 정적 파일이 **같은 두 Azure VM(VM1, VM2)** 에 있고, Azure Application Gateway가 두 VM을 backend로 사용한다는 전제의 CD만 다룬다. CI가 이미 Blob에 immutable release ID, checksum, manifest를 게시했다고 가정한다.

CD 조정 도구는 다음 두 가지를 함께 비교한다.

1. GitHub Actions만 사용: 중앙 deployment repository의 GitHub Actions가 배포 요청을 queue로 조정한다.
2. Azure Pipelines CD 사용: GitHub Actions는 CI만 수행하고 Azure Pipelines Environment approval/exclusive lock이 CD 요청 순서를 조정한다.

두 선택지는 traffic drain, VM 파일 적용, Tomcat 재기동, readiness, rollback을 자동으로 대신해 주지 않는다. 이 작업은 VM마다 같은 배포 절차로 남는다.

## 전체 흐름

```text
source repository CI
  └─ Blob: release ID + WAR/JAR + Vue files + manifest + checksums
                         │
                         ▼
              CD 조정기 중 하나 선택
     ┌───────────────────┴────────────────────┐
     │ A. GitHub Actions deployment repository │
     │ B. Azure Pipelines Environment           │
     └───────────────────┬────────────────────┘
                         │  shared-pair lock / queue
                         ▼
              VM1 배포 → 검증 → VM2 배포
                         │
        ┌────────────────┴────────────────┐
        │ Application Gateway custom probe │
        │ Nginx readiness marker           │
        │ Tomcat same-port restart         │
        │ Vue current 전환                 │
        └─────────────────────────────────┘
```

CD는 source repository의 Actions Artifact를 직접 받아 배포하지 않는다. Blob의 `release-id`와 manifest digest를 입력으로 고정한다. 이 방식이면 GitHub Actions CD와 Azure Pipelines CD 모두 같은 release를 소비하고, 재실행·수동 rollback도 같은 release ID로 수행할 수 있다.

## 공유 VM 쌍에서 지켜야 할 배포 불변조건

같은 VM 쌍에 여러 서비스가 있으므로, 서비스별 workflow가 서로 다른 VM을 동시에 drain하면 두 VM이 동시에 Application Gateway에서 제외될 수 있다. 이 경우 다른 서비스까지 포함해 사용자 요청을 처리할 backend가 없어질 수 있다.

다음 조건을 CD 조정 도구와 무관하게 지킨다.

| 조건 | 의미 |
|---|---|
| pair 단위 단일 배포 | 한 쌍에서는 서비스가 달라도 한 배포만 VM 변경·drain을 수행한다. |
| peer healthy 사전 확인 | VM1을 제외하기 전 VM2가 Gateway에서 Healthy이고, 해당 서비스의 현재 release가 준비 상태인지 확인한다. |
| VM1 성공 뒤 VM2 | VM1 drain·배포·readiness·안정 관찰이 성공한 뒤에만 VM2를 시작한다. |
| 실패 시 HOLD | probe/readiness/Run Command 결과가 실패 또는 Unknown이면 VM2와 다음 서비스 배포를 막고 상태를 기록한다. |
| 고정 release | 시작 전에 service, release ID, manifest digest, target pair, rollback 후보를 고정한다. `latest`는 입력으로 사용하지 않는다. |

`concurrency` 또는 Azure Pipelines exclusive lock은 요청 순서를 관리한다. VM 밖의 수동 작업, 이전 Run Command의 잔류 실행, lock 유실까지 막으려면 pair 상태·lease·fencing 정보를 별도 control store에 기록하고 배포 시작 시 확인해야 한다. runner/agent step 안에서 lock을 얻을 때까지 sleep/polling하지 않고, 획득 실패는 종료 또는 HOLD로 처리한다.

## CD 조정 방식 비교

| 항목 | A. GitHub Actions만 사용 | B. GitHub Actions CI + Azure Pipelines CD |
|---|---|---|
| CI | GitHub Actions | GitHub Actions |
| CD 실행 위치 | 중앙 deployment repository의 GitHub Actions | Azure DevOps Pipeline |
| 요청 순서 | GitHub Actions concurrency와 pair state/lock | Azure DevOps Environment approval + exclusive lock(`sequential`)과 pair state/lock |
| 승인 | GitHub Environment reviewer·branch/tag 규칙 | Azure DevOps Environment approval/check |
| Azure 인증 | GitHub OIDC → Entra federated credential → Azure RBAC | Azure DevOps workload identity federation service connection → Azure RBAC |
| 적합한 경우 | GitHub 안에서 CI·승인·CD를 일관되게 운영할 때 | Azure DevOps Environment 중심의 승인·배포 이력·대기열 운영이 필요한 때 |
| 여전히 필요한 것 | Gateway probe/drain, VM 순차 배포, Blob pull, HOLD/rollback | 동일 |

Azure Pipelines를 선택해도 GitHub OIDC 설정을 그대로 재사용하는 것이 아니다. Azure DevOps service connection의 workload identity federation, Environment 권한, Azure RBAC을 별도로 구성하고 검증한다.

### A. GitHub Actions만 사용하는 경우

중앙 deployment repository가 모든 source service의 CD 요청을 받는다. `pair-<environment>-<pair-id>` 형태의 concurrency group으로 queue를 만들고, protected Environment에서 승인을 받은 뒤 배포 job을 실행한다. 같은 repository 안의 concurrency만으로는 충분하지 않으므로 모든 shared-pair CD가 이 중앙 repository를 통과해야 한다.

장점은 GitHub Environment, OIDC, workflow 이력을 한 곳에서 관리한다는 점이다. 단점은 release 요청 전달 방식과 pair lock/control store의 운영 책임을 GitHub 측에서 직접 가져야 한다는 점이다.

### B. Azure Pipelines를 CD에 사용하는 경우

GitHub Actions CI가 Blob release를 만든 뒤 Azure Pipelines가 release ID를 event 또는 수동 입력으로 받는다. Azure DevOps Environment의 approval과 exclusive lock을 `sequential`로 설정해 shared-pair CD 요청을 순차 처리한다.

장점은 Azure DevOps Environment의 승인·deployment history·exclusive lock을 CD 운영에 사용할 수 있다는 점이다. 단점은 GitHub와 Azure DevOps의 권한, service connection, 비용·사용량, 운영 화면을 함께 관리해야 한다는 점이다. exclusive lock은 VM traffic을 drain하지 않으므로, 아래 VM 절차를 생략할 수 없다.

## VM별 무중단 배포 절차

Application Gateway custom probe는 Nginx의 `/_deploy/ready` 같은 endpoint를 사용한다. 이 endpoint는 Nginx marker, Tomcat application readiness, 기대 release ID가 모두 맞을 때만 200을 반환한다.

```text
1. pair lock 획득, 대상 release/manifest와 peer VM Healthy 확인
2. VM1의 Nginx deploy marker off
3. Application Gateway Backend Health에서 VM1 Unhealthy 확인
4. 기존 요청 종료/connection draining 기준을 확인
5. VM1이 Blob에서 고정 release ID와 checksum을 내려받음
6. Tomcat을 중지하고 WAR/JAR·설정을 교체한 뒤 같은 port로 기동
7. Vue 파일을 준비하고, frontend 정책에 따라 current HTML을 전환
8. application readiness, release ID, smoke test 성공 확인
9. marker on → Gateway가 VM1 Healthy로 판정 → 안정 관찰
10. 성공할 때만 VM2에 2~9를 반복
```

probe를 **수동으로 제외**한다는 말은 배포 script가 Nginx marker를 off로 전환해 의도적으로 readiness endpoint를 실패시키는 것을 뜻한다. probe의 **자동 감지**는 Application Gateway가 그 endpoint를 주기적으로 확인하여 VM을 Healthy/Unhealthy로 분류하는 것을 뜻한다. marker만 내렸다고 기존 연결이 즉시 끝난다고 가정하지 말고, connection draining과 실제 요청·WebSocket·SSE 종료 조건을 별도로 확인해야 한다.

VM1을 제외하기 전에 VM2가 Healthy인지 확인하는 것은 필수다. VM2가 Unhealthy, 용량 부족, maintenance 중, 이전 배포의 상태 Unknown이면 VM1 marker를 내리지 않고 HOLD한다. 한 VM을 배포하는 동안 남은 VM이 전체 요청을 감당할 수 있는지와 drain deadline은 Pilot에서 수치로 검증한다.

## Blob download와 VM 전송 방식

두 방식 모두 Blob에서 release를 받는다는 공통 전제를 둔다.

| 선택 | 명령 전달 | Blob download 주체 | 장점 | 주의점 |
|---|---|---|---|---|
| A. 직접 push | GitHub Actions 또는 Azure Pipeline agent가 SSH/배포 endpoint로 VM을 제어 | runner/agent가 download한 뒤 VM에 전송하거나 VM에서 script 실행 | 흐름을 pipeline에서 직접 제어 | VM inbound 경로, SSH/endpoint 인증서·키, private runner network가 필요. GitHub OIDC는 SSH 인증을 대체하지 않음 |
| B. 명령 push + 파일 pull | CD job이 Azure Managed Run Command로 VM script만 시작 | VM Managed Identity가 Blob에서 release ID를 download | VM inbound 배포 port를 열지 않고, CI/deployer identity와 VM runtime identity를 분리 | VM Agent, Blob/Key Vault outbound 경로, Run Command timeout·잔류 실행을 운영해야 함 |

현재 파일럿의 GitHub-hosted runner, passwordless identity, Blob release 전제를 따르면 **B. 명령 push + 파일 pull**이 권장안이다. GitHub Actions CD와 Azure Pipelines CD 모두 Azure control plane에서 Run Command를 호출하고, VM Managed Identity가 Blob에서 파일을 받는 방식으로 동일하게 적용할 수 있다. 직접 push는 조직에 이미 검증된 private runner network와 SSH certificate/broker 또는 deployment endpoint가 있을 때만 선택한다.

## Java/Tomcat 및 Nginx 처리

Tomcat은 별도 port나 병렬 JVM 없이 기존 JVM을 같은 port에서 교체한다. 배포 script는 marker를 내린 뒤 Tomcat을 멈추고, Blob에서 검증한 WAR/JAR·필요 설정을 staged directory에 준비한 후 교체한다. 기동 뒤에는 단순 process 존재 여부가 아니라 application readiness, 의존성 연결, expected release ID, 최소 smoke test를 확인한다.

Nginx는 다음 세 가지 역할을 가진다.

1. Application Gateway probe가 호출할 deploy readiness endpoint 제공
2. Vue HTML과 versioned static asset 제공
3. marker on/off에 따라 신규 요청 수신 가능 여부를 노출

Nginx reload 또는 Tomcat restart 실패 시 marker는 off로 유지한다. CD job은 VM2로 진행하지 않고, 동일 release의 rollback 또는 운영자 HOLD 판단에 필요한 command ID, manifest digest, Gateway health를 남긴다.

## Vue 정적 파일: 이전 버전 호출을 보호하는 선택

새 HTML을 받은 브라우저 탭은 이전 HTML이 참조하던 JS/CSS chunk를 잠시 더 요청할 수 있다. VM1과 VM2가 순차 전환되는 동안에도 이 요청은 발생한다.

| 선택 | 처리 | 장점 | 영향 |
|---|---|---|---|
| A. 단일 active slot | `current` HTML과 asset만 교체하고 이전 파일을 제거 | 배포와 cleanup이 단순 | 오래 열린 탭의 이전 JS/CSS 요청이 404가 되어 새로고침 또는 사용자 영향이 발생할 수 있음 |
| B. hash/version asset 2버전 공존 | HTML은 새 release로 전환하되 이전·새 asset URL을 Nginx에서 함께 제공 | 이전 HTML의 asset 요청을 보호하고 rollback이 단순 | asset 보존량, cleanup 규칙, cache header와 release별 URL 구성이 필요 |

권장안은 **B**다. 모든 VM에 새 release의 versioned asset을 먼저 준비하고 이전·새 asset URL이 모두 200인지 확인한 다음, VM1과 VM2의 `current` HTML을 순차 전환한다. 예를 들면 `/assets/r101/app.a1b2.js`와 `/assets/r102/app.c3d4.js`를 동시에 제공하고, 새 HTML만 r102 path를 참조하게 한다. 공용 `/assets/app.js` 경로를 새 파일로 덮어쓰면 이전 HTML도 새 파일을 요청하므로 2버전 공존 목적을 달성하지 못한다.

이전 asset은 최소 `previous`, in-flight, rollback-pinned release 집합에서 보호한다. 보존 시간은 사용자 세션·browser cache·지원 정책을 확인해 정하며, cleanup은 두 VM의 current/previous 상태와 active 배포를 읽은 뒤 dry-run으로 수행한다. 아직 정해지지 않은 frontend 동작을 이 초안에 추가 가정으로 넣지 않는다.

## 실패·복구 기준

| 상황 | CD 동작 |
|---|---|
| peer VM Unhealthy 또는 용량 부족 | 배포 시작 전 HOLD. 대상 VM marker를 내리지 않음 |
| Gateway가 marker off 뒤에도 상태를 갱신하지 않음 | probe 설정·backend health를 확인하고 HOLD |
| Blob digest/checksum 불일치 | Tomcat/Vue 전환 전에 중단, VM2 미실행 |
| Tomcat readiness 또는 smoke test 실패 | marker off 유지, VM2와 다음 서비스 CD 차단, rollback 또는 HOLD |
| VM1 성공 뒤 VM2 실패 | VM1을 즉시 되돌린다고 가정하지 않음. 이미 정상 release인 VM1과 VM2의 상태·영향을 기록하고 정해진 rollback 절차를 실행 |
| pipeline/job 취소 또는 timeout | Run Command와 VM 실제 상태를 조회해 상태 Unknown을 해소하기 전 다음 배포 차단 |

## Pilot에서 확인할 항목

1. 서로 다른 서비스의 동시 CD 요청이 shared-pair queue/lock에서 하나씩 실행되는지
2. peer VM이 Healthy가 아닐 때 대상 VM drain이 시작되지 않는지
3. marker off → Gateway Unhealthy → 기존 요청 처리 → Tomcat 교체 → readiness → Healthy 순서가 기록되는지
4. VM1의 실패가 VM2와 다음 서비스 배포를 차단하는지
5. GitHub Actions와 Azure Pipelines 각각에서 승인·대기·실행 시간 및 비용 표시가 어떻게 다른지
6. 직접 push와 명령 push + Blob pull에서 network·identity·실패 복구 조건이 충족되는지
7. 이전·새 Vue asset URL이 VM1/VM2 모두에서 순차 전환 중 200으로 제공되는지
8. 허용되지 않은 source repository, branch, Environment 또는 Azure DevOps service connection이 Azure 작업을 수행하지 못하는지

## 근거

- [Application Gateway health probes](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-probe-overview)
- [Application Gateway backend HTTP settings and connection draining](https://learn.microsoft.com/en-us/azure/application-gateway/configuration-http-settings)
- [Application Gateway backend health](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-backend-health)
- [GitHub Actions concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency)
- [Azure Pipelines approvals and checks](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops)
- [Azure Pipelines environments](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/environments?view=azure-devops)
- [Azure Pipelines workload identity federation](https://learn.microsoft.com/en-us/azure/devops/pipelines/release/configure-workload-identity?view=azure-devops)
- [Azure Managed Run Command](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command-managed)

공식 문서는 기능과 설계 근거를 설명한다. 실제 Azure tenant, VM, Application Gateway, probe, Blob network, GitHub/Azure DevOps 조직과 권한은 이 Assessment에서 아직 확인하거나 변경하지 않았다.
