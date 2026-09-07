# DEC-2026-008 — Frontend 정적 자산 rollout 모델

## Status

PROPOSED

## Traceability

- Assessment: ASM-2026-001
- Applicable Environment IDs: ENV-02
- Applicable Workload / Integration IDs: WL-02, INT-07
- Step: STEP-01 / DISCOVERY; STEP-05 준비 조사
- Approval Level: USER_DECISION
- Created / Updated: 2026-09-07

## Context / Requirement / Constraints

Nginx current/slot 전환 중 이전 HTML을 가진 browser가 이전 정적 자산을 요청할 수 있다. REQ-10, REQ-17을 따른다. 확인되지 않은 frontend 이슈는 사용자 확인 전 이 결정의 조건으로 추가하지 않는다.

## Options

### Option A — 단일 active slot

**Advantages**: 저장·정리·운영이 단순하다.

**Disadvantages**: 이전 정적 자산 요청이 실패할 수 있으며 사용자 영향 또는 재로딩 필요성을 수용해야 한다.

**Cost / Operational / Security / Global Impact**: 저장 공간은 작다. VM1/VM2 전환 중 자산 요청 실패를 감시해야 한다. 별도 보안·global 영향은 현재 확인되지 않았다.

### Option B — hash/version 자산 두 버전 공존

**Advantages**: 이전 HTML의 자산 URL과 새 release 자산을 함께 제공한다.

**Disadvantages**: 두 release 자산의 저장·보존·cleanup·양 VM 준비를 관리한다.

**Cost / Operational / Security / Global Impact**: 최소 두 release 자산의 저장량과 배포 시간을 관리한다. current/previous 보호 집합과 cleanup 검증이 필요하다. 별도 보안·global 영향은 현재 확인되지 않았다.

**Implementation Example**: [선택안의 r101→r102 예시](../designs/frontend-static-asset-rollout-options.md)를 따른다. 두 VM에 previous/current 자산을 먼저 준비하고, versioned URL을 유지한 뒤 HTML current를 순차 전환한다.

## Agent Recommendation

Option B를 권장한다. Option A는 이전 자산 요청의 사용자 영향을 명시적으로 수용할 때 선택한다.

## User Decision

- Selected: 미결정 — Option A 또는 B 선택 필요
- Decision Date / Owner / Reason / Approval Evidence: 없음

## Implemented Reality

- Actual: 미적용. Nginx path, hash/version asset, current pointer, retention/cleanup 설정을 변경하지 않았다.

## Risks Accepted

없음.

## Mitigations

- Option B는 두 VM에 current/previous 자산을 준비·검증하고 protected release를 cleanup에서 제외한다.
- 실제 browser 자산 요청은 파일럿에서 검증한다.

## Revisit Trigger

- 사용자 영향 또는 자산 요청 실패 관측
- asset 크기·보존 비용 증가
- frontend release/rollback 정책 변경

## Evidence

- [Frontend 정적 자산 rollout 선택안](../designs/frontend-static-asset-rollout-options.md)
- [배포 release 생명주기](../designs/build-cache-and-release-lifecycle-options.md)
