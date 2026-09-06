# Hosted 기반 전환 — push/pull·네트워크·Key Vault·빌드·정적 버전 전환

기준일: 2026-09-07. 상태: 후속 요구사항 반영 및 대안 비교. 실제 리소스 설정·배포 미수행.
이 문서는 기존 조사에서 Nexus 유지, Ansible 재사용, 다른 포트 JVM 병행을 제안한 부분을 대체한다.

## 1. 사용자 확정 사항

- 기존 내부 GitLab/Jenkins/Nexus/Ansible을 새 파이프라인에서 사용하지 않는다. 실제 기존 시스템 삭제·해지·데이터 폐기는 이번 범위가 아니다.
- GitHub Enterprise Cloud/Actions와 Azure를 사용하며 의존성 캐시는 Actions Cache를 사용한다.
- Key Vault를 도입한다. vault 분리·네트워크·개별 항목/RBAC는 아래 제안이다.
- 추가 모듈은 빌드 runner에서 사용하는 도구·라이브러리다. VM 실행용 컨테이너 전환 요구가 아니다.
- 다른 포트에서 새 JVM을 동시에 기동하지 않는다. 서비스별 기존 포트·단일 JVM의 두 VM rolling으로 검토한다.
- 세션은 JWT Cookie 방식이다. 서버 로컬 로그인 세션 이동은 주요 제약에서 제외한다.
- 사용자는 DB schema/API의 구·신 버전 공존 위험을 감수하는 방향을 제시했다. 장애가 없어지는 것이 아니라 수용 위험으로 기록한다.

## 2. push와 pull의 기준

명령을 누가 시작하는지와 배포 파일을 누가 내려받는지는 서로 다른 축이다.

| 형태 | 명령·파일 흐름 | 인증/네트워크 | 장점 | 비용·운영·실패 처리 |
|---|---|---|---|---|
| 직접 push | Actions runner가 VM에 연결해 파일 복사·명령 실행 | runner→VM SSH/배포 endpoint의 경로와 별도 인증 필요; VNet runner면 사설 ingress로 가능 | 단계별 제어가 직관적 | 원격 접근·host 권한 관리가 필요; 인터넷 공개가 필수인 것은 아님 |
| 명령 push + 파일 pull — 권장 | Actions가 OIDC로 ARM Run Command 호출, VM Agent가 실행, VM MI가 Blob release 다운로드 | VM 배포용 인터넷 ingress 불필요; Agent outbound·Blob 접근 필요 | GitHub 승인·순서 제어와 VM의 passwordless 다운로드 결합 | Azure Agent/원격 상태 polling 필요; runner 취소 후 원격 실행 잔존 대응 |
| 자율 pull | Actions가 승인된 desired manifest 게시, VM의 timer/service가 조회·적용 | VM→Blob/Key Vault 및 상태 저장소 outbound | 외부에서 VM 명령 실행 권한을 줄일 수 있음 | 로컬 poller의 코드·주기·상태·복구를 운영해야 함; 반영 지연과 두 VM 조정 복잡도 |

자율 pull의 timer/service는 GitHub self-hosted runner는 아니지만 직접 관리할 배포 실행 코드다. 별도 중앙 서버 없이도 구현 가능하나 운영 책임이 사라지는 것은 아니다.
두 VM이 같은 desired version을 읽고 동시에 restart하는 구현은 금지한다. VM1 완료 후 VM2에 허가하는 공통 pair 잠금/상태 머신이 필요하다.
선택된 pull/push 형태와 관계없이 승인된 release ID/digest, 대상 서비스·VM 및 이전 버전을 고정하고 잘못된 manifest/경로를 거부한다.

Run Command가 Azure VM Agent로 실행되는 제품 기능은 [Managed Run Command](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/run-command-managed)에 근거한다. 위 세 가지 배포 형태와 추천은 이 Assessment의 설계 비교다.

## 3. Azure/GitHub만 쓰면 외부 허용이 필요 없는가

