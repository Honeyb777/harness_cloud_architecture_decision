# Decision Register

전체 검토의 결정 색인이다. 원본은 각 Assessment의 `decisions/`에 보관하고 여기에는 복사하지 않는다.
[Decision 템플릿](../templates/DECISION-TEMPLATE.md)과 [검토별 구조](../sot/ASSESSMENT-STRUCTURE.md)를 따른다.
Decision ID는 저장소 전체에서 연도별로 중복 없이 부여한다.

| ID | Assessment / Step | 적용 환경·워크로드 | 제목 | 승인 등급 | Status | 원본 |
|---|---|---|---|---|---|---|
| DEC-2026-001 | ASM-2026-001 / STEP-01 | ENV-01/02/03, WL-01/02/03 | 사용자 지정 파일럿 범위 | USER_DECISION | DECIDED | [원본](../assessments/ASM-2026-001-idc-azure-vm-pilot/decisions/DEC-2026-001-pilot-scope.md) |
| DEC-2026-002 | ASM-2026-001 / STEP-03/05 준비 조사 | ENV-02/03, WL-01/02/03 | 운영 배포 신뢰·실행 경계 | USER_DECISION | PROPOSED | [원본](../assessments/ASM-2026-001-idc-azure-vm-pilot/decisions/DEC-2026-002-deployment-trust-boundary.md) |

| DEC-2026-003 | ASM-2026-001 / STEP-01 | ENV-02/03, WL-01/02/03 | Hosted 전환 및 배포 제약 | USER_DECISION | DECIDED | [원본](../assessments/ASM-2026-001-idc-azure-vm-pilot/decisions/DEC-2026-003-hosted-only-scope.md) |

Recommendation, User Decision, Implemented Reality는 원본에서 각각 관리한다.
