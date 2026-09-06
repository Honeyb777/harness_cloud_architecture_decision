# Cloud Architecture Decision Harness

클라우드(Azure, AWS 및 필요 시 기타 플랫폼) 아키텍처 검토를 단발성 설계가 아니라
**반복 가능한 의사결정 프로세스와 누적 지식 체계**로 운영하기 위한 Repository다.

## 목표

- 아키텍처 요구사항을 Step 단위로 분석한다.
- 여러 대안을 비교하고 권장안을 제시한다.
- 사용자가 결정해야 하는 사항은 구현/설계 확정 전에 Decision Gate로 분리한다.
- `권장 결정`, `사용자 결정`, `실제 적용 결정`을 구분해 기록한다.
- 실제 적용 결과가 권장안과 다른 경우 제약과 사유를 남긴다.
- 비용, 태깅, 글로벌 확장성, 보안, OIDC, Kubernetes Landing Zone, 감사 대응을 함께 검토한다.
- Storage/Database의 Backup, Retention, Tiering, Lifecycle을 비용 관점에서 함께 평가한다.
- 의사결정 결과를 1-page 보고서와 상세 보고서로 각각 생성할 수 있게 한다.

## 기본 작업 흐름

`INTAKE → DISCOVERY → OPTIONS → DECISION_GATE → TARGET_DESIGN → VALIDATION → ACCEPTED → EVIDENCE → REPORT`

다음 Step의 변경성 작업은 이전 Step이 완료된 이후에만 수행한다.
읽기/조사/비교는 다음 Step 준비를 위해 허용할 수 있다.

## SOT

핵심 기준은 `sot/` 디렉터리에 저장한다.

- `sot/CHARTER.md` : 하네스의 목적과 범위
- `sot/GUARDRAILS.md` : 공통 가드레일
- `sot/DECISION-POLICY.md` : 사용자 승인 기준
- `sot/TAGGING-STANDARD.md` : 비용/조직/서비스 구분을 위한 태깅 기준
- `sot/QUALITY-GATES.md` : Step 완료 기준
- `sot/REPORTING-STANDARD.md` : 보고서 작성 기준
- `sot/WORKFLOW.md` : Step/Phase, 상태, 완료 기준 및 식별자
- `sot/ASSESSMENT-STRUCTURE.md` : 검토 건별 디렉터리와 환경 경계
- `sot/AGENT-LIFECYCLE.md` : 역할 선택·평가·개선 기준

## 주요 증적

- `assessments/` : 검토 건별 환경·요구사항·결정·설계·증적·평가·보고서·이력
- `decisions/` : 전체 검토의 결정 원본을 연결하는 색인
- `history/` : 하네스 변경 및 검토 간 요약 이력
- `reports/` : 공통 보고서 템플릿; 결과물은 해당 Assessment 내부에 저장
- `examples/` : 실제 검토에 포함하지 않는 예시

## 시작하기

이 저장소는 Markdown 기반 문서 하네스다. 패키지 설치, 빌드 또는 클라우드 인증 설정은 현재 필요하지 않다.

1. [작업 지침](AGENTS.md)과 [Bootstrap Prompt](PROMPT.md)를 읽는다.
2. [Assessment 목록](assessments/README.md)에서 작업 대상을 선택한다. 새로운 대상이면 [검토별 구조](sot/ASSESSMENT-STRUCTURE.md)에 따라 별도 Assessment를 생성한다.
3. [Workflow](sot/WORKFLOW.md)와 [Quality Gates](sot/QUALITY-GATES.md)에 따라 현재 Step을 진행한다.
4. 주요 선택은 [Decision register](decisions/README.md)에 연결하고 권장안·사용자 결정·실제 적용을 각각 기록한다.
5. 매 역할 사용 후 평가를 기록한다. 완료 증적과 보고서는 해당 Assessment 안에 저장하고 전역 색인·요약을 갱신한다.

새 검토의 전체 구조는 [Assessment 템플릿](templates/ASSESSMENT-TEMPLATE.md)을 사용한다.
`agents/` 파일은 전문 역할별 검토 지침이며 실행 프로그램은 아니다.

## 검토 범위와 역할 개선

매 검토는 여러 서비스와 서로 다른 환경을 포함할 수 있다. 공통 기준과 검토별 사실·결정을 분리하며 Cloud, IDC, Hybrid 및 공용 플랫폼 검토를 지원한다.

- DevOps: Java/Tomcat, Kotlin/Tomcat, Vue/npm 등의 빌드·실행 환경, CI/CD, artifact 승격, 배포·롤백.
- Network/IDC: 회선·라우팅·DNS·방화벽, 기존 인증·DB·파일·배치·API 연동과 단계적 이전.
- 역할 운영: [역할 목록](agents/README.md)에서 필요한 역할을 선택하고 [사용 후 평가](sot/AGENT-LIFECYCLE.md)에 따라 유지·보완·추가·통합·비활성 여부를 판단한다.

커밋 메시지를 요청하면 변경 내용을 요약한 한 줄 한국어 메시지로 작성한다.
