# DEC-2026-007 — Azure VM 배포 전송 모델

## Status

PROPOSED

## Traceability

- Assessment: ASM-2026-001
- Applicable Environment IDs: ENV-02/03
- Applicable Workload / Integration IDs: WL-01/02/03, INT-04/05/06/07
- Step: STEP-01 / DISCOVERY; STEP-04/05 준비 조사
- Approval Level: USER_DECISION
- Created / Updated: 2026-09-07
- Supersedes: 없음

## Context / Requirement / Constraints

GitHub-hosted runner가 Azure VM에 배포할 때, 직접 SSH 등의 push 방식과 Azure Run Command·VM Managed Identity를 결합한 명령 push·파일 pull 방식이 후보이다. REQ-03, REQ-04, REQ-08~14를 따른다. GitHub OIDC는 Azure API 인증이며 SSH login 인증을 제공하지 않는다.

## Options

### Option A — SSH 등 직접 push

**Advantages**

runner가 파일 전송과 명령 실행을 직접 통제한다. Azure VM Agent 없이도 구현할 수 있다.

**Disadvantages**

runner→VM 관리 network와 SSH/endpoint credential·host key·접근 회수를 별도 설계해야 한다. 장기 SSH private key는 현재 passwordless 방향과 맞지 않는다.

**Cost / Operational / Security / Global Impact**

VNet larger runner 또는 secure ingress, SSH certificate/broker·network 운영 비용이 생길 수 있다. runner가 VM 변경 권한을 직접 갖고, region·pair 증가마다 network/credential 경계를 확장한다.

### Option B — Managed Run Command 명령 push + VM MI 파일 pull

**Advantages**

runner의 VM SSH ingress가 필요 없고, GitHub OIDC·Azure RBAC와 VM MI를 분리할 수 있다. Blob canonical release를 VM이 직접 읽는다.

**Disadvantages**

VM Agent·Run Command 상태/timeout·egress에 의존하며, deployer Run Command 권한은 강한 VM code execution 권한이다.

**Cost / Operational / Security / Global Impact**

Azure control plane/Agent·Blob request/network와 상태 polling을 운영한다. prod deployer trust는 중앙 repository·Environment로 좁히고 VM MI는 runtime/read 권한만 부여한다. region·pair 증가는 동일 패턴의 RBAC·network·state 확장으로 관리한다.

## Agent Recommendation

Option B를 권장한다. Option A는 private runner network와 short-lived SSH credential broker를 이미 조직 표준으로 운영하고, Azure Agent/Blob pull 제약이 있을 때 선택한다.

## User Decision

- Selected: 미결정 — Option A 또는 B 선택 필요
- Decision Date / Owner / Reason / Approval Evidence: 없음

## Implemented Reality

- Actual: 미적용. SSH endpoint/credential, Managed Run Command, VM MI, Blob RBAC를 만들지 않았다.
- Applied Date: 없음

## Risks Accepted

없음. VM Agent/egress·SSH network·remote command cancellation·drain/rollback은 실제 검증 전 수용하지 않는다.

## Mitigations

- pair lock, VM1/VM2 순차 배포, Run Command/SSH 원격 실행 상태 확인, HOLD·rollback 절차를 적용한다.
- dev에서 OIDC/RBAC 또는 SSH credential 거부, artifact digest, network 단절, 취소·timeout을 시험한다.

## Revisit Trigger

- VNet/private endpoint 또는 SSH certificate 표준 도입
- VM Agent/Blob network 제약
- self-hosted/larger runner 비용 또는 운영 정책 변경
- 원격 실행·rollback 장애

## Evidence

- [전송 모델 비교](../designs/server-deployment-transport-options.md)
- [검증 계획](../evidence/pilot-validation-plan.md)
