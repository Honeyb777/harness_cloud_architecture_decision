# GitHub Actions CI 빌드·산출물·Azure OIDC 운영 모델

- 대상: IDC 서비스의 Azure VM 전환 파일럿
- 작성일: 2026-09-08
- 상태: DRAFT — 선택지와 권장 운영 모델을 설명하는 문서다. GitHub, Azure, ACR, Blob에 실제 설정·권한 부여·빌드 실행은 하지 않았다.

## 이 문서가 설명하는 범위

이 문서는 소스 변경부터 검증된 release를 보관하는 시점까지의 **GitHub Actions CI**를 설명한다. 서버 배포, Application Gateway drain, Nginx readiness, Tomcat 교체, Azure Pipelines CD는 다루지 않는다.

목표는 두 가지다.

1. 개발자가 어느 repository에서 빌드하더라도 같은 JDK·Node·공통 CLI 환경으로 재현 가능한 build를 만든다.
2. 캐시, CI 증적, 배포 원본을 분리해 속도·감사·rollback 요구가 서로 충돌하지 않게 한다.

## 전체 흐름

```text
PR / merge / release tag
          │
          ▼
┌──────────────────────── GitHub Actions CI ────────────────────────┐
│  1. GitHub-hosted runner 시작                                      │
│  2. private build image pull: JDK + Node + 공통 CLI                │
│  3. Actions Cache restore: Maven/Gradle/npm 재사용 가능 데이터     │
│  4. build · test · SBOM 생성                                       │
│  5. Actions Artifacts 저장: test report · SBOM · 로그 · job 전달물 │
│  6. release 조건 충족 시 Blob에 release와 manifest 게시            │
└───────────────────────────────────────────────────────────────────┘
          │
          ▼
이후 CD가 release ID + manifest digest를 선택해 배포
```

CI가 Blob에 release를 게시하는 일과 CD가 그 release를 VM에 적용하는 일은 별도 책임이다. CI가 성공했다고 서버 배포가 완료된 것은 아니며, CI identity가 VM 또는 Gateway를 제어해서도 안 된다.

## 네 저장소의 역할

| 구성 요소 | 보관하는 것 | 해결하는 문제 | 해결하지 않는 문제 |
|---|---|---|---|
| **GHCR 또는 ACR** | JDK, Node, 공통 CLI가 들어 있는 OCI build image | 모든 CI job의 toolchain 버전 고정 | dependency 다운로드나 release 보관 |
| **Actions Cache** | Maven/Gradle/npm 의존성, 재생성 가능한 중간 결과 | 반복 build의 다운로드·설치 시간 절감 | 배포 원본, 증적 장기 보존, secret 보관 |
| **GitHub Actions Artifacts** | test report, SBOM, 로그 요약, job 간 전달 파일 | 동일 workflow의 전달과 감사 증적 | 장기 rollback 원본 |
| **Azure Blob** | 검증이 끝난 WAR/JAR, frontend bundle, manifest, checksum | CD와 VM이 읽을 수 있는 고정 release 원본 | build 환경 제공, 의존성 cache |

이 분리가 필요한 이유는 lifecycle이 다르기 때문이다. cache는 없어져도 다시 만들 수 있어야 하고, artifact는 보존기간 후 사라질 수 있으며, release 원본은 rollback 기간 동안 정확한 digest로 남아야 한다.

GitHub Actions cache는 repository별 기본 10 GB이며 최근 7일 동안 접근하지 않은 cache는 제거될 수 있다. 따라서 cache hit는 성능 향상일 뿐 정상 build의 전제 조건이 될 수 없다. [GitHub cache 정책](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching)

Actions Artifacts의 기본 보존기간은 90일이며 private/internal repository에서는 1–400일 범위에서 설정할 수 있다. 만료될 수 있는 artifact를 유일한 rollback 원본으로 쓰면 안 되는 이유다. [GitHub artifact 보존 정책](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization)

## 빌드 이미지 선택: private GHCR 또는 ACR

GitHub-hosted runner image와 build image는 다르다. runner는 job을 실행하는 VM이고, build image는 그 job 안에서 JDK·Node·CLI를 고정하는 container다. ACR과 GHCR도 runner 자체를 제공하지 않는다.

| 선택 | 사용 방식 | 장점 | 확인할 사항 |
|---|---|---|---|
| **A. private GHCR** | GitHub Packages의 private OCI image를 job 시작 전 pull | repository와 package 권한을 GitHub 안에서 관리, GitHub 중심 운영에 단순 | package의 Actions access, pull 대상 repository, image patch/cleanup, digest 보존 |
| **B. ACR** | Azure Container Registry의 image를 Azure OIDC로 pull | Azure registry·network·runtime container 요구가 있을 때 통합 가능 | ACR 비용, Azure RBAC, hosted runner 네트워크와 pull 인증 |

