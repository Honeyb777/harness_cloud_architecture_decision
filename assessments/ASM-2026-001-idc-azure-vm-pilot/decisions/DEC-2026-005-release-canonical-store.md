# DEC-2026-005 — 승인된 VM 배포 release의 canonical store

## Status

PROPOSED

## Traceability

- Assessment: ASM-2026-001
- Applicable Environment IDs: ENV-02/03
- Applicable Workload / Integration IDs: WL-01/02/03, INT-05/06
- Step: STEP-01 / DISCOVERY; STEP-06 설계 선행 조사
- Approval Level: USER_DECISION
- Created / Updated: 2026-09-07
- Supersedes: 없음

## Context

빌드 결과물은 dev/prod에서 같은 digest로 승격하고 rollback할 수 있어야 한다. Actions Cache는 이를 보관할 저장소가 아니며, GitHub Packages는 사내 dependency와 image용 registry다. VM이 passwordless로 release를 pull하는 경로와 보존·정리 책임을 선택해야 한다.

## Requirement

- REQ-07, REQ-13, REQ-19 및 [release lifecycle 선택안](../designs/build-cache-and-release-lifecycle-options.md)을 따른다.
- current/previous/in-flight/rollback-pinned release와 frontend 호환 자산은 정리에서 보호한다.

## Options

### Option A — Azure Blob을 canonical release store로 사용하고 Artifacts는 증적으로 사용

**Advantages**

- VM Managed Identity가 Storage RBAC로 승인된 release를 직접 pull한다.
- immutable release 경로·digest·lifecycle·soft delete를 release 수명과 연결하기 쉽다.

**Disadvantages**

- Storage network, RBAC, lifecycle, cleanup identity를 설계·검증해야 한다.

**Cost / Operational / Security / Global Impact**

Storage 용량·요청·전송·보호 기능 비용을 관리한다. VM MI에는 read만, publisher와 cleanup identity에는 분리된 최소 data-plane 권한을 준다. private endpoint/region·DR 요구가 생기면 storage topology를 확장한다.

### Option B — GitHub Actions Artifacts만 canonical release store로 사용

**Advantages**

- CI run과 결과물 연결이 단순하다.

**Disadvantages**

- VM에 GitHub artifact download 권한과 정확한 run/artifact ID 경로를 추가로 설계해야 한다.
- retention 만료·run 삭제·교차 repository 권한이 rollback 원본에 영향을 준다.

**Cost / Operational / Security / Global Impact**

Artifacts·Packages shared storage와 1~400일 retention 상한을 관리한다. Azure VM에서 GitHub 접근 token을 다루는 경로가 늘어난다. GitHub 서비스·도메인 정책 의존성이 커진다.

## Agent Recommendation

Option A를 권장한다. Actions Artifacts는 테스트·SBOM·진단·단기 job 전달 증적에 남기고, Blob만 VM 배포·rollback의 canonical release 원본으로 사용한다.

## User Decision

- Selected: 미결정 — Option A 또는 B 선택 필요
- Decision Date / Owner / Reason / Approval Evidence: 없음

## Implemented Reality

- Actual: 미적용. Blob container, Artifact retention, VM access token, lifecycle/cleanup을 구성하지 않았다.
- Applied Date: 없음

## Risks Accepted

없음. N, rollback 기간, frontend asset 보호 기간, storage protection 및 cleanup 권한이 미결정이다.

## Mitigations

- immutable release ID·digest와 보호 집합을 사용한다.
- cleanup dry-run, 재검사, 별도 cleanup identity와 허용·거부 시험을 적용한다.

## Revisit Trigger

- private-only storage·VNet runner 요구
- 보존 기간·감사 요구 변경
- release 크기·서비스 수·rollback 빈도 증가
- Azure/GitHub 비용·가용성 요구 변화

## Evidence

- [Build cache·release lifecycle 선택안](../designs/build-cache-and-release-lifecycle-options.md)
- [검증 계획](../evidence/pilot-validation-plan.md)
