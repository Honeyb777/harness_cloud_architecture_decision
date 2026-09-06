# 배포 원본·Artifacts·빌드 캐시 대안

상태: 조사 및 권장안. 실제 보존 개수·기간·가격·스토리지 설정은 미확정.

## 1. Blob과 Actions Artifacts

| 기준 | Azure Blob | GitHub Actions Artifacts |
|---|---|---|
| 이번 검토의 권장 역할 | VM에 전달할 승인된 release 원본 | CI 작업 결과, job 간 전달, 시험·배포 증적 |
| VM 접근 | VM MI와 Entra RBAC로 읽기 가능 | VM이 GitHub API/다운로드 권한을 얻는 추가 경로 필요 |
| 보존 기준 | lifecycle는 시간 기반; N개 보존은 manifest·정리 로직 | 기간 기반; N개 보존은 별도 API 정리 필요 |
| 환경 승격 | 동일 release ID/digest를 dev/prod에서 참조 | run/artifact ID 고정 및 만료 전 승격 관리 필요 |
| 운영 | private endpoint·DNS·RBAC·backup/soft delete 설계 | GitHub 보존 정책·storage 과금·run 삭제 영향 관리 |
| 감사/복구 | release manifest·승인 이력과 연결 | 장기 필수 증적은 별도 보존 정책에 따라 이관 |
| 비용 | 저장 용량·복제·요청·전송·private endpoint·보호 기능 | artifact 누적 용량과 runner·전송 등 계약 기준 확인 |

