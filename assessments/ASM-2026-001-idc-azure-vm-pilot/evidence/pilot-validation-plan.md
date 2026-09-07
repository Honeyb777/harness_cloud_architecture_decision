# 파일럿 검증 계획

Status: NOT_RUN. 공식 문서 확인과 문서 정합성 검사는 실제 환경 시험을 대체하지 않는다.
각 시험의 실행자·시점·run URL·Azure operation/command ID·결과/로그 링크를 기록해야 한다.
운영 트래픽 시험 전 dev/파일럿에서 합의된 부하와 오류 주입 범위를 정한다.

| ID | 시험 | 통과 조건 |
|---|---|---|
| VAL-01 | 올바른 repo/environment OIDC | 기대한 Identity·tenant로만 인증, 불필요한 권한 없음 |
| VAL-02 | 다른 repo/branch/Environment/issuer 및 비승인 job | prod 인증·실행 거부, token 본문 로그 없음 |
| VAL-03 | 승인·거절·self-review·bypass | 지정 운영자만 승인, 정책대로 거절/자기 승인 차단 |
| VAL-04 | Run Command 최소 RBAC | 지정 VM에서만 실행, 다른 VM·role 변경 불가; 실제 실행 exitCode 확인 |
| VAL-05 | VM MI Blob 접근 | 지정 release 읽기 가능, 쓰기·삭제·불필요한 데이터 접근 거부 |
| VAL-06 | hosted runner 네트워크 | 의존성 원본/Cache/GHCR/Blob 및 Entra/ARM·GitHub의 필요한 경로만 동작 |
| VAL-07 | VM Agent 정상·장애/egress 차단 | 정상은 결과 확인, 장애/Unknown은 신규 drain 차단 |
| VAL-08 | marker 제거·probe 반영 시간 | probe만 503, 일반 경로 계속 처리; 신규 요청 제외 시점 기록 |
| VAL-09 | 장시간 HTTP/WebSocket/SSE 및 affinity | 합의한 완료·재접속 조건 충족; probe 방식과 명시적 drain 비교 |
| VAL-10 | Tomcat 느린 기동·기동 실패 | 준비 전 재투입 없음, peer 유지, VM2 배포 금지 |
| VAL-11 | VM1 재투입 후 불안정 | 안정 관측 실패 시 다음 서버 진행 금지 |
| VAL-12 | JWT 검증·DB migration·배치 영향 관측 | 양 VM의 JWT 검증/키 회전 확인; DB/API 공존은 수용 위험으로 영향·복구 한계 기록 |
| VAL-13 | 두 서비스 동시 배포 요청, 같은/다른 repo | 동일 서버 쌍에서 동시에 drain/중지하지 않음 |
| VAL-14 | 수동 배포·VM 패치·Gateway 변경 경합 | 공통 잠금/권한/절차로 우회 방지 |
| VAL-15 | runner 강제 종료·workflow 취소·timeout | 원격 명령이 남아도 다른 배포가 진행하지 않음; HOLD 증거 |
| VAL-16 | lease/상태 저장소 장애·stale worker | 새 파괴적 단계 금지, 이전 실행 정지 확인 전 인계 금지 |
| VAL-17 | Vue 양 서버에 새/구 자산 혼재 요청 | 어느 서버에서 HTML/JS를 받아도 chunk 404·HTML fallback 오응답 없음 |
| VAL-18 | 오래 열린 탭·lazy chunk·cache/service worker | 합의한 지원 기간 안에 이전 화면·자산이 정상 동작 |
| VAL-19 | frontend/backend N/N-1 조합 | 기능 오류·rollback 한계를 관측해 사용자 수용 위험에 연결; 호환 재설계를 강제하지 않음 |
| VAL-20 | 버전 정리 중 배포·승격·rollback 경합 | current/in-flight/pinned/호환 자산 삭제 없음 |
| VAL-21 | 한 VM으로 전체 부하·peer 장애 상태 | 용량 기준 충족, peer 이미 장애면 배포 시작 금지 |
| VAL-22 | 잘못된 release/digest·경로·중복 command | 실행 거부 또는 idempotent 재개, 임의 script/artifact 실행 금지 |
| VAL-23 | 잠금 강제 복구·run 재실행 | 미종료 remote/local 작업과 실제 상태 확인 후만 복구 |
| VAL-24 | 감사 증적 및 rollback 복원 | 승인·commit·digest·대상·명령·결과·복구 이력 추적 가능 |

## 승인 전 수치화할 항목

- 허용 HTTP 오류·p95/p99 지연, 세션 영향, WebSocket 재접속 기준.
- probe interval/timeout/threshold, 최대 요청 시간, drain deadline.
- Tomcat boot/warm-up deadline, 재투입 후 안정 관측 기간.
- 한 대 수용 용량, 배포 중 추가 장애 요구, 복구 목표.
- N개 버전의 N, rollback 보호 기간, frontend asset 호환 지원 기간, 감사 보존 기간.

