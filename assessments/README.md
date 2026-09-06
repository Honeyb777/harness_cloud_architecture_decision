# Assessments

서로 다른 검토 대상은 별도 Assessment로 관리한다. 한 검토가 여러 서비스·환경 및 Cloud/IDC 연동을 포함할 수 있다.
구조는 [검토 건별 디렉터리 기준](../sot/ASSESSMENT-STRUCTURE.md), 상태는 [Workflow](../sot/WORKFLOW.md)를 따른다.

## 검토 목록

| ID | 목적 / 범위 | 포함 환경 | Status | 현재 Step / Phase | 문서 |
|---|---|---|---|---|---|

현재 실제 검토 건은 없다. 대상 미정으로 만들었던 초기 접수 문서는 [예시](../examples/initial-intake.md)로 옮겼다.

## 새 검토 시작

1. 대상과 목적이 확인되면 사용하지 않은 `ASM-YYYY-NNN`을 부여하고 `ASM-YYYY-NNN-<scope>/`를 만든다.
2. [Assessment 템플릿](../templates/ASSESSMENT-TEMPLATE.md)을 `README.md`, [Context 템플릿](../templates/CONTEXT-TEMPLATE.md)을 `context.md`로 복사한다.
3. 요구사항은 `requirements.md`에 기록하고 README에서 참조한다. 검토에 필요한 산출물 디렉터리를 생성한다.
4. 환경·워크로드·연동 범위와 초기 상태를 기록하고 위 목록에 등록한다.
5. 결정·보고서·이력·역할 평가는 해당 검토 안에 보관하고 전역 목록에는 원본 링크를 남긴다.

기존 검토의 연속 작업이면 동일 Assessment를 갱신한다. 독립된 검토라면 새로 생성하고 관련 검토를 연결한다.
