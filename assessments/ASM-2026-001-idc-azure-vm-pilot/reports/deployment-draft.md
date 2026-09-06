# Azure VM 파일럿 배포 초안

- Assessment: ASM-2026-001
- Date: 2026-09-07
- Status: DRAFT — 사용자 설명을 구조화한 배포 절차. Azure/GitHub 적용 및 파일럿 시험은 미수행.
- 적용 대상: Azure Application Gateway 뒤의 이중화 VM, 서비스별 Tomcat/JVM 및 Vue 정적 파일

## 목표 배포 방식

GitHub Actions가 승인·릴리스 선택·서버 쌍 순서를 통제한다. Actions는 GitHub OIDC로 Azure에 단기 인증하고 Azure Managed Run Command로 VM에 명령을 전달한다. 각 VM은 Managed Identity로 승인된 release 파일을 Blob에서 내려받아 적용한다.

이는 **명령 push + 파일 pull** 방식이다. VM의 SSH나 별도 배포 endpoint를 인터넷에 공개하지 않는 것을 목표로 한다. VM의 Azure Agent, Blob, Key Vault에 대한 필요한 outbound 경로와 권한은 별도 구성·시험한다.

## 사전 구성 초안

| 영역 | 초안 |
|---|---|
| 소스·CI | GitHub Enterprise Cloud repository와 GitHub-hosted runner |
| 인증 | GitHub OIDC → Azure federated credential. 장기 Azure 비밀번호·서비스 계정 키 미사용 |
| 승인 | GitHub Environment의 production 승인, branch 제한, 배포 저장소 권한 분리 |
| 배포 파일 | Blob Storage에 release ID와 digest를 포함하여 보관 |
| VM 실행 | Azure Managed Run Command와 VM Managed Identity |
| 비밀값 | Key Vault. VM 런타임 소비자와 Gateway 인증서 소비자의 최소 권한 분리 |
| 의존성 | Actions Cache. 캐시는 원본 registry를 대체하지 않음 |
| 추가 빌드 도구 | hosted runner 설치를 우선 사용하고 필요할 때 GHCR 공통 빌드 이미지 검토 |
| 이미지 registry | 현재 ACR 미도입. Azure VM이 private runtime container를 pull하는 요구가 생기면 재검토 |
| 동시 배포 제어 | 동일 서버 쌍의 모든 서비스에 적용되는 공통 pair 잠금과 영속 상태 |

기존 GitLab, Jenkins, Nexus, Ansible은 새 파이프라인의 구성요소로 사용하지 않는다. Nexus에만 존재하는 사내 Maven/npm 패키지는 cache가 아닌 영구 package registry로 이전해야 한다.

## 배포 절차

### 1. 빌드와 릴리스 생성

1. 보호 branch의 merge 또는 허용된 tag가 workflow를 시작한다.
2. GitHub-hosted runner가 JDK·Node 및 추가 도구를 준비한다. npm/Maven/Gradle의 재생성 가능한 다운로드는 Actions Cache를 사용한다.
3. backend는 배포 패키지, frontend는 정적 build 결과를 생성하고 테스트한다.
4. release ID, Git commit SHA, 파일 digest, 대상 서비스 및 rollback 후보를 포함한 immutable release manifest를 생성한다.
5. 배포 파일을 Blob에 업로드한다. Actions Artifacts는 build-to-deploy 전달, 테스트 결과, SBOM, 진단 로그처럼 단기 workflow 산출물에 사용한다.

### 2. 운영 승인과 Azure 인증

1. production job은 GitHub Environment에 진입하면서 지정된 승인자의 승인을 기다린다.
2. 승인 후 Actions가 OIDC ID token으로 Azure에 로그인한다.
3. Azure는 repository, branch 또는 tag, Environment가 일치하는 federated credential만 허용한다.
4. 배포 job은 release manifest와 대상 VM 쌍을 고정하고, 실행 중 다른 commit이나 최신 파일로 바뀌지 않게 한다.

승인자는 GitHub Environment의 배포 승인 권한을 가지며, Azure의 런타임 secret 읽기 권한은 기본적으로 갖지 않는다. Azure 권한은 Run Command와 필요한 Application Gateway 상태 조회 등 최소 작업으로 제한한다.

### 3. 서버 쌍 잠금과 VM 1 배포

1. 대상 서비스가 속한 **서버 쌍 전체**의 배포 잠금을 획득한다.
2. 배포 상태에서 VM 2가 정상이며 한 대만으로도 수용 가능한지 확인한다.
3. VM 1의 readiness marker를 변경해 Application Gateway probe가 신규 요청을 보내지 않게 한다.
4. Application Gateway backend health와 진행 중 요청의 종료 기준을 확인한다. probe unhealthy와 connection draining은 같은 기능으로 간주하지 않는다.
5. Actions가 Run Command를 호출한다.
6. VM 1의 Managed Identity가 Blob에서 고정된 release ID와 digest의 파일만 다운로드한다.

### 4. 서비스별 적용

