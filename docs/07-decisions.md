# 설계 결정

| 순서 | 문서 | 이동 |
|---|---|---|
| 6 | [운영](06-operations.md) | 이전 |
| **7** | **설계 결정** | **현재** |
| 8 | [현재 상태](08-status.md) | 다음 |

[0. 프로젝트 개요](../README.md) · [전체 읽기 순서](README.md) · [부록 목록](README.md#실행-기록과-원자료)

스택별 선택 이유와 대안, 운영 중 바꾼 판단을 모았습니다. **항목을 펼치면 본문과 근거를 이 페이지에서 읽을 수 있습니다.**

| 분야 | 결정 번호 | 다루는 내용 |
|---|---|---|
| [인프라와 네트워크](#infra) | 0001~0009 | 저장소 분리·OKE·Terraform·Gateway·접근 제어 |
| [개발과 배포](#delivery) | 0010~0017 | Jenkins·템플릿·CI·검사·서명·GitOps |
| [데이터와 시크릿](#data) | 0018~0021 | OpenBao·MySQL·Redis·NATS |
| [관측과 알림](#observability) | 0022~0023 | Prometheus·metrics-server·Discord |

<a id="decisions"></a>

<a id="infra"></a>

## 인프라와 네트워크

워커·진입점·DNS·인증서·관리 접근을 공통 기반으로 두고, 서비스별 설정과 권한의 소유 위치를 나눴습니다.

<a id="adr-0001"></a>

<details>
<summary>0001 · 인프라·템플릿·CI·배포 상태의 책임 분리</summary>

- 상태: 채택
- 기록일: 2026-09-22

**배경**

인프라 설정, 서비스 시작점, 빌드 절차, 앱 버전 배포는 서로 다른 이유로 바뀝니다. 이를 같은 변경 단위로 취급하면 개발자 소스 변경에 플랫폼 운영 변경이 섞이고, 공통 절차도 서비스마다 달라질 수 있습니다.

이 프로젝트는 AI를 애플리케이션 개발자로 두고, 플랫폼 담당자가 공통 실행·배포 경로를 제공하는 방식으로 진행합니다.

**현재 선택**

| 책임 | 원본 |
|---|---|
| 클라우드/클러스터와 플랫폼 운영 | oci-always-free-k8s |
| 언어별 서비스 시작 템플릿 | app-templates |
| 공통 테스트·빌드·검사·서명·태그 갱신 | jenkins-shared-library |
| 앱의 원하는 배포 상태 | k8s-gitops |
| 업무 코드·테스트 | 개발자가 관리하는 svc-* |
| 전체 설명·설계 판단·검증 근거 | ggang-platform |

CI는 앱 소스 저장소에 배포용 태그 변경을 되돌려 쓰지 않고 GitOps 저장소를 갱신합니다. ArgoCD는 GitOps의 선언을 적용합니다.

템플릿은 독립 저장소로 유지하며 업무 기능을 포함하지 않습니다. 생성 후의 앱 코드는 개발자가 소유하며, 템플릿 수정은 기존 앱에 자동 전파되지 않습니다.

**대안 비교**

| 방식 | 장점 | 감수할 점 |
|---|---|---|
| 하나의 저장소에 전부 보관 | 한 변경에서 전체 관계를 보기 쉬움 | 소스·운영·공통 절차의 변경/권한 경계를 내부 규칙으로 관리해야 함 |
| 서비스마다 빌드·배포 절차 복사 | 각 개발자가 독립적으로 조정하기 쉬움 | 공통 개선 반영과 규약 일치 비용이 커짐 |
| 현재의 책임별 저장소 분리 | 공통 경로와 desired state의 소유 위치가 명확함 | 저장소 간 호환성·접근 권한·버전 추적 필요 |

**결과와 대가**

- 개발자는 공통 CI를 호출하고 플랫폼 규약에 맞는 실행 정보를 제출합니다.
- 플랫폼 담당자는 앱 내부를 구현하지 않아도 빌드·배포 경로를 관리할 수 있습니다.
- 언어 추가나 툴체인 변경은 템플릿과 Shared Library를 함께 검토해야 합니다.
- 새 서비스 등록에는 namespace·AppProject·시크릿 등 저장소 밖/사이의 연결 작업이 남습니다.
- 공통 라이브러리 변경은 여러 서비스에 영향을 줄 수 있어 검증과 버전 기록이 중요합니다.

**재검토 조건**

공통 변경 때마다 여러 저장소의 동시 수정이 반복되거나, 개발자 온보딩 시간이 수동 연결 작업 때문에 늘어나면 원인을 측정합니다. 먼저 계약·자동화·버전 관리를 개선하고, 저장소 재구성은 그 비용과 효과가 확인될 때 검토합니다.

**구조 변경 이력**

6/28에는 앱 app-of-apps와 Shared Library를 분리하고, 6/30에는 앱 소스와 배포 태그의 저장소를 나눴습니다. 7/3에는 템플릿을 독립 저장소로 분리했습니다. 초기에는 앱 매니페스트 일부가 서비스 저장소에 있었지만 현재는 k8s-gitops가 소유합니다.

세부 결정은 [템플릿](#adr-0011), [공통 CI](#adr-0012), [GitOps](#adr-0017)에 정리했습니다. CI의 배포 변경은 GitOps 저장소에 한정하며, 실제 토큰 권한을 이 범위로 제한하는 작업은 별도 점검 항목입니다.

**관련 구현**

- 범위: 네 플랫폼 저장소와 개발자 산출물의 경계
- 근거: [아키텍처](01-architecture.md), [인프라 저장소](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/README.md), [공유 라이브러리](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/README.md)

</details>

<a id="adr-0002"></a>

<details>
<summary>0002 · OCI OKE Basic과 ARM64 워커 두 대</summary>

- 상태: 채택
- 기록일: 2026-09-30

**배경**

상시 플랫폼 컴포넌트와 일시적인 CI 빌드를 개인 OCI 환경에서 함께 운영합니다. 관리형 Kubernetes를 사용하면서 워커 자원을 제한하고, 클라우드 구성은 재현 가능한 코드로 관리할 필요가 있었습니다.

**선택**

OKE Basic, Flannel overlay, A1.Flex ARM64 워커 두 대를 사용합니다. 워커당 2 OCPU·12GB, 합계 4 OCPU·24GB이며 같은 availability domain에 배치합니다. Terraform은 networking·oke·iam·database·kms·object-storage 모듈로 나눕니다.

API·공개 LB·워커·DB 서브넷을 분리합니다. 워커와 DB는 사설 서브넷에 두고, 워커의 인터넷 발신은 NAT Gateway, OCI 서비스 접근은 Service Gateway 경로로 구성합니다. 관리 접속은 [Tailscale 결정](#adr-0008)과 연결됩니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| OKE Enhanced | 당시 필요한 기능에 비해 추가 관리형 기능·비용을 감수할 이유가 작아 Basic 선택 |
| 소형 AMD 인스턴스 | 당시 자원 비교에서 플랫폼 동시 수용에 ARM 구성이 유리하다고 판단 |
| ARM64 워커 두 대 | 채택. 빌드·런타임·보조 도구까지 ARM64 지원을 확인하는 비용 수용 |

**결과와 검증**

[9/30 관찰](09-evidence/E05-2026-09-30-cli-observation.md)에서 ARM64 노드 두 대가 Ready 상태였습니다. 같은 AD와 다수 단일 replica를 사용하므로 노드 장애 시 서비스 중단이 발생할 수 있습니다.

PAYG 환경에서 4 OCPU·24GB를 설계 기준으로 삼았습니다. [OCI 비용 조회 화면](09-evidence/E06-2026-09-30-platform-measurements.md#비용-조회)에 표시된 2026년 6~9월의 월별 합계와 기간 총합은 0.00 SGD입니다.

**재검토 조건**

ARM64 미지원 의존성이 필요하거나, 동시 빌드와 서비스 부하가 노드 여유를 초과하거나, 다른 AD를 포함한 가용성 요구가 생기면 워커·클러스터 구성을 재검토합니다. 비용은 해당 테넌시의 사용량과 청구 내역으로 판단합니다.

**관련 구현**

- 범위: OCI, Terraform networking·oke·iam 모듈
- 근거: [루트 README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/README.md), [Terraform README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/terraform/README.md), [OKE 설정](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/terraform/modules/oke/main.tf), [네트워크 설정](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/terraform/modules/networking/main.tf)

</details>

<a id="adr-0003"></a>

<details>
<summary>0003 · Terraform state를 OCI Object Storage로 분리</summary>

- 상태: 채택; 원격 state 복구 검증 필요
- 기록일: 2026-09-30 / 결정 기록: 2026-07-23

**배경**

초기 state가 노트북의 로컬 파일에만 있어 장비 유실과 함께 상태를 잃을 수 있었습니다. state에는 DB 자격 정보도 포함될 수 있으므로 Git에 넣지 않는 것만으로 보존·접근 문제를 해결할 수 없었습니다.

**선택**

`backend "oci" {}`와 Terraform `>= 1.12.0`을 선언합니다. 버킷·객체 경로 등 환경 설정은 추적 제외된 `backend.local.hcl`로 공급합니다. README의 부트스트랩 절차는 state 전용 버킷의 버저닝 활성화, state 이관, plan 확인 순서입니다.

인프라 CI는 `fmt`와 `init -backend=false`·`validate`를 수행합니다. OCI 자격과 원격 state가 필요한 plan/apply는 이 CI의 범위에 넣지 않습니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| 로컬 state 유지 | 실제 이전 방식. 장비와 state의 장애 영역을 분리하려고 교체 |
| OCI 네이티브 backend | 채택. 기존 OCI 환경에서 원격 저장과 잠금을 사용하려는 결정 |
| 다른 backend·잠금 서비스 추가 | 회고 비교. 저장소·자격·운영 대상이 추가됨 |

**결과와 검증**

backend 선언과 초기화 예제를 소스로 관리하고, 자격 정보가 필요한 원격 state 작업은 정적 검사 CI와 분리했습니다.

state 버킷과 `object-storage` 모듈의 `bao-snapshots` 버킷은 목적과 부트스트랩 경로가 다릅니다. state 저장소가 준비되기 전에 해당 backend를 초기화할 수 없으므로 사전 생성 단계가 남습니다.

**재검토 조건**

운영자가 늘어나거나 자동 apply가 필요하면 접근 권한, 잠금 충돌 처리, 버전 복원 절차를 검증합니다.

**관련 구현**

- 범위: Terraform backend, Object Storage, 인프라 CI
- 근거: [Terraform README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/terraform/README.md), [provider.tf](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/terraform/provider.tf), [backend 예제](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/terraform/backend.hcl.example), [CI 정의](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/.github/workflows/terraform-ci.yml)

</details>

<a id="adr-0004"></a>

<details>
<summary>0004 · Gateway API와 OCI NLB로 공개 진입점 통일</summary>

- 상태: 채택
- 기록일: 2026-09-30

**배경**

여러 서비스와 CI webhook을 공개하되 TLS·L7 라우팅 책임을 공통 진입점에 모으고 싶었습니다. 플랫폼은 공통 listener를, 각 서비스의 GitOps 선언은 host·path·backend를 관리해야 했습니다.

**선택**

OCI NLB가 TCP 트래픽을 Istio Gateway로 전달하고, Gateway에서 TLS 종료와 HTTP 라우팅을 맡습니다. Gateway API CRD는 별도 부트스트랩 대상으로 관리하고, Istio가 `Gateway` 선언에 따라 데이터플레인 Deployment·Service를 생성합니다.

`public-gateway`의 HTTP listener는 308 redirect에 사용합니다. HTTPS apex·wildcard listener는 공통 인증서 Secret을 참조합니다. 앱 HTTPRoute는 앱 GitOps 저장소가 소유합니다. 외부 라우팅은 Gateway API로 통일하고 구 Istio `VirtualService`를 추가하지 않는 프로젝트 규칙을 택했습니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| OCI L7 LB와 Gateway에서 각각 TLS·라우팅 관리 | 미채택. 인증서·라우팅 운영 지점을 줄이고 싶었음 |
| 별도 Gateway Helm release 관리 | 이 프로젝트는 Gateway API의 자동 프로비저닝 경로 선택 |
| Gateway API + NLB | 채택. 네트워크 전달과 HTTP 정책의 소유자를 구분 |

**결과와 검증**

[9/30 요청 경로 검증](09-evidence/E05-2026-09-30-cli-observation.md)에서 Gateway Accepted·Programmed와 HTTPRoute 6개의 Accepted·ResolvedRefs를 확인했습니다. 외부 웹·core readiness는 TLS 인증서 검증을 포함해 HTTP 200이었습니다.

auth·core·web의 OutOfSync는 HTTPRoute 참조의 기본 필드 차이였습니다. 정규화 비교에서 host·path·backend·rewrite는 같았습니다. kps·Istio 등의 차이는 각 Application의 리소스별로 관리합니다.

**재검토 조건**

서로 다른 팀의 진입점 소유권, 별도 인증서·보안 경계, gRPC 등 추가 프로토콜 요구가 생기면 listener와 Gateway 분리부터 검토합니다. CRD·Istio 업그레이드는 함께 호환성을 확인합니다.

**관련 구현**

- 범위: Gateway API CRD, Istio Gateway, OCI NLB, 서비스 HTTPRoute
- 근거: [Gateway API README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/gateway-api/README.md), [Istio README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/istio/README.md), [Gateway](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/istio/gateway.yaml), [HTTP redirect](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/istio/http-redirect.yaml)

</details>

<a id="adr-0005"></a>

<details>
<summary>0005 · Flannel을 유지하고 Istio Ambient를 선택</summary>

- 상태: Ambient 채택, Cilium chaining 미채택; STRICT·AuthorizationPolicy는 미구현
- 기록일: 2026-09-30 / 변경 결정: 2026-07-24

**배경**

제한된 메모리에서 서비스 간 통신을 메시로 관리하면서, 기존 Flannel이 집행하지 않는 NetworkPolicy의 공백도 검토했습니다. 스테이징 클러스터 없이 네트워크 구성 요소를 늘리는 위험이 판단에 영향을 주었습니다.

**선택**

Pod마다 sidecar를 넣는 대신 노드의 ztunnel과 Istio CNI를 사용하는 Ambient를 택하고, namespace 라벨로 참여 대상을 지정합니다. Istio는 Helm values·ArgoCD 경로로 관리합니다. 필요하지 않은 L7 waypoint는 추가하지 않습니다.

7/10 검토한 Cilium chaining은 실행 전에 7/24 철회했습니다. Flannel → Cilium → Istio CNI의 추가 운영 부담 대신, 동서 통제는 Istio 신원 기반 정책으로 보강하는 방향을 정했습니다.

**대안과 이유**

| 대안 | 근거와 판단 |
|---|---|
| Envoy sidecar | Pod별 자원·주입 관리 비용을 고려해 Ambient 선택 |
| Flannel 위 Cilium chaining | 실제 검토 후 미도입. 네트워크 전체에 영향을 주는 체인을 늘릴 위험 수용이 어려웠음 |
| Cilium primary로 재구축 | 당시 채택하지 않음. 클러스터 재생성 시 재검토 대상으로 남김 |
| Ambient + 후속 접근 정책 | Ambient는 적용, 정책 보강은 별도 구현 과제 |

**결과와 검증**

[9/30 관찰](09-evidence/E05-2026-09-30-cli-observation.md)에서 istiod·CNI·ztunnel과 enrollment 라벨을 확인했습니다. `PeerAuthentication`·`AuthorizationPolicy`·`NetworkPolicy`는 구성하지 않았으며, STRICT 강제와 서비스별 접근 통제는 후속 작업입니다.

7/24에는 Istio 신원 기반 정책을 우선하는 방향을 정했습니다. mesh 밖 호출자와 CIDR egress, 노드 metadata 접근 통제는 별도로 보강해야 합니다.

**재검토 조건**

실제 호출 관계를 정리해 STRICT·접근 정책을 단계적으로 검증합니다. mesh 밖 egress 통제가 필요하거나 클러스터를 재구축하게 되면 CNI 선택을 다시 평가합니다.

**관련 구현**

- 범위: Flannel, Istio base·istiod·CNI·ztunnel, namespace enrollment
- 근거: [Istio README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/istio/README.md), [istiod values](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/istio/istiod.values.yaml), [namespace 선언](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/namespaces/namespaces.yaml)

</details>

<a id="adr-0006"></a>

<details>
<summary>0006 · HTTPRoute에서 Cloudflare DNS 레코드 관리</summary>

- 상태: 채택
- 기록일: 2026-09-30

**배경**

서비스의 host·path를 GitOps로 관리하면서 DNS를 콘솔에서 별도로 편집하면 서비스 추가·철거 때 두 상태를 맞춰야 했습니다. Gateway가 공개·사설 주소를 함께 제공한 사례에서는 공개 DNS에 사설 주소가 들어가는 문제도 생겼습니다.

**선택**

기존 도메인의 Cloudflare를 provider로 사용하고 `gateway-httproute`·`gateway-grpcroute`를 감시합니다. TXT registry와 `txtOwnerId: oci-oke`로 소유권을 추적하며 `policy: sync`로 관리 대상의 생성·변경·삭제를 반영합니다.

`--exclude-target-net=10.0.0.0/8`을 적용해 해당 사설 대역이 공개 레코드의 대상으로 들어가지 않도록 합니다. API token은 Kubernetes Secret 참조로 공급합니다. TLS 종료는 [Gateway](#adr-0004)가 맡습니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| DNS 수동 편집 | 회고 비교. Route와 DNS의 변경 시점이 분리됨 |
| `upsert-only` | README의 비교 대상. 삭제가 자동 반영되지 않아 철거 작업이 남음 |
| `sync` + TXT 소유권 | 채택. Route 수명에 맞추되 삭제 권한과 owner 설정을 관리해야 함 |
| Gateway 주소를 필터 없이 등록 | 과거 이중 IP 문제의 경로. 현재 사설 대역 제외로 보완 |

**결과와 검증**

[9/30 외부 확인](09-evidence/E05-2026-09-30-cli-observation.md)에서 DNS 해석과 웹·API readiness 접근을 확인했습니다.

`sync`는 관리 대상 삭제까지 반영하므로 인스턴스별 owner를 구분해야 합니다. DNS 자격은 현재 Kubernetes Secret으로 공급하며 OpenBao/ESO 전환은 후속 작업입니다.

**재검토 조건**

DNS provider나 관리 도메인이 늘어나면 owner·domain 범위를 분리합니다. Gateway 주소·CIDR 변경 시 사설 주소 제외 규칙도 함께 확인합니다.

**관련 구현**

- 범위: external-dns, Cloudflare, Gateway API
- 근거: [external-dns README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/external-dns/README.md), [values](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/external-dns/values.yaml), [ArgoCD Application](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/argocd/apps/external-dns.yaml)

</details>

<a id="adr-0007"></a>

<details>
<summary>0007 · cert-manager DNS-01로 공통 TLS 인증서 관리</summary>

- 상태: 채택
- 기록일: 2026-09-30

**배경**

여러 서비스가 같은 도메인 아래에 추가되므로 서비스마다 인증서 발급·갱신 절차를 복사하지 않고 공통 진입점에 TLS 운영을 모으려 했습니다.

**선택**

Cloudflare DNS-01을 사용하는 Let's Encrypt staging·production ClusterIssuer와 명시적인 `Certificate`를 둡니다. 인증서는 apex와 wildcard를 포함하고 `istio-system/public-wildcard-tls` Secret에 저장합니다. Gateway의 HTTPS listener가 이를 참조합니다.

Certificate에는 ECDSA P-256, `rotationPolicy: Always`, `duration: 2160h`, `renewBefore: 360h`가 선언돼 있습니다. 이는 원하는 설정이며 실제 발급·갱신 시각은 Certificate status로 확인합니다. external-dns와 cert-manager의 DNS 자격 참조도 각각 관리합니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| HTTP-01 | wildcard 요구와 별도 challenge Route 운영을 고려해 미채택 |
| 서비스별 수동 인증서 | 회고 비교. 발급·갱신 작업과 Secret 관리 지점이 늘어남 |
| annotation에만 발급 의존 | 명시적인 Certificate를 선택해 인증서의 소유·수명·참조 위치를 분리 |
| 공통 DNS-01 인증서 | 채택. 공통 Secret에 여러 host가 의존하는 대가 수용 |

**결과와 검증**

[9/30 관찰](09-evidence/E05-2026-09-30-cli-observation.md)에서 Certificate Ready, 만료 11/5·갱신 예정 10/21 상태를 확인했습니다. 공개 readiness 연결에서도 인증서 체인·호스트 검증과 TLS 1.3 연결을 확인했습니다.

공통 인증서 Secret은 여러 서비스가 공유하므로 손상 시 여러 진입점이 함께 영향을 받습니다. DNS 자격 교체와 인증서 갱신은 함께 관리해야 합니다.

**재검토 조건**

도메인 소유자가 분리되거나 서비스별 인증서 격리가 필요하면 인증서와 listener를 나눕니다. DNS 자격 교체·issuer 변경 시 갱신 경로를 검증합니다.

**관련 구현**

- 범위: cert-manager, Let's Encrypt, Cloudflare DNS-01, Gateway TLS
- 근거: [cert-manager README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/cert-manager/README.md), [ClusterIssuer](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/cert-manager/cluster-issuer.yaml), [Certificate](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/cert-manager/certificate.yaml)

</details>

<a id="adr-0008"></a>

<details>
<summary>0008 · Tailscale subnet router로 관리 접근 분리</summary>

- 상태: 채택
- 기록일: 2026-09-30

**배경**

이동하는 개인 기기에서 Kubernetes API·DB·관리 UI에 접근해야 했습니다. 서비스 트래픽과 관리 접근을 구분하고, 별도 VM의 OS 운영 부담을 늘리지 않는 방향을 선택했습니다.

**선택**

클러스터의 userspace subnet router Pod가 VCN·Service CIDR을 tailnet에 광고합니다. 태그 기반 신원과 라우트 승인, 클라이언트의 route 수락을 함께 사용합니다. 노드 state는 특정 Kubernetes Secret에 유지하고, `TS_ACCEPT_DNS=false`로 클러스터 DNS 설정을 유지합니다.

관리 UI의 공개 Route는 제거하고 Jenkins webhook 경로는 별도로 공개합니다. Tailscale 배포는 ArgoCD prune에 관리 접근이 함께 사라지지 않도록 수동 부트스트랩 대상으로 둡니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| 별도 VM subnet router | 클러스터 장애와 분리되는 장점이 있지만 관리할 OS가 늘어나 미채택 |
| 집과 OCI의 site-to-site VPN | 고정 거점·CPE 운영보다 기기별 로밍 접근을 우선 |
| kernel mode | 장치·네트워크 capability 대신 userspace 구성 선택 |
| 클러스터 내부 router Pod | 채택. 기존 배포 모델을 사용하지만 클러스터 장애에 함께 영향받음 |

**결과와 검증**

[9/30 관찰](09-evidence/E05-2026-09-30-cli-observation.md) 중 사설 API timeout이 발생했고, 관리 연결을 확인한 뒤 다시 조회하자 접근이 복구됐습니다.

클러스터 내부 router는 클러스터 장애에 함께 영향을 받습니다. 비상 접근을 위해 공인 API와 허용 규칙을 이용한 절차를 별도로 두었으며, Tailscale 도입 후에도 공인 endpoint와 NSG 허용 범위를 관리합니다.

**재검토 조건**

클러스터 장애 중에도 즉시 접근해야 하는 요구가 생기면 router를 외부에 두거나 독립 복구 경로를 검증합니다. 기기·운영자가 늘어나면 tailnet ACL 범위와 태그·키 회전 절차를 재검토합니다.

**관련 구현**

- 범위: 관리 PC, Tailscale, OKE API·DB·관리 UI
- 근거: [Tailscale README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/tailscale/README.md), [Deployment](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/tailscale/deployment.yaml), [RBAC](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/tailscale/rbac.yaml), [Jenkins 공개 경로](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/jenkins/httproute.yaml)

</details>

<a id="adr-0009"></a>

<details>
<summary>0009 · namespace별 PSA와 컴포넌트별 RBAC</summary>

- 상태: 채택; 네트워크 접근 정책은 후속 과제
- 기록일: 2026-09-30 / 서비스별 namespace 결정: 2026-06-26

**배경**

서비스 워크로드와 플랫폼·빌드는 필요한 권한이 다릅니다. Kaniko 때문에 CI 전체의 Pod 보안 수준을 풀거나 서비스별 설정이 암묵적으로 생기는 일을 피하려 했습니다.

**선택**

namespace를 플랫폼의 명시적 매니페스트로 관리합니다. 앱 서비스는 namespace마다 `restricted`, 플랫폼·데이터 컴포넌트는 역할에 맞게 `baseline`, 빌드는 별도 `build` namespace의 `privileged`로 나눕니다. Istio 등 예외도 현재 선언에 명시합니다.

RBAC 파일은 해당 컴포넌트와 함께 둡니다. Jenkins controller가 build Pod를 관리하는 권한은 RoleBinding으로 부여하고, 빌드 Pod는 `automountServiceAccountToken: false`를 사용합니다. Tailscale state Secret의 접근 범위도 별도로 정의합니다. ArgoCD·Prometheus의 chart 기본 RBAC가 최소 권한을 충족하는지는 별도 확인 항목입니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| 모든 서비스를 단일 `app`에 배치 | 초기 방향에서 서비스별 namespace로 변경. 향후 정책·자격 소유 경계를 서비스와 맞춤 |
| 빌드를 위해 `cicd` 전체 권한 완화 | 미채택. 별도 build namespace 사용 |
| chart마다 namespace 자동 생성 | 미채택. 라벨·수명 관리 위치를 단일화 |
| RBAC YAML 중앙 폴더 집중 | 설계 표만 중앙에 두고 실제 SA·권한은 컴포넌트와 동거 |

**결과와 검증**

[9/30 관찰](09-evidence/E05-2026-09-30-cli-observation.md)에서 PSA·Ambient·이미지 검증 라벨을 확인했습니다. namespace 분리와 PSA는 호출자 간 네트워크 격리를 자동으로 제공하지 않습니다. 현재 AuthorizationPolicy 부재는 [ADR-0005](#adr-0005)에 기록합니다.

7/4 기록에서는 새 namespace의 `ghcr-pull` 누락으로 첫 이미지 pull이 실패했습니다. 시크릿·AppProject·CI 등록까지 온보딩에 남는 이유입니다. `privileged` PSA는 해당 namespace에서 허용하는 수준이지 모든 컨테이너가 `privileged: true`로 실행된다는 뜻은 아닙니다.

**재검토 조건**

서비스·개발자가 늘어나면 실효 RBAC, namespace 자원 할당, 호출자 정책을 검증합니다. 빌드 자격과 앱 작성자의 신뢰 경계는 [공통 CI 결정](#adr-0012)과 함께 재검토합니다.

**관련 구현**

- 범위: namespaces, Pod Security Admission, ServiceAccount·Role
- 근거: [Namespaces README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/namespaces/README.md), [namespace 선언](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/namespaces/namespaces.yaml), [RBAC README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/rbac/README.md), [Jenkins RBAC](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/jenkins/rbac.yaml)

</details>

<a id="delivery"></a>

## 개발과 배포

서비스 생성부터 테스트·이미지 검사·서명·GitOps 갱신까지 공통 경로를 제공합니다. 실패 사례와 도구 호환성에 따라 초기 선택을 바꿨습니다.

<a id="adr-0010"></a>

<details>
<summary>0010 · JCasC와 동적 agent로 Jenkins 상태 최소화</summary>

- 상태: 채택
- 기록일: 2026-09-30 / 관련 결정: 2026-06-23, 2026-07-04

**배경**

제한된 볼륨과 메모리를 사용하는 개인 클러스터에서 Jenkins를 유지하면서, UI에만 남는 설정과 영구 빌드 agent를 줄이려 했습니다. 기존 Jenkins 경로를 다른 CI로 옮기는 비용도 고려했습니다.

**선택**

controller의 home은 `emptyDir`로 두고 설정은 JCasC로 재구성합니다. controller executor는 0, Kubernetes agent 동시 수는 2로 제한합니다. `svc-*` organizationFolder와 공통 라이브러리를 JCasC로 연결하고, 콜드부트·주기 스캔으로 작업을 발견합니다.

플러그인 목록을 버전으로 고정하고 `installLatestPlugins: false`를 사용합니다. 관리자·GitHub 등 자격은 Git 밖의 Secret 참조로 공급합니다. 실제 변경 반영은 [ArgoCD 소유권 결정](#adr-0017)을 따릅니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| Jenkins home 전용 PVC | 미채택. 볼륨 예산을 NATS·Prometheus에 배분하고 설정 재생성 선택 |
| controller에서 직접 빌드 | 채택하지 않음. 동적 agent Pod로 실행 환경 분리 |
| 다른 CI로 전면 이전 | 7/10 결정 기록의 회고에서 기존 사용·전환 비용 때문에 Jenkins 유지 |
| JCasC + ephemeral agent | 채택. 플러그인·JCasC·이미지 조합의 호환성 관리 필요 |

**결과와 검증**

7/4에는 controller 재생성 뒤 빈 organizationFolder가 자동으로 채워지지 않아 콜드부트 스캔을 보완했습니다. [9/30 기존 batch 빌드 추적](09-evidence/E02-2026-09-30-onboarding-deployment.md)에서 공통 경로의 실행 결과를 연결했습니다.

`emptyDir`의 데이터는 같은 Pod 안의 컨테이너 재시작에는 유지되지만, **Pod 삭제·교체 시 빌드 이력·로컬 아티팩트를 잃을 수 있습니다.** Git·GHCR에는 소스·이미지·태그가 남으므로 Jenkins 로그와 보고서는 별도 보존 경로가 필요합니다.

**재검토 조건**

빌드 감사 기록 보존이 필요하면 외부 로그·아티팩트 저장이나 영속성을 먼저 검토합니다. 동시 빌드 중 서비스 부하·대기 시간이 관찰되면 agent cap과 빌드 자원을 조정합니다.

**관련 구현**

- 범위: Jenkins controller, JCasC, Kubernetes plugin, 빌드 이력
- 근거: [Jenkins README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/jenkins/README.md), [values](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/jenkins/values.yaml), [RBAC](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/jenkins/rbac.yaml)

</details>

<a id="adr-0011"></a>

<details>
<summary>0011 · 업무 기능을 제외한 언어별 서비스 시작 템플릿</summary>

- 상태: 채택
- 기록일: 2026-09-30 / 분리 결정: 2026-07-03, 최소 기능 정정: 2026-09-15

**배경**

새 서비스를 시작할 때마다 Dockerfile·probe·CI 호출·배포 선언을 작성하는 반복을 줄이려 했습니다. 특정 업무 서비스에 의존하는 템플릿은 독립적으로 실행하기 어려워지므로 플랫폼 규약까지만 제공하기로 했습니다.

**선택**

`sed-template.sh`로 `svc-<이름>` 소스와 `gitops-<이름>` 배포 조각을 함께 생성합니다. 생성 후 소스와 배포 상태는 각각의 저장소가 소유합니다. 템플릿 업데이트가 기존 서비스에 자동 전파되지는 않습니다.

| 템플릿 | 시작점·현재 계약 |
|---|---|
| [Go](https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/go-app/README.md) | Go 1.25·chi, distroless nonroot, 상태 확인·런타임 메트릭 |
| [Java Gradle](https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/java-app/gradle/README.md) | Java 21·Spring Boot 3.4, Gradle 8.12 이미지, Actuator·Micrometer |
| [Node.js](https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/node-app/README.md) | Node 22·TypeScript·Fastify, lockfile·Vitest, distroless 런타임 |
| [JavaScript](https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/javascript-app/README.md) | 순수 JS·Node 테스트/빌드·비특권 nginx, 자체 readiness, ServiceMonitor 없음 |
| [Python](https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/python-app/README.md) | Python 3.13 표준 라이브러리 HTTP·unittest, 내부 Service·기본 메트릭 |
| [Java Maven](https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/java-app/maven/README.md) | 참고용 변형. 현재 `language: java`의 Gradle 테스트 게이트와 호환되지 않음 |

**대안과 이유**

| 대안 | 판단 |
|---|---|
| 인프라 저장소 안에 템플릿 유지 | 초기 구조에서 분리. 배포할 자원과 재사용할 생성 도구의 책임이 달랐음 |
| auth/core 연결·알림 워커를 템플릿에 포함 | 9/15 최초안에서 철회. 자체 상태 확인 중심의 최소 기능으로 범위를 축소 |
| 모든 언어의 공통 런타임 추상화 | 회고 비교. 현재는 언어 관용을 유지하고 CI·배포 계약만 통일 |
| 가벼운 생성 스크립트 + 명시적 온보딩 | 채택. namespace·Secret·AppProject·CI 등록은 별도 |

**결과와 검증**

[9/30 생성 검증](09-evidence/E01-2026-09-30-template-generation.md)에서 새 JavaScript 출력의 테스트 6/6·정적 빌드·Kustomize 렌더링이 성공했습니다.

Go·Node·Java 생성기는 출력 하위 폴더를 재생성하므로 새 출력 위치를 사용해야 합니다. JS·Python 생성기는 기존 출력 덮어쓰기를 거부합니다.

**재검토 조건**

동일한 온보딩 오류가 반복되면 해당 연결만 자동화합니다. 새 프레임워크나 의존성은 실제 서비스 요구와 [언어 테스트 계약](#adr-0012)을 함께 확인한 뒤 추가합니다.

**관련 구현**

- 범위: app-templates의 Go·Java·Node.js·JavaScript·Python
- 근거: [공통 README](https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/README.md), 아래 템플릿별 README, [온보딩](03-service-onboarding.md)

</details>

<a id="adr-0012"></a>

<details>
<summary>0012 · Shared Library에서 파이프라인과 언어 테스트 계약 관리</summary>

- 상태: 채택
- 기록일: 2026-09-30 / 중앙화: 2026-07-21, 테스트 계약: 2026-07-24

**배경**

초기에는 공통 라이브러리를 개별 빌드 step으로 제공하고 서비스가 절차를 조립했습니다. 서비스가 늘면서 검사·서명 정책과 언어별 테스트 명령을 각 Jenkinsfile에서 관리하는 부담이 커졌습니다.

**선택**

앱 Jenkinsfile은 `@Library('shared')`와 `ci(service: ...)`를 호출합니다. 공통 파이프라인은 Test → Build & Push → Image Scan → Sign → Bump를 조립합니다. 언어·테스트 이미지·명령과 서비스별 scan/sign 설정은 플랫폼이 소유한 `services.yaml`에 둡니다.

기본 테스트는 Docker daemon 없이 실행돼야 합니다. 현재 Go·Node·Java Gradle·Python 계약이 있으며 서비스별 테스트 명령 override는 두지 않습니다. 통합 테스트는 앱이 기본 명령에서 분리해 Docker가 있는 별도 환경에서 실행하도록 합니다. 언어별 차이가 이미지·명령 데이터이므로 Groovy 실행 경로는 하나로 유지합니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| 서비스별 Jenkinsfile에서 단계·정책 조립 | 이전 방식. 7/21 중앙화로 변경 |
| 언어별 Groovy 구현 | 7/24 실제 비교. 문자열 차이에 코드 경로를 복제할 필요가 작음 |
| DinD 또는 운영 DB를 테스트에 공유 | 7/24 검토 후 미채택. agent 구조와 테스트 상태 격리에 맞지 않음 |
| YAML 계약 + 단일 실행 경로 | 채택. 앱이 계약에 맞춰 테스트 구성을 정리해야 함 |

**결과와 검증**

[batch #13](09-evidence/E04-2026-09-30-ci-failure-boundary.md)은 Test 단계의 `compileTestJava` 실패로 중단됐고 이후 네 단계는 skip됐습니다. [batch #23](09-evidence/E02-2026-09-30-onboarding-deployment.md)은 테스트부터 GitOps 갱신까지 성공했습니다.

7/24에는 부모 agent의 namespace가 상속되지 않아 `build`를 명시하도록 수정했습니다. 공통 CI를 유지하려면 plugin·JCasC·언어 도구 버전을 함께 맞춰야 합니다. 테스트 수·커버리지·통합 테스트는 앱별로 관리합니다.

정책 중앙화는 **신뢰한 Jenkinsfile이 공통 경로를 사용하는 계약**입니다. 앱 작성자가 Jenkinsfile을 임의 변경할 수 있고 빌드 Pod가 서명·push 자격을 가지는 상황까지 격리하는 보안 경계는 아닙니다. 9/25 검토의 이 한계는 남아 있습니다.

**재검토 조건**

비신뢰 기여자의 코드를 실행하거나 정책 예외가 늘어나면 자격 공급·서명·배포 권한과 공통 계약을 재검토합니다. 도구 버전은 테스트 이미지와 앱 Dockerfile을 함께 변경합니다.

**관련 구현**

- 범위: Jenkins Shared Library, services.yaml, 서비스 Jenkinsfile
- 근거: [Shared Library README](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/README.md), [ci.groovy](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/vars/ci.groovy), [services.yaml](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/resources/ci/services.yaml)

</details>

<a id="adr-0013"></a>

<details>
<summary>0013 · ARM64 Kaniko 빌드와 GHCR SHA 태그</summary>

- 상태: 채택
- 기록일: 2026-09-30 / 관련 결정·문제 해결: 2026-06-26, 2026-07-25

**배경**

A1 ARM64 워커에서 서비스를 빌드·배포하면서 Docker daemon 운영을 추가하지 않고 Jenkins Kubernetes agent에 맞는 빌드 경로가 필요했습니다.

**선택**

`build` namespace의 Kaniko debug 컨테이너에서 Dockerfile을 빌드하고 GHCR에 push합니다. 공통 CI는 전체 소스 Git SHA를 태그로 전달합니다. `latest` 동시 발행은 제거했습니다. `kanikoBuild`를 직접 호출할 때의 기본 짧은 SHA와 공통 CI의 전체 SHA를 구분합니다.

Kaniko의 digest 파일은 [cosign 서명](#adr-0015)에 넘깁니다. 레이어 캐시를 사용하고 `/busybox`·`/home/jenkins`를 snapshot 제외 경로로 지정해 Jenkins 실행 셸과 workspace를 보존합니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| Docker daemon을 agent에 추가 | 기존 daemon 없는 빌드 방향과 맞지 않아 채택하지 않음 |
| Kaniko 기본 이미지 | 셸·sleep이 필요한 agent 운용에 맞춰 debug 이미지 선택 |
| `latest`와 SHA를 함께 발행 | 이전 방식. 실행 버전 추적을 위해 7/25 SHA-only로 변경 |
| GHCR private 패키지 | 기존 GitHub·CI 경로와 결합해 사용. pull·push·서명 조회 자격 관리 비용 수용 |

Docker daemon은 사용하지 않지만, 빌드 Pod의 권한과 서명·push 자격은 별도로 관리해야 합니다. 비신뢰 코드의 자격 접근을 격리하는 구성은 아닙니다.

**결과와 검증**

6/26 기록에는 이미지 push는 끝났지만 Kaniko가 Jenkins 실행 셸을 훼손해 작업이 끝나지 않는 문제와 ignore-path 대응이 있습니다. [batch #23 추적](09-evidence/E02-2026-09-30-onboarding-deployment.md)은 소스 SHA·출력 digest·실행 imageID를 연결합니다.

Git SHA 형태의 태그도 레지스트리에서 재지정될 수 있습니다. 현재 GitOps는 digest 참조가 아니며 불변성 강제 여부는 별개입니다. ARM64 기본 이미지·agent·서명 도구의 아키텍처 호환도 함께 유지해야 합니다.

**재검토 조건**

빌더 유지보수·취약점 대응, 새 아키텍처, 캐시 정확성 또는 빌드 시간이 실제 제약이 되면 대체 빌더를 평가합니다. 검증한 이미지를 그대로 실행하는 보장은 scan·sign·GitOps의 digest 연결과 함께 강화합니다.

**관련 구현**

- 범위: Kaniko, GHCR, Jenkins build namespace, 이미지 식별자
- 근거: [Jenkins README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/jenkins/README.md), [agent 설정](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/jenkins/values.yaml), [kanikoBuild](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/vars/kanikoBuild.groovy), [ci](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/vars/ci.groovy)

</details>

<a id="adr-0014"></a>

<details>
<summary>0014 · Trivy 결과를 보존한 뒤 취약점 게이트 판정</summary>

- 상태: 채택; 현재 scanGate 기본값 true
- 기록일: 2026-09-30 / 도입: 2026-07-14, 과거 warn 전환: 2026-07-16

**배경**

빌드 성공만으로 배포하지 않고 이미지 취약점을 확인하되, 실패한 빌드에서 원인 리포트를 볼 수 있어야 했습니다. 검사·서명 정책이 서비스 문서마다 다른 값으로 적히는 문제도 있었습니다.

**선택**

push된 이미지를 Trivy로 한 번 JSON 스캔하고 같은 결과를 HTML로 변환합니다. 결과 파일이 있으면 JSON·HTML을 아카이브·게시한 다음 캡처한 종료 코드로 게이트를 판정합니다.

현재 기본 기준은 `CRITICAL`, `ignoreUnfixed: true`, `scanGate: true`입니다. `gate: false`는 비정상 스캔 결과를 UNSTABLE로 표시하고 후속 경로를 계속할 수 있는 예외입니다. 리포트 파일조차 없으면 스캔 자체 실패로 중단합니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| 첫 비정상 종료에서 즉시 파이프라인 종료 | 리포트 게시에 도달하지 못하므로 종료 코드 캡처 후 판정 |
| JSON·HTML을 별도 스캔으로 생성 | 같은 결과 변환으로 일관성을 유지하고 재스캔 비용 제거 |
| warn-only | 7/16에 사용한 방식. 현재 서비스 기본값은 차단 |
| 중앙 정책 + 결과 보존 | 채택. 서비스별 정책 예외와 필터를 중앙 설정에서 관리 |

**결과와 검증**

[batch #23](09-evidence/E02-2026-09-30-onboarding-deployment.md)의 리포트는 CRITICAL·unfixed 제외 조건에서 취약점 0건이었습니다. 게이트 판정은 이 필터 범위에 한정됩니다.

순서는 push → scan이므로 게이트 실패 이미지도 GHCR에 남을 수 있습니다. 차단하는 것은 정상 파이프라인의 후속 서명·GitOps 갱신입니다. 이미지 태그 재지정과 빌드 자격 경계는 별도 한계입니다.

**재검토 조건**

실제 예외·오탐·패치 지연 사례를 근거로 심각도와 fix 제외 정책을 조정합니다. 스캔·서명 대상의 동일성 요구가 강화되면 digest 참조를 일관되게 사용하도록 검토합니다.

**관련 구현**

- 범위: Trivy, Jenkins HTML Publisher, 공통 CI
- 근거: [Shared Library README](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/README.md), [스캔 step](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/vars/trivyImageScan.groovy), [정책 값](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/resources/ci/services.yaml), [Jenkins 설정](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/jenkins/values.yaml)

</details>

<a id="adr-0015"></a>

<details>
<summary>0015 · 검증기 호환성에 맞춘 cosign v2 digest 서명</summary>

- 상태: 채택
- 기록일: 2026-09-30 / 최종 피벗: 2026-07-17

**배경**

CI에서 만든 이미지와 신뢰한 서명키를 연결하려 했습니다. 초기 cosign v3 bundle과 당시 Kyverno의 raw 공개키 검증 조합에서 인증·포맷 문제가 이어져 서명기 단독 성공만으로 체인을 완성할 수 없었습니다.

**선택**

cosign v2.4.1 바이너리와 셸을 포함한 ARM64 이미지를 만들어 Jenkins agent에서 사용합니다. Kaniko 출력 digest를 읽어 key-based 서명하고 레거시 서명 아티팩트를 GHCR에 보관합니다.

`--tlog-upload=false`와 Kyverno의 `rekor.ignoreTlog: true`를 함께 사용합니다. 개인키·비밀번호는 빌드 namespace의 Secret으로 공급하고 공개키는 정책에서 참조합니다.

**대안과 이유**

아래는 7/17 기록의 실제 시행·수정 경로입니다.

| 대안 | 결과와 판단 |
|---|---|
| cosign v3 bundle + raw 공개키 | 당시 사용한 Kyverno 조합에서 검증 실패 |
| bundle에 공개 tlog 포함 | 재서명해도 당시 문제를 해결하지 못해 최종안에서 철회 |
| 공식 distroless 서명 이미지 | 셸·sleep을 사용하는 Jenkins agent 구성과 맞지 않았음 |
| 셸 포함 자체 v2 이미지 + 기본 Cosign 검증 | 최종 채택. 이미지·바이너리 업데이트를 직접 관리 |

**결과와 검증**

7/17 최종 조합에서 PolicyReport PASS를 확인했습니다. [batch #23 추적](09-evidence/E02-2026-09-30-onboarding-deployment.md)에서도 CI가 서명한 digest와 실행 digest가 일치했습니다.

서명 검증의 대상은 신뢰한 키와 이미지 digest의 연결입니다. 테스트·스캔 통과 attestation과 SBOM·빌드 출처 검증은 포함하지 않습니다. 공개 tlog 대신 키 보관·회전과 서명 이력을 직접 관리해야 합니다.

**재검토 조건**

서명기·검증기 버전 또는 포맷을 올릴 때 실제 ARM64 agent와 registry 자격을 포함해 서명 → 검증 → admission 호환성을 다시 확인합니다.

**관련 구현**

- 범위: cosign, GHCR 서명 아티팩트, Kyverno 검증 포맷
- 근거: [서명 step](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/vars/cosignSign.groovy), [서명 이미지 Dockerfile](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/jenkins/cosign-image/Dockerfile), [Jenkins README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/jenkins/README.md), [Kyverno 정책](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/kyverno/policies/verify-image-signature.yaml)

</details>

<a id="adr-0016"></a>

<details>
<summary>0016 · 서비스 이미지 서명을 Kyverno admission에서 검사</summary>

- 상태: Enforce 채택; digest 고정은 미도입
- 기록일: 2026-09-30 / 도입: 2026-07-17, Enforce 전환: 2026-07-30

**배경**

CI의 서명만으로는 클러스터가 어떤 이미지를 허용하는지 결정하지 못합니다. 배포 시점에 신뢰한 키의 서명을 검사하면서 새 서비스 추가 때 정책의 namespace 목록을 매번 고치지 않으려 했습니다.

**선택**

`verify-images: "true"` namespace와 `ghcr.io/ggingggang/svc-*` 이미지 패턴에 서명 검증을 적용합니다. Audit 리포트 확인 뒤 Enforce로 전환했습니다. 검증기는 rule의 `imageRegistryCredentials`로 private GHCR의 서명을 조회합니다.

현재 `background: false`, `mutateDigest: false`, `verifyDigest: false`입니다. admission·reports controller를 사용하고 불필요한 background·cleanup controller는 끕니다. 단일 admission replica의 가용성 한계를 수용합니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| Audit 유지 | 초기 적용 방식. 7/30 당시 대상 namespace의 PASS 확인 후 차단 모드로 전환 |
| namespace 이름을 정책에 나열 | 라벨 멤버십으로 바꾸어 새 서비스 편입 위치를 namespace 선언에 모음 |
| GHCR public 전환 | 7/17 검토 후 private 유지와 검증용 자격 공급을 선택 |
| Enforce와 digest 고정을 함께 적용 | 초기 후속안에서 변경. 7/30은 태그 기반 배포를 유지하며 두 digest 옵션을 false로 확정 |

**결과와 검증**

[9/30 관찰](09-evidence/E05-2026-09-30-cli-observation.md)에서 Enforce 정책과 auth·batch·core·web의 PolicyReport PASS를 확인했습니다.

정책 패턴 밖 이미지와 라벨 없는 namespace까지 금지하는 전체 이미지 allowlist는 아닙니다. 또한 SHA 태그를 digest로 변환·고정하지 않아 admission 조회와 실제 pull의 식별자를 고정하는 보장이 부족합니다. 서명 검증과 digest 고정은 서로 다른 속성입니다.

초기에는 OpenBao 이관 후 Enforce로 전환할 계획이었으나, 7/30에 Enforce를 먼저 적용했습니다. 시크릿 통합과 회전 자동화는 후속 작업으로 남았습니다.

**재검토 조건**

digest 기반 배포, 정책 범위 확대, registry 자격 회전, admission 장애 시 동작을 검증합니다. 신뢰하지 않는 앱 작성자가 빌드 자격에 접근하는 문제는 [ADR-0012](#adr-0012)와 함께 다룹니다.

**관련 구현**

- 범위: Kyverno, namespace 라벨, GHCR 검증 자격
- 근거: [Kyverno README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/kyverno/README.md), [정책](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/kyverno/policies/verify-image-signature.yaml), [values](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/kyverno/values.yaml)

</details>

<a id="adr-0017"></a>

<details>
<summary>0017 · ArgoCD·Kustomize로 GitOps 배포 상태 관리</summary>

- 상태: 채택; Deployment PR 전환은 철회됨
- 기록일: 2026-09-30 / 주요 결정: 2026-06-26~07-03, 최종 정정: 2026-09-28

**배경**

앱 소스에 매번 배포 태그를 되쓰면 webhook 재빌드와 개발 브랜치 변경이 섞입니다. 플랫폼 Helm release가 ArgoCD 관리로 넘어간 뒤에도 직접 Helm upgrade를 실행하면 서로 같은 필드의 소유권을 주장했습니다.

**선택**

플랫폼은 `platform` AppProject, 서비스는 `apps` AppProject와 별도 app-of-apps로 관리합니다. 서비스 매니페스트는 기존 구조에 맞춰 Kustomize를 사용하고, CI는 `k8s-gitops/manifests/<service>/kustomization.yaml`의 이미지 태그를 GitOps main에 직접 push합니다.

서비스 Application은 automated·prune·selfHeal을 사용합니다. 플랫폼 root·meta 자동 동기화와 개별 플랫폼 컴포넌트 동기화는 구분하며, ArgoCD 자신과 Jenkins·kps 등은 해당 Application 정책을 따릅니다. Helm은 chart 렌더링·초기 설치에 사용하더라도 ArgoCD가 관리하는 리소스의 일반 변경은 Git → ArgoCD로 반영합니다.

공통 GitOps 저장소 push 경합은 최대 5회 시도로 처리합니다. 거부되면 **빌드의 임시 clone**을 최신 main으로 갱신하고 해당 서비스 태그만 재적용합니다. 강제 push를 사용하지 않습니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| 앱 소스 저장소에 배포 태그 기록 | 초기 구조에서 분리. 코드 변경과 배포 이력·webhook 경로를 구분 |
| 앱용 Helm chart 도입 | 6/26 실제 저장소를 확인한 뒤 기존 Kustomize를 선택 |
| ArgoCD adopt 뒤에도 직접 Helm upgrade | 6/29 Jenkins·7/25 kps에서 실제 소유권 충돌. 일반 변경 경로로 사용하지 않음 |
| Deployment PR + 병합 후 배포 | 9월 로컬 구현을 검토했으나 9/28 최종적으로 철회 |
| GitOps main 직접 push | 최종 선택. 서비스 간 경합 재시도와 별도 배포 완료 확인 필요 |

**결과와 검증**

[9/26 실제 경합](09-evidence/E03-2026-09-26-gitops-push-conflict.md)에서 core 변경이 먼저 push된 뒤 batch가 재시도해 core 태그를 보존했습니다. [배포 추적](09-evidence/E02-2026-09-30-onboarding-deployment.md)은 해당 변경을 ArgoCD history·실행 digest까지 연결합니다.

9/28에는 로컬에서 검토한 main/최신 SHA 가드도 철회하고 기존 원격 구현을 유지했습니다. **현행 코드에는 빌드 브랜치 제한이나 최신 앱 main SHA 비교가 없습니다.** 재시도는 서로 다른 서비스의 변경 경합을 처리하지만 같은 서비스의 오래된 빌드가 나중에 덮어쓰는 문제까지 해결하지 않습니다.

Jenkins 성공은 GitOps push까지입니다. 배포 완료는 ArgoCD Sync·Health·실행 이미지·readiness로 별도 판단합니다. kps의 과거 failed Helm revision과 현재 workload Health도 동일한 상태가 아닙니다.

**재검토 조건**

승인·환경 승격·동일 서비스의 버전 역행 방지가 필요하면 PR·브랜치/최신 소스 검사·digest 배포를 새 결정으로 검토합니다. 자동 sync 확대 전에는 실제 diff 원인과 복구 경로를 확인합니다.

**관련 구현**

- 범위: ArgoCD·Helm adopt, Kustomize, AppProject, deployBump
- 근거: [플랫폼 ArgoCD README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/argocd/README.md), [앱 GitOps README](https://github.com/GGingGGang/k8s-gitops/blob/3ef5320819bf94240db7d35924bd7858979d0167/argocd/README.md), [앱 project](https://github.com/GGingGGang/k8s-gitops/blob/3ef5320819bf94240db7d35924bd7858979d0167/argocd/project.yaml), [deployBump](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/vars/deployBump.groovy), [ci](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/vars/ci.groovy)

</details>

<a id="data"></a>

## 데이터와 시크릿

데이터마다 필요한 보존 수명이 다릅니다. 볼륨 배분과 저장 수명, 서비스를 사용하는 앱의 데이터 용도를 기준으로 구성요소를 관리합니다.

<a id="adr-0018"></a>

<details>
<summary>0018 · OpenBao·OCI KMS와 시크릿 저장 수명</summary>

- 상태: OpenBao·KMS·emptyDir 채택; 시크릿 이관·정기 스냅샷·복원 검증은 미완료
- 기록일: 2026-09-30 / 저장 방식 변경: 2026-07-22~25

**배경**

흩어진 플랫폼 Secret을 통합할 기반이 필요했습니다. 동시에 볼륨 예산을 NATS·Prometheus와 나눠야 했습니다. 당시 OpenBao 저장량이 작고 본격적인 시크릿 이관이 시작되지 않은 상황이 저장 방식 변경의 전제였습니다.

**선택**

당시 라이선스 변화 리스크와 프로젝트 방향을 고려해 OpenBao를 선택했습니다. Raft 단일 replica와 OCI KMS auto-unseal을 사용하고, API key 파일 대신 instance principal로 특정 unseal 키를 사용하게 합니다. 키는 software-protected로 구성합니다.

7/22에는 OpenBao PVC를 반납해 NATS에 배분했습니다. 현재 `/openbao/data`는 `emptyDir`입니다. 7/25에는 이를 유지하고 정기 스냅샷·복원 리허설로 보완하는 방향을 정했습니다. 스냅샷 버킷은 선언했으며, 자동 스냅샷 작업과 복원 검증은 후속 과제입니다.

Secret 공급은 런타임 파일용 Injector와 Pod 생성 전에 필요한 Secret용 ESO를 구분하는 설계입니다. imagePullSecret 같은 객체는 Pod 생성 전에 준비해야 합니다. 전체 Secret의 이관은 미완료입니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| Vault | 당시 라이선스·거버넌스 변화를 고려해 OpenBao 선택 |
| API key 파일로 KMS 접근 | 키 배포·회전 부담 때문에 instance principal 선택 |
| OpenBao PVC 유지 | 7/22 실제 반납. 작은 사용량과 이관 전 상태를 전제로 자원 재배분 |
| emptyDir + 정기 스냅샷 | 7/25 방향 채택. 현재 emptyDir 사용, 자동 백업·복구는 후속 과제 |
| Injector 하나로 모든 Secret 공급 | 사전 Kubernetes Secret과 런타임 파일의 수명이 달라 미채택 |

**결과와 검증**

[9/30 관찰](09-evidence/E05-2026-09-30-cli-observation.md)에서 OpenBao의 실행과 emptyDir를 확인했습니다. `ha.enabled: true`라는 chart 값에도 replica는 1이므로 고가용성이 아닙니다.

Pod 삭제·교체로 emptyDir가 없어지면 Raft 데이터도 사라집니다. 같은 Pod 내 컨테이너 재시작과는 구분합니다. auto-unseal은 암호화 키 접근을 자동화하며 유실된 데이터를 복구하지 않습니다. 7/23에는 data mount 누락으로 기동하지 못한 문제도 있었습니다.

instance principal은 Pod별 신원이 아니므로 node metadata 접근 통제의 공백이 남습니다. KMS 권한의 키 범위 제한과 메시 enrollment가 이를 모두 대신하지는 않습니다.

**재검토 조건**

실제 시크릿 원본을 이관하기 전에 보존·복원 경로를 확보하고 빈 환경에서 복구를 검증합니다. RPO·RTO는 측정 후 기록하며, 필요하면 PVC 배분 또는 외부 시크릿 원본 유지 방식을 다시 선택합니다.

**관련 구현**

- 범위: OpenBao, OCI KMS·IAM, Agent Injector, 후속 ESO
- 근거: [OpenBao README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/openbao/README.md), [values](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/openbao/values.yaml), [KMS](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/terraform/modules/kms/main.tf), [IAM](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/terraform/modules/iam/main.tf), [Object Storage](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/terraform/modules/object-storage/main.tf)

</details>

<a id="adr-0019"></a>

<details>
<summary>0019 · HeatWave MySQL과 서비스별 DB 계정</summary>

- 상태: 채택; 삭제·복원 보호 보강은 후속 과제
- 기록일: 2026-09-30 / DB 온보딩 결정: 2026-07-07

**배경**

서비스가 늘어날 때 DB·계정을 매번 수동 생성하는 작업이 반복됐습니다. 데이터 원본은 워커 Pod 수명과 분리하고, 각 서비스가 자기 스키마에 접근하는 계약을 제공할 필요가 있었습니다.

**선택**

사설 DB 서브넷의 HeatWave `MySQL.Free`와 50GB 설정을 Terraform으로 관리합니다. DB 운영을 관리형 서비스에 맡겨 워커와 DB의 수명을 분리했습니다.

DB 사용이 필요한 서비스에만 `<service>_db`·`app_<service>`를 만들고, 해당 namespace의 `db-creds` Secret에 연결 정보를 제공합니다. 반복 작업은 `scripts/`로 분리하고, 기존 계정의 추가 스키마 권한은 별도 grant 도구로 처리합니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| Kubernetes 안에 DB와 PVC 운영 | 회고 비교. 노드·볼륨·DB 복구 운영을 추가해야 함 |
| 서비스마다 수동 SQL·Secret 작성 | 7/7의 반복 작업을 스크립트로 정리 |
| 모든 서비스가 동일 관리자 계정 사용 | 현재는 서비스별 사용자·스키마로 소유 경계를 표현 |
| 동적 DB 자격 발급 | 장기 방향. 현재 정적 계정·Secret을 대체한 상태는 아님 |

**결과와 검증**

7/7에 core·auth·batch DB 온보딩을 완료했습니다. 초기 접속 timeout은 DB inactive 상태가 원인이었습니다.

현재 Terraform에는 delete protection false, 자동 백업 삭제, final backup 생략이 명시돼 있습니다. DB 삭제 시 데이터와 복구본을 보존하려면 삭제 보호와 별도 백업 경로를 보강해야 합니다.

`onboard-app-db.sh`는 재실행 시 기존 사용자 비밀번호와 새 Secret이 달라질 수 있습니다. 기존 계정에 스키마를 추가할 때는 grant 스크립트를 사용하고, 자격 회전은 별도 절차로 관리해야 합니다.

**재검토 조건**

사용자 데이터의 보존 요구가 생기면 삭제 보호·외부 복구본·복원 실험을 우선 검토합니다. 계정 수·회전 부담이 증가하면 OpenBao의 동적 자격 공급을 별도 결정으로 검증합니다.

**관련 구현**

- 범위: Terraform database, HeatWave MySQL, DB 온보딩 도구
- 근거: [Terraform README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/terraform/README.md), [database 모듈](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/terraform/modules/database/main.tf), [scripts README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/scripts/README.md), [DB 생성](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/scripts/onboard-app-db.sh), [추가 권한](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/scripts/onboard-app-grant.sh)

</details>

<a id="adr-0020"></a>

<details>
<summary>0020 · Redis를 비영속 캐시로 단순하게 운영</summary>

- 상태: 기존 선택 유지; 실제 소비 계약과의 불일치는 재검토 필요
- 기록일: 2026-09-30 / 도입 기록: 2026-06-22

**배경**

초기 요구는 DB를 원본으로 둔 cache-aside였습니다. 재생성할 수 있는 캐시에 전용 볼륨과 operator를 추가하지 않고 제한된 메모리 안에서 운용하려 했습니다.

**선택**

공식 Redis 이미지의 단일 Deployment·ClusterIP Service를 raw YAML로 관리합니다. RDB·AOF는 끄고 `/data`는 emptyDir로 둡니다. `maxmemory 256mb`·`allkeys-lru`와 컨테이너 limit 384Mi를 사용합니다.

데이터 namespace는 앱과 나누고 Ambient에 참여시킵니다. Redis 자체 인증은 없으며, 호출자를 제한하는 [접근 정책](#adr-0005)도 후속 작업입니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| Helm·Operator로 Redis 관리 | 단일 캐시에 비해 운영 구성이 늘어난다고 판단 |
| Redis PVC·영속 로그 | 초기 cache-aside 전제에서는 재생성 가능해 미채택 |
| raw Deployment + ephemeral cache | 채택. Pod 교체 후 캐시를 다시 채우는 비용 수용 |

**결과와 검증**

[9/30 관찰](09-evidence/E05-2026-09-30-cli-observation.md)에서 실행 상태와 emptyDir를 확인했습니다.

9/25에 소비 서비스의 Redis 사용을 살펴보니 refresh/session 상태와 락에도 사용하고 있었습니다. **재생성 가능한 캐시라는 초기 전제와 현재 세션·락의 보존 요구가 맞지 않아 재검토가 필요합니다.**

Pod 교체 시 전체 상태가 사라지는 문제와 메모리 압박으로 일부 키가 축출되는 문제를 각각 다뤄야 합니다. 세션·락까지 DB에서 모두 재생성할 수 있는 구조는 아닙니다.

**재검토 조건**

캐시와 세션·락의 소비 계약을 구분하고, 키 축출·Pod 교체 시 기대 동작을 검증합니다. 필요하면 별도 저장소나 정책으로 나누며, 단순히 noeviction으로 바꾸는 경우에도 쓰기 실패 처리를 확인합니다.

**관련 구현**

- 범위: Redis Deployment·Service, 메모리·데이터 수명
- 근거: [Redis README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/redis/README.md), [redis.yaml](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/redis/redis.yaml), [운영·복구](06-operations.md)

</details>

<a id="adr-0021"></a>

<details>
<summary>0021 · Kafka를 NATS JetStream으로 교체</summary>

- 상태: 교체 채택; NATS 단일 서버·oci-bv 구성
- 기록일: 2026-09-30 / 교체 결정: 2026-07-10, 저장 방식 정정: 2026-07-22

**배경**

일정 이벤트에는 재전달, 명시적 ack, 재처리·실패 격리가 필요했습니다. 반면 기존 Kafka의 파티션 확장·다중 대규모 소비 요구는 사용하지 않았고 broker·operator의 운영·메모리 비용은 계속 발생했습니다.

7/10 기록의 당시 측정은 Kafka broker와 Strimzi operator 합계 약 1.1Gi였습니다. 이 값은 교체를 판단한 과거 관측이며, NATS의 동일 부하 대비 절감률 측정값은 아닙니다.

**선택**

Kafka·Strimzi를 제거하고 단일 NATS 서버의 JetStream file store를 사용합니다. 최종 저장 방식은 `oci-bv` 50Gi PVC와 45Gi file store 상한입니다. 메모리 store·상주 nats-box·PDB는 사용하지 않습니다.

플랫폼은 서버·저장소·exporter·PodMonitor를 제공합니다. stream·durable consumer·ack 시점·재시도·DLQ 라우팅은 소비 앱에서 구성합니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| Kafka 유지 | 7/6 유지 결정 후 7/10 실제 요구·운영 비용 재평가로 변경 |
| 기존 Redis의 Streams | 7/10 비교. 캐시와 이벤트 보존 책임을 함께 두는 대가 고려 |
| NATS + local-path | 7/10 초안. 7/22 볼륨 재배분으로 철회, local-path provisioner도 도입하지 않음 |
| NATS + oci-bv | 최종 채택. OpenBao가 반납한 볼륨 예산을 이벤트 저장에 사용 |

**결과와 검증**

[9/30 관찰](09-evidence/E05-2026-09-30-cli-observation.md)에서 NATS 서버와 50Gi PVC를 확인했습니다. 단일 서버이므로 고가용성 구성이 아니며, PVC의 메시지 데이터는 별도 백업·복원 경로가 필요합니다.

NATS Application은 Healthy지만 StatefulSet 차이로 OutOfSync였습니다. 정상 실행과 선언 일치는 구분합니다. 현재 계정·사용자 인증과 호출자 제한도 완료된 상태가 아닙니다.

**재검토 조건**

처리량·consumer 수·replay 보존량을 측정해 단일 서버가 요구를 만족하는지 확인합니다. 독립 장애 영역·고가용성, 멀티테넌트 subject 권한, 실제 데이터 복원 요구가 생기면 복제·인증·보존 정책을 재검토합니다.

**관련 구현**

- 범위: NATS JetStream, 기존 Kafka·Strimzi, 이벤트 저장 책임
- 근거: [NATS README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/nats/README.md), [values](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/nats/values.yaml), [ArgoCD Application](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/argocd/apps/nats.yaml)

</details>

<a id="observability"></a>

## 관측과 알림

현재 자원, 장기 지표, 사람에게 도달하는 알림의 역할을 나누고 실제로 연결한 범위를 기록했습니다.

<a id="adr-0022"></a>

<details>
<summary>0022 · kube-prometheus-stack과 Resource Metrics API 분리</summary>

- 상태: 채택
- 기록일: 2026-09-30

**배경**

노드·Pod의 현재 자원과 시간에 따른 지표를 모두 확인해야 했습니다. 실제 사용량과 requests 장부가 달라지는 운영 문제도 있어 두 값을 구분해서 관찰할 필요가 있었습니다.

**선택**

kube-prometheus-stack으로 operator·Prometheus·Alertmanager·Grafana·node-exporter·kube-state-metrics를 함께 관리합니다. OKE에서 수집 대상으로 쓰지 않는 관리형 제어면 endpoint는 비활성화합니다. 서비스는 ServiceMonitor, NATS는 PodMonitor로 연결합니다.

Prometheus는 `oci-bv` 50Gi, retention 15d·45GiB를 선언하고 Grafana·Alertmanager는 ephemeral로 둡니다. Grafana 관리자 자격은 existingSecret으로 Git 밖에서 공급합니다. 관리 UI 경로는 [Tailscale](#adr-0008)을 따릅니다.

`kubectl top`의 Resource Metrics API는 별도 metrics-server가 제공합니다. 현재 한 replica와 `--kubelet-insecure-tls`를 사용하므로, kubelet 인증서 검증 우회를 제거할 수 있도록 인증서 경로를 점검해야 합니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| Prometheus 계열을 개별 설치 | CRD·수집·기본 대시보드 정합 관리 비용을 고려해 stack 선택 |
| 장기 지표로만 현재 자원 확인 | Metrics API와 사용처가 달라 metrics-server를 별도로 설치 |
| 모든 관측 컴포넌트에 PVC | 저장 예산 안에서 지표 history를 우선 보존 |
| 중앙 로그·trace·HPA까지 함께 도입 | 후속 도입 후보로 분리 |

**결과와 검증**

[9/30 측정](09-evidence/E06-2026-09-30-platform-measurements.md)에서 두 노드의 메모리 working set은 allocatable 대비 약 86.96%·82.87%, CPU는 약 11.79%·9.38%였습니다. Pressure 조건은 False였고 당시 build Pod는 없었습니다.

kps의 Grafana Secret 차이와 과거 failed Helm revision은 [CLI 기록](09-evidence/E05-2026-09-30-cli-observation.md)에 정리했습니다. metrics-server는 자원 조회에 사용하며 HPA는 구성하지 않았습니다. Prometheus PVC의 복원 검증은 후속 과제입니다.

**재검토 조건**

빌드 중 부하·장기 추세·알림 정확도를 측정하고 지표 보존 기간을 조정합니다. kubelet 인증서 검증 우회는 실제 인증서 경로 확인 후 제거 가능성을 평가합니다. 로그·trace는 해결하려는 관측 공백이 생길 때 추가합니다.

**관련 구현**

- 범위: Prometheus Operator·Prometheus·Grafana·exporter·metrics-server
- 근거: [Monitoring README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/monitoring/README.md), [kps values](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/monitoring/values.yaml), [metrics-server README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/metrics-server/README.md), [metrics-server values](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/infra/metrics-server/values.yaml)

</details>

<a id="adr-0023"></a>

<details>
<summary>0023 · AlertmanagerConfig와 Discord로 운영 알림 연결</summary>

- 상태: 채택; Discord 수신 확인
- 기록일: 2026-09-30 / Discord 경로 결정: 2026-07-25

**배경**

CPU overcommit 알림이 발생했지만 수신처가 null이라 통보되지 않았습니다. 운영 알림을 확인하는 Discord로 경로를 연결하고, 라우팅은 Git에 남기되 자격은 분리하기로 했습니다.

**선택**

`AlertmanagerConfig`를 global 설정으로 참조합니다. route·receiver·inhibit 규칙은 Git에 두고 Discord webhook은 Secret key 참조로 공급합니다. global 모드에서 필요한 inhibition과 Watchdog → null 규칙도 함께 선언합니다.

초기 `webhook_url_file` 방식은 당시 Prometheus Operator 파서에서 거부되어 CRD의 Secret 참조 방식으로 바꿨습니다. 변경은 Helm 직접 upgrade가 아니라 ArgoCD의 monitoring-resources·kps 경로로 반영합니다.

공통 CI의 `post.failure`는 `JenkinsBuildFailed`를 Alertmanager에 전달합니다. 서비스·작업·빌드 식별자와 URL을 담고 10분 후 만료됩니다. UNSTABLE·ABORTED와 실제 CD 완료는 이 알림의 대상이 아닙니다.

**대안과 이유**

| 대안 | 판단 |
|---|---|
| null receiver 유지 | 실제 이전 상태. 관측 결과가 사람에게 전달되지 않아 교체 |
| 전체 config를 Secret에 숨김 | 당시 비교 후 미채택. 라우팅 변경을 Git에서 검토하고 싶었음 |
| values의 `webhook_url_file` | Alertmanager 바이너리 검증은 통과했지만 operator 파서가 거부한 실제 실패 경로 |
| global AlertmanagerConfig + Secret 참조 | 최종 채택. CRD와 operator를 포함한 전체 설정 경로로 검증 |
| CI 알림만을 위한 MQ 추가 | 7/10·7/25 검토에서 단순 수신 경로에는 webhook/API로 충분하다고 판단 |

**결과와 검증**

7/25에는 CRD 검증 → operator 생성 설정 → 런타임 설정 순서로 경로를 점검한 뒤 첫 실제 알림을 Discord에서 수신했습니다.

Jenkins 전송 실패는 로그에 남기고 원래 빌드 실패 결과를 유지합니다. Alertmanager와 Prometheus는 같은 클러스터에 있고 Watchdog는 null로 보냅니다. 클러스터 전체 장애를 감지하려면 외부 heartbeat·probe가 필요합니다.

**재검토 조건**

알림량이 늘면 severity별 라우팅·중복 억제를 조정합니다. 전체 클러스터 장애 통보가 필요하면 외부 heartbeat·probe를 검증하고, CI 실패와 실제 배포 상태를 구분한 알림을 추가합니다.

**관련 구현**

- 범위: Alertmanager, Prometheus Operator, Discord, Jenkins 실패 알림
- 근거: [Monitoring README](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/monitoring/README.md), [AlertmanagerConfig](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/monitoring/alertmanager-config.yaml), [kps values](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/monitoring/values.yaml), [Jenkins 실패 알림](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/vars/ci.groovy)

</details>

<details>
<summary>소스 커밋 · 주요 결정 변경 이력</summary>

**소스 기준**

| 저장소 | 고정 커밋 |
|---|---|
| oci-always-free-k8s | [bd4b1b7](https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/README.md) |
| app-templates | [dff19b1](https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/README.md) |
| jenkins-shared-library | [f762a6d](https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/README.md) |
| k8s-gitops | [3ef5320](https://github.com/GGingGGang/k8s-gitops/blob/3ef5320819bf94240db7d35924bd7858979d0167/README.md) |

**주요 결정의 변화**

| 이전 선택 | 변경 내용 |
|---|---|
| 6/26 thin library·서비스별 절차 | 7/21 `ci()`·services.yaml 중앙화, 7/24 언어 테스트 계약 |
| 7/6 Kafka 유지 | 7/10 NATS 교체 결정, 7/22 Kafka 제거·볼륨 재배분 |
| 7/10 Cilium chaining | 7/24 실행 전 철회. Ambient와 후속 접근 정책 방향 |
| 7/10 NATS local-path | 7/22 oci-bv로 변경. local-path provisioner 미도입 |
| 7/17 cosign v3·bundle·tlog 조정 | 같은 날 최종 v2·레거시 서명·tlog 미사용으로 피벗 |
| Enforce 때 digest 옵션 복원·OpenBao 선행 계획 | 7/30 Enforce 전환, digest 옵션 false 유지; 시크릿 이관 미완료 |
| 7/25 OpenBao 주기 스냅샷 결정 | 정기 스냅샷 방향 채택; 자동 스냅샷 작업과 복원 검증은 후속 과제 |
| 9/15 서비스 연동·워커를 담은 템플릿 초안 | 같은 날 최소 실행·자체 상태 확인으로 정정 |
| 9/28 Deployment PR·main/최신 SHA 가드 검토 | 기존 직접 main push 방식 유지, 추가 가드 미적용 |
| Redis는 모든 상태가 재생성 가능한 캐시 | 9/25 소비 계약 검토를 반영해 세션·락 용도 불일치 명시 |

</details>

---

<!-- reading-nav -->
[← 6. 운영](06-operations.md) · [8. 현재 상태 →](08-status.md) · [전체 읽기 순서](README.md)
<!-- /reading-nav -->
