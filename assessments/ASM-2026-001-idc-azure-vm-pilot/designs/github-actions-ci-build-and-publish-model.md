# GitHub Actions CI: build, cache, artifact, release publish 권한 모델

- Assessment: ASM-2026-001
- Step / Phase: STEP-01 / DISCOVERY
- Status: PROPOSED — 보고서가 아닌 설계 초안이며 GitHub·Azure 설정이나 빌드 실행은 수행하지 않았다.
- 확인일: 2026-09-08

## 목적과 범위

이 문서는 GitHub Actions의 **CI 구간**만 다룬다. 소스 변경을 빌드·검증하고, 검증된 결과를 다음 CD 단계가 읽을 수 있는 release로 게시하는 경계를 정리한다.

다음은 이 문서의 범위 밖이다.

- Azure Pipelines 또는 GitHub Actions를 이용한 CD 대기열·승인·서버 배포
- Application Gateway drain, Nginx readiness, Tomcat 교체와 rollback 실행
- VM Managed Identity의 runtime Blob/Key Vault 접근

따라서 여기의 Azure OIDC는 CI가 ACR 또는 Blob에 접근해야 할 때만 사용한다. VM Run Command, Application Gateway, Key Vault secret 읽기 권한은 CI identity에 주지 않는다.

## 한눈에 보는 흐름

```text
source repository의 PR / merge / release tag
                    │
                    ▼
          GitHub-hosted runner
                    │
        private build image pull
          ┌─────────┴──────────┐
          │ GHCR 또는 ACR       │  JDK · Node · 공통 CLI의 고정된 버전
          └────────────────────┘
                    │
                    ▼
         Actions Cache restore
          Maven/Gradle/npm의 재생성 가능한 의존성·중간 결과
                    │
                    ▼
         build · test · SBOM 생성
             │              │
             │              └── Actions Artifacts
             │                  테스트 보고서·SBOM·로그·job 간 전달물
             ▼
     release 후보 승인 조건 충족
                    │
                    ▼
     Azure OIDC로 Blob에 release 게시
       immutable release ID + commit SHA + artifact digest + manifest
                    │
                    ▼
      이후 CD가 고정된 release ID만 선택하여 소비
```

네 저장소는 이름이 비슷해도 역할이 다르다. build image는 실행 환경을 재현하고, cache는 다운로드 시간을 줄이며, artifact는 workflow 결과와 증적을 전달하고, Blob은 배포·복구에 쓸 release 원본 후보가 된다. 서로 대체하지 않는다.

## 저장소별 책임과 초기 정책

| 구분 | 저장 대상 | 사용하는 시점 | 원본으로 삼는가 | 초기 정책 및 미결정 사항 |
|---|---|---|---|---|
| GHCR 또는 ACR | JDK, Node, 공통 CLI가 포함된 OCI build image | job 시작 전 container pull | 예. 단 image digest로 고정한다. | 둘 중 하나의 registry 선택은 아직 미결정이다. GHCR을 선택하면 package는 `private`로 유지한다. `latest`는 CI 기준 tag로 쓰지 않는다. |
| Actions Cache | Maven/Gradle/npm 의존성, 재생성 가능한 중간 결과 | build 전 restore, 종료 시 save | 아니오 | lockfile·wrapper·OS 등을 포함한 key를 사용한다. secret, release 파일, `node_modules` 전체를 배포 원본으로 넣지 않는다. 기본 10 GB/repository와 7일 미접근 정리 동작을 기준으로 Pilot 측정 후 보존 한도를 정한다. |
| GitHub Actions Artifacts | test report, SBOM, 로그 요약, job 간 build 전달물 | 동일 workflow의 후속 job 또는 감사 | 아니오 | 기본 보존은 90일이며 private/internal repository는 1–400일로 정할 수 있다. 기간 만료가 가능한 증적 저장소이므로 rollback의 유일한 원본으로 쓰지 않는다. 보존일은 별도 결정이 필요하다. |
| Azure Blob | release WAR/JAR, frontend bundle, manifest, checksum | release publish 후 이후 CD/VM이 읽음 | 예. 채택 시 canonical release store | release ID와 digest를 경로·manifest에 기록하고 overwrite를 막는다. 보존 세대, soft delete/versioning, lifecycle 및 Blob public/private network 정책은 아직 미결정이다. |

Actions Cache는 cache hit 여부와 관계없이 정상 build가 가능해야 한다. cache에 들어간 데이터는 신뢰 경계 밖에서 온 것으로 다루며 실행 가능한 release나 자격증명을 보관하지 않는다. Artifacts도 image 안에 사전 설치한 JDK·Node·CLI를 대신하지 않으며, 매 job에서 `install` 시간을 없애는 용도가 아니다.

## build image 선택지

| 선택지 | 적합한 경우 | CI 접근 방식 | 권한 경계 |
|---|---|---|---|
| A. private GHCR | GitHub 중심으로 공통 toolchain image를 여러 repository가 사용할 때 | `GITHUB_TOKEN`으로 job 시작 전 pull. image 발행 workflow만 `packages: write` | source repository job은 `packages: read`만 받는다. package의 Actions access를 소비 repository 또는 reusable workflow에 명시한다. |
| B. ACR | Azure registry 운영 기준을 통합해야 하거나 ACR 사용 요구가 확인될 때 | Azure OIDC로 읽기 전용 identity를 로그인에 사용하여 pull | CI reader와 ACR image publisher를 분리한다. ACR image publish가 없으면 CI에 쓰기 권한을 주지 않는다. |

