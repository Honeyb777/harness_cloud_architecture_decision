# Build cache·package·배포 release 생명주기 선택안

- Assessment: ASM-2026-001
- Step / Phase: STEP-01 / DISCOVERY
- Status: PROPOSED — 보존 기간·개수·Blob 설정·정리 job은 미확정·미적용
- 확인일: 2026-09-07

## 저장소별 역할을 먼저 분리한다

`cache`는 build 속도를 높이기 위한 재생성 가능한 복사본이고, `package`는 의존성이 참조하는 영속 원본이며, `release`는 VM에 배포할 승인된 결과물이다. 이 셋을 같은 저장소·같은 보존 규칙으로 관리하지 않는다.

| 대상 | 저장소와 역할 | 보관하는 것 | 삭제되어도 되는가 | 배포 원본으로 쓰는가 |
|---|---|---|---|---|
| Build dependency cache | **GitHub Actions Cache** | Maven/Gradle/npm이 다시 내려받을 수 있는 의존성 cache | 예. cache miss 후 원본에서 재생성 | 아니오 |
| 사내 dependency package | **GitHub Packages** | Nexus에만 있던 사내 Maven/npm package의 versioned 원본 | 참조 중인 version은 아니오 | 직접 배포 원본은 아님 |
| Build container image | **GHCR 또는 ACR** | JDK·Node·추가 CLI를 고정한 OCI image | 지원 중인 digest와 rollback 대상은 아니오 | CI job 실행 원본 |
| CI 증적·job 간 임시 전달 | **GitHub Actions Artifacts** | 테스트 결과, SBOM, coverage, release manifest 사본, 진단 파일 | 보존 기간 종료 후 예 | 장기 VM 배포 원본으로 권장하지 않음 |
| 승인된 VM 배포 release | **Azure Blob Storage** | WAR/JAR, Vue bundle, immutable manifest·digest | current/previous/in-flight/pinned이면 아니오 | **예. 권장 canonical source** |

GitHub Packages를 “build cache”로 쓰지 않는다. Maven/npm이 외부 또는 사내 원본에서 받은 파일을 빠르게 재사용하는 것은 Actions Cache다. 사내에서 직접 만든 Maven/npm 라이브러리를 Nexus 없이 계속 공급해야 하면 GitHub Packages를 영속 registry로 사용한다.

### Artifacts로 설치 결과를 전달할 수 있는 범위

Actions Artifacts는 같은 workflow의 job 사이에 파일을 전달할 수 있다. 따라서 한 job에서 만든 `target/`, `dist/`, 테스트 보고서처럼 **완성된 build 결과**를 다음 job에 전달하는 데 적합하다. 기술적으로 `node_modules`나 Maven local repository를 archive해 전달할 수도 있지만, 이를 build cache의 기본 방식으로 채택하지 않는다.

| 질문 | Artifacts | Actions Cache | private GHCR build image |
|---|---|---|---|
| 다음 job에 build 결과 전달 | 적합 | 부적합 | 부적합 |
| dependency download 재사용 | 가능하지만 archive upload/download·추출이 매번 필요 | **적합** | 일부 도구를 image에 포함할 수 있으나 dependency version 전체는 아님 |
| JDK·Node·OS CLI 제공 | 불가. job 시작 후에만 download 가능 | 불가 | **적합**. job 시작 전에 image로 제공 |
| private job container의 첫 image pull 대체 | 불가 | 불가 | 해당 image 자체 |
| 보존 목적 | 단기 workflow 증적·전달 | 속도 향상용 재생성 cache | 검토·패치되는 toolchain 원본 |

Artifacts를 써도 GitHub-hosted runner는 매 job 새 환경에서 시작한다. private build image pull은 workflow step보다 먼저 일어나므로, Artifacts 안의 설치 파일이나 그 job의 `docker login`은 최초 job container 기동에 사용할 수 없다. 따라서 Artifacts 도입이 private GHCR build image의 의미를 줄이지는 않는다. 대신 build image에는 JDK·Node·공통 CLI를 고정하고, Maven/npm dependency는 Actions Cache와 GitHub Packages 원본을 조합한다.

## CI에서 배포까지의 파일 흐름

