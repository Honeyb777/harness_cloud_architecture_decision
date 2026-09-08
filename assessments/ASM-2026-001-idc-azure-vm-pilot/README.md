# ASM-2026-001 — IDC 일부 서비스의 Azure VM 파일럿 이전

- Status: IN_PROGRESS
- Current Step: STEP-01
- Phase: DISCOVERY
- Created / Updated: 2026-09-07
- Owner: 미지정
- 범위: 기존 IDC CI/CD에서 GitHub Enterprise Cloud·Actions·Azure VM으로 일부 이전
- 근거: 현재 대화의 사용자 설명 및 후속 답변; 환경 직접 조회는 수행하지 않음

## 현재 결론

GitHub Enterprise Cloud, Azure VM, Application Gateway, 서비스별 별도 Tomcat/JVM은 사용자 확인 사항이다.
GitHub-hosted runner, Azure OIDC, Managed Run Command, VM Managed Identity와 Blob을 조합하는 방안을 우선 검토한다.
기존 GitLab/Jenkins/Nexus/Ansible은 새 파이프라인에서 제외한다. Actions Cache와 Key Vault 도입은 사용자 결정이다.
추가 모듈은 빌드 runner용이므로 ACR은 기본안에서 제외하고 필요 시 GHCR 공통 빌드 이미지를 검토한다.
Blob을 private-only로 사용할 경우 Azure VNet 연결 GitHub-hosted larger runner를 비교한다.
Tomcat은 다른 port의 병행 JVM 없이 rolling하며, JWT Cookie를 사용한다. DB/API 공존 문제는 사용자 수용 위험으로 구분한다.
중앙 배포 저장소와 서버 쌍 공통 잠금, probe 기반 신규 요청 제외와 실제 요청 종료 확인, Vue 정적 자산의 양쪽 서버 선배치를 제안한다.
이 내용은 조사·권장안이며 Target Design 승인이나 실제 적용 완료가 아니다.

## 산출물

- [최신 후속 비교: push/pull·네트워크·Key Vault·빌드·정적 전환](designs/hosted-only-push-pull-key-vault-static.md)
- [현재 환경 및 미확인 항목](context.md)
- [요구사항](requirements.md)
- [OIDC·권한·승인 및 runner 대안](designs/identity-and-delivery-options.md)
- [GitHub repository 권한·Azure Identity 거버넌스 선택안](designs/github-access-and-identity-governance-options.md)
- [drain·Tomcat·Vue·동시 배포 제어](designs/availability-and-rollout-options.md)
- [Blob·Artifacts·Actions Cache·이미지 및 비용](designs/artifact-and-build-options.md)
- [CI/CD 단계와 runner·GHCR·ACR build image 선택](designs/ci-cd-stages-and-build-image-options.md)
- [GitHub Actions CI: build·cache·artifact·release publish 권한 모델](designs/github-actions-ci-build-and-publish-model.md)
- [Build cache·GitHub Packages·배포 release 생명주기](designs/build-cache-and-release-lifecycle-options.md)
- [CD 순차 제어·runner 대기·Azure OIDC 준비](designs/cd-sequencing-and-azure-oidc-preparation.md)
- [CD 운영 모델: 중앙 자동 조정 / 개별 수동 조정](designs/cd-operation-model-options.md)
- [VM 배포 전송 모델: 직접 push / 명령 push·파일 pull](designs/server-deployment-transport-options.md)
- [Tomcat rollout: same-port drain 후 교체](designs/tomcat-rollout-model-options.md)
- [Frontend 정적 자산: 단일 slot / hash 자산 2버전 공존](designs/frontend-static-asset-rollout-options.md)
- [공식 출처](evidence/official-sources.md)
- [파일럿 검증 계획](evidence/pilot-validation-plan.md)
- [결정 목록](decisions/README.md)
- [중간 요약](reports/executive-brief.md)
- [배포 절차 초안](reports/deployment-draft.md)
- [Microsoft 협의용 질문 목록](reports/ms-meeting-question-list.md)
- [Microsoft 제품 소개·도입 협의 질문지 v2](reports/ms-product-introduction-question-list-v2.md)
- [GitHub Enterprise Cloud 권한·배포 보호 모델](reports/github-enterprise-cloud-access-model.md)
- [GitHub Actions CI 빌드·산출물·Azure OIDC 운영 모델](reports/github-actions-ci-model.md)
- [GitHub Actions CD와 Azure Pipelines CD: 공유 이중화 VM 배포 운영 모델](reports/github-actions-cd-shared-vm-model.md)
- [Azure 관리·Identity·권한 모델](reports/azure-management-and-identity-model.md)
- [GitHub-Azure OIDC 연결·사용 모델](reports/github-azure-oidc-integration.md)
- [의사결정 1장 요약 (HTML)](reports/decision-summary-one-page.html)
- [상세 의사결정 보고서 (HTML)](reports/decision-detail.html)
- [역할 평가](evaluations/2026-09-07-discovery.md)
- [이력](history/2026-09.md)

