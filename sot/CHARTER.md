# Harness Charter

## Mission

회사 또는 개인 환경에서 Cloud Architecture를 설계할 때
기술적 적합성뿐 아니라 운영, 비용, 보안, 감사, 확장성, 복구 가능성까지 함께 검토하고
그 결정 근거를 장기간 재사용 가능한 형태로 축적한다.

## Scope

다음 영역을 기본 검토 범위로 한다.

- Cloud provider 및 account/subscription 구조
- Landing Zone
- Identity / Federation / OIDC
- Kubernetes / Container Platform
- Network / Connectivity / DNS
- Security / Compliance / Audit
- Logging / Monitoring / Observability
- Cost / FinOps / Tagging
- Compute / Autoscaling
- Storage lifecycle
- Database / Backup / Retention / DR
- Multi-region / Global expansion
- CI/CD 및 Infrastructure as Code
- Secrets / Keys / Certificates
- Service alternatives / managed service comparison
- Governance / Policy as Code

## Non-Goals

- 특정 Cloud Provider를 항상 우선하지 않는다.
- 신규 기술을 사용한다는 이유만으로 기존 서비스를 교체하지 않는다.
- 단순한 서비스 가격 비교만으로 Architecture를 결정하지 않는다.
- 모든 Workload에 동일한 표준을 강제하지 않는다.

## Core Principle

`Recommendation != Final Decision != Implemented Reality`

세 값은 각각 독립적으로 기록될 수 있어야 한다.

실제 적용안이 권장안과 다른 경우 반드시 다음을 기록한다.

- 변경된 결정
- 변경 이유
- 제약사항
- 위험
- 보완책
- 재검토 조건 또는 시점

## 반복 검토 및 확장 범위

Assessment는 검토 목적과 승인 범위별로 분리하며 하나 이상의 서비스·환경을 포함할 수 있다.
Cloud뿐 아니라 기존 IDC 유지·연동·이전과 Hybrid 구성을 검토한다.
애플리케이션 빌드·런타임(Java/Tomcat, Kotlin/Tomcat, Vue/npm 등), CI/CD와 artifact·설정·배포·롤백 관리도 아키텍처 범위에 포함한다.
각 전문 역할의 기여와 누락을 사용 후 평가하고 필요에 따라 역할 구성을 개선한다.
