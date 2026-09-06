# Tagging Standard

비용 귀속, 운영 책임, 감사, 자동화에 활용할 수 있도록 공통 태그를 정의한다.

## Required Tags

| Tag | 목적 | 예시 |
|---|---|---|
| owner | 운영 책임자/팀 | platform |
| department | 비용 귀속 부서 | commerce |
| service | 서비스명 | checkout |
| environment | 환경 | prod |
| cost-center | 비용센터 | CC-1024 |
| workload | Workload 구분 | api |
| data-classification | 데이터 등급 | internal |
| criticality | 중요도 | tier-1 |
| managed-by | 관리 방식 | terraform |
| lifecycle | 자원 수명 | persistent |
| region-scope | 지역 전략 | global |

## Optional Tags

- project
- product
- tenant
- customer
- compliance
- backup-policy
- retention-policy
- expiration-date

## Rules

- Cloud provider별 제한을 고려해 실제 Key 표준은 프로젝트 단계에서 확정한다.
- 비용 분석에 사용할 Tag는 Billing/Cost Management에서 실제 활성화 가능한지 검증한다.
- Tag inheritance 기능만 믿지 않고, 실제 비용 귀속 가능성을 검증한다.
- 개인정보/Secret을 Tag 값으로 저장하지 않는다.
