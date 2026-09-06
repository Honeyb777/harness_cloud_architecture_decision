# Reporting Standard

모든 주요 Architecture 검토는 두 가지 보고서 형태를 지원한다.

## Executive One-Page

A4 1페이지 수준을 목표로 한다.

필수 내용:
- 목적
- 현황/문제
- 결정사항
- 핵심 Architecture
- 주요 비용 영향
- 주요 Risk
- 사용자/경영진 결정 필요사항
- Next Step

세부 기술 구현은 제외한다.

## Full Architecture Report

페이지 제한 없음.

포함 가능 항목:
- Executive Summary
- Requirements
- Constraints
- Current State
- Target Architecture
- Alternatives
- Decision Records
- Identity
- Network
- Kubernetes
- Security
- Audit
- Cost / FinOps
- Data / Backup / Lifecycle
- Reliability / DR
- Global Strategy
- Operations
- IaC / Delivery
- Risks
- Exceptions
- Roadmap
- Appendices

## 환경별 산출물

공통 양식은 `reports/`에 유지하고 완성된 보고서는 `assessments/<assessment>/reports/`에 저장한다.
두 보고서 모두 Assessment ID와 적용 서비스·환경·검토 상태를 명시한다.
상세 보고서는 스택/런타임 호환성, CI/CD·배포·롤백, 네트워크/IDC 의존성·연결·이전, 역할 평가에 따른 보완 및 미검증 사항을 포함한다. 해당 없음은 사유를 남긴다.
