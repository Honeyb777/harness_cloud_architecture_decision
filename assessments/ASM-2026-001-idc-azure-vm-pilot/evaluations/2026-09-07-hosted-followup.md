# Hosted 전환 후속 역할 평가

- 날짜: 2026-09-07 / ASM-2026-001
- 수행·평가: 동일 에이전트의 역할별 자기 검토. 별도 에이전트 실행이나 독립 평가가 아니다.
- 산출물: [설계 비교](../designs/hosted-only-push-pull-key-vault-static.md), [사용자 결정](../decisions/DEC-2026-003-hosted-only-scope.md), [공식 출처](../evidence/official-sources.md), [검증 계획](../evidence/pilot-validation-plan.md).
- 요구사항 반영·근거 확인은 문서 범위다. 실행·운영 가능성은 전 역할 미평가이며 실제 시험은 NOT_RUN이다.

| 역할 | 수행·평가 근거 | 누락·중복 및 후속 판단 |
|---|---|---|
| Orchestrator / Cloud | 사용자 확정과 혼합 배포 추천 분리, Gate 유지 | 기존 역할 유지; 네트워크 정책 미정 |
| DevOps | push/pull 비교, 빌드 도구·GHCR·캐시·원본 구분 | 유지; 실제 빌드와 cold cache 시험 필요 |
| Security | Key Vault 소비자별 MI/RBAC, JWT·인증서 회전 | 유지; 권한 거부 시험 필요 |
| Network | VM inbound와 Agent outbound, private data 경로 분리 | 유지; 실제 DNS·방화벽 시험 필요 |
| Data / FinOps | 캐시의 비영구성, Blob 보존·GHCR·larger runner 비용 요인 | 유지; 용량·보존·사용량 미정 |
| Operations | 같은 포트 JVM 순차 배포, 정적 파일의 VM별 전환과 전체 동시 전환 구분 | 유지; 용량·drain·복구 시험 필요 |
| Research / Reviewer | 공식 근거, ACR 필요 조건, 기능 위험과 인프라 가용성 구분 | 유지; 도구 제외와 오래된 추천의 충돌 확인 |
| Historian | DEC-2026-003, 색인·이력 연결 | 유지; 적용 완료로 기록하지 않음 |

기존 전문 역할의 책임 범위로 요청을 처리할 수 있어 새 역할을 추가하지 않는다. 이번 미사용인 Kubernetes 역할은 VM 대상 범위에 해당하지 않으며 비활성화할 근거는 없다. 문서 역할의 중복은 종합 검토에서 조정했고, 남은 누락은 구현 및 환경 시험으로 명시했다.
