# CI/CD 단계와 빌드 이미지 선택안

- Assessment: ASM-2026-001
- Step / Phase: STEP-01 / DISCOVERY
- Status: PROPOSED — 실행 workflow·registry·Azure 리소스는 아직 만들지 않음
- 확인일: 2026-09-07

## 먼저 구분할 세 가지

GitHub Actions workflow는 repository의 `.github/workflows/*.yml`에 버전 관리되는 실행 정의다. push, pull request, tag, 수동 실행 등의 이벤트가 workflow를 시작하고, 각 job은 `runs-on`으로 지정한 runner에서 실행된다.

| 항목 | 무엇인가 | 이번 파일럿에서 필요한가 | 보관 또는 제공 위치 |
|---|---|---|---|
| GitHub-hosted runner image | GitHub가 매 job에 제공하는 일회성 VM의 OS·기본 도구 이미지. `runs-on: ubuntu-latest` 등이 선택함 | **필수**. build가 실행될 기반 | GitHub가 제공·관리 |
| job container / build image | runner 안에서 build job을 실행할 Docker image. `jobs.<job>.container.image`로 선택 | **필수 요구사항**. JDK·Node·추가 CLI 버전을 고정 | GHCR 또는 ACR |
| ACR | Azure의 private OCI container registry. runner image도 workflow도 아님 | **현재 기본안에서는 불필요** | Azure Container Registry |
| GHCR | GitHub Packages의 OCI container registry. runner image도 workflow도 아님 | 선택. GitHub 중심 공통 build image에 적합 | GitHub Container Registry |

즉, GitHub-hosted runner image는 CI를 실행하려면 항상 사용한다. 이번 파일럿에서는 별도의 build container image도 사용하며, 그 이미지를 GHCR 또는 ACR 중 하나에 저장한다. ACR은 runner image 자체가 아니다.

```text
source repository의 workflow YAML
        │ 이벤트 (PR / merge / tag / manual)
        ▼
GitHub-hosted runner image ── 선택 사항 ──> job container image (GHCR 또는 ACR)
        │ JDK·Node·추가 도구, build/test
        ▼
WAR/JAR + Vue 정적 결과 + manifest
        │
        ├─ Actions Artifacts: job 간 전달, 테스트·SBOM·진단 증적
        └─ Blob: 승인된 VM 배포 release 원본
```

## 권장 CI/CD 흐름

| 단계 | GitHub Actions가 하는 일 | 결과 | Azure 권한 필요 여부 |
|---|---|---|---|
| 1. 변경 접수 | PR, 보호 branch merge, 허용 tag 또는 `workflow_dispatch`로 workflow 시작 | 실행 run ID·commit SHA | 없음 |
| 2. CI 실행 환경 준비 | GitHub-hosted runner를 시작하고 JDK/Node와 필요한 build CLI를 준비. Actions Cache는 재생성 가능한 다운로드만 재사용 | 재현 가능한 build 환경 | 없음 |
| 3. Build·test | Maven/Gradle과 npm의 실제 lockfile/wrapper 기준으로 build·단위/통합 검사를 수행 | WAR/JAR, Vue 정적 bundle, 테스트 결과 | 없음 |
| 4. CI 증적 보관 | 테스트 보고서, SBOM, 로그 요약, release 후보 manifest를 Actions Artifacts에 보관 | 검토 가능한 단기 증적 | 없음 |
| 5. Release publish | 승인된 source·digest로 release manifest를 만들고 Blob의 고유 release 경로에 게시 | 변경 불가능한 release ID·digest | publisher workload identity의 Blob upload 범위만 |
| 6. Dev CD | dev Environment의 job이 같은 release ID를 선택하고 dev VM 쌍에 순차 적용·검증 | dev 배포 결과 | dev deployer workload identity의 최소 권한 |
| 7. Prod 승인 | `prod` Environment에서 release ID·digest·시험 결과·rollback 후보를 확인한 뒤 승인 | 승인 또는 거절 기록 | 승인자는 Azure 권한 불필요 |
| 8. Prod CD | prod job이 OIDC로 Azure에 로그인하고 Run Command·pair 잠금·VM MI Blob pull을 통해 순차 배포 | 감사 가능한 prod 배포·복구 결과 | prod deployer workload identity의 최소 권한 |