배포를 위해 VM의 SSH를 인터넷에 열 필요는 없도록 구성할 수 있다. 하지만 GitHub-hosted runner가 사용자 Azure VNet에 자동 포함되지는 않으며, Azure 서비스 인증 성공이 network firewall 통과를 의미하지도 않는다.

| 통신 구간 | 인터넷 inbound 공개 필요 여부 | 필요한 경로·설정 |
|---|---|---|
| GitHub/패키지 registry→빌드 runner 작업 | 사용자 서버를 공개할 필요 없음 | runner가 GitHub, cache, GHCR 및 의존성 원본에 outbound 연결 |
| runner→Entra/ARM | VM 공개 불필요 | OIDC 및 Azure API HTTPS 접근 |
| ARM/Agent→VM 배포 실행 | VM SSH 공개 불필요 | Azure VM Agent/extension 통신과 결과 반환 egress |
| runner→Blob release 업로드 | Blob 공용 endpoint 또는 private 연결 필요 | standard runner이면 허용된 공용 데이터 경로, private-only면 VNet runner 등 필요 |
| VM→Blob/Key Vault | 공용 데이터 endpoint 없이 구성 가능 | Private Endpoint + DNS/route + VM MI/RBAC |
| App Gateway→VM | 배포용 외부 공개와 무관 | 필요한 backend/probe 사설 port만 허용 |
| App Gateway→Key Vault 인증서 | private 경로로 구성 가능 | Gateway UAMI·KV 권한·private DNS/endpoint |
| 사용자→서비스 | 서비스 공개 정책에 따름 | 기존 listener 접근 정책; 배포망과 구분 |

가능한 조합:

- **공용 Blob endpoint에 인증된 접근을 허용하는 안**: standard hosted runner가 OIDC로 Blob upload/ARM 제어. VM ingress는 닫고, Key Vault는 VM/Gateway만 private endpoint로 읽게 할 수 있다. Blob 익명 접근 허용과 공용 endpoint의 Entra 인증 접근은 다르다.
- **Blob/Key Vault 데이터를 모두 private-only로 두는 안**: 배포/publish job은 Azure VNet 연결 GitHub-hosted larger runner에서 실행한다. 빌드 job은 public 의존성/GHCR만 사용하면 standard runner에 둘 수 있다. 두 job 간 artifact 전달, 서로 다른 권한, NSG·private DNS와 허용 egress를 구성한다.

