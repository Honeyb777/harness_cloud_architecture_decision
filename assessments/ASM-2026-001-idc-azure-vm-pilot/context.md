# Context — ASM-2026-001

## 확인된 사실과 사용자 의향

| 구분 | 내용 | 근거 |
|---|---|---|
| 현행 | IDC 이중화 서버에서 Tomcat/Java 및 Vue/npm 산출물을 Nginx와 함께 서비스 | 사용자 최초 설명 |
| 현행 | GitLab push 이벤트를 Jenkins가 수신, Secrets 값을 참조해 Jenkinsfile 단계 실행 | 사용자 최초 설명; Secrets 제품은 미확인 |
| 현행 | Nexus에 배포 파일을 올리고 빌드 의존성·컨테이너 이미지를 캐싱 | 사용자 최초 설명 |
| 현행 | dev는 Ansible inventory의 host 접근 정보를 이용해 SSH와 같은 방식으로 playbook 자동 실행 | 사용자 최초 설명 |
| 현행 | 운영은 SE가 L4 네트워크 팜에서 서버를 제외, 배포·시험 후 재투입 | 사용자 최초 설명 |
| 목표/진행 중 | IDC 일부를 Azure VM 파일럿으로 이전, Azure에서는 Application Gateway 사용 | 사용자 최초 설명; 실제 리소스 조회 없음 |
| 확정 의향 | GitHub Enterprise Cloud와 GitHub Copilot 사용 예정 | 최초 설명 및 후속 답변 |
| 선호 | GitHub Actions 최대 활용, self-hosted runner 가능하면 미사용, 장기 비밀번호·서비스 계정 비밀값 제거 | 사용자 최초 설명 |
| 검토 요청 | Blob 또는 Actions Artifacts에 일정 개수 버전 보관, Artifacts의 별도 용도 검토 | 사용자 최초 설명 |
| 검토 요청 | probe용 파일로 제외/복귀, 이중화 VM에 여러 서비스가 있을 때 동시 배포 방지 | 사용자 최초 설명 |
| 추가 확인 | 서비스마다 별도 Tomcat/JVM | 사용자 후속 답변 |
| 사용자 결정 | 기존 GitLab/Jenkins/Nexus/Ansible을 새 구성에서 미사용 | 후속 요청; 기존 제품 삭제 실행 요청 아님 |
| 사용자 결정 | Actions Cache 및 Key Vault 도입 | 후속 요청 |
| 추가 확인 | 설치 모듈은 빌드 runner 도구·라이브러리 | 후속 질문 답변 |
| 사용자 결정 | 다른 port의 JVM 병행 기동 미사용 | 후속 요청 |
| 추가 확인 | JWT Cookie 세션 방식 | 후속 요청 |
| 수용 위험 | DB schema/API 공존 문제를 감수하는 방향 | 후속 요청; 상세 영향·복구는 미검증 |
| 미확인 | VM/Blob/Key Vault 공용·사설 접근 정책 | 기존 미정 답변 및 외부 접근 비교 요청 |

## 환경 목록

| ID | 환경 | 용도 | 확정 범위 | 미확인 |
|---|---|---|---|---|
| ENV-01 | 기존 IDC | 현재 dev/운영 및 내부 DevOps | GitLab/Jenkins/Nexus/Ansible/L4 | 사이트·망·담당자·각 dev/운영 호스트 매핑 |
| ENV-02 | Azure 파일럿 | 일부 서비스 이전 | VM, 이중화 구성 설명, Application Gateway | 구독·RG·리전·OS·VM명·SKU, dev/prod 실제 분리 여부 |
| ENV-03 | GitHub Enterprise Cloud | 코드·CI/CD·Copilot | Cloud 형태 사용 예정 | Enterprise/Org/Repo, github.com 또는 데이터 거주형 도메인, 관리자·팀 |

문서의 dev/prod와 vm-01/vm-02는 논리적 예시다. 같은 서버를 dev/prod가 공유한다고 가정하지 않는다.
Linux/Nginx·WAR 기준 상세 설명은 잠정 가정이며 OS·패키징·버전 확인 후 수정한다.

## 워크로드

