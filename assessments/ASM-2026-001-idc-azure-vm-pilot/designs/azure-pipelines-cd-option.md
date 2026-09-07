# Azure Pipelines를 CD 조정기로 사용하는 선택지

- Assessment: ASM-2026-001
- Step / Phase: STEP-01 / DISCOVERY; STEP-05 준비 조사
- Status: PROPOSED — GitHub Actions CI는 유지하고 Azure Pipelines를 CD에만 추가하는 후보. Azure DevOps 조직, Pipeline, Environment, service connection, 실제 배포는 만들지 않았다.
- 확인일: 2026-09-07

## 질문과 결론

GitHub Enterprise Cloud/Copilot/GitHub Actions를 사용하면서 **Azure Pipelines만 CD 조정기로 추가**할 수 있다. 소스 repository와 CI는 GitHub에 두고, CI가 만든 immutable release ID와 digest를 Azure Pipelines에 전달하거나 수동으로 선택해 배포한다.

Azure Pipelines의 Environment approval 및 exclusive lock은 배포 승인과 Azure Pipelines를 통한 배포 요청의 순서를 조정한다. 그러나 Application Gateway가 backend VM으로 새 요청을 보내지 않게 하거나 이미 연결된 요청을 종료시키지는 않는다. 따라서 Nginx readiness marker, Application Gateway probe/connection draining, VM 1→검증→VM 2 순차 적용, 실패 시 HOLD/rollback은 이 선택지에서도 필수다.

## 권장 흐름 후보

```text
GitHub source repository
  → GitHub Actions CI: build/test/release manifest 생성
  → Blob: immutable release 파일과 release ID/digest 보관
  → Azure Pipelines CD: GitHub release event 또는 수동 release ID 입력
  → Azure DevOps Environment: production-pair approval / exclusive lock(sequential)
  → Azure control plane: VM 1 Run Command, Gateway/probe 상태 확인
  → VM 1: Blob에서 정해진 release pull → drain → Tomcat 교체 → readiness
  → VM 2: 같은 release로 반복
  → Azure Pipelines: 결과·HOLD·rollback 이력 기록
```

CI와 CD가 서로 다른 실행기여도 배포 파일은 Artifacts의 run별 임시 전달이 아니라 Blob의 immutable release ID/digest를 기준으로 선택한다. Azure Pipelines가 GitHub repository에서 직접 CI를 다시 실행하거나 `latest` 파일을 배포 기준으로 삼지 않는다.

## 무엇이 바뀌고 무엇이 남는가

| 항목 | Azure Pipelines가 제공하는 것 | 별도로 유지할 것 |
|---|---|---|
| 여러 배포 요청의 순서 | 같은 Azure DevOps Environment에 exclusive lock을 설정하고 `sequential`로 대기열 처리 | Azure Pipelines 밖에서 실행된 수동 작업, Run Command의 잔류 실행, VM 상태 불일치를 막기 위한 pair 상태 확인과 HOLD 규칙 |
| 배포 승인 | Environment approval, branch control, artifact 평가 등 stage 시작 전 check | 승인 이후 실제 VM 상태·release digest·peer health 확인 |
| 배포 실행 | deployment job의 순차 step, Azure service connection을 통한 Azure API 호출 | VM Agent/egress, Blob 파일 검증, Run Command timeout·취소·복구 처리 |
| 트래픽 전환 | 없음 | Nginx marker, Application Gateway probe, connection draining, 기존 요청·WebSocket 처리 |
| 배포 이력 | Azure DevOps Environment와 Pipeline run 이력 | release manifest, Azure operation/command ID, VM별 실제 상태 및 rollback 증적 |

Application Gateway connection draining은 backend가 제거되거나 backend 상태가 바뀌는 동안 새 요청을 제외하고 기존 요청을 설정된 시간 동안 완료하도록 돕는 기능이다. Pipeline의 exclusive lock은 실행 순서 잠금이므로 connection draining과 같은 기능으로 간주하지 않는다.

## 기존 CD 운영 모델에 추가되는 선택지

