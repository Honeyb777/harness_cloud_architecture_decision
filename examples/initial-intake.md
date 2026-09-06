# 예시 — 아키텍처 검토 접수

실제 검토나 사용자 입력 대기 건이 아닌 양식 예시다. 기존 초기 접수 문서를 이동했으며 새로운 검토에서 ID와 날짜를 부여한다.

- Assessment ID: ASM-YYYY-NNN
- Status: PROPOSED
- Current Step: STEP-01
- Phase: INTAKE
- Created: <접수일>
- Owner: 미확인

## 확인된 사실

- 사용자는 프로젝트 초안 확인 및 기본 설정을 요청했다.
- 저장소는 클라우드 아키텍처 검토와 의사결정 기록을 위한 문서 하네스다.
- 실제 검토 대상 서비스와 클라우드 환경은 아직 제공되지 않았다.

## 접수 항목

| 항목 | 현재 값 |
|---|---|
| 서비스 / 프로젝트 및 검토 목적 | 미확인 |
| Business / Technical owner, 부서 | 미확인 |
| 신규 구축 / 기존 환경 개선 여부 | 미확인 |
| 현재 Cloud / Account / Subscription / Tenant | 미확인 |
| 환경, 사용자 규모, 대상 지역 | 미확인 |
| 데이터 등급, 규제 및 데이터 거주 제약 | 미확인 |
| 예산, 목표 일정 | 미확인 |
| 가용성, RTO / RPO, 보존 요구사항 | 미확인 |

## Requirements / Current State / Constraints

미확인. 입력을 받으면 [전체 Assessment 템플릿](../templates/ASSESSMENT-TEMPLATE.md)에 따라 확장한다.

## Unknowns

| ID | 확인할 내용 | 담당자 | 확인 방법 | 차단 여부 |
|---|---|---|---|---|
| UNK-001 | 검토 대상, 목적, 현재 환경 | 미지정 | 사용자 입력 / 기존 요구사항 문서 | INTAKE 완료 차단 |
| UNK-002 | 예산, 일정, 보안·데이터·복구 제약 | 미지정 | 대상 확정 후 요구사항 확인 | 관련 설계 확정 차단 |

## Decision Steps

### STEP-01 — Organization / Account / Subscription

- Status: PROPOSED
- Phase: INTAKE
- Completion Gate: 미충족 — 검토 대상과 목적 확인 필요
- Decision records: 없음

STEP-02~STEP-10은 미착수이며 전체 템플릿의 선행 Step 규칙을 적용한다.

## 다음 행동

검토할 서비스, 신규 구축/기존 환경 여부, 현재 클라우드와 우선 해결할 문제를 접수한다.
현재 추천안, 사용자 아키텍처 결정, 실제 적용 결과는 없다.
