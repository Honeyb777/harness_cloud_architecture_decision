# Frontend 정적 자산 rollout 선택 — 단일 slot 또는 hash 자산 2버전 공존

- Assessment: ASM-2026-001
- Step / Phase: STEP-01 / DISCOVERY; STEP-05 준비 조사
- Status: PROPOSED — 선택·Nginx 설정·실제 browser 시험은 미수행

Nginx current/slot 전환 중 이전 HTML을 가진 browser가 이전 JavaScript/CSS 파일을 요청할 수 있다. 이 자산 요청을 어떻게 처리할지 선택한다.

| 선택지 | 전환 뒤 이전 자산 요청 | 운영 형태 |
|---|---|---|
| A. 단일 active slot | 이전 파일이 없으면 실패/재로딩 전 사용자 영향 가능 | 새 release만 제공 |
| B. hash/version 자산 2버전 공존 | 이전 release hash URL도 계속 제공 | 새·이전 release 자산을 함께 제공 |

## Option A — 단일 active slot

Nginx `current`를 새 release로 전환하고 이전 release 자산을 더 이상 제공하지 않는다. 이미 열린 페이지가 이전 JS/CSS를 요청하면 실패할 수 있으며, 그 사용자 영향 또는 재로딩 필요성을 수용한다. 저장·정리·운영은 단순하다.

## Option B — hash/version 자산 두 버전 공존

release별 hash 또는 version 경로를 사용해 새 release와 이전 release의 정적 자산을 함께 제공한다. HTML 진입점은 새 release로 전환하되, 이전 HTML이 참조한 자산 URL도 두 VM에서 계속 제공한다.

```text
/srv/<service>/frontend/
  releases/<new-release>/index.html
  assets/<previous-release>/...
  assets/<new-release>/...
  current -> releases/<new-release>
```

최소 `current`와 `previous` 두 release 자산을 cleanup 보호 집합으로 둔다. 두 VM에서 새·이전 자산이 준비된 뒤 HTML 전환을 진행하고, 보호 해제 전에는 이전 자산을 지우지 않는다. 저장·보존·cleanup 책임은 증가한다.

### 예시: release `r101`에서 `r102`로 전환

`r101`의 HTML이 `/assets/app.a1b2.js`를 참조하고, `r102`의 HTML이 `/assets/app.c3d4.js`를 참조한다고 가정한다. 파일 이름의 hash는 예시이며 실제 frontend build가 어떤 이름을 만드는지는 아직 확인하지 않았다.

```text
/srv/web-portal/frontend/
  releases/
    r101/
      index.html                 # /assets/app.a1b2.js 참조
    r102/
      index.html                 # /assets/app.c3d4.js 참조
  assets/
    r101/
      app.a1b2.js
      app.e5f6.css
    r102/
      app.c3d4.js
      app.g7h8.css
  current -> releases/r102
```

Nginx는 HTML entry와 versioned asset 경로를 구분해 제공한다. 다음은 경로 역할을 설명하는 예시이며 실제 server block·cache header·TLS 설정을 확정한 구성은 아니다.

```nginx
# 새 방문자는 current HTML을 받는다.
location = / {
    try_files /current/index.html =404;
}

# 이전과 새 release의 자산 URL은 version별 디렉터리에서 계속 제공한다.
location /assets/r101/ {
    alias /srv/web-portal/frontend/assets/r101/;
}

location /assets/r102/ {
    alias /srv/web-portal/frontend/assets/r102/;
}
```

실제 build 결과의 asset URL도 위 구조와 맞게 release별 경로를 포함해야 한다. 예를 들어 r101 HTML이 `/assets/r101/app.a1b2.js`를, r102 HTML이 `/assets/r102/app.c3d4.js`를 참조한다. `current/assets/...`처럼 공통 경로만 쓰면 current를 r102로 바꾼 뒤 r101 HTML이 r102 경로를 요청할 수 있으므로, 두 release 공존의 목적을 달성하지 못한다.

### 두 VM 배포 순서 예시

| 순서 | VM1 | VM2 | 확인 값 |
|---|---|---|---|
| 1 | r101 자산 유지, r102 자산을 staging에 준비 | r101 자산 유지, r102 자산을 staging에 준비 | r102 파일 digest, r101/r102 asset URL 모두 200 |
| 2 | r102 자산을 versioned path에 게시 | r102 자산을 versioned path에 게시 | 어느 VM에서 요청해도 r101/r102 asset URL이 존재 |
| 3 | `current`를 r102 HTML로 전환 | r101 HTML 유지 | VM1 r102 HTML과 r101/r102 asset URL 확인 |
| 4 | r102 HTML·자산 확인 | `current`를 r102 HTML로 전환 | 두 VM의 current=r102, 이전 asset URL 유지 |
| 5 | r101 asset 보호 상태 유지 | r101 asset 보호 상태 유지 | current=r102, previous=r101 기록 |

VM별 `current` 전환은 원자적으로 할 수 있지만 두 VM이 같은 순간에 전환되지는 않는다. 그래서 **HTML 전환보다 먼저 양 VM에 r101과 r102 자산을 모두 준비**한다. 이 순서는 사용자가 요청한 이전 파일 호출 처리를 위한 조건이며, 별도의 frontend 동작을 가정하지 않는다.

### rollback과 cleanup 예시

- r102 배포 뒤 문제가 나면 두 VM의 `current`를 r101로 되돌린다. r101 asset은 보호 중이므로 다시 게시하지 않아도 된다.
- r103이 정상 배포되면 `current=r103`, `previous=r102`가 된다. r101 asset은 보호 집합에서 빠진 뒤에야 cleanup 후보가 된다.
- cleanup job은 release 상태와 양 VM의 current/previous 기록을 먼저 읽고 dry-run 목록을 남긴다. `current`, `previous`, in-flight, rollback-pinned 자산은 삭제하지 않는다.
- 실제 보존 기간과 cleanup 시점은 아직 미결정이다. 이 예시는 “두 버전을 즉시 합쳐 두고, 이전 버전 자산을 보호한다”는 B안의 최소 동작만 설명한다.

이 문서는 이전 정적 자산 호출 문제만 다룬다. frontend의 추가 이슈는 사실과 영향이 확인되기 전까지 이 설계·결정에 반영하지 않는다. 확인이 필요한 이슈가 생기면 현상, 영향 범위, 재현 방법을 먼저 기록하고 사용자 확인 뒤 후속 설계에 반영한다.

현재 권장안은 Option B다. 사용자 선택은 [DEC-2026-008](../decisions/DEC-2026-008-frontend-static-asset-rollout.md)에 기록한다.
