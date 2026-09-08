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

## GitHub 권한·Azure Identity 분류 후속

| 역할 | 수행·평가 근거 | 누락·중복 및 후속 판단 |
|---|---|---|
| Orchestrator / Historian | STEP-01 Gate를 유지하고 권한 층·선택 항목·이력 연결을 정리 | 유지; 실제 team/repo/scope 매핑 후 Decision Gate 진행 |
| Security | 사람 권한, job token, OIDC federation, Azure RBAC, VM MI를 분리 | 유지; 실제 claim·최소 권한·거부 시험 필요 |
| DevOps / Operations | 중앙 deployment repository와 공유 VM pair 잠금의 책임 경계 연결 | 유지; 실제 manifest 반입과 긴급 운영 경합 절차 확인 필요 |
| Research / Reviewer | GitHub Enterprise Cloud의 repository/custom/CI-CD 역할과 Azure OIDC 근거 재확인 | 유지; 권한 누적과 prod 우회 경로를 실제 설정 검토에서 확인 |

이번 보완은 문서 조사 범위이며 실행·운영 가능성은 미평가다. 기존 역할로 다룰 수 있어 역할 추가·통합·비활성화는 하지 않았다.

CI/CD 단계와 build image 선택 보완에서 DevOps는 workflow·runner·artifact 경계를, Security는 build job과 Azure 배포 권한 분리를, FinOps는 registry·runner 사용량 비용 요인을 재확인했다. 실제 build, private registry pull, cold-cache 성능은 미평가이며 역할 구성은 유지한다.

사용자 요구에 따라 build container image registry를 GHCR/ACR 선택으로 좁혔다. DevOps·Security·FinOps 역할의 기존 범위로 충족하며, GitHub Packages allowance와 GHCR Container registry 현행 과금 정책은 공식 문서로 재확인했다. 실제 계약·billing 설정, image pull, 보안 갱신 절차는 미평가다.

Build cache, 사내 package registry, CI 증적, VM deployment release의 보존·삭제 책임을 분리했다. DevOps·Data·FinOps·Security의 기존 역할 범위로 충족하며, 실제 lifecycle·soft delete·cleanup·rollback 시험은 미평가다. 역할 구성은 유지한다.

Artifacts와 cache·private GHCR build image의 실행 시점과 권한 경계를 보완했다. DevOps·Security 역할의 기존 범위로 충족하며, 실제 private image pull·cross-repository package access·artifact 전달 시험은 미평가다.

FinOps와 DevOps 관점에서 registry/cache/artifact 처리가 GitHub-hosted runner 실행 시간에 포함되는 점과 단계별 측정 항목을 보완했다. 실제 실행 분·비용·cache hit율은 미평가이며 역할 구성은 유지한다.

CD의 cross-repository concurrency, VM pair lock, Environment 대기와 Azure OIDC/RBAC·Key Vault 역할 분리를 DevOps·Operations·Security·FinOps 관점에서 보완했다. 실제 queue·billable minutes·OIDC 거부·lock recovery 시험은 미평가이며 역할 구성은 유지한다.

중앙 자동 조정과 개별 수동 조정 CD 운영 모델을 DevOps·Operations·Security 관점에서 비교했다. 실제 운영 인력·배포 빈도·공통 lock 책임자는 미확인이므로 선택과 실행 가능성은 미평가다.

VM 직접 push와 Managed Run Command·VM MI hybrid transport를 DevOps·Network·Security·Operations 관점에서 비교했다. SSH credential/runner private network 또는 Agent/Blob egress의 실제 적합성은 미평가다.

Tomcat same-port drain 교체와 별도 port 병행 rollout을 DevOps·Network·Operations 관점에서 비교했다. 현재 별도 port 방식은 사용자 결정으로 범위에서 제외하며, probe/drain·warm-up·capacity 증적은 미평가다.

사용자 지시에 따라 Tomcat 별도 port 방식의 상세 검토를 제거하고 same-port 절차만 유지했다. Frontend 자산의 단일 slot·두 버전 공존 선택은 DevOps·Operations·FinOps 범위로 기록했으며, 추가 frontend 이슈는 확인 전 반영하지 않는다.

Frontend B안의 release 경로·asset URL·두 VM 순서·cleanup 예시를 DevOps·Operations 관점에서 보완했다. 실제 build asset naming·Nginx 설정·browser 요청은 미확인이며 추가 이슈로 확장하지 않는다.

기존 전문 역할의 책임 범위로 요청을 처리할 수 있어 새 역할을 추가하지 않는다. 이번 미사용인 Kubernetes 역할은 VM 대상 범위에 해당하지 않으며 비활성화할 근거는 없다. 문서 역할의 중복은 종합 검토에서 조정했고, 남은 누락은 구현 및 환경 시험으로 명시했다.
## 의사결정 보고서 작성 및 자체 검토

