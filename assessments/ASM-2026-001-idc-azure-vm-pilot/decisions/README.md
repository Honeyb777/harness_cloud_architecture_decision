# ASM-2026-001 결정 목록

| ID | Step | 상태 | 내용 | 원본 |
|---|---|---|---|---|
| DEC-2026-001 | STEP-01 | DECIDED | 사용자가 지정한 파일럿 범위와 실행 전제 | [범위](DEC-2026-001-pilot-scope.md) |
| DEC-2026-002 | STEP-03/05 준비 조사 | PROPOSED | 운영 배포 Identity와 저장소 실행 경계 | [대안](DEC-2026-002-deployment-trust-boundary.md) |
| DEC-2026-004 | STEP-01 | PROPOSED | CI build container image registry (GHCR / ACR) | [대안](DEC-2026-004-build-image-registry.md) |
| DEC-2026-005 | STEP-01 / STEP-06 준비 조사 | PROPOSED | 승인된 VM 배포 release canonical store (Blob / Artifacts) | [대안](DEC-2026-005-release-canonical-store.md) |
| DEC-2026-006 | STEP-01 / STEP-05 준비 조사 | PROPOSED | 공유 VM 쌍 CD 운영 모델 (중앙 자동 / 개별 수동) | [대안](DEC-2026-006-cd-operation-model.md) |
| DEC-2026-007 | STEP-01 / STEP-04/05 준비 조사 | PROPOSED | Azure VM 배포 전송 모델 (직접 push / 명령 push·파일 pull) | [대안](DEC-2026-007-server-deployment-transport.md) |
| DEC-2026-008 | STEP-01 / STEP-05 준비 조사 | PROPOSED | Frontend 정적 자산 rollout (단일 slot / hash 자산 2버전) | [대안](DEC-2026-008-frontend-static-asset-rollout.md) |

runner 네트워크, Blob 보존, drain 상세 방식은 조사 문서의 후보이며 승인된 설계가 아니다.
현재 Gate는 STEP-01 DISCOVERY다. 후속 Decision 확정은 선행 범위 확인 후 진행한다.

후속 사용자 결정: [DEC-2026-003 — Hosted 전환 제약](DEC-2026-003-hosted-only-scope.md), STEP-01 / DECIDED. DEC-2026-002의 배포 신뢰 경계는 계속 PROPOSED다.

[GitHub repository 권한·Azure Identity 거버넌스 선택안](../designs/github-access-and-identity-governance-options.md)은 DEC-2026-002를 구체화하는 STEP-01 조사 자료다. 중앙 deployment repository와 환경별 workload identity 유형은 아직 사용자 결정이 아니다.
