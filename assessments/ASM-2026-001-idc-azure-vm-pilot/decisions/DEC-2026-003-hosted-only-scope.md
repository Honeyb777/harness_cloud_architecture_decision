# DEC-2026-003 — Hosted 기반 전환과 애플리케이션 배포 제약

## Status

DECIDED

## Traceability

- Assessment: ASM-2026-001
- Applicable Environments: ENV-02/03
- Applicable Workloads: WL-01/02/03
- Step: STEP-01
- Approval Level: USER_DECISION
- Date: 2026-09-07

## Context / Options

[기존 범위](DEC-2026-001-pilot-scope.md)를 후속 답변으로 구체화한다. 상세 비교는 [Hosted 전환 설계](../designs/hosted-only-push-pull-key-vault-static.md)에 기록한다.

## User Decision

- 기존 내부 GitLab, Jenkins, Nexus, Ansible을 목표 구성에서 제외한다.
- 의존성 캐시는 Actions Cache를 사용하고 Key Vault를 도입한다.
- 추가 도구·라이브러리는 빌드 runner에 필요하다.
- 다른 포트에서 새 JVM을 함께 기동하는 방식은 사용하지 않는다.
- 세션은 JWT Cookie 방식이다. DB 스키마·API 공존에 따른 위험은 감수하는 방향으로 진행한다.
- Decision Owner: 사용자. Decision Date: 2026-09-07.
- Approval Evidence: 기존 도구 제외·Cache·Key Vault·배포 제약에 관한 후속 메시지와 '빌드 runner에 필요한 도구·라이브러리' 답변.

## Agent Recommendation

명령은 ARM Managed Run Command로 전달하고 파일은 VM Managed Identity로 Blob에서 받는 혼합 방식을 우선 검토한다. 추가 빌드 도구는 hosted runner 설치로 시작하고 필요하면 GHCR 공통 빌드 이미지를 사용한다. ACR은 기본 구성에서 제외하는 것을 추천한다.

위 추천은 사용자 확정과 구분한다. push/pull 최종 방식, Blob 공개 데이터 엔드포인트 허용 여부, VNet hosted larger runner, 보존 개수, 운영 승인자 및 잠금 구현은 미확정이다.

## Implemented Reality

문서 반영만 수행했다. Azure/GitHub 구성 적용·배포·기존 IDC 도구 삭제는 수행하지 않았다.

## Risks / Revisit

- DB/API 위험 수용은 파괴적인 스키마 변경이나 특정 장애 범위의 사전 승인을 뜻하지 않는다. 릴리스별 영향과 관측 결과를 기록한다.
- JWT 검증 키는 두 VM에서 일관되어야 한다. 키 교체 시 기존 토큰의 검증 기간을 고려한다.
- Actions Cache는 영구 패키지 저장소가 아니다. Nexus에만 존재하는 자체 패키지는 별도 영구 원본으로 이전해야 한다.
- 실제 통신·권한·배포 검증은 [검증 계획](../evidence/pilot-validation-plan.md)의 NOT_RUN 상태다. STEP-01 DISCOVERY를 유지한다.
