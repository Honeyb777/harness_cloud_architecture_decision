# 역할 사용 후 평가 — IDC/Azure 파일럿 Discovery

- Assessment: ASM-2026-001
- Date: 2026-09-07
- 수행/평가 주체: 현재 대화의 동일 에이전트
- 실행 방식: 단일 에이전트 역할 검토
- 검토 독립성: 자기 검토; 별도 에이전트 실행/독립 검토 아님
- 입력: 사용자 현행·목표 설명, Enterprise Cloud/네트워크 미정/서비스별 JVM 후속 답변, 공식 문서
- 산출물: [Assessment](../README.md), [공식 출처](../evidence/official-sources.md), [검증 계획](../evidence/pilot-validation-plan.md)

기준은 문서 조사 범위에서 평가한다. 실제 운영 가능성은 환경에 접속하거나 시험하지 않아 미평가다.
중복·누락의 충족은 책임 및 이번 요청의 확인 항목이 문서에 반영되었다는 뜻이다.

| 역할·파일 | 수행 범위 | 요구사항 충족 | 근거 정확성 | 실행·운영 가능성 | 중복 | 누락 영역 | 증거·사유 | 판단·후속 행동 |
|---|---|---|---|---|---|---|---|---|---|
| [Orchestrator](../../../agents/00-orchestrator.md) | 범위·Gate·질문 | 충족 | 충족 | 미평가 | 충족 | 충족 | 새 검토 생성, 제품 형태·JVM 분리 확인, 미정 정책 분리 | 유지; 범위 매핑 후 다음 Gate |
| [Cloud](../../../agents/10-cloud-architect.md) | VM/Cloud·배포 경로 통합 | 충족 | 충족 | 미평가 | 충족 | 충족 | 사용자가 지정한 Azure/VM을 재선택하지 않음 | 유지; 리전·계정 확인 |
| [DevOps](../../../agents/25-devops-application-delivery.md) | OIDC job·배포·Tomcat·Vue | 충족 | 충족 | 미평가 | 충족 | 충족 | repo 범위 concurrency와 정적 asset version skew 발견 | 보완; 공통 체크리스트에 반영 |
| [Security](../../../agents/30-security-identity-audit.md) | 신뢰·승인·RBAC | 충족 | 충족 | 미평가 | 충족 | 충족 | 개인/워크로드 분리, 실제 subject와 VM 실행/MI 경계 확인 | 보완; 거부 시험/subject 검증 반영 |
| [Network](../../../agents/35-network-hybrid-connectivity.md) | hosted VNet·IDC·probe | 충족 | 충족 | 미평가 | 충족 | 충족 | 사설 접근은 OIDC로 해결되지 않으며 probe/drain 구분 필요 | 보완; 제어/데이터 경로와 drain 의미 반영 |
| [FinOps](../../../agents/40-finops.md) | 캐시·runner·보존 비용 | 충족 | 충족 | 미평가 | 충족 | 충족 | 입력 없이 가격 추정하지 않고 driver 기록 | 유지; N·용량·실행 시간 수집 |
| [Data](../../../agents/50-data-storage.md) | Blob·보존·DB 호환 | 충족 | 충족 | 미평가 | 충족 | 충족 | N개와 보호 버전 분리, DB rollback 별도 | 유지; 보존/정합성 기준 확인 |
| [Operations](../../../agents/60-reliability-operations.md) | 장애·취소·잠금·용량 | 충족 | 충족 | 미평가 | 충족 | 충족 | workflow 종료와 원격 종료 차이, HOLD·reconcile 필요 | 보완; 실패 안전 절차 반영 |
| [Research](../../../agents/70-alternative-researcher.md) | 공식 문서·변경된 기능 | 충족 | 충족 | 미평가 | 충족 | 충족 | immutable OIDC subject, queue: max와 권한 표기 차이 확인 | 유지; 구현 시 재확인 |
| [Reviewer](../../../agents/80-reviewer.md) | 전제·과장·경합 검토 | 충족 | 충족 | 미평가 | 충족 | 충족 | readiness marker만으로 보장 금지, lease의 강제 범위 제한, 실제 시험 분리 | 유지; VAL-01~24 실제 결과 검토 |
| [Historian](../../../agents/90-historian-reporter.md) | 결정·색인·증적·요약 | 충족 | 충족 | 미평가 | 충족 | 충족 | 사용자 결정과 추천·실제 적용을 분리 | 유지 |

## 개선 실행

- DevOps, Security, Network, Operations 역할에 재사용 가능한 검토 항목을 추가했다.
- 기존 역할로 요청 범위를 담당할 수 있어 새 전문 역할을 추가하지 않았다. 이번 비사용은 Platform/Kubernetes이며 사용자가 VM을 지정했고 Kubernetes 채택 검토가 필요하지 않기 때문이다. 역할을 비활성화할 근거는 아니다.
- 필수 미해결: 실제 권한·통신·배포·세션·실패 복구 시험. 현재 분석 문서는 완료했지만 Assessment를 ACCEPTED로 처리하지 않는다.
- 후속 담당 역할: Orchestrator가 환경/책임 매핑을 확인하고 Security/DevOps/Network/Operations가 해당 시험을 수행·검토한다. 실제 조직 담당자는 미지정.