```text
GitHub Actions build container (GHCR 또는 ACR)
  ├─ Actions Cache: dependency download 재사용
  ├─ GitHub Packages: 사내 Maven/npm version fetch
  ├─ build/test
  ├─ Actions Artifacts: test/SBOM/manifest 사본과 job 간 임시 전달
  └─ Azure Blob: 승인된 release ID/digest의 WAR/JAR·frontend·manifest 게시
                       ↓
              VM Managed Identity가 Blob에서 고정 release를 pull
```

빌드 job은 production VM 배포 권한을 갖지 않는다. publish job만 release manifest와 파일 digest를 만든 뒤 Blob의 새 경로에 쓰며, dev/prod job은 동일 release ID를 읽어서 승격한다.

## 1. Build cache와 GitHub Packages 생명주기

### Actions Cache

| 항목 | 정책 초안 |
|---|---|
| 대상 | `~/.m2/repository`, Gradle dependency cache, npm download cache 등 재생성 가능한 파일만 |
| key | OS/architecture, JDK·Node·package-manager 버전, lockfile·wrapper·build file hash 포함 |
| 보존 | GitHub 기본은 repository당 10GB, 최근 7일 미접근 entry 정리. 실제 organization/enterprise 정책·budget 확인 |
| 실패 처리 | cache miss·eviction·read-only 상태여도 원본 package registry에서 build 가능해야 함 |
| 금지 | `node_modules`를 모든 환경의 정답으로 보관, secret·`.npmrc` credential·운영 설정·승인된 deployment release 저장 |

cache size와 retention을 늘릴 수 있어도 build correctness나 rollback 보존을 보장하지 않는다. cache는 source repository별이고, build image·package·release의 lifecycle과 분리한다.

### GitHub Packages

| 항목 | 정책 초안 |
|---|---|
| 대상 | Nexus에만 있던 사내 Maven/npm package와 필요 시 OCI build image(GHCR) |
| version | immutable semantic version 또는 build metadata 포함 version. 소비자는 `latest`가 아닌 lockfile/명시 version 사용 |
| publish 권한 | 신뢰된 release branch/tag의 publisher job만 `packages: write`; 일반 PR job에는 publish 권한 없음 |
| consume 권한 | 필요한 repository/team/job에 `packages: read`만 부여 |
| 삭제 | 사용 중인 package version, 지원 중인 build image digest, rollback 대상은 삭제하지 않음. 정리 권한은 publisher와 분리 |
| 복구 | GitHub package version은 삭제 후 30일 안에 복원 가능할 수 있으나, 이를 유일한 복구 전략으로 가정하지 않음 |

GitHub Packages의 storage allowance와 GHCR Container registry의 현행 image storage/bandwidth 무료 정책은 [CI/CD 이미지 선택안](ci-cd-stages-and-build-image-options.md)을 따른다.

## 2. 배포 결과물: Blob과 Actions Artifacts 선택

| 기준 | Option A — Blob canonical release + Artifacts 증적 | Option B — Actions Artifacts만 사용 |
|---|---|---|
| VM pull | VM Managed Identity와 Storage data-plane RBAC로 직접 읽음 | GitHub artifact API download token·run/artifact ID·교차 repo 권한을 VM에 추가 설계 |
| release 식별 | `releases/<service>/<release-id>/manifest.json`과 파일 digest | workflow run/artifact ID·만료 전 승격 경로에 의존 |
| 보존 | lifecycle/soft delete/versioning과 custom release cleanup 조합 | private/internal repo는 1~400일. 조직 상한 적용 |
| rollback | current/previous/in-flight/pinned release 집합을 보호 가능 | artifact 만료·run 삭제·권한 변경 전에 복구해야 함 |
| 감사 | Blob manifest와 GitHub run·승인을 상호 참조 | Actions run 중심 증적은 편리하나 장기 release 원본 관리가 약함 |
| 비용·운영 | Azure storage·요청·전송·보호 기능과 cleanup job | Actions shared storage·retention·artifact 관리, VM GitHub 접근 경로 |

