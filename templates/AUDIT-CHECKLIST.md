# Audit / Compliance Checklist

- Assessment ID / Environment IDs:
- 검토일 / 증적 위치:

해당 없음은 사유를 기록하고 검토 완료와 실제 시험 완료를 구분한다.

## Identity
- [ ] Human / workload identity 분리
- [ ] MFA / Conditional Access 검토
- [ ] OIDC/Federation 검토
- [ ] Least privilege
- [ ] Privileged access 기록

## Resource Governance
- [ ] Account/Subscription ownership
- [ ] Required tags
- [ ] Policy enforcement
- [ ] Exception record

## Logging
- [ ] Control plane audit
- [ ] Authentication logs
- [ ] Kubernetes audit 필요성
- [ ] Network/security logs
- [ ] Retention
- [ ] Immutable archive 필요성

## Data
- [ ] Data classification
- [ ] Encryption at rest
- [ ] Encryption in transit
- [ ] Backup
- [ ] Restore test
- [ ] Retention / deletion
- [ ] Residency

## Operations
- [ ] Change traceability
- [ ] Incident evidence
- [ ] Break-glass procedure
- [ ] Key/secret rotation
- [ ] Patch/upgrade ownership

## DevOps / Supply Chain

- [ ] 빌드·배포 권한 분리와 환경별 승인 증거
- [ ] runner 격리·자격증명·OIDC 및 비밀값 노출 검토
- [ ] 의존성·artifact 추적, 취약점·라이선스 검사 필요성
- [ ] 환경 승격·배포 이력·긴급 변경·rollback 증적

## Network / IDC Integration

- [ ] 통신 매트릭스와 방화벽·라우팅 변경 승인
- [ ] 회선·DNS·TLS·사내 CA 및 인증 의존성
- [ ] 외부/폐쇄망 연결과 데이터 이동 통제
- [ ] 장애 전환·cutover·rollback 시험 및 조직별 책임