VNet 연결은 larger runner 기능이다. runner 자체로 들어오는 inbound는 필요하지 않으며 필요한 outbound와 사설 목적지 경로를 설정한다. [GitHub VNet runner](https://docs.github.com/en/organizations/managing-organization-settings/about-azure-private-networking-for-github-hosted-runners-in-your-organization)
Private Endpoint를 만드는 것만으로 Blob 공용 접근이 자동 비활성화되는 것은 아니다. 공용 접근 정책은 따로 설정한다. [Storage private endpoints](https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints)
Key Vault의 trusted services 설정은 임의의 Microsoft/GitHub 실행 코드를 전부 허용하는 설정이 아니다. 실제 지원 목록과 network 경로를 확인한다. [Key Vault network security](https://learn.microsoft.com/en-us/azure/key-vault/general/network-security)

추천은 runtime secret을 runner에 주입하지 않고 VM이 Key Vault에서 읽도록 하는 것이다. runner가 secret 값을 읽을 이유가 없으면 Key Vault를 위해 runner용 공용 접근을 추가할 필요도 없다. 초기 secret 등록·rotation 관리자의 접근 경로는 별도로 필요하다.

## 4. Key Vault에 넣을 항목

| 항목 | 유형 | 접근 주체와 사용 방식 |
|---|---|---|
| Application Gateway HTTPS 인증서 | Certificates | Gateway UAMI가 certificate의 backing secret 참조를 읽음 |
| 외부 API key·webhook signing secret | Secrets | 필요한 서비스의 VM MI만 읽기 |
| passwordless 미지원 DB·메일·연계 시스템 자격증명 | Secrets | 런타임 MI로 조회; 지원 시 해당 자격증명 자체를 MI 방식으로 대체 |
| 자체 발급 JWT의 HMAC 서명 secret | Secrets | 실제 발급/검증에 필요한 서비스만; 양 VM의 검증 일관성과 버전 전환 관리 |
| 자체 발급 JWT의 비대칭 서명 private key | Keys 또는 앱 호환 방식 | RSA/EC sign API 사용이 가능한 경우 private key를 내보내지 않는 설계 검토; 앱 변경·성능·권한 필요 |
| 데이터 암호화/키 wrapping key | Keys | 실제 암호화 요구가 있는 서비스만 암호 작업 권한 |
| 앱의 mTLS 등 client certificate | Certificates | 앱의 저장 형식·갱신·키 접근 방식에 맞춰 관리 |

JWT를 외부 IdP가 발급하고 앱이 공개키로 검증만 한다면 앱 vault에 발급 private key를 추가하지 않는다.
공개 검증키/JWKS URL, API URL, port, release ID, feature flag, Azure client/tenant/subscription ID 같은 일반 설정은 Key Vault보다 GitHub vars·배포 설정·필요시 App Configuration에 둔다.
GitHub/Azure OIDC access token, GITHUB_TOKEN, 실제 사용자 JWT/cookie, 캐시 파일, WAR/정적 파일을 Key Vault에 저장하지 않는다. Vue bundle에도 어떤 secret도 포함하지 않는다.

Key Vault는 Secrets/Keys/Certificates를 보호하며 일반 설정 저장소로 쓰지 않는 것이 공식 권장이다. vault는 환경과 애플리케이션/신뢰 경계별로 나누고 MI/RBAC, soft delete·purge protection, 감사 로그·만료 알림·갱신 절차를 구성한다. [Key Vault 보안 권장](https://learn.microsoft.com/en-us/azure/key-vault/general/secure-key-vault)
임의 API secret을 vault에 넣는 것만으로 원본 서비스까지 자동 rotation되지는 않는다. 원본 갱신·새 버전 게시·소비자 갱신·기존 버전 철회 순서를 정의한다.

### 권한 제안

- VM MI: 필요한 vault의 Key Vault Secrets User 또는 작업에 맞는 최소 권한.
- 서명 주체: 필요한 Key의 sign 등 crypto operation만. 일반 secret read와 구분한다.
- Gateway UAMI: 인증서 backing secret 읽기. 앱의 DB/JWT secret을 함께 읽을 필요가 없도록 인증서 vault 경계를 검토한다.
- secret 등록/rotation 담당: 필요한 쓰기 권한, 런타임 reader와 분리.
- GitHub deployer: 기본적으로 runtime secret 읽기 없음. secret 관리 workflow가 필요하면 별도 Identity·Environment로 제한.

실제 역할 범위는 [Key Vault RBAC](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide)에 따라 확인한다. 같은 VM의 MI는 서비스별 완전 격리를 자동 제공하지 않으므로 vault만 나누고 서비스 격리가 완료되었다고 선언하지 않는다.
애플리케이션은 필요한 값을 기동/갱신 시 읽고 적절하게 캐시한다. 모든 사용자 요청마다 Key Vault를 읽도록 만들지 않는다. 버전 갱신과 장애 시 동작은 앱에 명시한다.

### Gateway 인증서와 JWT 회전

App Gateway Key Vault 연동은 v2 조건을 확인한다. 일반적인 연동은 exportable private key를 가진 PFX 인증서와 UAMI, version 없는 backing secret URI를 사용해 갱신을 추적한다. 이는 비추출 서명 Key를 쓰는 JWT 설계와 다르다. 인증서 갱신은 모든 인스턴스에서 즉시 완료된다고 가정하지 않는다. [Gateway Key Vault certificates](https://learn.microsoft.com/en-us/azure/application-gateway/key-vault-certs)
JWT Cookie는 사용자 확인대로 로컬 session migration 문제에서 제외한다. 다만 양 VM이 같은 issuer/audience 및 올바른 서명키 집합을 사용해야 한다. 자체 키를 회전할 때 기존 JWT 만료까지 구 검증키를 유지하고 새 키를 순차 반영한다. 이 확인은 세션 저장소 재설계 요구가 아니다.

## 5. Actions Cache·빌드 runner·GHCR·ACR

추가 모듈은 빌드 runner용으로 확인되었다. 따라서 ACR을 기본안에 추가하지 않는다.

| 선택 | 용도 | 현재 권장 |
|---|---|---|
| GitHub-hosted runner image + setup/install 단계 | JDK/Node와 추가 CLI·OS 도구 설치 | 먼저 사용. 도구 버전·설치 스크립트를 관리 |
| 공통 Docker build image를 GHCR에 게시 | 반복 설치가 길거나 여러 repo의 도구 환경을 고정 | 필요해지면 추가. 승인된 base와 도구 버전, digest, 갱신 책임 필요 |
| ACR | Azure runtime에서 private container image를 MI로 pull, Azure 사설 registry 요구 | 현재는 필요 없음. runtime container 요구 발생 시 재검토 |

`runs-on: ubuntu-...`로 선택하는 runner VM 이미지와 `container.image: ghcr.io/...`의 job 컨테이너 이미지는 다르다. GHCR은 registry이며 거기에 있는 모든 이미지가 GitHub가 제작·보증한 이미지는 아니다. [Hosted runner](https://docs.github.com/en/actions/reference/runners/github-hosted-runners), [Container jobs](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/run-jobs-in-a-container)

추가 모듈이 있다고 반드시 자체 이미지를 만들 필요는 없다. 반복 설치 시간·재현성 때문에 유리하면 Dockerfile에 모듈을 포함한 공통 이미지를 만들어 GHCR에 보관한다.
GitHub Actions에서 GHCR은 package 접근이 허용된 repo의 GITHUB_TOKEN으로 사용할 수 있다. publish는 packages: write, consume은 packages: read를 필요한 job에만 둔다. private image는 container job 시작 전에 받아야 하므로 job의 container credentials와 package 접근을 미리 구성한다. 나중 step의 docker login만으로 job container 초기 pull을 해결할 수 없다. [GHCR 인증](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)

Azure VM에서 private GHCR을 직접 pull하는 것은 GitHub Actions 안의 token 사용과 다르며 Azure MI가 GHCR에 자동 통하지 않는다. VM runtime 이미지가 필요해지면 MI 인증이 가능한 ACR이 자연스러운 후보다. ACR 역할은 registry의 RBAC/ABAC 모드에 맞게 결정한다. [ACR MI 인증](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication-managed-identity)

### 의존성 cache 운영

- npm 다운로드 cache, Maven local repository, Gradle dependency cache 등 도구별 재생성 가능한 항목을 캐시한다.
- OS/architecture·toolchain·lockfile/빌드 파일 hash를 key에 반영한다. 캐시 적중으로 install/의존성 검증을 무조건 생략하지 않는다.
- `npm ci` 등 실제 install 단계는 유지한다. npm cache와 node_modules·OS 설치 패키지는 구분한다.
- cache miss/퇴거 시 공식 dependency 원본으로 다시 받을 수 있어야 한다. cache는 사내 패키지의 유일한 원본이나 package registry가 아니다.
- Nexus에만 존재하는 사내 Maven/npm 패키지가 있으면 GitHub Packages 등 영속 registry로 원본을 이전한다. 캐시만 복사하고 Nexus를 새 파이프라인 의존성에서 제거한 것으로 처리하지 않는다.
- cache에 인증 파일·secret·운영 설정을 넣지 않는다. PR과 publish의 권한·cache 신뢰 경계를 나눈다.

GitHub cache의 범위·퇴거와 데이터 보관 한계는 [Dependency caching](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching)를 따른다.

## 6. 정적 파일의 slot과 포인터

정적 release 두 개를 slot처럼 관리하는 것은 가능하지만 Azure VM에 App Service식 배포 slot/swap이 내장되는 것은 아니다. 이 파일럿은 A/B 두 폴더를 계속 덮어쓰기보다 고유 release 디렉터리와 `current` 포인터를 권장한다.

```text
/srv/service-a/frontend/
  releases/
    r001/index.html
    r002/index.html
  assets/
    r001/...
    r002/...
  current -> releases/r002
```

- Nginx의 HTML 경로는 current를 바라보고, `/assets/<release>/...`는 current와 독립된 고정 경로에서 제공한다.
- 새 파일을 temp/staging에 모두 준비하고 digest·권한을 검사한 뒤 완성된 release로 이동한다.
- 배포 스크립트는 새 symlink를 만든 뒤 같은 파일시스템의 rename으로 current를 교체한다. 링크를 지운 뒤 다시 만드는 중간 공백은 만들지 않는다.
- current 내용만 바꾸는 경우 Nginx 설정을 변경하지 않으면 reload/restart는 보통 필요하지 않다. open_file_cache·disable_symlinks·파일 권한 등 실제 Nginx 설정에 따라 포인터 반영을 시험한다.
- `active-release.json` 같은 일반 파일을 써도 되지만 Nginx가 그 JSON을 자동 읽어서 root를 바꾸지는 않는다. 배포 스크립트가 manifest를 해석해 포인터를 바꾸도록 구현해야 한다.
- 배포 manifest의 release/digest는 '어떤 버전을 배포할지', readiness marker는 '트래픽을 받을지'를 나타낸다. 하나의 파일로 두 역할을 합치지 않는다.

Nginx의 파일 경로·cache 동작은 [NGINX core module](https://nginx.org/en/docs/http/ngx_http_core_module.html)을 기준으로 시험한다. 위 디렉터리 구조와 포인터 절차는 자체 설계 제안이다.

### 두 VM 전환

1. pair 공통 배포 잠금을 잡고 동일 release manifest를 고정한다.
2. 양 VM 모두에 새/구 정적 자산을 해당 URL로 준비하고 검증한다.
3. VM1 current를 전환하고 HTML·JS·CSS를 실제 경로로 검증한다.
4. VM2 current를 같은 release로 전환·검증한다.
5. 성공 상태를 기록한다. 한쪽 실패 시 다음 임의 작업으로 넘어가지 않고 이전 포인터로 복구하거나 승인된 재시도로 일치시킨다.

각 VM의 pointer 변경은 원자적이지만 두 VM 전체의 동시 원자적 전환은 아니다. Blob의 단일 manifest를 바꿔도 각 VM의 관측 시점은 다르다. 기존 브라우저는 이미 내려받은 HTML/JS로 계속 실행될 수 있다.
새·구 자산을 양쪽에서 제공하고 짧은 진입점 버전 혼재를 허용하는 것이 이번 VM 기반 정적 배포의 권장 의미다. fleet 전체의 정확한 순간 전환이 별도 요구라면 라우팅·서비스 제공 방식을 추가 검토해야 한다.
단순 정적 파일 전환에는 JVM slot, 추가 port, Nginx 중지, VM 전체 drain을 도입하지 않는다. 공유 Nginx 설정/VM 변경이 있는 배포는 기존 pair 잠금/검증 절차를 따른다.

## 7. 수용 위험과 추천 결론

DB schema/API 공존 문제를 감수한다는 사용자 지시를 수용한다. 이번 단계에서 expand/contract나 API 하위 호환성 구현을 필수 선행 과제로 강제하지 않는다.
이 선택은 배포 중·직후 기능 오류, 구 frontend의 API 실패, 이전 WAR만으로 복구되지 않는 schema 변경 위험을 포함할 수 있다. 데이터 삭제·임의 오류 허용·모든 향후 schema 변경의 포괄 승인은 아니다. 변경별 실제 영향·복구 가능성을 기록한다.

권장 조합: hosted runner + Actions Cache → 필요 시 GHCR 공통 빌드 이미지 → Blob release → GitHub 승인/공통 잠금 → Run Command 명령 push + VM MI 파일 pull → VM MI의 Key Vault runtime 조회.
Tomcat은 같은 port의 순차 재기동, Vue는 release/current 전환을 사용한다. push/pull 최종 선택·Blob 공용 여부·Key Vault 세부 경계·GHCR 추가 여부는 아직 미결정이다.