- 범위: GitHub·Azure 권한 과제를 제외한 미정 정책을 1장 요약과 상세 보고서로 정리하고, 기존 결정 기록·검증 계획과의 일치 여부를 점검했다.
- 산출물: [1장 HTML 보고서](../reports/decision-summary-one-page.html), [상세 HTML 보고서](../reports/decision-detail.html), [Assessment 인덱스](../README.md).
- Historian / Reporter: 충족. 사용자 결정 필요(DEC-2026-004~008), 권장안, 실제 적용 미수행을 분리하고 assessment 내부 보고서와 이력을 연결했다.
- Reviewer: 보완 필요. 보고서상 설계 선택지는 기존 결정 기록과 일치하지만, 실제 repository·VM·Application Gateway·network 조건과 Pilot validation은 모두 미확인/NOT_RUN이므로 설계 또는 적용 완료로 판정하지 않는다.
- 후속 조치: 사용자가 선택한 항목만 해당 Decision record에 기록하고, 환경 매핑과 VAL-32~37을 포함한 Pilot 증적을 추가한다.
## 독립 보고서 보완 및 자체 검토

- Historian / Reporter: 충족. 두 HTML 보고서를 저장소 내부 문서 링크 없이 읽을 수 있게 고치고, 권장안·미정 정책·실제 적용/검증 미수행 상태를 본문에 분리했다.
- Reviewer: 보완 필요. 배포·VM 전달·frontend asset의 흐름 그림은 비교 이해를 돕지만 실제 환경 동작을 증명하지 않는다. VM/Gateway/network/runtime 사실 확인 및 Pilot 검증이 후속 조건이다.
- 후속 조치: 보고서 수신자의 질문과 선택 결과를 받은 뒤 해당 결정 기록과 Pilot 증적을 갱신한다.
## GitHub Actions CI 독립 리포트 검토

- Historian / Reporter: 충족. CI 범위를 build image·cache·Artifacts·Blob release 원본·OIDC publisher로 한정하고, 선택지·권장안·미결정 값·실제 적용 미수행을 한 문서 안에 분리했다. 내부 결정 코드와 내부 상대 링크는 본문에서 제외하고 공식 공개 문서 링크만 근거로 제시했다.
- DevOps / Security / Data / FinOps: 보완 필요. GHCR/ACR 실제 pull, cache cold/warm 시간, Artifact 보존, Blob upload 및 OIDC 거부 경로는 모두 NOT_RUN이다. 실제 registry, Blob lifecycle, release Environment, custom role 가능 여부를 확정한 뒤 Pilot 증적을 추가해야 한다.

## 공유 이중화 VM CD 초안 검토

- DevOps / Operations: 충족. GitHub Actions 단독 CD와 Azure Pipelines CD의 조정 책임을 비교하고, 어느 쪽도 Gateway drain·Tomcat 전환·readiness를 대신하지 않는다는 경계를 명시했다. shared pair에서 다른 서비스가 두 VM을 동시에 제외하지 않도록 pair 단위 queue/lock·peer Healthy·VM1 이후 VM2 조건을 추가했다.
- Security / Network / Data: 보완 필요. GitHub OIDC와 Azure DevOps service connection federation은 별도 trust로 검증해야 하며, Run Command·VM Managed Identity Blob pull, probe·connection draining, private network와 checksum 검증은 모두 NOT_RUN이다.
- Reviewer: 보완 필요. Vue versioned asset 2버전 공존은 이전 HTML의 chunk 요청을 보호하는 설계이며, 실제 asset URL·cache header·session 보존 기간은 frontend 구현과 Pilot 증적으로 확인해야 한다.

- Historian / Reporter: 충족. 공유 VM CD 리포트는 잠금·순차 실행과 Gateway/Nginx/Tomcat traffic 전환을 별도 흐름으로 설명하고, 선택지·권장안·미결정 값·실제 적용 미수행을 본문에 포함했다.

- Historian / Reporter: 충족. Microsoft 제품 소개·도입 협의 질문지 v2는 보고서 다섯 종류의 순서를 유지하고, 비기술 참여자를 위해 용어 전체 이름·한글 뜻·주제별 목적을 질문 앞에 배치했다. 면접형 평가나 내부 결정 강요 대신 제품 설명·시연·도입 조건 확인으로 표현했다.

## Azure Pipelines CD 선택지 검토

- DevOps / Operations: 충족. Azure Pipelines Environment approval·exclusive lock의 배포 조정 역할과 Application Gateway drain·VM 순차 전환의 runtime 역할을 분리했다.
- FinOps: 보완 필요. 월 1,800분 무료 구간과 실제 approval/lock/deployment 시간의 billing 표시는 조직별 Pilot run으로 확인해야 한다.
- Security: 보완 필요. Azure Pipelines service connection의 workload identity federation, Environment 권한, Azure RBAC은 GitHub Actions OIDC 설정과 별도로 결정·검증해야 한다.