현재 권장안은 **Azure private registry 또는 VM runtime container 요구가 확정되기 전에는 private GHCR**이다. GHCR을 택하면 image는 반드시 `private`로 두고, 소비 job에는 `packages: read`만 준다. image 발행 job만 `packages: write`를 갖는다. GitHub는 private package의 repository 연결 또는 granular access 설정을 지원한다. [GitHub Container registry](https://docs.github.com/en/enterprise-cloud@latest/packages/working-with-a-github-packages-registry/working-with-the-container-registry)

어느 registry를 택하든 workflow는 `latest`가 아니라 image digest를 고정한다. 예를 들어 `build-java-node:2026.09` tag를 사람이 읽기 쉽게 쓰더라도 실제 CI 입력은 해당 tag가 가리키는 `sha256:...` digest로 기록한다. 이렇게 해야 다음 달에 tag가 바뀌어도 과거 release를 재현할 수 있다.

## 권한 모델: CI job마다 필요한 것만 부여

```text
build-test
  └─ GitHub: contents read, 필요한 경우 packages read
  └─ Azure: 없음

publish-build-image
  └─ GHCR: packages write
  └─ ACR 선택 시: ACR publisher OIDC identity만

publish-release
  └─ GitHub: contents read + id-token write
  └─ Azure: 대상 Blob container에 게시·검증만 가능한 CI publisher identity

cd-deployer (후속 단계)
  └─ CI identity와 별도 identity
  └─ VM Run Command·Gateway 상태 확인 등 CD 권한

VM runtime
  └─ VM Managed Identity로 Blob download와 필요한 runtime secret만
```

`id-token: write`는 GitHub Actions job이 OIDC token을 요청할 수 있게 하는 권한이다. 이것만으로 Azure 자원에 접근할 수 있는 것은 아니다. Azure Entra workload identity의 federated credential이 어느 GitHub repository·branch·Environment를 신뢰할지 결정하고, Azure RBAC이 실제 Azure 작업 범위를 결정한다. [GitHub OIDC](https://docs.github.com/en/actions/reference/security/oidc), [Azure Login with OIDC](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect)

Azure OIDC를 쓸 때 workflow에 넣는 client ID, tenant ID, subscription ID는 identity를 식별하는 값이다. client secret은 필요하지 않다. 이 식별값은 GitHub Environment variable로 관리할 수 있지만, 실제 보호 경계는 GitHub job의 OIDC subject와 Azure RBAC이다.

### release publish의 Azure 권한

`publish-release` job에는 다음만 허용한다.

- 정해진 Blob container에 release 파일과 manifest를 upload
- upload 뒤 checksum 및 manifest를 읽어 검증
- 필요한 경우 release 경로 목록을 조회

다음은 허용하지 않는다.

- VM Run Command 실행
- Application Gateway 설정 또는 backend 변경
- Key Vault secret read
- Azure role assignment 변경
- release 삭제

제공 역할에 삭제 권한이 함께 포함될 수 있으므로, Pilot에서 upload·검증에 필요한 data-plane API를 확인한 뒤 삭제 없는 custom role을 만들 수 있는지 검증한다. release cleanup은 별도 identity와 승인 절차로 분리하는 것이 권장안이다.

## release를 만드는 방법

test가 성공한 뒤 CI는 build 결과를 단순 압축 파일로만 남기지 않고, 무엇을 배포하는지 증명하는 manifest를 함께 만든다.

```text
releases/<service>/<release-id>/
  backend/app.war                 또는 app.jar
  frontend/<bundle files>
  manifest.json                   commit SHA, build image digest, artifact digest, 생성 시각
  checksums.txt
```

CD는 `latest`나 workflow run 번호를 입력으로 삼지 않는다. 승인된 `release-id`와 manifest digest를 입력으로 받고, Blob에 완전한 파일·checksum·manifest가 모두 존재할 때만 적용한다. upload 도중 실패한 디렉터리는 완성 release로 표시하지 않는다.

## 초기 lifecycle 정책

| 대상 | 초기 원칙 | 별도 확정이 필요한 값 |
|---|---|---|
| build image | digest pinning, patch와 취약점 대응, 사용 중 digest 삭제 금지 | 이미지 보존 수·보존일, cleanup 담당 |
| Actions Cache | 재생성 가능한 데이터만 저장, cache miss에서도 build 성공 | 용량 상향 여부, key 구성, 운영 측정 기준 |
| Actions Artifacts | test/SBOM/로그와 job handoff에 사용 | 보존일, 저장 대상, 감사 보존 요구 |
| Blob release | release ID와 digest 기준 immutable 경로, cleanup identity 분리 | 보존 세대, rollback 기간, soft delete/versioning, lifecycle, network 공개 범위 |

## Pilot에서 확인할 것

1. private GHCR 또는 ACR image를 허용된 CI job만 digest 기준으로 pull할 수 있는지
2. cold/warm cache의 restore·install·build 시간을 분리해 기록하고, cache miss에서도 build/test가 성공하는지
3. Artifacts의 보존 종료와 무관하게 Blob release의 manifest·checksum·파일 digest가 일치하는지
4. 다른 repository, branch, Environment의 workflow는 Blob publisher OIDC login이 거절되는지
5. CI publisher identity가 VM Run Command, Gateway 변경, Key Vault secret read에 실패하는지
6. 같은 release ID의 overwrite가 차단되고, 실패한 publish가 CD 후보로 나타나지 않는지

## 선택 및 구체화가 필요한 내용

| 항목 | 선택지 또는 확인 내용 |
|---|---|
| build image registry | private GHCR / ACR |
| CI 증적 보존 | Artifact 보존기간과 대상 파일 |
| release 원본 | Blob을 canonical store로 채택할지, 채택 시 lifecycle과 rollback 보존 |
| Blob publisher | 삭제 없는 custom data-plane role 적용 가능 여부 |
| release publish 조건 | 허용할 tag·branch, `release-publish` Environment, 승인 필요 여부 |
| 네트워크 | Blob/ACR을 public endpoint 또는 private endpoint로 둘지, hosted runner 접근 경로 |

## 결론

파일럿의 CI는 private build image로 toolchain을 고정하고, Actions Cache는 속도 향상용, Artifacts는 workflow 증적용, Blob은 release 원본용으로 분리하는 구성이 가장 명확하다. Azure OIDC는 Blob 또는 ACR이 필요한 publish job에만 부여하고, CD와 VM runtime identity는 별도로 유지한다.

이 문서는 설계 초안이다. registry, retention, Blob lifecycle, OIDC subject와 RBAC scope가 확정되고 Pilot 검증을 통과하기 전까지 실제 적용으로 간주하지 않는다.
