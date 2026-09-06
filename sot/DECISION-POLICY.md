# Decision Policy

의사결정을 다음 세 단계로 구분한다.

## AUTO

Agent가 기존 정책과 요구사항을 근거로 결정할 수 있다.

예:
- 문서 형식
- 비교표 구조
- 기존 Tagging Standard에 맞춘 태그명 적용
- 명백한 Naming Convention 적용
- Read-only 조사 방식

## USER_DECISION

사용자가 Step 시작 전에 선택해야 한다.

예:
- Cloud Provider
- Account / Subscription / Tenant 구조
- Region 전략
- Network topology
- Kubernetes 채택 여부
- 주요 Database 선택
- Multi-region 여부
- Backup/Retention 기준
- RTO/RPO
- 비용 상한선
- 주요 Managed Service 선택
- Public/Private connectivity 정책
- 데이터 저장 국가/지역
- 감사/규제 기준
- IaC 도구 표준
- 공유 플랫폼과 서비스팀 책임 경계

## EXPLICIT_APPROVAL_REQUIRED

높은 위험 또는 조직적 영향이 큰 결정.

예:
- Production 데이터 삭제
- Backup/Retention 축소
- 암호화 또는 보안통제 완화
- Public exposure 확대
- Identity trust 변경
- Root/Global Admin 수준 권한 변경
- Landing Zone 정책 예외
- Multi-account/subscription 구조 대규모 변경
- 감사 로그 비활성화
- 실제 Cloud Resource 생성/삭제

## Decision Gate Format

각 Decision Gate는 다음을 포함한다.

- Decision ID
- Context
- Constraints
- Option A/B/C
- 장점
- 단점
- 비용 영향
- 운영 영향
- 보안/감사 영향
- 글로벌 확장 영향
- Recommended
- User Decision
- Decision Date

## DevOps / IDC 적용

문서 역할의 추가·보완·평가와 검토별 디렉터리 정리는 AUTO다.
CI/CD 표준, 주요 런타임 변경, artifact/runner 운영 방식, 환경 승격 정책, IDC 연결 및 이전 전략은 USER_DECISION으로 분리한다.
실제 운영 배포, 운영 runner/배포 권한 변경, IDC 방화벽·라우팅·회선·장비 변경과 cutover는 EXPLICIT_APPROVAL_REQUIRED로 다룬다.
기존 사용자 승인은 대상 환경·워크로드·변경 범위가 일치할 때 재사용한다. dev나 타 검토의 승인을 확대 해석하지 않는다.