**Tomcat 서비스**는 기존 포트에서 기존 JVM을 중지·교체·재기동한다. 다른 포트의 새 JVM을 먼저 올리는 방식은 사용하지 않는다. 기동 뒤 실제 readiness endpoint, JVM 상태, 핵심 API와 로그를 확인한다.

**Vue 정적 파일**은 고유 release 디렉터리에 먼저 완전히 배치하고 digest·권한을 검사한다. HTML의 `current` 심볼릭 링크는 임시 링크를 만든 뒤 같은 파일시스템의 rename으로 바꾼다. JS·CSS 등 자산은 release ID가 포함된 고정 URL에서 제공하고, 구버전 자산을 즉시 삭제하지 않는다.

```text
/srv/<service>/frontend/
  releases/<release-id>/index.html
  assets/<release-id>/...
  current -> releases/<release-id>
```

`current` 변경은 VM 한 대 안에서는 원자적으로 만들 수 있지만 두 VM 전체를 같은 순간에 전환하지는 못한다. 새·구 자산을 양 VM에 먼저 준비하고 VM별로 전환한다. 릴리스 manifest는 목표 버전을 정하고 readiness marker는 트래픽 수용 상태를 나타내므로 별도로 관리한다.

### 5. 검증·재편입·VM 2 배포

1. VM 1에서 readiness, backend 핵심 API, frontend HTML·JS·CSS 경로를 확인한다.
2. 실패하면 VM 2에는 진행하지 않는다. 이전 WAR 또는 이전 `current` 포인터로 복구하거나 상태를 HOLD로 남겨 운영자가 판단한다.
3. 성공하면 VM 1이 정상 probe 상태로 복귀했는지 확인한다.
4. 동일한 고정 release manifest를 사용해 VM 2에 3~5단계를 반복한다.
5. 두 VM이 정상 상태가 되면 잠금을 해제하고 배포 결과를 기록한다.

## 비밀값과 세션 처리

Key Vault에는 Application Gateway TLS 인증서, 외부 API key, passwordless를 지원하지 않는 연계 자격증명, 필요한 JWT 서명 키 등을 둔다. 일반 URL·포트·release ID·feature flag, 배포 파일, 캐시, GitHub OIDC token, 사용자 JWT Cookie는 저장하지 않는다.

JWT Cookie 방식이므로 서버 로컬 session migration은 필요하지 않다. 다만 양 VM의 issuer, audience 및 검증 키 집합은 같아야 한다. 키 교체 시에는 기존 토큰이 만료될 때까지 구 검증 키를 유지하는 순서가 필요하다.

## 실패 처리 기준 초안

| 상황 | 처리 |
|---|---|
| 빌드·테스트 실패 | Blob release를 승격하지 않고 workflow 실패 |
| 승인 거부 또는 만료 | 배포를 시작하지 않음 |
| VM 1 drain·배포·검증 실패 | VM 2 배포 금지, 이전 버전 복구 또는 HOLD |
| Actions job 취소 | 원격 Run Command가 계속될 수 있으므로 실제 VM 상태를 조회한 뒤 HOLD 또는 복구 판단 |
| VM 1 정상 복귀 실패 | pair 잠금 유지, 수동 확인 전 다음 서비스 배포 금지 |
| VM 2 실패 | VM 1의 버전과 서비스 영향 확인 후 승인된 복구 절차 실행 |

DB schema와 API 구·신 버전 공존 위험은 사용자 수용 범위로 기록되어 있다. 이는 데이터 삭제나 모든 변경의 무조건적 승인을 뜻하지 않는다. 각 릴리스에서 영향·복구 가능성·관측 결과를 남긴다.

## 착수 전에 확정할 항목

1. GitHub 조직·배포 repository와 Azure tenant, subscription, resource group, dev/prod VM 쌍의 실제 매핑
2. Blob 공개 인증 접근 또는 VNet 연결 GitHub-hosted larger runner 중 네트워크 방식
3. Key Vault 분리 단위, Managed Identity, OIDC subject와 Azure RBAC 최소 권한
4. Application Gateway의 backend pool, HTTP setting, probe, affinity, 장기 연결 및 drain 수용 기준
5. 서버 한 대의 용량 기준, pair 잠금의 저장 위치·만료·복구 책임, 수동 배포 통제 방식
6. Blob 보존 버전 수, rollback 보존 기간, frontend 구버전 자산 유지 기간

## 검증 상태

이 문서는 설계 초안이다. GitHub Environment 승인, OIDC subject 거부, Azure RBAC, private DNS·방화벽, Run Command, Blob MI download, Application Gateway drain, Tomcat 재기동, static pointer 전환, 취소·잠금·복구는 실제 파일럿에서 검증해야 한다.

상세 근거와 검증 항목은 [설계 비교](../designs/hosted-only-push-pull-key-vault-static.md), [공식 출처](../evidence/official-sources.md), [검증 계획](../evidence/pilot-validation-plan.md)을 따른다.