| ID | 내용 | 배치 | 확인 사항 | 미확인 |
|---|---|---|---|---|
| WL-01 | Java/Tomcat 서비스군 | ENV-01 일부 → ENV-02 | 서비스별 독립 JVM, JWT Cookie, 다른 port의 JVM 병행 제외 | 서비스명·수, Java/Tomcat 버전, WAR/JAR, JWT 키 구성, DB/배치 |
| WL-02 | Vue/npm 프론트 서비스군 | ENV-01 일부 → ENV-02 | Nginx 정적 배포 요구 | Vue/Node/npm, Vite 여부, URL 경로, 캐시·service worker |
| WL-03 | 빌드/배포 체계 | ENV-01 → ENV-03 + ENV-02 | 목표는 내부 DevOps 4개 제품 미사용, Actions Cache, Key Vault | 빌드 도구 목록, 사내 패키지 원본 이전, runner 경로, 승인자 |

## 통신 목록

| ID | 출발 → 도착 | 목적 | 상태 / 다음 확인 |
|---|---|---|---|
| INT-01 | GitLab → Jenkins | push 이벤트 | 현행; webhook 인증 방식 미확인 |
| INT-02 | Jenkins ↔ Nexus | 의존성/이미지 취득, 배포 파일 게시 | 현행; 제품 버전·인증·프록시 구성 미확인 |
| INT-03 | Ansible → IDC host | dev 배포 | 현행; 실제 SSH/계정 정책 미확인 |
| INT-04 | GitHub runner → Entra/ARM | OIDC 교환, VM 명령, App Gateway 상태 | 권장안; 443 outbound 정책 확인 |
| INT-05 | GitHub runner → 의존성 원본/Cache/GHCR/Blob | 빌드·이미지·릴리스 업로드 | Nexus 제외; Blob private-only면 VNet 경로 필요 |
| INT-06 | Azure VM → Azure 플랫폼/Blob/Key Vault | VM Agent 통신, MI artifact·secret 조회 | 권장안; Agent·egress·private DNS 확인 |
| INT-07 | App Gateway → Nginx/서비스 | 사용자 요청·서비스별 probe | 사용 중이라는 설명; host/port/path/settings 매핑 미확인 |
| INT-08 | Azure 서비스 → IDC DB/인증/API/파일 | 이전하지 않은 의존성 | 존재 여부부터 확인; 대역폭·지연·TLS·회선·장애 경로 |

## Unknowns

담당자는 아직 지정되지 않았다. 담당 역할은 확인을 조정할 역할이며 실제 조직 담당자를 임의로 배정한 것이 아니다.

| ID | 확인 항목 | 담당 역할 | 확인 방법 | 차단 Gate |
|---|---|---|---|---|
| UNK-01 | tenant/subscription/RG, 조직·repo·환경·서버 쌍 및 책임자 | Orchestrator / Security | 사용자·리소스 목록 | DISCOVERY / OIDC 확정 |
| UNK-02 | 리전·GitHub 도메인·데이터 등급/거주·감사 기준 | Cloud / Security | 정책 및 계약 | STEP-02 / 신뢰 issuer 확정 |
| UNK-03 | Blob·Key Vault·VM·Azure Agent egress 및 잔여 업무 IDC 연결 | Network | 통신 매트릭스·DNS/접속 확인 | runner/배포 경로 확정 |
| UNK-04 | OS·Java/Tomcat/Vue/Node/npm 버전, 빌드·실행 형태 | DevOps | pom/build.gradle/package-lock/Jenkinsfile 등 | 배포 스크립트 확정 |
| UNK-05 | pool/settings/probe, 공유 Nginx·host/port, affinity 설정 | Network / Operations | App Gateway 및 Nginx 설정 | drain 시험 |
| UNK-06 | JWT 키·검증 일관성, WebSocket/SSE/장시간 요청·batch, timeout | DevOps / Operations | 앱·트래픽·로그 | 무중단 수용 기준 |
| UNK-07 | 한 대의 모든 서비스 수용 용량·장애 허용·SLO | Operations | 부하 시험 | 운영 배포 허용 |
| UNK-08 | 보관 버전 N, 최대 브라우저 자산 수명, rollback 기간, 예산 | Data / FinOps | 사용자 기준 | 보존/정리 정책 |
| UNK-09 | 승인자·분리 의무·SE 역할·긴급 복구 및 break-glass | Security / Operations | 조직 정책 | prod 승인 Gate |
| UNK-10 | IDC DB/인증 의존성·DB migration·백업·복구 기준 | Data / Network | 의존성 조사 | 이전·rollback |

## 후속 결정 참조

[DEC-2026-003](decisions/DEC-2026-003-hosted-only-scope.md)에 목표 도구 제외·Key Vault/Cache·JVM·JWT 및 수용 위험을 기록했다.
ENV-01의 도구·연결 설명은 현행 이력이며 목표 구성에 유지한다는 뜻이 아니다. 업무 DB/API 등 잔여 IDC 의존성은 별도로 확인한다.
