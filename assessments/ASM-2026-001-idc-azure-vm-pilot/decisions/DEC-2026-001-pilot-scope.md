# DEC-2026-001 — 사용자 지정 Azure VM 파일럿 범위

## Status

DECIDED

## Traceability

- Assessment: ASM-2026-001
- Applicable Environments: ENV-01/02/03
- Applicable Workloads: WL-01/02/03
- Step: STEP-01
- Approval Level: USER_DECISION
- Date: 2026-09-07

## Context / Requirement / Constraints

IDC 일부 서비스의 Azure 이전을 검토한다. GitHub Enterprise Cloud, Actions, Copilot, Azure VM을 사용하려는 의향이 명시되었고 서비스별 Tomcat/JVM이 확인되었다.

## Options

사용자가 이미 대상 플랫폼과 실행 방식을 지정한 접수 결정이다. Cloud provider/Kubernetes/Enterprise Server를 다시 선택하도록 요구하지 않는다. 이후 runner·OIDC·배포·저장소 대안은 별도 비교한다.

## Agent Recommendation

지정한 범위 안에서 단기 인증·VM 순차 배포·안전한 버전 승격을 검토한다.

## User Decision

- Selected: Azure VM 파일럿, GitHub Enterprise Cloud 및 Copilot/Actions 사용 예정, self-hosted runner 가능하면 미사용.
- Decision Date: 2026-09-07
- Decision Owner: 사용자
- Approval Evidence: 최초 환경 설명, 'GitHub Enterprise Cloud 사용 예정', '서비스마다 별도 Tomcat/JVM' 후속 답변.
- Reason: 사용자 지정 전제; 플랫폼 선택 사유의 상세 비교는 제공되지 않음.

## Implemented Reality

- Actual: 사용자 설명상 일부 Azure 이전 진행 및 Application Gateway 사용. 직접 조회/검증 없음.
- GitHub OIDC·RBAC·Actions 배포 구성의 실제 적용: 미확인.
- Applied Date: 미확인.
- Difference from Recommendation / User Decision: 아직 비교 가능한 적용 증적 없음.

## Risks / Mitigations / Revisit

네트워크·세션·용량·승인 범위 미확인. [Context](../context.md)와 [검증 계획](../evidence/pilot-validation-plan.md)에 따라 확인한다.
GitHub 제품 형태·VM 배치·대상 서비스 범위가 바뀌면 재검토한다.
