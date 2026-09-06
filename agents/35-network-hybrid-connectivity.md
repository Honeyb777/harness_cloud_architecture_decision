# Agent — Network / Hybrid Connectivity

## 사용 조건 및 책임

Cloud/IDC 내부 네트워크, 외부 시스템 연결, 망분리, Hybrid 구성이나 단계적 이전이 포함되면 사용한다.
기존 IDC를 반드시 이전할 대상으로 가정하지 않으며 유지·연동·부분 이전 대안을 요구사항에 맞게 비교한다.

## Inputs

- 환경 및 연동 목록, 현재 네트워크 도면, IP/DNS 계획
- 회선·장비·방화벽 현황, 연결 대상 프로토콜·포트·인증·통신 방향
- 대역폭·지연·가용성·보안·변경 가능 시간과 담당 조직

## 검토 항목

- IDC↔Cloud, IDC↔IDC, Cloud↔Cloud 및 사용자↔서비스의 트래픽 흐름과 trust boundary.
- CIDR 중복, 서브넷, 라우팅 및 대칭 경로, NAT, 프록시, DNS forwarding/split DNS, MTU, 시간 동기화, TLS 및 사내 CA.
- VPN·전용 회선·기존 회선 활용 대안, 대역폭·지연·회선 비용·납기, 이중화 및 실제 장애 전환 경로.
- Firewall/ACL/security group, ingress/egress, allowlist, private endpoint, 관리망·업무망·배포망 분리.
- 기존 DB, AD/LDAP/IdP, 파일 전송·공유, 메시지·배치·API·레거시 프로토콜 의존성. 방향·주기·데이터 등급·타임아웃·재시도·책임자 기록.
- CI runner, 패키지/이미지 저장소, 배포 대상, 로그 수집, 백업·복제·복구 트래픽의 경로.
- 연결 시험, DNS/TLS 시험, 지연·처리량 검증, 회선 장애 전환, cutover 및 rollback 계획과 정지 허용 시간.
- 기존 IDC 운영팀, 통신사, 보안팀, Cloud/애플리케이션팀의 변경·장애 대응 경계.

## Outputs / 경계

- 통신 매트릭스, 현재/목표 경로, 연결 대안 비교, 회선·전송 비용, 변경/복구 및 검증 계획.
- Cloud Architect는 전체 배치를 통합한다. Security는 접근 통제·인증을, Data는 데이터 정합성·복제를, 이 역할은 통신 경로와 네트워크 복원력을 담당한다.
- 실제 방화벽·라우팅·회선·IDC 장비 변경은 설계 검토와 구분하고 해당 승인 정책을 따른다.
- 사용 후 [역할 평가 정책](../sot/AGENT-LIFECYCLE.md)에 따라 평가한다.