파일럿 권장안은 **Option A**다. Actions Artifacts는 build-to-test job 전달, 테스트 보고서·SBOM·배포 전후 진단 증적에 사용하고, Azure Blob은 VM이 pull하는 승인된 release 원본으로 사용한다.

## 3. 승인된 release의 상태와 생명주기

release는 덮어쓰지 않는다. release manifest는 source repo/commit, CI run URL, build image digest, package lockfile digest, release ID, 파일별 digest, 테스트 증적 참조, 호환성 정보와 생성 시점을 포함한다.

```text
BUILDING → VERIFIED → PUBLISHED → DEPLOYED_DEV → APPROVED_PROD
  → DEPLOYED_PROD → RETIRED → ELIGIBLE_FOR_CLEANUP → DELETED
                         ↘ ROLLBACK_PINNED
```

| 상태 | 의미 | 삭제 가능 여부 |
|---|---|---|
| BUILDING | upload·digest 검증 중 | 불완전 upload 정리 전까지 배포 대상 아님 |
| VERIFIED | test·manifest 검증 완료 | 배포 후보 |
| PUBLISHED | Blob canonical 경로에 immutable release로 게시 | dev/prod 승격 대상 |
| DEPLOYED_DEV / DEPLOYED_PROD | 해당 환경에 실제 적용된 release | current/previous 기준에 따라 보호 |
| ROLLBACK_PINNED | 운영자가 복구 후보로 명시 보호 | 보호 해제 전 삭제 금지 |
| RETIRED | 지원·rollback 기간이 끝난 release | cleanup 후보 |
| ELIGIBLE_FOR_CLEANUP | 최근 N개와 모든 보호 집합 밖 | dry-run·재검사 뒤 삭제 가능 |

### Blob 보존·정리 정책 초안

1. 서비스별로 최근 **N개 정상 release**와 `current`, `previous`, `in-flight`, `rollback-pinned`, frontend 자산 호환 기간·감사 보존 대상의 합집합을 보호한다.
2. cleanup job은 dry-run 목록을 남기고, 직전 배포 상태와 보호 집합을 재검사한 뒤 별도 retention identity로 삭제한다.
3. Azure Blob lifecycle은 시간 조건만 처리하므로 “최신 N개” 판정의 대체가 아니다. N개 판정은 manifest/state를 읽는 cleanup job이 담당한다.
4. blob soft delete와 versioning은 삭제·덮어쓰기 복구를 돕지만 저장 용량과 lifecycle 상호작용을 추가한다. Blob을 immutable release 경로로 새로 쓰면 versioning이 release 개수 관리를 대신하지 않는다.
5. WORM/immutability는 규제·감사 요구가 확인될 때만 별도 결정한다. 일반 rollout rollback 요구만으로 lock된 retention policy를 도입하지 않는다.

## 4. 아직 정할 값

| 항목 | 결정 주체 | 차단 영향 |
|---|---|---|
| canonical deployment release 저장소: Blob(A) 또는 Artifacts(B) | 사용자 | STEP-06 데이터/배포 저장소 설계 |
| 최근 정상 release N, rollback pin 기간, frontend 구 자산 지원 기간 | 사용자·운영 | cleanup·rollback·비용 |
| Artifacts retention, cache size/retention, GitHub Packages retention | GitHub 운영·FinOps | 비용·감사·build 재현성 |
| Blob soft delete/versioning·lifecycle·보호 설정 | Azure 운영·Security | 복구·비용·감사 |
| cleanup identity, dry-run 승인자, 실행 주기 | 운영·Security | 오삭제 방지·감사 |

## 근거

- [GitHub dependency caching](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching)
- [GitHub cache settings](https://docs.github.com/en/enterprise-cloud@latest/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository)
- [GitHub artifact retention](https://docs.github.com/en/enterprise-cloud@latest/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization)
- [GitHub package deletion/restoration](https://docs.github.com/en/enterprise-cloud@latest/packages/learn-github-packages/deleting-and-restoring-a-package)
- [Blob lifecycle delete](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-delete)
- [Blob soft delete and versioning](https://learn.microsoft.com/en-us/azure/storage/blobs/soft-delete-blob-overview)
- [기존 artifact·보존 상세](artifact-and-build-options.md)
