# Application Gateway·Tomcat·Vue 무중단 배포 대안

상태: 설계 비교 및 검증 계획. 서비스마다 별도 Tomcat/JVM은 사용자 확인 사항이다.
VM 두 대, 서비스 A/B는 설명용 식별자다. 서비스별 라우팅·부하는 미확인이다. JWT Cookie와 별도 port JVM 병행 제외, DB/API 공존 위험 수용은 [후속 결정](../decisions/DEC-2026-003-hosted-only-scope.md)에 반영했다.

## 1. probe 실패와 connection draining의 차이

| 방법 | 확인된 제품 동작 | 제안 적용 | 남는 문제 |
|---|---|---|---|
| readiness/probe 실패 | unhealthy 대상으로 신규 요청 전달을 중단하고 건강 회복 시 재투입 | 파일/앱 readiness를 조합해 배포 대상 서비스의 신규 요청 제외 | probe 전파 중 요청, 처리 중 요청·세션·장기 연결을 별도 확인 |
| pool에서 명시적으로 제거 + connection draining | 제거 중 backend의 기존 연결을 설정 시간 동안 유지 | 문서화된 connection drain이 필수인 운영 요구에 비교 | Gateway 설정 변경 권한·시간·동시 수정, affinity 예외·timeout |

probe 실패는 pool 구성에서 backend를 제거하는 것과 다르다. 따라서 marker 파일 제거만으로 Azure의 connection draining timeout이 적용된다고 가정하지 않는다. [Probe 동작](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-probe-overview)
명시적 drain에도 gateway-managed affinity 요청의 예외가 있으며, 설정 변경은 drain timeout 이후 연결을 종료할 수 있다. 장기 다운로드·WebSocket/SSE의 수명과 재접속 정책을 확인해야 한다. [Backend HTTP settings](https://learn.microsoft.com/en-us/azure/application-gateway/configuration-http-settings)

파일럿 권장: 파일 기반 방식의 가능성을 dev에서 먼저 시험한다. 기존 요청 무손실을 입증하지 못하면 명시적 pool 제거와 connection drain 대안을 비교한다. 별도 port JVM 병행은 현재 검토 대상에서 제외한다. 증적 없이 '무중단 보장'으로 승인하지 않는다.

## 2. 사용자가 제안한 파일 방식

가능한 설계는 서비스별 `/_deploy/ready` 경로를 두고 아래 조건을 모두 만족할 때만 200을 반환하는 것이다.

```text
ready(service, vm) =
    배포 허용 marker 존재
    AND 해당 서비스의 실제 readiness 통과
    AND 의도한 release의 smoke test/기동 검증 완료
```

- marker가 없으면 probe 경로만 503을 응답한다. 일반 사용자 경로는 기존 Nginx/Tomcat으로 계속 처리한다.
- backend는 marker만 보는 Nginx 정적 200 대신 Tomcat/앱 readiness까지 검사한다. Tomcat 포트가 열렸다는 사실이나 PID 존재만으로 복귀시키지 않는다.
- frontend는 marker와 release manifest/index·필수 자산 검증을 사용한다. 배포 폴더와 marker를 분리해 artifact 압축 해제가 실수로 online 상태를 만들지 않도록 한다.
- probe는 서비스 host/port/path와 정확히 연결하고 성공 status를 200으로 제한한다. 로그인 redirect나 SPA fallback으로 200이 되어서는 안 된다.
- marker 파일 조회가 캐시되어 오래된 상태를 반환하지 않도록 Nginx 설정과 실제 응답을 시험한다.
- marker 변경은 인증된 배포 경로에서만 가능해야 한다. 공개 HTTP on/off API는 만들지 않는다.
- VM 재부팅·배포 실패 시 marker를 자동으로 켜지 않는다. readiness와 배포 상태를 재조정한 뒤 켠다.
- 서비스 A/B의 backend HTTP settings와 custom probe를 분리한다. 동일 port를 쓰는 Nginx에서도 가상 host·path와 App Gateway 연결을 확인한다. 파일 이름만 서비스별로 나눠서는 격리가 보장되지 않는다.

marker 제거 후에는 해당 pool/settings/VM 조합이 Unhealthy로 관측될 때까지 제한 시간 내 polling한다. Backend Health API 호출이 probe 자체를 즉시 갱신하는 것은 아니다. 처리 중 요청이 끝났는지는 별도 앱 지표/로그 및 시험으로 확인한다. 단순 `sleep 30`이나 TCP 연결 수만을 완료 조건으로 삼지 않는다. [Backend health](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-backend-health)

## 3. 공유 서버 쌍의 배포 잠금

### 잠금 범위

권장 key: `availability-domain/<subscription>/<environment>/<vm-pair>`.
서비스명·branch·repo·개별 VM 이름을 기준으로 각각 잠그면 A-vm1과 B-vm2가 동시에 제외될 수 있다.
한 pair에 속한 배포·Nginx 재시작·VM 패치·수동 배포 작업을 같은 통제 범위에 둔다.

서비스별 pool/probe/JVM이 완전히 독립이면 A-vm1과 B-vm2의 제외가 곧 모든 서비스 장애라는 뜻은 아니다. 하지만 공유 Nginx·VM 자원·운영 명령이 영향을 줄 수 있으므로 파일럿은 pair 전체를 직렬화한다. 이후 영향 범위가 증명된 경우에만 서비스별 동시성을 허용한다.

### GitHub concurrency의 범위

`concurrency`는 repository 범위다. 여러 repo에 같은 group 문자열을 넣어도 서로 잠그지 않는다. 중앙 reusable workflow 파일을 호출하는 것만으로 실행·잠금이 중앙 repo로 이동하지도 않는다. [Concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency), [Reusable workflow context](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations)

파일럿은 모든 운영 배포 workflow run이 실제로 중앙 배포 repo에서 시작되는 구조를 권장한다. 아래는 그 repo 안에서 사용하는 제어 부분 예시이며 완성된 배포 workflow가 아니다.

```yaml
concurrency:
  group: prod-shared-vm-pair-01
  cancel-in-progress: false
  queue: max
```

group은 신뢰된 서버 매핑에서 결정하며 요청자가 임의의 key를 입력해 우회할 수 없게 한다. lock은 VM1과 VM2 전체 배포가 끝날 때까지 유지한다.
`queue: max`는 현재 최대 100개 대기를 지원한다. 기본값은 대기 1개이며 새 요청이 이전 pending 요청을 대체한다. 대기열 진입 순서와 release 생성 순서는 다를 수 있으므로 stale release를 실행하지 않도록 manifest의 순서/승인 상태도 확인한다.
`matrix.max-parallel: 1`, Environment 승인, `cancel-in-progress: false`만으로 서로 다른 실행/저장소와 수동 배포가 조정되는 것은 아니다.

### 취소·runner 장애까지 고려한 실제 배포 상태

GitHub job이 끝나거나 취소되어도 Azure에서 이미 시작한 명령이 계속 실행 중일 수 있다. 따라서 concurrency 외에 pair별 영속 배포 상태와 외부 잠금을 두는 안을 제안한다.

- 별도 Blob의 무기한 lease + 상태 기록을 파일럿 대안으로 사용한다. 성공/검증된 rollback 완료 시만 정상 해제하며 불명확한 실패는 HOLD로 남긴다.
- lease는 Blob 쓰기·삭제만 강제한다. VM·Gateway에 대한 배포를 Azure가 자동 차단하는 기능이 아니다. 모든 실행 경로가 같은 잠금/상태를 사용하고 deploy 권한도 통제되어야 한다. [Blob Lease](https://learn.microsoft.com/en-us/rest/api/storageservices/lease-blob)
- 상태에는 deployment ID, release/digest, owner run URL, 대상 서비스·VM, 현재 단계, 변경한 marker/pool, 원격 command ID, heartbeat를 기록한다.
- 상태/잠금이 조회되지 않으면 신규 제외·중지·재배포를 금지한다. 단순 TTL 만료 뒤 새 작업을 시작하지 않는다.
- 유한 lease를 쓰려면 갱신 실패·실행자 교체에서 stale worker의 side effect를 막을 fencing을 실행 경로 전체에 설계해야 한다. Blob lease ID만 추가했다고 VM 명령까지 fencing되는 것은 아니다.
- 무기한 lease의 가용성 비용은 수동 복구다. 운영자가 기존 Run Command 종료와 VM 로컬 작업 종료를 확인하고 건강 상태를 대조한 뒤에만 break/reconcile한다.
- VM 내부에도 중복 실행을 막는 로컬 lock과 deployment ID 검사를 둔다. 로컬 lock은 pair 전역 잠금의 대체물이 아니다.
- `finally`/`if: always()`만 믿고 온라인 복귀·잠금 해제를 실행하지 않는다. 실패한 버전은 제외 상태로 두고 반대 VM을 유지한다.
- 원격 동작은 idempotent한 prepare/drain/activate/verify/rejoin 단계로 나누고, 재실행은 기록된 상태에서 이어간다.

## 4. 두 VM의 rolling 절차

```text
승인된 release 확정
→ pair 잠금 + 미완료 작업 검사
→ 양 VM 및 모든 영향 서비스의 health/capacity 확인
→ artifact 선배치·digest 검증
→ VM1 서비스 제외
→ 제외 관측 + 처리 중 요청 종료 조건 확인
→ VM1 배포/기동/warm-up/직접 smoke test
→ VM1 재투입 + Gateway Healthy + 관측 기간 통과
→ VM2 제외/배포/기동/검증/재투입
→ 양쪽 release·서비스 상태 확인
→ 배포 증적 기록 + 잠금 해제
```

각 drain 직전 `해당 변경 후 모든 영향 서비스의 ready backend 수 >= 1`을 확인한다. 두 서버 중 이미 한 대가 고장이라면 다른 서버를 배포하지 않는다.
1대가 전체 필요한 트래픽을 감당할 수 있어야 한다. 2대 rolling은 배포 중 추가 1대 장애까지 견디는 구조가 아니므로 그 요구가 있으면 임시 3번째 VM/별도 green pool 등을 비교한다.

VM1에서 실패하면 VM2에 진입하지 않는다. 이전 release로 VM1 복구→readiness→재투입을 검증하거나, 실패 대상을 제외하고 HOLD/운영 개입으로 종료한다. Gateway health Unknown도 성공으로 간주하지 않는다.
App Gateway 구성을 직접 변경하는 대안에서는 동일 Gateway를 수정하는 다른 배포/IaC도 직렬화하고 최신 구성을 기준으로 제한적으로 변경한다. 원래 설정·pool 멤버를 기록해 복구하며 실패 시 양 VM 전체 목록을 덮어쓰지 않는다.

## 5. Tomcat 기동 전후

기본안은 VM1의 해당 서비스만 제외·중지하고 새 release를 시작하는 동안 VM2가 계속 처리하는 것이다. 서비스별 JVM이므로 A 배포에 B의 JVM을 재시작할 필요는 없다.
앱 readiness에는 실제 context 초기화·필수 의존성 사용 가능 여부와 예상 release 식별이 포함되어야 한다. JWT 검증, connection pool 및 캐시 warm-up을 확인하고 잠깐 200이 나온 것만으로 재투입하지 않는다.

별도 port의 JVM 병행과 Tomcat parallel deployment는 현재 적용안에서 제외한다. 기존 port의 서비스 JVM을 순차 재기동하고 peer VM이 요청을 처리하도록 한다.
JWT Cookie 방식이므로 로컬 세션 이동을 전제한 공유 세션 저장소 도입은 요구하지 않는다. JWT 발급·검증키/issuer/audience가 양쪽에서 일치하고 key rotation 시 이전 token 검증이 유지되는지는 확인한다.
DB/API N/N-1 공존 위험은 사용자가 수용하는 방향으로 지정했다. 호환성 개선을 강제 선행 조건으로 두지 않고 실제 영향과 rollback 한계를 기록한다. WAR rollback이 DB 변경을 되돌리지는 않는다.

## 6. Vue/Nginx 정적 배포

정적 배포에서는 Node/npm은 빌드 시점 도구이며 산출물은 Nginx가 제공할 수 있다. 실제 SSR 여부는 확인한다.
라이브 폴더를 삭제하거나 실행 중인 `dist`에 파일을 하나씩 덮어쓰지 않는다. release 디렉터리에 완성본을 준비하고 동일 파일시스템에서 원자적 포인터/rename으로 HTML 진입점을 전환한다.

중요한 점은 포인터를 바꾼 서버 밖에서도 자산이 일치해야 한다는 것이다.

```text
1. /assets/<release-id>/... 형태의 주소 또는 고유 hash 주소로 새 자산을 준비
2. VM1과 VM2 모두에 새 자산을 먼저 배치하고 digest·접근 가능성 확인
3. 구버전 자산도 두 VM 모두에서 계속 같은 URL로 제공
4. VM1 index/entry 전환 → 검증 → VM2 index/entry 전환
5. 이전 HTML을 가진 브라우저의 지원 기간과 rollback 기준을 만족한 뒤 자산 정리
```

새 HTML을 VM1에서 받고 JS를 VM2에서 요청해도 성공해야 한다. `current/assets`가 새 release만 가리키면 구 파일을 디스크에 보관해도 예전 URL에서 접근하지 못할 수 있다. 버전별 URL 또는 공통 immutable asset 경로가 필요하다.
HTML과 실행 설정은 `Cache-Control: no-cache` 등 재검증 정책, hash/version 고정 자산은 적절한 장기 cache 정책을 설계한다. SPA fallback이 없는 JS를 HTML 200으로 돌려주지 않도록 한다. service worker가 있다면 별도 갱신·구 캐시 공존을 시험한다.
Vite 역시 구 chunk 삭제에 따른 기존 브라우저 오류를 문서화한다. Vite 사용은 미확인이지만 이 version skew는 이번 정적 배포의 검증 항목이다. [Vite build](https://vite.dev/guide/build), [Vite troubleshooting](https://vite.dev/guide/troubleshooting)

정적 파일 전환만이라면 Nginx 재시작이나 VM 전체 drain 없이도 설계할 수 있다. Nginx 설정 변경·공유 서비스 영향이 있으면 pair 잠금 안에서 처리한다. 프론트/백엔드 API 버전 혼재는 사용자 수용 위험이다. 정적 파일 404 방지와 API 기능 호환성은 별도 결과로 기록한다.
