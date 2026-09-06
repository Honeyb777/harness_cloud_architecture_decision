# Quality Gates

Architecture Step을 완료하기 위한 공통 기준이다.

## Discovery Gate

- Requirement가 정리되었다.
- Unknown이 구분되었다.
- 기존 환경이 확인되었다.
- Constraint가 기록되었다.

## Option Gate

- 가능한 대안이 비교되었다.
- 비용/운영/보안/확장성 차이가 기록되었다.
- Recommendation이 근거와 함께 작성되었다.

## Decision Gate

- 사용자가 선택할 항목이 분리되었다.
- Decision이 기록되었다.
- 미결정 항목이 명확하다.

## Target Design Gate

- 실제 채택 Architecture가 문서화되었다.
- 권장안과 다른 부분이 있다면 이유가 기록되었다.
- Ownership과 책임 경계가 정의되었다.

## Validation Gate

최소 다음 축을 검토한다.

- Security
- Cost
- Reliability
- Operability
- Auditability
- Scalability
- Data lifecycle
- Global readiness

## Evidence Gate

- Decision record가 갱신되었다.
- 실제 적용 결과가 존재하면 기록되었다.
- History가 간단히 업데이트되었다.

## 환경·DevOps·IDC 및 역할 평가

- Discovery: 적용 환경·워크로드·연동을 식별하고 DevOps와 네트워크/IDC 검토의 필요 여부 및 제외 근거를 기록한다.
- Options: 적용 대상의 런타임·배포·연동 대안을 비교하고 호환성·운영·비용·보안 차이를 남긴다.
- Target Design: 환경별 빌드/배포 경로, artifact·설정·권한, 통신 매트릭스, 책임 경계, rollout/cutover·rollback 계획이 존재한다.
- Validation: 해당 범위의 빌드 재현성·런타임 호환성·배포/복구와 DNS/TLS·통신·회선 장애 전환 등을 증거로 확인한다. 미수행은 통과로 기록하지 않는다.
- 설계 검토로 한정된 경우 검토 방식·근거·한계 및 실제 적용 전 필수 시험의 담당자와 시점을 명시한다. 실제 시험이 요구사항이면 계획만으로 통과하지 않는다.
- Evidence: 사용한 모든 역할의 평가와 변경·후속 조치가 있고, 필수 누락이 해소되어 있다. 평가·결정·보고서가 해당 Assessment에 연결되어 있다.
- 적용되지 않는 검증은 사유를 기록한다. 단순 문서 보완에 실제 인프라 검증을 요구하지 않는다.