## Gate 및 Step

INTAKE는 대상·목적·미확인 항목이 식별되어 완료했다. DISCOVERY는 tenant/subscription·repo·환경/서버 매핑과 책임자 확인이 남았다.
사용자가 요청한 OIDC·배포·저장소 대안은 후속 Step의 조사로 병행했다. 후속 설계 확정이나 Azure/GitHub 변경은 수행하지 않았다.

| Step | 영역 | Status | 이번 작업 |
|---|---|---|---|
| STEP-01 | 조직·환경·책임·구독 범위 | IN_PROGRESS | 사용자 선택 기록, 환경 식별 및 Unknowns |
| STEP-02 | 리전·데이터 위치 | BLOCKED_BY_STEP_01 | 리전·GitHub 도메인·데이터 거주 미확인 |
| STEP-03 | OIDC·접근·승인 | BLOCKED_BY_STEP_02 | 공식 조사와 권한 초안 |
| STEP-04 | 네트워크·IDC 연동 | BLOCKED_BY_STEP_03 | hosted runner의 접근 대안 조사 |
| STEP-05 | VM·Tomcat·Vue·CI/CD | BLOCKED_BY_STEP_04 | 롤링 배포·전역 잠금 대안 조사 |
| STEP-06 | 배포 저장소·데이터 | BLOCKED_BY_STEP_05 | Blob/Artifacts 비교, DB 호환성 확인 과제 |
| STEP-07 | 보안·관측·감사 | BLOCKED_BY_STEP_06 | 승인·실행·배포 상태 증적 계획 |
| STEP-08 | 비용·태그 | BLOCKED_BY_STEP_07 | 비용 driver, 보존 정책 검토 |
| STEP-09 | 전체 대안 통합 | BLOCKED_BY_STEP_08 | 현재 문서는 추천 초안 |
| STEP-10 | 검증·최종 보고 | BLOCKED_BY_STEP_09 | 파일럿 시험 미수행; ACCEPTED 아님 |

## 다음 확인

1. GitHub 조직·source/배포 저장소 경계, 실제 team·repository role과 Azure tenant/subscription/RG, dev/prod 서버 쌍 매핑.
2. Blob/Key Vault의 공용·사설 접근 정책 및 현재 App Gateway의 서비스별 pool/settings/probe 연결.
3. JWT 서명·검증키 일관성, WebSocket/장시간 요청, 1대 수용 용량, 배포 버전 보존 개수와 복구 기준.

후속 기록: [사용자 결정](decisions/DEC-2026-003-hosted-only-scope.md), [역할 평가](evaluations/2026-09-07-hosted-followup.md), [배포 절차 초안](reports/deployment-draft.md).
- [Azure Pipelines를 CD 조정기로 사용하는 선택지](designs/azure-pipelines-cd-option.md)
- [GitHub Actions CD: 공유 이중화 VM·Application Gateway 배포 모델](designs/github-actions-cd-shared-vm-model.md)
