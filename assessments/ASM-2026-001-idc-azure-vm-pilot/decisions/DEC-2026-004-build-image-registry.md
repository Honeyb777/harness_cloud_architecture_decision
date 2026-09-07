# DEC-2026-004 — CI build container image registry

## Status

PROPOSED

## Traceability

- Assessment: ASM-2026-001
- Applicable Environment IDs: ENV-03; Azure 연동 시 ENV-02
- Applicable Workload / Integration IDs: WL-03, INT-05
- Step: STEP-01 / DISCOVERY
- Approval Level: USER_DECISION
- Created / Updated: 2026-09-07
- Supersedes: 없음

## Context

사용자는 JDK·Node·추가 CLI 버전이 고정된 build container image를 CI에 사용하고, 이를 GHCR 또는 ACR에 보관하도록 지정했다. GitHub-hosted runner image는 job 실행 기반이며 이 결정의 registry 대상이 아니다.

## Requirement

- REQ-19: GHCR 또는 ACR의 versioned build container image를 사용한다.
- source repository workflow가 private image를 job 시작 전에 pull할 수 있어야 한다.
- image digest와 Dockerfile·도구 버전·보안 갱신 책임을 추적할 수 있어야 한다.

## Constraints

- GitHub Enterprise Cloud와 GitHub-hosted runner를 사용한다.
- 현재 VM은 Tomcat/Nginx 기반이며 VM runtime container pull은 확인되지 않았다.
- GitHub Enterprise Cloud의 Packages 50GB allowance와 GHCR container image의 현행 무료 정책은 구분한다.

## Options

### Option A — GHCR

**Summary**

`ghcr.io/<organization>/build-java-node:<version>`처럼 GitHub Container Registry에 build image를 저장한다.

GHCR을 선택하면 image는 private package로 운영한다. 허가된 CI repository/job만 package read와 `GITHUB_TOKEN` 기반 pull을 할 수 있게 한다.

**Advantages**

- GitHub repository·package 접근 정책과 가까우며 GitHub Actions의 package token 흐름에 맞는다.
- Azure registry와 별도 Azure network/RBAC 구성이 필요 없다.
- 현행 GitHub 정책에서 GHCR image storage·bandwidth는 무료다.

**Disadvantages**

- GitHub Packages 권한·image 갱신·취약점 대응을 별도로 운영해야 한다.
- Azure private registry 표준이나 VM runtime image 요구가 생기면 registry를 추가하거나 재검토해야 한다.

**Cost Impact**

Enterprise Cloud Packages allowance는 50GB storage와 월 100GB transfer다. GHCR container image storage·bandwidth는 현재 무료 정책이나 변경 가능성을 budget·usage monitoring으로 관리한다.

**Operational Impact**

private job container의 사전 pull credentials, image digest pinning, Dockerfile patch release와 retention 정책을 운영한다.

**Security / Audit Impact**

package publish/pull을 최소 GitHub team/job 권한으로 제한하고, build image 변경을 code review와 digest로 추적한다.

**Global Expansion Impact**

GitHub Enterprise Cloud data residency를 쓰는 경우 실제 Container registry endpoint와 package 접근 정책을 확인한다.

### Option B — ACR

**Summary**

Azure Container Registry에 동일한 build image를 저장하고 GitHub Actions job이 pull한다.

**Advantages**

- Azure private registry·network·runtime container 표준이 있는 경우 정렬된다.
- 향후 Azure VM runtime container pull 요구가 생기면 registry를 통합할 수 있다.

**Disadvantages**

- ACR 비용, GitHub-hosted runner pull 인증, 네트워크 경로와 Azure RBAC를 추가로 운영한다.
- 현재 VM의 WAR/JAR 배포만으로는 ACR runtime 통합 이점이 확정되지 않았다.

**Cost Impact**

ACR SKU·저장량·전송·private networking 조건을 실제 region과 사용량으로 견적해야 한다.

**Operational Impact**

registry 운영, Azure authentication/network 및 image lifecycle을 관리한다.

**Security / Audit Impact**

GitHub job의 ACR pull 신뢰와 ACR RBAC를 최소화하고, VM runtime MI와 CI pull 권한을 분리한다.

**Global Expansion Impact**

Azure region·private endpoint·복제 요구가 생기면 registry topology를 확장한다.

## Agent Recommendation

Azure private registry 또는 VM runtime container 요구가 아직 확인되지 않았으므로, 파일럿은 **Option A (GHCR)** 를 권장한다. ACR이 조직 표준이거나 private Azure registry 요구가 확인되면 Option B를 선택한다.

## User Decision

- Selected: 미결정 — GHCR 또는 ACR 선택 필요
- Decision Date / Owner / Reason / Approval Evidence: 없음

## Implemented Reality

- Actual: 미적용. registry, image, workflow, package/ACR 권한을 만들지 않았다.
- Applied Date: 없음
- Difference from Recommendation / User Decision: 해당 없음

## Risks Accepted

없음. private image pull·patching·취약점 대응·비용/usage monitoring을 검증 전 수용한 것으로 표시하지 않는다.

## Mitigations

- digest pinning, Dockerfile code review, 취약점/patch lifecycle, package/registry 최소 권한을 적용한다.
- dev CI에서 private image 사전 pull, build/test, image 갱신 rollback을 검증한다.

## Revisit Trigger

- Azure private registry 표준 또는 private endpoint 요구
- VM runtime container 전환
- GitHub Container registry 과금/정책 변경
- image 용량·pull 시간·취약점 대응 목표 미충족

## Evidence

- Requirement: [REQ-19](../requirements.md)
- Design: [CI/CD 단계와 build image 선택](../designs/ci-cd-stages-and-build-image-options.md)
- Official sources: [GitHub Packages billing](https://docs.github.com/en/enterprise-cloud@latest/billing/concepts/product-billing/github-packages), [GitHub Actions billing](https://docs.github.com/en/enterprise-cloud@latest/billing/concepts/product-billing/github-actions), [ACR managed identity](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication-managed-identity)