## 종료 기준

모든 필수 시험 통과 또는 정책에 맞는 예외 승인·보완·재검토 조건이 있어야 운영 적용 판단을 한다.
이번 단계에서는 어떤 실제 배포 시험도 통과로 표시하지 않는다.

## 후속 조건의 추가 시험 — 모두 NOT_RUN

| ID | 시험 | 확인 기준 |
|---|---|---|
| VAL-25 | 기존 DevOps 의존성 제거·cache cold start | GitLab/Jenkins/Nexus/Ansible 접근 없이 원본에서 install/build 가능; 사내 패키지 원본 이전 확인 |
| VAL-26 | GHCR 공통 이미지가 필요한 경우 | 허용 job token만 pull/push, digest 고정, job 시작 전 private image 인증 |
| VAL-27 | Key Vault runtime 접근 | VM/Gateway 필요한 항목만 사용, build/deploy job의 불필요한 secret read 없음 |
| VAL-28 | private Blob/Key Vault와 standard/VNet runner | 선택한 경로만 성공; OIDC만으로 private endpoint 접근된다고 오판하지 않음 |
| VAL-29 | JWT 키/인증서 rotation | 이전 JWT 검증 기간과 새키 반영 일관성; 인증서 갱신·권한 장애 관측 |
| VAL-30 | 정적 current 교체·재실행·한쪽 실패 | VM 내 공백 없는 pointer 전환, 양 VM 자산 접근, 불일치 검출/복구 |
| VAL-31 | 자율 pull을 선택할 경우 | 양 VM 독립 restart 방지, 승인 manifest만 적용, poller 중복/실패/재개 검증 |

## CI 시간·저장소 측정 — NOT_RUN

| ID | 시험 | 기록할 값 |
|---|---|---|
| VAL-32 | private GHCR 또는 ACR build image pull | image digest·크기, pull 경과 시간, runner OS/region, 실패·재시도 |
| VAL-33 | Actions Cache hit/miss와 GitHub Packages dependency fetch | cache restore/save, install, build/test 시간을 구분하고 lockfile key·용량 기록 |
| VAL-34 | Artifacts·Blob release 처리 | upload/download/publish 시간, 파일 크기, retention·보호 상태, runner 실행 분 |

| VAL-35 | CD concurrency와 Environment 대기 | 같은 pair group의 `queue: max` 순차 실행, pending/approval/runner-step lock wait별 job execution·billable minutes 비교 |
| VAL-36 | 교차 repository·수동 배포 경합 | 중앙 deployment repository와 pair lock을 우회한 동시 VM drain·Run Command가 차단 또는 HOLD되는지 확인 |
| VAL-37 | Azure OIDC 허용·거부 | 올바른 prod repo/Environment만 해당 deployer identity로 login, 다른 repo/branch/Environment는 거부 |

VAL-35~37은 실제 repository·Environment·Azure scope가 정해진 뒤 실행한다. 문서의 concurrency/OIDC 구조는 실제 실행·과금·RBAC 증적을 대체하지 않는다.

VAL-32~34는 registry 또는 cache가 runner 시간을 없앤다는 가정을 검증하기 위한 비용·운영 측정이다. 결과를 받기 전에는 GHCR, ACR, cache 또는 Artifacts의 비용 우위를 확정하지 않는다.

VAL-12/19는 DB/API 공존에 대한 사용자 위험 수용을 반영한 관측 항목이다. 실제 시험 결과나 수용 범위가 없는 오류를 통과로 기록하지 않는다.
## Azure Pipelines CD 선택 시 추가 검증 — 모두 NOT_RUN

| ID | 시나리오 | 통과 조건 |
|---|---|---|
| VAL-38 | 서로 다른 source service의 Azure Pipelines CD 요청 | 같은 Azure DevOps Environment exclusive lock과 `sequential` 설정에서 한 deployment stage만 실행되고 순서가 기록됨 |
| VAL-39 | approval 거절·timeout·lock 대기·Pipeline 취소 | VM drain, Run Command, VM 2 배포가 시작되지 않거나 중단/HOLD 상태가 추적됨 |
| VAL-40 | Pipeline이 VM 1 배포 중 drain/readiness 실패 | VM 2 stage가 실행되지 않고 같은 release의 rollback 또는 HOLD 증적이 남음 |
| VAL-41 | Azure Pipelines 무료 구간의 대기·실행 시간 | approval/lock wait, agentless validation, 실제 deployment job 시간을 분리해 1,800분 및 병렬 job 사용량과 billing 화면에 기록 |

Azure Pipelines exclusive lock은 Azure DevOps Environment를 사용하는 Pipeline stage의 실행 순서를 제어하는 기능이다. Application Gateway connection draining, Nginx readiness marker, VM 외부 수동 변경과 잔류 Run Command를 대체하는 검증으로 해석하지 않는다.
