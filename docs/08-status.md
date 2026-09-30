# 플랫폼 운영 현황

| 순서 | 문서 | 이동 |
|---|---|---|
| 7 | [설계 결정](07-decisions.md) | 이전 |
| **8** | **현재 상태** | **현재** |
| 부록 | [부록 목록](README.md#실행-기록과-원자료) | 다음 |

[0. 프로젝트 개요](../README.md) · [전체 읽기 순서](README.md) · [부록 목록](README.md#실행-기록과-원자료)

기준일: **2026-09-30**. 클러스터와 배포 상태는 [CLI 기록](09-evidence/E05-2026-09-30-cli-observation.md)에, 자원과 빌드 시간은 [측정 기록](09-evidence/E06-2026-09-30-platform-measurements.md)에 정리했습니다.

## 구현과 운영 결과

| 영역 | 구현·운영 결과 | 관련 기록 |
|---|---|---|
| OCI/OKE | Terraform으로 기반을 구성하고 ARM64 워커 2대를 운영합니다. 노드 2/2와 실행 Pod 52개의 컨테이너가 Ready 상태입니다. | [인프라](02-infrastructure.md), [CLI](09-evidence/E05-2026-09-30-cli-observation.md) |
| 외부 요청 | 웹·core readiness 경로에서 DNS·TLS와 HTTP 200 응답을 확인했습니다. | [요청 경로](05-network-entry.md) |
| 서비스 템플릿 | JavaScript 템플릿의 로컬 생성·테스트 6/6·정적 빌드·Kustomize 렌더링을 통과했습니다. | [생성 기록](09-evidence/E01-2026-09-30-template-generation.md) |
| 공통 CI·GitOps | batch #23의 소스·라이브러리·이미지 digest·GitOps 커밋·ArgoCD 이력·실행 imageID를 연결했습니다. | [배포 추적](09-evidence/E02-2026-09-30-onboarding-deployment.md) |
| 동시 배포 | batch의 GitOps push 재시도에 성공했고, 먼저 반영된 core 변경을 보존했습니다. | [경합 해결](09-evidence/E03-2026-09-26-gitops-push-conflict.md) |
| 실패 차단 | batch #13의 Test 컴파일 실패 후 이미지 빌드·검사·서명·배포 태그 갱신을 건너뛰었습니다. | [실패 기록](09-evidence/E04-2026-09-30-ci-failure-boundary.md) |
| 관측·알림 | 관측 도구가 Ready 상태이며, 9/24 batch TargetDown의 발생·해소 알림을 Discord에서 수신했습니다. | [운영](06-operations.md) |
| 자원·빌드 시간 | 노드 자원 표본과 batch·core·web의 최근 성공 빌드 각 5건의 소요 시간을 기록했습니다. | [측정](09-evidence/E06-2026-09-30-platform-measurements.md) |
| OCI 비용 | 비용 조회 화면의 2026년 6~9월 월별 합계와 기간 총합은 0.00 SGD입니다. | [비용 조회](09-evidence/E06-2026-09-30-platform-measurements.md#비용-조회) |

## GitOps 동기화 상태

노드 2대·실행 Pod 52개가 Ready이고 ArgoCD 앱 32개는 Healthy입니다. 23개는 Synced, 9개는 OutOfSync입니다.

auth·core·web은 HTTPRoute만 차이가 나며 live 참조 group/kind·weight를 보완하면 Git spec과 일치합니다. 세 서비스의 host·path·rewrite·port와 실행 이미지 버전은 일치합니다.

istio-gateway·istiod·kps·kyverno·nats 등의 차이는 별도로 남아 있습니다. kps의 과거 Helm failed 이력과 현재 ArgoCD Healthy·grafana Secret OutOfSync도 구분합니다. 대상은 [관찰 기록](09-evidence/E05-2026-09-30-cli-observation.md)을 따릅니다.

---

<!-- reading-nav -->
[← 7. 설계 결정](07-decisions.md) · [부록 목록 →](README.md#실행-기록과-원자료) · [전체 읽기 순서](README.md)
<!-- /reading-nav -->
