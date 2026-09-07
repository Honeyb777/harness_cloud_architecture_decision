# CD 운영 모델 선택 — 중앙 자동 조정 또는 개별 수동 조정

- Assessment: ASM-2026-001
- Step / Phase: STEP-01 / DISCOVERY; STEP-05 준비 조사
- Status: PROPOSED — 선택·workflow·권한·lock 설정은 미수행
- 확인일: 2026-09-07

## 선택의 기준

여러 service repository가 같은 Azure VM 쌍을 공유한다. 이때 “누가 배포 순서를 조정하는가”를 선택한다. GitHub Actions concurrency는 repository 안에서만 동작하므로, 개별 repository에 같은 concurrency group 이름을 써도 서로를 기다리지 않는다.

| 선택지 | 실제 CD 실행 위치 | 순서 조정 주체 | GitHub concurrency 역할 | 공통 pair lock 필요 |
|---|---|---|---|---|
| A. 중앙 자동 조정 | 중앙 deployment repository | 중앙 workflow·queue | pair별 queue로 자동 직렬화 | 필요 |
| B. 개별 수동 조정 | 각 service repository | 지정 운영자 | 해당 repository의 중복 실행 방지 | 필요 |

## Option A — 중앙 deployment repository가 자동으로 조정

```text
source repository CI/publish
  → release ID·digest·pair ID 전달
  → 중앙 deployment repository queue
  → pair lock·VM1·검증·VM2·기록
```

| 항목 | 내용 |
|---|---|
| 사람의 역할 | prod Environment 승인, release·영향·rollback 후보 검토. 실행 순서를 매번 직접 조정하지 않음 |
| 순차 처리 | 중앙 repository의 `concurrency: group: deploy-<env>-pair-<id>, queue: max`로 요청을 대기시킴 |
| Azure identity | 중앙 deployment repository의 dev/prod Environment별 deployer identity만 Azure Run Command·Gateway·control storage 권한을 가짐 |
| 수동 재배포 | 중앙 repository의 `workflow_dispatch`로 기존 immutable release ID와 pair ID를 선택 |
| 장점 | 모든 서비스의 shared pair 변경·승인·감사·OIDC trust를 한곳에 모음. 사람의 배포 순서 조정 부담이 작음 |
| 단점 | release 요청 전달 방식과 중앙 repository 운영 책임을 추가로 정해야 함 |

## Option B — 각 service repository에서 사람이 순서를 조정

```text
운영자
  → service repository의 workflow_dispatch
  → 공통 pair lock 획득 확인
  → VM1·검증·VM2·기록
```

| 항목 | 내용 |
|---|---|
| 사람의 역할 | 어떤 service repository의 배포를 언제 시작할지 직접 결정하고, 이전 배포의 종료·HOLD·pair 상태를 확인 |
| 순차 처리 | repository별 concurrency는 같은 service의 중복만 방지. 서로 다른 service repository의 경합은 사람이 조정하고 공통 pair lock이 최종 차단 |
| Azure identity | 각 service repository의 dev/prod Environment와 해당 repository만 신뢰하는 deployer identity/RBAC를 별도로 구성 |
| 수동 재배포 | 해당 service repository의 `workflow_dispatch`로 기존 immutable release ID를 선택 |
| 장점 | 서비스팀별 배포 자율성과 단순한 workflow 소유권 |
| 단점 | 사람의 조정 누락·권한 분산·감사 경로 분산 위험. 서비스 수만큼 OIDC trust·Environment·RBAC·runbook이 늘어남 |

개별 수동 조정은 승인자가 Azure portal에서 직접 VM을 변경한다는 뜻이 아니다. 사람은 GitHub Environment 승인과 workflow 시작을 담당하고, Azure 실행은 여전히 repository별 OIDC workload identity와 최소 RBAC를 사용한다.

## 두 모델 공통의 안전 조건

- 같은 pair에 대한 다른 배포·VM patch·Gateway 변경이 진행 중이면 다음 VM drain을 시작하지 않는다.
- 공통 pair lock은 GitHub concurrency와 별도다. lock 획득 실패 시 runner 안에서 무한 대기하지 말고 종료/HOLD하고 운영자가 재조정한다.
- approved release ID·digest·대상 pair·rollback 후보를 시작 전에 고정한다.
- VM1 실패, peer health unknown, Run Command 잔존, lock 상태 unknown이면 VM2와 다음 service 배포를 시작하지 않는다.
- 어떤 모델이든 실제 Azure RBAC는 deployer workload identity에만 부여하며, 개인 Azure account나 GitHub Secrets의 Azure client secret을 사용하지 않는다.

## 권장안과 사용자 선택

공유 VM 쌍의 서비스가 여러 개이므로 파일럿 권장안은 **Option A**다. 자동 queue가 순서를 조정하고 사람은 승인·예외 판단에 집중할 수 있다.

Option B는 배포 빈도가 낮고, 지정 운영자가 변경 순서를 직접 책임지며, 공통 pair lock과 상태 확인을 실제로 운영할 수 있을 때 선택할 수 있다. 이 경우에도 GitHub concurrency만으로 교차 repository 배포를 막는다고 표시하지 않는다.

선택은 [DEC-2026-006](../decisions/DEC-2026-006-cd-operation-model.md)에 기록한다.
