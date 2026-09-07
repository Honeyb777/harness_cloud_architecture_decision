# Requirements — ASM-2026-001

모든 항목은 2026-09-07 사용자 설명에서 도출했다. 수치와 미제공 정책을 확정값으로 만들지 않는다.

| ID | 요구사항 | 검토 위치 |
|---|---|---|
| REQ-01 | IDC 일부를 Azure VM으로 파일럿 이전 | context, DEC-2026-001 |
| REQ-02 | GitHub Enterprise Cloud / Copilot / Actions 활용 | Identity / CI 대안 |
| REQ-03 | 가능하면 self-hosted Actions runner 운영을 피함 | runner 비교 |
| REQ-04 | OIDC 단기 인증과 비밀번호 없는 배포 지향 | 신뢰·RBAC·MI 분리 |
| REQ-05 | GitHub 승인 절차 및 누가 무엇을 승인하는지 확인 | 승인 표·설정 절차 |
| REQ-06 | Nexus를 포함한 내부 DevOps 4개 제품 미사용, Actions Cache 사용 | hosted 후속 비교 |
| REQ-07 | 배포 버전 일정 개수 유지, Blob/Artifacts 비교 | 보존 알고리즘 |
| REQ-08 | App Gateway probe 파일 기반 제외 후 순차 배포 가능성 | drain 비교·시험 |
| REQ-09 | 별도 port의 JVM 병행 없이 Tomcat rolling; JWT Cookie 검증 일관성 | rolling/JWT 키 검증 |
| REQ-10 | Vue 정적 파일 배포 중 서비스 연속성 | 양 서버 선배치·구버전 자산 |
| REQ-11 | 여러 서비스가 같은 서버 쌍을 공유해도 양쪽 동시 제외 방지 | 공통 잠금·상태 복구 |
| REQ-12 | 기존 IDC 업무 의존성이 남는 경우 고려; 내부 DevOps 의존성은 제거 | 통신·인증·복구 검증 |
| REQ-13 | 명령과 배포 파일의 push/pull 방식 비교 | hosted 후속 비교 |
| REQ-14 | Azure/GitHub만 사용 시 외부 접근 필요 구간 확인 | 네트워크 표 |
| REQ-15 | Key Vault 도입 항목·권한·소비 주체 구성 | Key Vault 표 |
| REQ-16 | 빌드 runner 추가 모듈 및 GitHub image/GHCR/ACR 필요성 비교 | runner/image 비교 |
| REQ-17 | 정적 slot·파일 포인터 전환 및 두 VM 일괄 적용 한계 | static 전환 |
| REQ-18 | DB/API 버전 공존 위험 수용 방향 반영 | DEC-2026-003 |
| REQ-19 | JDK·Node·추가 CLI 버전을 고정한 build container image를 GHCR 또는 ACR에서 관리 | CI/CD build image 결정 |
| REQ-20 | Azure VM 배포에서 직접 push와 명령 push·파일 pull 전송 모델을 비교·선택 | DEC-2026-007 |
| REQ-21 | Frontend 정적 자산에서 단일 slot 영향 수용 또는 hash/version 자산 2버전 공존을 선택 | DEC-2026-008 |

## 설계에서 도출한 수용 조건 — 확정 전 제안

- 영향을 받는 모든 서비스에 대해 최소 하나의 ready backend와 필요한 잔여 용량을 유지한다.
- 고장·타임아웃·취소·잠금 상실 시 다음 VM 배포를 시작하지 않는다.
- 승인된 release ID/commit/digest를 고정하고 dev/prod에서 임의 재빌드하거나 latest로 치환하지 않는다.
- HTTP 성공, JWT 키 일관성, 지연, WebSocket, 프론트 chunk 404를 시험한다. DB/API 공존은 수용 위험으로 기록하고 영향·복구 가능성을 관측한다. 호환성 재설계 완료를 이번 구성의 강제 선행 조건으로 삼지 않는다.
- 단기 인증은 장기 비밀값 제거를 뜻한다. Azure RBAC 할당의 자동 만료나 모든 기존 제품의 비밀번호 제거를 뜻하지 않는다.
- 사용자 트래픽 무중단의 정확한 정의와 허용 지연·오류·세션 영향 수치는 미확인이다.
| REQ-22 | GitHub Actions CI를 유지하면서 Azure Pipelines를 CD 조정기로만 추가하는 선택지와, 대기열·승인·잠금·비용·drain 책임 경계를 비교 | Azure Pipelines CD option |