권장: WAR/JAR 또는 frontend bundle을 Blob의 고유 release 경로에 두고, VM은 MI로 읽는다. Azure access token이 곧 GitHub artifact 다운로드 token인 것은 아니다. [AzCopy MI](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-authorize-managed-identity), [Blob Entra 접근](https://learn.microsoft.com/en-us/azure/storage/blobs/authorize-access-azure-active-directory)
Artifacts를 배포 저장소로 쓰는 것도 가능하다. 다만 workflow가 정확한 run/artifact를 내려받은 뒤 VM에 전달하는 경로, 만료·삭제·재실행·교차 repo 권한 및 장기 rollback을 해결해야 한다. no-password VM pull 요구에는 Blob이 더 직접적이라는 판단이다.

## 2. release와 보존 정책

경로 예시: `releases/<service>/<release-id>/app.war`, `frontend.tar.gz`, `manifest.json`.
manifest는 source repo/commit, build run, release ID, 각 파일 digest, 테스트 결과 참조, 생성 시점, 호환성 범위를 포함한다.
동일 release ID를 덮어쓰지 않고 배포할 digest를 승인 전에 고정한다. mutable `latest`를 rollback 원본으로 사용하지 않는다. 저장소의 기술적 immutability/WORM 도입은 요구사항과 삭제 정책을 함께 비교해 별도 결정한다.

보존 정리 알고리즘 제안:

1. 서비스별 release 목록과 배포 상태를 읽고, 모든 환경·VM의 current/previous/in-flight/rollback-pinned release를 보호 집합으로 만든다.
2. 실패·미완성 upload를 완성된 release로 집계하지 않는다. 완성 manifest가 게시된 정상 버전에서 최근 N개를 선택한다.
3. 최근 N개와 보호 집합의 합집합을 유지한다. 브라우저 자산 호환 기간·감사/법적 보존 대상도 삭제하지 않는다.
4. 정리는 배포·승격과 동일한 조정 정책 또는 일관된 snapshot/lock으로 수행한다. 삭제 직전 참조를 재검사한다.
5. 먼저 dry-run 목록을 남기고, 확정된 정책에 따라 별도 정리 Identity가 실행한다. 보호된 버전이 많으면 N개를 넘을 수 있음을 보고한다.

정확히 N개로 강제하면 사용 중인 버전이나 오래 열린 브라우저의 자산을 지울 수 있다. 권장 의미는 '최신 N개 + 사용/복구/정책상 보호 버전'이며 N과 보호 기간은 사용자 결정 항목이다.
Blob versioning은 동일 객체 변경 이력이며 애플리케이션 release 개수 관리와 다르다. Blob lifecycle의 실행 조건은 시간 기반이라 최신 N개 정리의 직접 대체가 아니다. [Lifecycle policy](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-structure)
Actions artifact 기본 보존은 90일이며 private repo에서는 1~400일 범위와 조직/Enterprise 상위 정책을 따른다. 개별 retention 값만으로 N개가 유지되거나 영구 rollback이 보장되지 않는다. [Artifacts 보존](https://docs.github.com/en/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization)

## 3. Artifacts를 사용할 다른 용도

- JUnit 등 테스트 보고서, coverage, UI/E2E 스크린샷·실패 영상.
- SBOM, 의존성·취약점 검사 결과, release manifest·digest 목록.
- build→integration-test→publish job 간 임시 산출물 전달.
- 배포 전후 health snapshot, version 확인, 승인 run 참조와 rollback 결과.
- 정리 dry-run, 비교 보고서 등 사람이 검토할 파일.

출력에 token·비밀번호·민감한 설정·원본 운영 데이터를 포함하지 않는다. 필수 장기 감사 증적은 짧은 artifact 보존과 분리한다.
Artifacts는 job 결과 보존/전달, Actions cache는 재생성 가능한 의존성 캐시라는 목적 차이가 있다. Maven/Gradle/npm 캐시나 Docker 레지스트리의 완전한 대체로 사용하지 않는다. [Workflow artifacts](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts)

## 4. Nexus 제외 후 역할 구성

사용자는 기존 GitLab/Jenkins/Nexus/Ansible을 새 구성에서 사용하지 않기로 했다. 초기 조사에서 제안했던 Nexus 유지와 Ansible 재사용은 대상에서 제외한다.

| 기능 | 목표 구성 | 남은 확인 |
|---|---|---|
| Maven/Gradle/npm 의존성 cache | Actions Cache | miss 시 원본 접근, cache key·권한 |
| 사내 자체 패키지 원본 | 필요 시 GitHub Packages 등 영속 registry | Nexus에만 존재하는 패키지 목록과 이전 |
| 빌드 runner 도구·라이브러리 | hosted runner setup/install; 반복성이 필요하면 GHCR 공통 build image | 실제 모듈·버전·설치 시간 |
| 배포 WAR/JAR/프론트 원본 | Blob 우선 권장 | N·보존 기간·접근 범위 |
| 시험·배포 결과 | Actions Artifacts | 보존·용량·감사 |

추가 모듈은 빌드 runner용이므로 ACR을 기본안에 추가하지 않는다. GHCR private package는 권한이 있는 Actions job의 GITHUB_TOKEN으로 사용 가능하다. ACR은 Azure runtime의 private image/MI pull 필요 시 재검토한다.
상세 근거와 비교는 [최신 후속 검토](hosted-only-push-pull-key-vault-static.md)를 따른다.

## 5. Java/Tomcat·Vue/npm 빌드와 승격

- 기존 Jenkinsfile에서 trigger·JDK/Node·wrapper·단계·secret 참조·Nexus 주소·artifact 명명·Ansible 인자를 목록화한다.
- Java는 실제 Maven/Gradle wrapper, JDK, target bytecode와 Tomcat/Servlet 호환성을 확인한다. 현재 Kotlin 사용은 확인되지 않았으므로 필요해질 때 추가한다.
- Vue는 lockfile과 실제 npm/Node 버전으로 재현 가능한 install/build를 구성한다. 빌드는 runner에서, VM은 완성 artifact를 실행/제공하도록 설계한다.
- build job에는 운영 VM 실행 권한을 주지 않는다. 승인된 source·release만 publisher Identity로 게시한다.
- dev 자동 배포→검증→동일 digest의 prod 승인 승격을 목표로 한다. 환경별 값은 런타임 설정으로 분리할 수 있는지 확인한다. Vue 빌드에 환경별 값이 내장돼 있으면 동일 artifact 승격 조건을 먼저 정리한다.
- 원본 digest 확인만으로 악의적 publisher를 방어할 수는 없다. 허용 repo/branch/build run과 서명·attestation 등 출처 검증 요구를 함께 결정한다.
- Actions cache가 없어도 빌드 가능해야 하며, 신뢰되지 않은 PR의 캐시·산출물을 prod publish 경로에서 그대로 실행하지 않는다.

## 6. Copilot 및 비용

Copilot은 기존 Jenkinsfile 분석, Actions/배포 스크립트 초안, 테스트·코드 리뷰 보조에 사용할 수 있다. prod token 발급, Environment 승인, peer health 확인을 대체하는 역할로 두지 않는 안을 제안한다.
Enterprise의 Copilot 사용/검토/승인 정책과 코드·지시문 접근 범위를 확인한다. 실제 기능 정책은 도입 시점 문서를 따른다. [Copilot review 설정](https://docs.github.com/en/enterprise-cloud@latest/copilot/how-tos/copilot-on-github/set-up-copilot/configure-code-review)

견적은 미산정이다. VM/App Gateway 기존 비용과 CI/CD 증분 비용을 분리하고 다음 입력을 수집한다.

| 비용 driver | 필요한 값 |
|---|---|
| hosted/larger runner | 빌드·시험·배포 대기 포함 실제 실행 분, SKU·동시성·월 횟수 |
| Blob/Artifacts | 평균 release 크기 × 유지 개수 × 서비스 수, 보호본·로그·soft delete·복제 추가량 |
| 네트워크 | IDC↔Azure 의존성/이미지/패키지 전송량, VPN/전용회선, private endpoint/DNS |
| 배포 방식 | 기존 port·단일 JVM rolling의 단일 VM 부하 여유, 필요시 임시 VM |
| 도구 | GitHub Enterprise/Copilot·보안 검사·GHCR·Key Vault 비용, ACR은 도입 시 재검토 |

빠른 rollback 원본은 즉시 읽을 수 있는 계층을 우선하고 Archive 이동은 복구시간과 함께 결정한다. region/DR·backup·retention·tag 정책은 해당 Step에서 미확인 값을 채운다.
