# DEC-2026-002 — 운영 배포 신뢰와 실행 경계

## Status

PROPOSED

## Traceability

- Assessment: ASM-2026-001
- Applicable Environments: ENV-02/03; prod는 논리적 환경 예시
- Applicable Workloads: WL-01/02/03
- Step: STEP-03/05 준비 조사; 현재 STEP-01 선행 확인 필요
- Approval Level: USER_DECISION
- Date: 2026-09-07

## Context / Requirements / Constraints

여러 서비스가 같은 두 VM을 공유한다. 서비스별 JVM은 분리되어 있으나 배포가 서로 다른 VM을 동시에 제외하면 공유 장애를 일으킬 수 있다.
GitHub-hosted runner와 단기 OIDC 인증을 지향하며 GitHub repo 구성은 미확인이다.

## Options

| 대안 | 장점 | 위험·운영 부담 | 비용 영향 | 보안/감사 | 글로벌 영향 |
|---|---|---|---|---|---|
| A. 중앙 배포 repo에서 실제 run 실행 + 환경별 OIDC Identity + pair 잠금/영속 상태 | prod 권한과 배포 규칙 집중, GitHub repo 범위 concurrency 활용 | build repo→중앙 repo 요청 경로와 운영 책임 필요, 중앙 workflow 변경 통제 중요 | 추가 storage 상태/조정 및 runner 시간; 수치 미산정 | 승인·release·실행 증거 일원화, publisher/deployer 분리 | 환경·리전·서버 쌍별 Identity/잠금 확장 |
| B. 각 서비스 repo에서 배포 + 분리 Identity + 전역 coordinator/lock | 서비스팀별 배포 자율성 | 모든 경로의 일관된 lock·실패 복구·fencing 구현 및 권한 분산 | coordinator/lock 운영과 검증 비용 증가 가능 | repo별 federated trust·승인·감사 유지 | 리전별 lock/조정 서비스 가용성 검토 |

중앙 reusable workflow 호출만으로 A가 구현되지는 않는다. 중앙 repo에서 workflow run을 시작해야 한다. source repo의 GITHUB_TOKEN을 교차 repo 만능 token으로 쓰지 않는다.

## Agent Recommendation

파일럿은 A를 권장한다. prod deployer를 신뢰된 중앙 repo의 보호 Environment로 좁히고, 모든 공유 VM 변경에 pair 잠금/영속 상태를 적용한다.
runner는 standard+Run Command와 VNet larger runner를 네트워크 요구에 따라 선택한다. Blob은 배포 원본으로 우선 추천하되 별도 확정한다.

## User Decision

- Selected: 미결정
- Decision Date / Owner / Approval Evidence: 없음
- 다음 확인: org/repo·환경·Azure scope와 중앙 배포 repo 운영 책임을 확인한 후 A/B를 결정한다.

## Implemented Reality

미적용. 실제 Identity·Federated Credential·RBAC·workflow·lock을 생성하지 않았다.

## Risks / Mitigations

Run Command는 강한 VM 실행 권한이다. 전역 lock은 코드·권한·수동 운영 경로가 함께 지켜야 하며 Blob lease가 VM 동작을 자동 차단하지 않는다.
취소·stale execution·lease 복구·health Unknown 시험을 필수로 둔다.

## Revisit Trigger / Evidence

저장소 운영 책임 분리, 서비스 수 증가, 리전/서버 쌍 증가, 더 강한 격리 요구 발생 시 재검토한다.
근거: [권한 대안](../designs/identity-and-delivery-options.md), [배포 안전성](../designs/availability-and-rollout-options.md), [공식 출처](../evidence/official-sources.md).