CI의 build/test job에는 VM Run Command, App Gateway 변경, production Blob publish 권한을 주지 않는다. dev/prod CD job만 OIDC와 필요한 Azure RBAC를 사용한다.

## build image 선택

| 선택지 | 방식 | 장점 | 제약 | 파일럿 판단 |
|---|---|---|---|---|
| A. GHCR 공통 build image | Dockerfile에 JDK·Node·CLI를 고정해 GHCR에 publish하고 job container로 사용 | GitHub repository·package 권한과 가깝고 여러 repo의 도구를 같은 digest로 고정 | image 갱신·취약점·package 권한 관리 필요. private image는 job 시작 전에 registry 인증 필요 | **기본 권장**. Azure private registry 요구가 없을 때 |
| B. ACR 공통 build image | ACR에 build image를 저장하고 GitHub job이 pull | Azure 정책·사설 registry 표준과 맞출 수 있고, 추후 VM runtime container와 registry를 통합 가능 | ACR 비용·네트워크·GitHub job의 pull 인증을 추가로 관리 | 조직 표준, private Azure registry 또는 runtime container 요구가 확인될 때 |

현재 확인된 워크로드는 Tomcat과 Nginx에서 WAR/JAR·정적 파일을 실행하는 VM 배포다. 따라서 ACR의 VM runtime container 연계는 아직 요구되지 않는다. build image registry는 GHCR(A) 또는 ACR(B) 중 사용자 결정으로 정한다.

`container.image`를 사용하면 이미지 pull은 workflow step보다 먼저 일어난다. private GHCR/ACR image라면 `jobs.<job>.container.credentials` 또는 동등한 사전 인증 경로를 설계해야 하며, step 안에서 뒤늦게 실행한 `docker login`으로 이를 해결할 수 없다.

### GHCR을 선택할 때의 private image 정책 초안

사용자 의향은 GHCR build image를 private package로 사용하는 것이다. registry 선택(DEC-2026-004)이 GHCR로 확정될 경우 다음을 적용한다.

| 항목 | 정책 초안 |
|---|---|
| visibility | GHCR package는 `private`. public 또는 internal로 완화하지 않음 |
| package 연결 | build-image repository와 연결하고 `org.opencontainers.image.source` label로 출처를 표시 |
| publish | 승인된 build-image repository의 protected branch/tag publisher job만 `packages: write` |
| pull | CI source repository와 허가된 reusable workflow에 package `Read` 및 job `packages: read`만 부여 |
| job 시작 전 pull | `container.credentials`로 해당 job의 `GITHUB_TOKEN`을 사용. 다른 private repository image는 package에 해당 repository의 Actions access를 명시적으로 부여 |
| version | `latest` 대신 toolchain version·build revision tag와 immutable digest를 사용 |
| 삭제 | 지원 중인 digest와 rollback 대상은 retention 정책 전 삭제 금지. cleanup job만 package delete 권한을 가짐 |

private GHCR package가 repository 권한을 자동으로 모두 상속한다고 가정하지 않는다. package를 repository에 연결해 상속을 택하거나 package의 granular access에서 CI repository를 명시해야 한다. 사람용 PAT를 shared workflow secret으로 두는 대신 job `GITHUB_TOKEN`과 package 최소 권한을 우선 사용한다.

## workflow를 repository에 둘 위치

| workflow 종류 | 저장 권장 위치 | 실행 권한 경계 |
|---|---|---|
| PR/merge CI | 각 source repository | source code read와 build/test에 필요한 최소 GitHub 권한 |
| release 후보 publish | 각 source repository 또는 승인된 중앙 publisher workflow | Blob publish만; production VM/Gateway 권한 없음 |
| dev/prod 배포 | 중앙 deployment repository 권장 | protected Environment와 환경별 deployer identity로 제한 |
| 공통 build logic | reusable workflow 또는 composite action | 호출 repo의 권한으로 실행됨. 재사용 자체가 prod 권한 집중을 뜻하지 않음 |

중앙 deployment repository 선택은 [DEC-2026-002](../decisions/DEC-2026-002-deployment-trust-boundary.md)의 사용자 결정 후보를 따른다. 아직 확정되지 않았으므로 source repository와 deployment repository의 실제 이름을 문서에 가정하지 않는다.

## 다음 확인

