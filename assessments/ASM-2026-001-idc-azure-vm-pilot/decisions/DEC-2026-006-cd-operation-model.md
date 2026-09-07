# DEC-2026-006 — 공유 VM 쌍의 CD 운영 모델

## Status

PROPOSED

## Traceability

- Assessment: ASM-2026-001
- Applicable Environment IDs: ENV-02/03
- Applicable Workload / Integration IDs: WL-01/02/03, INT-04/05/06/07
- Step: STEP-01 / DISCOVERY; STEP-05 준비 조사
- Approval Level: USER_DECISION
- Created / Updated: 2026-09-07
- Supersedes: 없음

## Context

여러 service repository가 동일 Azure VM 쌍을 공유한다. 배포 순서를 중앙 workflow가 자동 조정할지, 지정 운영자가 각 repository의 수동 deploy workflow를 조정할지 선택해야 한다.

## Requirement / Constraints

- REQ-08, REQ-09, REQ-11, REQ-13 및 [CD 운영 모델 선택안](../designs/cd-operation-model-options.md)을 따른다.
- GitHub concurrency는 repository 경계를 넘는 잠금이 아니다.
- Azure 배포는 개인 계정이 아닌 OIDC workload identity와 최소 RBAC로 실행한다.

## Options

### Option A — 중앙 deployment repository 자동 조정

**Advantages**

모든 shared pair deploy를 한 repository의 queue·Environment·OIDC trust·감사 기록으로 통합한다. 운영자는 승인과 예외 판단에 집중한다.

**Disadvantages**

release handoff·중앙 repository 책임·workflow 변경 통제를 설계해야 한다.

**Cost / Operational / Security / Global Impact**

중앙 queue pending과 pair lock을 운영한다. prod Azure identity를 하나의 신뢰된 repository로 제한한다. pair/region이 늘어도 group·identity·state를 확장할 수 있다.

### Option B — 각 service repository에서 수동 조정

**Advantages**

서비스팀이 자신의 deploy workflow와 시점을 직접 통제한다.

**Disadvantages**

사람이 repository 간 순서를 조정해야 하며, OIDC trust·Environment·RBAC·감사 기록이 분산된다. shared pair lock 없이는 동시 drain을 막지 못한다.

**Cost / Operational / Security / Global Impact**

대기·변경 조정의 인력 비용과 운영 오류 가능성이 늘어난다. repository별 prod identity가 늘어 권한 검토 범위가 커진다. 서비스·region 증가 시 수동 조정 부담이 비례해 증가한다.

## Agent Recommendation

Option A를 권장한다. Option B는 배포 빈도와 서비스 수가 작고, 운영자가 공통 pair lock·상태 확인을 실행할 책임이 명확할 때 선택한다.

## User Decision

- Selected: 미결정 — Option A 또는 B 선택 필요
- Decision Date / Owner / Reason / Approval Evidence: 없음

## Implemented Reality

- Actual: 미적용. 중앙 repository, repository별 deploy workflow, queue, lock, OIDC/RBAC를 만들지 않았다.
- Applied Date: 없음

## Risks Accepted

없음. 사람 수동 조정 또는 자동 queue의 실제 운영 적합성은 검증하지 않았다.

## Mitigations

- pair lock, HOLD, peer health, Run Command 잔존 확인, 승인된 release ID/digest 고정을 두 모델에 공통 적용한다.
- dev에서 교차 repository 동시 요청·수동 재배포·취소·lock recovery를 시험한다.

## Revisit Trigger

- 서비스 또는 shared VM pair 증가
- 배포 빈도 증가·긴급 배포 요구
- 감사·분리 의무 강화
- queue/lock 장애 또는 동시 배포 사고

## Evidence

- [CD 운영 모델 선택안](../designs/cd-operation-model-options.md)
- [CD 순차 제어·Azure OIDC 준비](../designs/cd-sequencing-and-azure-oidc-preparation.md)
- [파일럿 검증 계획](../evidence/pilot-validation-plan.md)
## Addendum — Azure Pipelines CD 조정기 후보

GitHub Actions CI를 유지하고 Azure Pipelines를 CD 실행·조정기로만 사용하는 세 번째 선택지를 추가 검토한다. Azure DevOps Environment approval과 exclusive lock의 `sequential` 동작은 같은 Environment를 사용하는 Azure Pipelines CD 요청을 한 건씩 진행하게 할 수 있다.

이 기능은 GitHub Actions 또는 수동 Azure 작업까지 포함한 VM pair의 실제 side effect를 완전히 잠그지는 않으며, Application Gateway의 신규 요청 제외·connection draining·Tomcat readiness를 수행하지 않는다. 따라서 pair 상태 확인, Nginx/Application Gateway drain, VM 1→검증→VM 2, HOLD/rollback은 모든 선택지에 공통으로 남는다.

Azure Pipelines CD는 중앙 GitHub deployment repository와 각 service repository 수동 조정에 더하는 선택지일 뿐 아직 선택되지 않았다. GitHub Actions CD와 Azure Pipelines CD를 같은 pair에 병행 허용하지 않으며, 선택 시 Azure DevOps workload identity federation과 Environment/service connection 권한을 별도로 결정·검증한다.

상세 비교: [Azure Pipelines CD 조정기 선택지](../designs/azure-pipelines-cd-option.md).
