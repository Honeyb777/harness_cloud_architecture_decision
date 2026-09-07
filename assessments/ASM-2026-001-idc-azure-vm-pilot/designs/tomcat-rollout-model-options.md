# Tomcat rollout — same-port drain 후 교체

- Assessment: ASM-2026-001
- Step / Phase: STEP-01 / DISCOVERY; STEP-05 준비 조사
- Status: 현재 사용자 선택 반영. 실제 probe·Tomcat·Gateway 설정/시험은 미수행

별도 port에서 새 JVM을 병행하는 방식은 현재 파일럿 범위에서 제외한다. 각 VM은 기존 Tomcat을 중지한 뒤 같은 port에서 승인된 release를 기동한다.

```text
VM1
  marker off → App Gateway probe Unhealthy 관측 → 기존 요청 종료 기준 확인
  → 기존 Tomcat stop → WAR/config 교체 → 같은 port에서 Tomcat start/warm-up
  → application readiness + expected release 확인 → marker on → Healthy 관측
  → VM2에 같은 순서 반복
```

| 단계 | 완료 기준 |
|---|---|
| 신규 요청 제외 | `/_deploy/ready`가 marker, application readiness, expected release를 모두 통과할 때만 200. marker off 뒤 Backend Health Unhealthy 확인 |
| 기존 요청 처리 | probe Unhealthy만으로 connection draining이 적용되지는 않음. 합의된 요청 종료·재접속 기준을 앱 지표·로그로 확인 |
| Tomcat 교체 | 기존 JVM stop 뒤 WAR/config를 바꾸고 동일 port에서 새 JVM start. 구/신 JVM을 병행하지 않음 |
| readiness | context 초기화, 필요한 의존성, JWT key/issuer/audience, 핵심 smoke test, 예상 release ID 확인 |
| 재투입 | marker on 뒤 Gateway Healthy와 안정 관측 기간 확인 |
| 실패 | marker off 유지, VM2 배포 금지. 이전 WAR 재기동 또는 HOLD 후 운영 판단 |

한 VM이 교체 중 해당 서비스 요청을 처리하지 않으므로 peer VM의 용량, drain deadline, warm-up 시간을 파일럿에서 검증한다. shared pair concurrency, persistent pair lock, release digest 고정은 이 절차의 선행 조건이다.

## 근거

- [Application Gateway health probes](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-probe-overview)
- [Application Gateway custom probe](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-create-probe-portal)
- [현재 사용자 결정](../decisions/DEC-2026-003-hosted-only-scope.md)
