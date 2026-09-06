# Azure VM 파일럿 CI/CD — 중간 검토 요약

- Assessment: ASM-2026-001
- 적용 범위: 기존 IDC 일부 서비스, Azure 이중화 VM, GitHub Enterprise Cloud
- Status: DISCOVERY / 권장안 검토 중
- Date: 2026-09-07

## 목적과 현황

GitLab/Jenkins/Nexus/Ansible과 SE의 L4 제외·투입 절차를 GitHub Actions 중심의 단기 인증·승인·순차 배포로 전환한다.
Azure VM 및 Application Gateway 사용, 서비스별 별도 Tomcat/JVM, GitHub Enterprise Cloud 도입 의향이 확인되었다.

## 권장안

중앙 배포 repo + 보호 Environment + 환경별 Azure OIDC Identity를 구성하고, Managed Run Command로 VM 배포를 실행하는 안을 우선 검토한다.
VM은 Managed Identity로 Blob release와 Key Vault runtime 항목을 읽는다. private-only Blob에 게시할 때 VNet 연결 GitHub-hosted larger runner를 비교한다.
GitLab/Jenkins/Nexus/Ansible은 새 파이프라인에서 제외하며 Actions Cache와 Key Vault 도입을 반영했다. 추가 모듈은 빌드용이므로 hosted runner setup/install과 필요시 GHCR 이미지를 사용하고 ACR은 현재 기본안에서 제외한다.
push/pull 비교 결과 명령 push + 파일 pull을 추천한다. Vue는 release/current 포인터를 VM별로 전환하며 두 VM 전체 원자적 전환으로 표현하지 않는다.

## 가용성과 주요 위험

- probe 실패를 Azure connection draining과 동일시하지 않는다. 신규 요청 제외와 처리 중 요청 종료를 각각 확인한다.
- 서버 쌍 전체 잠금과 영속 배포 상태로 다른 서비스·저장소·수동 작업의 동시 제외를 방지한다.
- Tomcat은 부팅·warm-up·실제 readiness 검증 후 재투입한다.
- Vue 새 정적 자산은 두 VM에 먼저 배치하고 이전 URL의 자산을 유지한다.
- JWT Cookie 사용과 별도 port JVM 병행 제외를 반영했다. VM 한 대의 용량·장기 연결은 미검증이며 DB/API 공존은 수용 위험으로 구분한다.

## 비용 및 결정 필요사항

가격은 미산정. runner 실행 시간, Blob/Artifacts 보호본, GHCR·Key Vault, 사설 네트워크 및 실행 용량이 주요 driver다.
org/repo와 tenant/subscription/환경 범위를 먼저 확인하고 중앙 배포 신뢰 경계를 선택한다. 네트워크 제약, N개 보존의 N·보호 기간, drain 수용 기준은 후속 결정이다.

## 다음 단계

범위 매핑 → OIDC/RBAC·Environment 확정 → dev의 인증/파일 전달 → 실패 주입 포함 rolling 시험 → prod 적용 판단.
상세: [권한](../designs/identity-and-delivery-options.md), [무중단 배포](../designs/availability-and-rollout-options.md), [산출물·비용](../designs/artifact-and-build-options.md), [검증](../evidence/pilot-validation-plan.md).

최신 상세: [hosted 전환·push/pull·Key Vault·정적 포인터](../designs/hosted-only-push-pull-key-vault-static.md).
