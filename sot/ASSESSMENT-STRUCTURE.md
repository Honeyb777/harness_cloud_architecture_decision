# 검토 건별 디렉터리와 환경 경계

## 관리 단위

하나의 Assessment는 하나의 의사결정 범위다. 서비스 한 개로 제한하지 않으며 여러 서비스, 공용 플랫폼, IDC, Cloud 및 Hybrid 환경을 함께 포함할 수 있다.
매번 다른 대상의 검토는 별도 Assessment로 생성한다. 같은 서비스라도 독립된 목적·일정·승인 범위라면 새 Assessment를 만들고 이전 검토를 참조한다.

공통 정책은 `sot/`, 역할은 `agents/`, 빈 양식은 `templates/`에 둔다. 환경별 요구사항이나 선택한 기술을 공통 정책에 자동으로 올리지 않는다.

## 디렉터리

아래는 실제 검토를 접수했을 때 만드는 구조다. 필요한 디렉터리만 생성하며 빈 검토 건을 미리 등록하지 않는다.

```text
assessments/
  README.md                         # 전체 검토 목록
  ASM-YYYY-NNN-<scope>/
    README.md                       # 범위, 상태, Step, 산출물 목록
    context.md                      # 조직·환경·워크로드·연동 현황
    requirements.md                 # 요구사항, 제약, 미확인 항목
    decisions/
      README.md                     # 이 검토의 결정 목록
      DEC-YYYY-NNN-<subject>.md      # 결정 원본
    designs/                        # 현재/목표 구조, 배포 및 네트워크 설계
    evidence/                       # 출처, 검증 요약, 승인 참조
    evaluations/                    # 역할별 사용 후 평가와 개선 내역
    reports/                        # 이 검토의 요약/상세 보고서
    history/                        # 이 검토의 변경 이력
decisions/README.md                  # 전체 결정 색인: 원본 링크만 관리
reports/                            # 공통 보고서 템플릿
history/                            # 하네스 변경 및 검토 간 요약 이력
examples/                           # 실제 검토에 포함하지 않는 예시
```

결정과 보고서의 원본은 해당 Assessment 안에만 둔다. 전역 목록에는 링크와 식별 정보만 기록하며 복사본을 만들지 않는다.
기존 원본을 이동할 때 링크와 색인을 함께 갱신하고 ID 및 승인 이력은 유지한다.

## 환경 및 워크로드 식별

- `context.md`는 [Context 템플릿](../templates/CONTEXT-TEMPLATE.md)에서 시작한다.
- 환경은 Assessment 내부의 `ENV-01`, 워크로드는 `WL-01`, 연동은 `INT-01`처럼 식별한다. 다른 검토를 참조할 때 Assessment ID를 함께 쓴다.
- dev/stage/prod는 예시이며 필수 구성이 아니다. Cloud, IDC, 폐쇄망, Hybrid 여부와 각 환경의 책임자·네트워크·제약을 개별 기록한다.
- 결정에는 적용 환경과 워크로드를 명시한다. dev 승인이나 타 프로젝트 결정을 prod 승인으로 확장하지 않는다.
- 동일 환경의 연속 작업은 기존 검토를 갱신할 수 있다. 검토 대상이 불명확하고 후보가 여러 개면 선택을 확인하며 최근 항목을 임의로 사용하지 않는다.
- 기존 검토는 참고 지식이다. 요구사항·호환성·가격·승인 범위의 재검증 없이 결론을 재사용하지 않는다.