두 선택지 모두 Dockerfile에서 toolchain 버전을 명시하고, workflow에는 image digest를 고정한다. base image patch와 취약점 조치, digest 보존 수, package cleanup 담당자는 별도 운영 정책으로 정해야 한다.

## CI job과 권한 분리

| job | GitHub 권한 | Azure OIDC / Azure 권한 | 금지할 권한 |
|---|---|---|---|
| `build-test` | `contents: read`, private build image를 쓸 때 `packages: read` | 없음 | Blob, ARM, Key Vault, App Gateway, VM Run Command |
| `publish-build-image` | GHCR 선택 시 `packages: write` | ACR에 image를 발행할 때만 ACR publisher identity | Blob release publish, 배포 권한 |
| `publish-release` | `contents: read`, `id-token: write` | `ci-release-publisher`: 대상 Blob container의 업로드와 게시 결과 검증에 필요한 data-plane 권한만 | VM Run Command, App Gateway 변경, Key Vault secret read, 역할 할당·삭제 |
| 이후 `cd-deployer` | CI와 별도 workflow/repository/Environment에서 부여 | CD 범위의 별도 workload identity | CI identity의 재사용 |

`id-token: write`는 Azure 로그인이 필요한 job에만 선언한다. Azure OIDC에는 client secret이 필요하지 않다. workflow에는 client ID, tenant ID, subscription ID 같은 식별값을 variable 또는 Environment variable로 두고, Entra workload identity의 federated credential이 repository·branch 또는 GitHub Environment subject를 정확히 신뢰하게 한다. 이 값 자체는 secret이 아니지만, Azure 권한이 생기는 조건은 subject와 RBAC이므로 공개 workflow에서도 최소 권한을 유지해야 한다.

release publish는 일반 PR build와 분리한다. 예를 들어 protected tag 또는 `release-publish` GitHub Environment에서만 `publish-release` job을 실행하고, federated credential도 그 subject만 허용한다. 이 경계가 없으면 모든 CI push가 Blob publisher 권한을 얻을 수 있다.

Blob의 기본 제공 역할에는 삭제처럼 release 불변성에 불필요한 작업이 포함될 수 있다. 따라서 Pilot에서는 먼저 필요한 upload·read verification·목록 조회 API를 확인하고, 삭제가 불필요하다면 해당 작업을 제외한 custom data-plane role의 실현 가능성을 검증한다. cleanup identity는 release publisher와 분리한다.

## release 게시의 입력과 결과

`publish-release`는 test가 성공한 뒤 다음을 한 묶음으로 만든다.

```text
releases/<service>/<release-id>/
  backend/app.war                 (또는 app.jar)
  frontend/<bundle files>
  manifest.json                   (commit SHA, build image digest, artifact digest, 생성 시각)
  checksums.txt
```

CD는 `latest`나 workflow run 번호만 보고 배포하지 않고, 승인된 `release-id`와 manifest digest를 입력으로 받는다. Blob upload가 중단되면 manifest를 완성된 release로 표시하지 않는다. upload 완료 후 checksum과 manifest가 일치할 때만 CD에 전달한다.

## Pilot에서 확인할 항목

| 검증 | 기대 결과 |
|---|---|
| private GHCR/ACR image pull | 허용된 repository/job만 고정 digest를 pull하고, 권한 없는 job은 실패한다. |
| cold/warm cache | cache miss와 hit의 restore·install·build 시간을 분리 기록하며, cache miss도 정상 build한다. |
| Artifact lifecycle | test/SBOM 증적과 job 간 전달은 가능하지만, 보존기간 만료 뒤 release 원본으로 의존하지 않는다. |
| Blob release integrity | 동일 release ID overwrite가 거절되고, manifest·checksum과 실제 파일 digest가 일치한다. |
| OIDC 거부 경로 | 다른 repository/branch/Environment subject는 Blob publish login이 거절된다. |
| 최소 권한 | CI publisher는 VM Run Command, Application Gateway, Key Vault secret read를 수행할 수 없다. |

## 결정이 필요한 내용

1. build image registry를 private GHCR과 ACR 중 어디로 선택할지
2. Artifacts 보존 기간과 대상(테스트 보고서, SBOM, 로그, build handoff)
3. Blob을 canonical release store로 채택할지, 채택한다면 보존 세대·soft delete/versioning·lifecycle
4. release publisher에 삭제 없는 custom Blob data-plane role을 적용할 수 있는지
5. `release-publish` Environment의 branch/tag 조건과 승인 필요 여부

## 근거

- [GitHub dependency caching](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching)
- [GitHub workflow artifacts](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts)
- [GitHub Actions artifact and log retention](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization)
- [GitHub Container registry](https://docs.github.com/en/enterprise-cloud@latest/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [GitHub Actions OIDC](https://docs.github.com/en/actions/reference/security/oidc)
- [Azure Login with OpenID Connect](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect)
- [Azure Storage access from GitHub Actions](https://learn.microsoft.com/en-us/azure/storage/blobs/assign-azure-role-data-access)

공식 문서는 기능과 권한 모델의 근거다. 실제 repository, package, Entra tenant, Blob container, network 경계와 RBAC은 아직 확인하거나 변경하지 않았다.