| 선택지 | CD 실행 위치 | 순서·승인 방식 | 적합한 경우 |
|---|---|---|---|
| 중앙 GitHub deployment repository | GitHub Actions | GitHub Environment와 중앙 queue/pair lock | CI/CD를 GitHub에서 일관되게 운영하고자 할 때 |
| 각 서비스 repository 수동 배포 | 각 GitHub Actions workflow | 지정 운영자와 공통 pair lock | 배포 빈도가 낮고 사람이 순서를 책임질 때 |
| **Azure Pipelines CD 조정기** | Azure DevOps Pipeline | Azure DevOps Environment approval + exclusive lock + pair 상태 확인 | Azure Pipelines의 월 무료 구간, Environment 기반 승인·배포 이력 또는 Azure DevOps 운영 경험을 활용할 때 |

세 번째 선택지는 GitHub Actions를 제거하는 선택지가 아니다. GitHub Actions CI + Azure Pipelines CD의 역할 분리 후보이며, GitHub Actions CD와 Azure Pipelines CD를 같은 VM pair에 동시에 허용하는 구성은 피한다.

## 비용과 대기 시간

Azure DevOps 조직에 유효한 Azure 구독을 연결하고 Microsoft-hosted 무료 병렬 작업을 사용하면 private project 기준 한 개 동시 job과 월 1,800분의 무료 구간이 있다. 무료 구간을 넘기거나 병렬 job을 추가하면 Azure DevOps에 연결한 Azure 구독으로 비용이 청구된다.

Environment approval/check는 stage 시작 전에 Pipeline을 pause할 수 있고, `ManualValidation` task는 agentless job에서 실행된다. 다만 exclusive lock 대기, 승인 대기, 실제 drain·Run Command polling이 무료 1,800분과 병렬 job 사용량에 어떤 방식으로 표시되는지는 실제 조직의 Pipeline run과 billing에서 확인해야 한다. runner 또는 agent step 안에서 polling/sleep으로 잠금을 기다리는 방식은 사용하지 않는다.

## Identity와 권한 경계

이 선택지는 Azure DevOps Environment와 Azure service connection을 추가한다. Azure 접근에는 Azure DevOps workload identity federation service connection을 사용하고, GitHub Actions OIDC 설정을 Azure Pipelines가 그대로 재사용한다고 가정하지 않는다. service connection의 scope, federated credential subject, Azure RBAC은 GitHub/Azure 권한 거버넌스 결정과 함께 별도로 정한다.

## Pilot 검증 조건

1. 같은 Azure DevOps Environment를 사용하는 서로 다른 GitHub source service의 CD 요청이 `sequential` lock으로 한 건씩 처리되는지 확인한다.
2. approval 거절·timeout·lock 대기·Pipeline 취소 때 VM 1 drain 또는 VM 2 배포가 시작되지 않는지 확인한다.
3. VM 1의 drain·Tomcat 교체·readiness 실패가 VM 2 실행을 차단하고 HOLD/rollback 증적을 남기는지 확인한다.
4. Approval/lock 대기와 실제 deployment job의 실행 시간·free minutes·비용 표시를 분리해 기록한다.
5. Azure Pipelines 외부의 수동 Run Command/Gateway 변경을 포함한 pair 상태 불일치가 다음 배포를 차단하는지 확인한다.

## 근거

- [Azure Pipelines approvals and checks](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops): Environment approval, protected resource checks, exclusive lock 및 `sequential` 동작.
- [Azure Pipelines environments](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/environments?view=azure-devops): deployment target·approval·deployment history.
- [Azure Pipelines parallel jobs](https://learn.microsoft.com/en-us/azure/devops/pipelines/licensing/concurrent-jobs?view=azure-devops): free tier, Azure subscription billing, Microsoft-hosted/self-hosted capacity.
- [Azure Pipelines GitHub repositories](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/github?view=azure-devops): GitHub repository를 source로 사용하는 trigger.
- [Application Gateway features](https://learn.microsoft.com/en-us/azure/application-gateway/features): connection draining의 backend 트래픽 처리 역할.