1. 각 서비스의 Maven/Gradle wrapper, JDK target, Node/npm lockfile, 추가 CLI 목록을 수집한다.
2. 선택한 registry의 Dockerfile로 `build-java-node:<version>` image를 만들고, image digest를 고정한다.
3. dev CI에서 private image pull, cold-cache build, lockfile 기반 build/test를 검증한다.
4. image patch·취약점 대응과 폐기 책임, 보존 개수와 registry 접근 권한을 정한다.

## GitHub Enterprise Cloud 용량과 과금 구분

GitHub Enterprise Cloud는 GitHub Packages에 **50GB storage**와 월 **100GB data transfer**를 포함한다. Actions artifacts와 일반 GitHub Packages storage는 shared storage allowance로 집계된다. Actions cache는 repository별 10GB의 별도 allowance다.

GHCR(Container registry)의 container image storage와 bandwidth는 현재 GitHub 정책상 무료다. 이는 Enterprise Cloud의 50GB Packages allowance와 같은 뜻이 아니며, GitHub가 정책 변경 시 최소 한 달 전에 통지한다고 명시한 현재 정책이다. 따라서 build image의 보관 한도를 50GB라고 가정하지 않는다. 조직의 billing 설정·budget과 실제 사용량은 별도로 확인한다. GHCR의 image layer 하나는 10GB 제한이 있다.

## runner 시간과 registry·cache의 비용 관계

GitHub-hosted runner는 job이 할당된 뒤 image pull, package download, cache restore/save, build/test, artifact upload/download을 처리한다. 따라서 이 작업들의 경과 시간은 runner 실행 시간에 포함된다. GHCR과 ACR은 storage/transfer·권한·네트워크 선택이지, runner가 해당 image나 package를 가져오는 시간을 없애는 선택은 아니다.

| 작업 | runner 시간 사용 | 줄이는 방법 | 다른 비용·제약 |
|---|---|---|---|
| private GHCR/ACR build image pull | 예 | image layer를 작게 유지하고 digest 재사용, registry와 runner의 network path 확인 | GHCR/ACR storage·transfer, private pull 인증 |
| JDK·Node·CLI 설치 | 예 | 검증된 build image에 고정 | build image patch·취약점 관리, 첫 pull 시간 |
| Maven/Gradle/npm dependency download | 예 | Actions Cache hit, lockfile 기반 cache key, 사내 package 원본 접근 최적화 | cache miss·eviction은 계속 발생 가능 |
| Actions Artifact upload/download | 예 | 전달 대상을 최종 build 결과·필수 증적으로 제한, 중복 archive 방지 | artifact storage·retention 비용 |
| WAR/JAR·frontend Blob publish | 예 | 파일 압축·변경량·publish 단계 최소화 | Blob storage·request·transfer 비용 |

파일럿의 비용 최적화 우선순위는 다음과 같다.

1. GHCR/ACR build image에 JDK·Node·공통 CLI만 넣어 setup 시간을 줄인다.
2. dependency 원본 전체를 image에 넣지 않고, Actions Cache와 GitHub Packages 원본을 사용한다. dependency를 image에 과도하게 넣으면 image pull·patch release가 커지고 cache 이점이 사라진다.
3. Artifacts에는 build 결과와 증적만 보관하고 `node_modules`·Maven cache를 매 workflow archive로 이동하지 않는다.
4. 실제 dev CI에서 `image pull`, `cache restore`, `install`, `build/test`, `artifact`, `publish` 시간을 분리해 측정하고, cache hit/miss별 실행 분을 기록한다.

GitHub-hosted runner의 custom image 기능은 larger runner에서만 사용할 수 있으며, larger runner 실행 분은 Enterprise Cloud의 standard runner 포함 분과 별도로 과금된다. 따라서 “매번 GHCR image를 pull하는 시간”을 피하려고 larger runner custom image로 바로 전환하지 않는다. 먼저 표준 runner + private build image의 실제 시간을 측정하고, 반복 실행 시간·동시성·네트워크 제약을 근거로 재검토한다.

## 근거

- [GitHub-hosted runner images](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)
- [GitHub Actions job container syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [GitHub container jobs](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/run-jobs-in-a-container)
- [ACR managed identity authentication](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication-managed-identity)
- [배포 원본·캐시·build 상세](artifact-and-build-options.md)
- [배포 절차 초안](../reports/deployment-draft.md)
