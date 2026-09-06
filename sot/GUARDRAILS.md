# Architecture Guardrails

## 1. Decision Integrity

모든 주요 결정은 Requirement와 Constraint를 근거로 한다.
근거 없는 선호 또는 특정 Vendor 편향만으로 결정하지 않는다.

## 2. Alternative Requirement

중요 서비스 선택 시 가능한 경우 최소 2개 이상의 대안을 검토한다.

비교 예:
- Azure native vs AWS native
- Managed service vs self-managed
- Kubernetes vs serverless/container service
- Relational DB vs NoSQL
- Regional vs multi-region
- Hot-only storage vs lifecycle tiering

## 3. Cost

비용 검토에는 가능하면 다음을 포함한다.

- Baseline cost
- Growth driver
- Data transfer / egress
- Backup
- Log retention
- Storage tier
- Reserved / Savings / Commitment 가능성
- Idle resource
- Non-production scheduling
- 비용 귀속 태그

## 4. Security

- Long-lived credential 사용을 최소화한다.
- Workload Identity/OIDC/Federation 방식을 우선 검토한다.
- Least privilege를 기본으로 한다.
- Human access와 Workload access를 구분한다.
- Break-glass 권한을 별도 관리한다.

## 5. Auditability

주요 변경과 결정은 추적 가능해야 한다.

최소 추적 대상:
- 누가 결정했는가
- 무엇을 결정했는가
- 어떤 대안을 검토했는가
- 왜 해당 대안을 선택했는가
- 실제 적용은 무엇인가
- 예외는 무엇인가

## 6. Global Readiness

현재 Single Region이어도 다음을 검토한다.

- Region dependency
- Data residency
- DNS / Traffic routing
- Replication
- RTO / RPO
- Stateful workload migration
- Cross-region cost
- Global identity dependency

## 7. Data Lifecycle

Storage와 Database는 생성 시점부터 다음을 검토한다.

- Backup
- Retention
- Archive
- Restore test
- Hot → Cool/IA → Archive 전환
- Legal/Audit retention
- Deletion policy

## 8. Change Budget

현재 Decision Scope에 필요한 최소 변경을 우선한다.
추가 개선사항은 FOLLOW-UP / TECH-DEBT / OUT-OF-SCOPE로 분리한다.
