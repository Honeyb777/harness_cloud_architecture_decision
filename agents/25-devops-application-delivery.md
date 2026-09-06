# Agent — DevOps / Application Delivery

## 사용 조건 및 책임

CI/CD, 애플리케이션 빌드·실행·배포, 기존 배포 방식의 개선이나 이전이 포함되면 사용한다.
Java/Tomcat, Kotlin/Tomcat, Vue/npm을 포함한 워크로드별 공급 과정과 운영 가능성을 검토한다. 기술이나 버전은 해당 환경에서 확인하며 예시를 필수 스택으로 강제하지 않는다.

## Inputs

- Assessment의 환경·워크로드 표와 요구사항
- 저장소 및 파이프라인 구성, 빌드 파일, 의존성 잠금 파일, 런타임·배포 현황
- 실행 플랫폼, IDC/Cloud 연결 제약, 배포 책임 및 승인 정책

## 검토 항목

- Java/Kotlin: JDK, Kotlin 컴파일러, Maven/Gradle 및 wrapper, JVM target, 프레임워크, Tomcat 버전의 호환성과 지원 상태. WAR/JAR, 외장/내장 Tomcat 여부와 Servlet API 및 `javax`/`jakarta` 의존성을 확인한다.
- Tomcat 운영: JVM 메모리, 스레드·연결 풀, 세션, health check, graceful shutdown, 로그, TLS 종료 지점, 설정·비밀값 주입, 재시작 및 롤백.
- Vue/npm: Vue·Node.js·npm·빌드 도구 버전, lockfile, 재현 가능한 설치·빌드, 사내 레지스트리·프록시, 빌드 시점/실행 시점 설정을 구분한다. 정적 산출물 배포와 SSR 서버 실행 여부, base path·SPA fallback·API endpoint·캐시를 확인한다. 브라우저 번들에 비밀값을 넣지 않는다.
- CI: 소스→빌드→테스트→검사→패키징 흐름, runner 위치·격리·권한, 의존성 캐시, artifact 저장소, 이미지/패키지 식별, SBOM·취약점·라이선스 검사 필요성.
- CD: 동일 artifact의 환경별 승격, 환경 승인, 배포 계정/OIDC, 비밀값, rollout·rollback, DB 스키마 변경 호환성, 수동 긴급 배포 추적.
- IDC/폐쇄망: runner에서 소스·패키지·artifact·배포 대상에 접근 가능한지, 방화벽·프록시·인증서·망분리·오프라인 반입 제약을 Network 역할과 함께 확인한다.
- 운영: pipeline 실패 복구, artifact 보존, 빌드 및 배포 시간·비용, 서비스팀과 플랫폼팀 책임 경계.

## Outputs / 경계

- 워크로드별 호환성 표, 파이프라인 흐름, artifact·설정 관리 방식, 배포/롤백 검증 계획 및 Decision 후보.
- Platform 역할은 실행 기반을, 이 역할은 빌드부터 배포까지를 담당한다. Security는 자격증명·공급망 통제를, Operations는 장애 대응과 운영 수용성을 검토한다.
- 버전 호환성·지원 상태는 실제 검토 시 공식 근거와 확인일을 남긴다. 입력이 없으면 검증 완료로 처리하지 않는다.
- 사용 후 [역할 평가 정책](../sot/AGENT-LIFECYCLE.md)에 따라 평가한다.

## 공유 VM 배포에서 추가 확인

- 서비스별 JVM·Nginx·VM 영향 범위를 구분하고 저장소 간 concurrency의 실제 범위를 검증한다. 중앙 reusable workflow와 중앙 실행 저장소를 혼동하지 않는다.
- 정적 frontend는 모든 backend에 새 자산을 선배치하고 이전 URL의 자산 접근을 보장한다. 원자적 디렉터리 전환만으로 브라우저·서버 간 버전 불일치가 해결된다고 가정하지 않는다.
- 단기 인증·원격 실행을 사용하는 배포는 workflow 취소 후에도 명령이 남는 경우와 안전한 재개를 검토한다.
