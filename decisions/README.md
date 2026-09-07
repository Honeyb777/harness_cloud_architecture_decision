# Decision Register

전체 검토의 결정 색인이다. 원본은 각 Assessment의 `decisions/`에 보관하고 여기에는 복사하지 않는다.
[Decision 템플릿](../templates/DECISION-TEMPLATE.md)과 [검토별 구조](../sot/ASSESSMENT-STRUCTURE.md)를 따른다.
Decision ID는 저장소 전체에서 연도별로 중복 없이 부여한다.

| ID | Assessment / Step | 적용 환경·워크로드 | 제목 | 승인 등급 | Status | 원본 |
|---|---|---|---|---|---|---|
| DEC-2026-001 | ASM-2026-001 / STEP-01 | ENV-01/02/03, WL-01/02/03 | 사용자 지정 파일럿 범위 | USER_DECISION | DECIDED | [원본](../assessments/ASM-2026-001-idc-azure-vm-pilot/decisions/DEC-2026-001-pilot-scope.md) |
| DEC-2026-002 | ASM-2026-001 / STEP-03/05 준비 조사 | ENV-02/03, WL-01/02/03 | 운영 배포 신뢰·실행 경계 | USER_DECISION | PROPOSED | [원본](../assessments/ASM-2026-001-idc-azure-vm-pilot/decisions/DEC-2026-002-deployment-trust-boundary.md) |

| DEC-2026-003 | ASM-2026-001 / STEP-01 | ENV-02/03, WL-01/02/03 | Hosted 전환 및 배포 제약 | USER_DECISION | DECIDED | [원본](../assessments/ASM-2026-001-idc-azure-vm-pilot/decisions/DEC-2026-003-hosted-only-scope.md) |
| DEC-2026-004 | ASM-2026-001 / STEP-01 | ENV-03, WL-03 | CI build container image registry | USER_DECISION | PROPOSED | [원본](../assessments/ASM-2026-001-idc-azure-vm-pilot/decisions/DEC-2026-004-build-image-registry.md) |
| DEC-2026-005 | ASM-2026-001 / STEP-01/06 준비 조사 | ENV-02/03, WL-01/02/03 | VM 배포 release canonical store | USER_DECISION | PROPOSED | [원본](../assessments/ASM-2026-001-idc-azure-vm-pilot/decisions/DEC-2026-005-release-canonical-store.md) |
| DEC-2026-006 | ASM-2026-001 / STEP-01/05 준비 조사 | ENV-02/03, WL-01/02/03 | 공유 VM 쌍 CD 운영 모델 | USER_DECISION | PROPOSED | [원본](../assessments/ASM-2026-001-idc-azure-vm-pilot/decisions/DEC-2026-006-cd-operation-model.md) |
| DEC-2026-007 | ASM-2026-001 / STEP-01/04/05 준비 조사 | ENV-02/03, WL-01/02/03 | Azure VM 배포 전송 모델 | USER_DECISION | PROPOSED | [원본](../assessments/ASM-2026-001-idc-azure-vm-pilot/decisions/DEC-2026-007-server-deployment-transport.md) |
| DEC-2026-008 | ASM-2026-001 / STEP-01/05 준비 조사 | ENV-02, WL-02 | Frontend 정적 자산 rollout | USER_DECISION | PROPOSED | [원본](../assessments/ASM-2026-001-idc-azure-vm-pilot/decisions/DEC-2026-008-frontend-static-asset-rollout.md) |

Recommendation, User Decision, Implemented Reality는 원본에서 각각 관리한다.
