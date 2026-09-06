# Workflow / State Model

## Step과 Phase

Step은 검토 영역의 순서다. Assessment 템플릿의 STEP-01부터 STEP-10까지를 기본으로 사용한다. 검토 범위에 맞게 적용 여부를 기록하며 IDC 검토에 Cloud 전용 항목을 강제하지 않는다.
Phase는 각 Step 안의 진행 단계다.

`INTAKE → DISCOVERY → OPTIONS → DECISION_GATE → TARGET_DESIGN → VALIDATION → ACCEPTED → EVIDENCE → REPORT`

- `INTAKE`: 목적, 책임자, 범위 및 입력 문서를 접수한다. 미확인은 Unknowns에 기록한다.
- `DISCOVERY`: 요구사항, 현재 환경, 제약을 확인한다.
- `OPTIONS`: 대안과 추천 근거를 작성한다. 대안 비교는 각 Step에서 수행하며 STEP-09에서 통합한다.
- `DECISION_GATE`: 해당 Step의 필수 결정을 기록한다.
- `TARGET_DESIGN`: 사용자 결정에 맞는 설계와 책임 경계를 작성한다.
- `VALIDATION`: 품질 기준에 따라 검증 결과와 증적을 기록한다.
- `ACCEPTED`: 해당 Step의 필수 결정 및 검증이 완료된 상태다. 실제 배포 완료를 의미하지 않는다.
- `EVIDENCE`: 결정 기록, 적용 결과가 있는 경우 그 증적, History를 갱신한다.
- `REPORT`: 완료 또는 진행 중 상태를 명시한 보고서를 작성한다.

## Step Status

- `PROPOSED`: 착수 대기.
- `IN_PROGRESS`: 검토 중. 별도 Phase 필드에 현재 단계를 기록한다.
- `BLOCKED_BY_STEP_NN`: 선행 Step 완료 대기.
- `WAITING_FOR_USER`: 다음 진행에 필수인 사용자 입력 또는 결정 대기.
- `ACCEPTED`: 해당 Step의 결정과 Validation 완료.
- `NOT_APPLICABLE`: 제외 사유와 근거를 기록한 적용 제외. 주요 범위 제외는 사용자 결정을 남긴다.

`Phase`와 `Status`는 구분한다. 예를 들어 `Phase: DECISION_GATE`, `Status: WAITING_FOR_USER`가 가능하다.
완료 후 새 요구사항이나 결정 변경이 생기면 영향을 받는 Step을 다시 열고 사유 및 후속 Step 영향을 기록한다.

## 전환 및 Completion Gate

세부 검토 항목은 [QUALITY-GATES.md](QUALITY-GATES.md)를 따른다.

1. INTAKE 완료: 목적과 검토 범위가 식별되고, 책임자·요구사항의 미확인 항목과 다음 확인 행동이 기록되어 있다.
2. DISCOVERY부터 VALIDATION까지 각 Gate를 순서대로 충족한다.
3. DECISION_GATE 완료: 현재 Step에 필요한 사용자 결정이 증거와 함께 기록되고, 진행을 막는 미결정 항목이 없다. AUTO 결정은 근거를 기록한다.
4. ACCEPTED 진입: 필수 Validation이 통과되고 미해결 차단 사항이 없다. 예외는 정책에 맞는 승인·보완책·재검토 조건을 기록한다.
5. Step 완료: ACCEPTED 또는 근거 있는 NOT_APPLICABLE 상태이며 Evidence Gate가 충족되어야 다음 Step의 설계 확정 작업을 시작한다.
6. Assessment ACCEPTED: 모든 필수 Step의 결정과 검증이 완료되어야 한다. 보고서 작성만으로 완료 처리하지 않는다.

REPORT는 전체 검토 완료 시 두 보고서 템플릿으로 작성한다. 중간 보고 요청은 현재 상태와 미결정 사항을 명시해 작성할 수 있다.

## 식별자 및 증적

- Assessment: `ASM-YYYY-NNN`, 경로 `assessments/ASM-YYYY-NNN-<slug>/README.md`.
- Decision: `DEC-YYYY-NNN`, 원본 경로 `assessments/ASM-YYYY-NNN-<scope>/decisions/DEC-YYYY-NNN-<slug>.md`. 전역 `decisions/README.md`는 원본 색인이다.
- 번호는 저장소 전체의 해당 연도 기존 목록을 확인해 중복 없이 부여한다.
- 결정 문서에 Assessment, 적용 환경·워크로드, Step, 승인 등급, 상태를 기록하고 해당 검토의 결정 목록 및 전역 `decisions/README.md`에 연결한다.
- Decision 상태: `PROPOSED`, `DECIDED`, `REJECTED`, `SUPERSEDED`. 실제 적용 여부는 Implemented Reality에서 별도 관리한다.
- 승인 증거는 사용자 지시의 날짜와 내용을 식별할 수 있게 기록한다. 접근 가능한 티켓·문서 링크가 있으면 연결한다.
- Unknown은 담당자, 확인 방법, 차단 여부를 포함한다. 미지정 담당자를 임의로 배정하지 않는다.

## 검토 시작 및 공통 검토 축

검토별 구조는 [ASSESSMENT-STRUCTURE.md](ASSESSMENT-STRUCTURE.md), 역할 선택과 매 사용 후 평가는 [AGENT-LIFECYCLE.md](AGENT-LIFECYCLE.md)를 따른다.
INTAKE에서 환경·워크로드·연동 범위를 구분하고 DISCOVERY에서 DevOps와 IDC 연동의 적용 여부를 확인한다.
네트워크 상세 검토는 STEP-04, 빌드·런타임·CI/CD 검토는 STEP-05에 포함하고 STEP-09에서 전체 대안을 통합한다.
보안·비용·복구·데이터·배포·연동 제약은 처음부터 수집한다. 해당 번호 Step까지 발견을 미루지 않는다.
모든 역할 작업 종료 시 평가를 남기고 필수 누락은 해당 Completion Gate 전에 보완한다.
