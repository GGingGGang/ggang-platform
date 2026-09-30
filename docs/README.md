# 문서 목차

루트 README의 개요를 읽은 뒤, 이 폴더의 `01`~`08` 문서를 순서대로 읽으시면 됩니다. 템플릿·배포·실패 대응·운영 측정의 상세 기록은 `09-evidence/`에 모았습니다. 번호는 읽기 순서이며, 실행 날짜는 각 기록의 파일명과 본문에 표시합니다.

<a id="구현-설명"></a>

## 본문 읽기 순서

| 순서 | 문서 | 확인할 내용 |
|---|---|---|
| 0 | [프로젝트 개요](../README.md) | 목적·구현 저장소·기여·대표 결과 |
| 1 | [아키텍처](01-architecture.md) | 저장소의 책임과 전체 연결 |
| 2 | [인프라](02-infrastructure.md) | OCI 자원·클러스터·초기 설치 |
| 3 | [서비스 온보딩](03-service-onboarding.md) | 템플릿 생성과 수동 준비 작업 |
| 4 | [CI/CD](04-delivery.md) | 테스트·이미지 검사·서명·GitOps 갱신 |
| 5 | [네트워크](05-network-entry.md) | 외부 요청의 DNS·TLS·라우팅 |
| 6 | [운영](06-operations.md) | 관측·알림·데이터 보존·복구 |
| 7 | [설계 결정](07-decisions.md) | 스택별 선택 이유와 대안 |
| 8 | [현재 상태](08-status.md) | 클러스터·배포·운영 현황 |

## 실행 기록과 원자료

본문의 근거를 확인할 때 읽으시면 됩니다. 각 부록에서 연결된 본문으로 돌아갈 수 있습니다.

| 순서 | 부록 | 연결된 본문 |
|---|---|---|
| E01 | [템플릿 생성 검증](09-evidence/E01-2026-09-30-template-generation.md) | [3. 서비스 온보딩](03-service-onboarding.md) |
| E02 | [배포 추적](09-evidence/E02-2026-09-30-onboarding-deployment.md) | [4. CI/CD](04-delivery.md) |
| E03 | [GitOps push 경합](09-evidence/E03-2026-09-26-gitops-push-conflict.md) | [4. CI/CD](04-delivery.md) |
| E04 | [기존 CI 실패 차단](09-evidence/E04-2026-09-30-ci-failure-boundary.md) | [4. CI/CD](04-delivery.md) |
| E05 | [플랫폼 CLI 관찰](09-evidence/E05-2026-09-30-cli-observation.md) | [5. 네트워크](05-network-entry.md) |
| E06 | [자원·빌드 시간·비용](09-evidence/E06-2026-09-30-platform-measurements.md) | [6. 운영](06-operations.md) |

원자료: [클러스터·배포·측정 JSON](09-evidence/data).

---

[← 0. 프로젝트 개요](../README.md) · [1. 아키텍처부터 읽기 →](01-architecture.md)
