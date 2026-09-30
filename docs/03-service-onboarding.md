# 서비스 템플릿과 개발자 온보딩

| 순서 | 문서 | 이동 |
|---|---|---|
| 2 | [인프라](02-infrastructure.md) | 이전 |
| **3** | **서비스 온보딩** | **현재** |
| 4 | [CI/CD](04-delivery.md) | 다음 |

[0. 프로젝트 개요](../README.md) · [전체 읽기 순서](README.md) · [부록 목록](README.md#실행-기록과-원자료)

## 문제와 개발자 경험

서비스마다 Dockerfile·Jenkinsfile·상태 확인·배포 YAML을 새로 구성하는 반복을 줄이기 위해 `app-templates`를 만들었습니다. **실행 가능한 서비스 템플릿과 배포 산출물**을 함께 생성하고, 업무 로직은 생성된 앱 저장소에서 개발하도록 나눴습니다. AI로 서비스를 구현할 때도 같은 실행·배포 규칙을 사용합니다.

## 지원하는 시작점

| 템플릿 | 제공하는 기반 | CI language |
|---|---|---|
| go-app | Go HTTP 서비스 | go |
| java-app/gradle | Java / Spring Boot 서비스 | java |
| node-app | TypeScript / Fastify 서비스 | node |
| javascript-app | JavaScript 정적 웹 + nginx | node |
| python-app | Python HTTP 서비스 | python |

Java Maven 변형도 있지만 현재 공통 `java` 테스트 명령은 Gradle입니다. Maven을 공통 파이프라인에서 사용하려면 CI 테스트 명령을 추가로 맞춰야 합니다.

구현: [서비스 템플릿][templates], [언어별 CI 계약][ci-config].

### 실행 계약

| 템플릿 | 컨테이너 포트 | 상태 확인 | 버전/환경 입력 |
|---|---:|---|---|
| Go | 8080 | /healthz · /readyz | HTTP_PORT, GIT_SHA를 빌드 ldflags에 반영 |
| Node/TypeScript | 3000 | /healthz · /readyz | HTTP_PORT, APP_VERSION |
| Java/Gradle | 8080 | startup/liveness /healthz, readiness /readyz | HTTP_PORT, APP_VERSION |
| JavaScript/nginx | 8080 | nginx 자체 /healthz · /readyz | GIT_SHA → APP_VERSION, runtime 파일은 /tmp |
| Python | 8080 | /healthz · /readyz | HTTP_PORT, SERVICE_NAME, APP_VERSION |

이 표는 템플릿 기본값입니다. 생성 후 개발자가 바꾸면 Dockerfile·애플리케이션·Deployment·Service의 설정을 함께 맞춰야 합니다. DB 등 업무 의존성을 probe에 반영할지는 앱별로 확인할 항목입니다.

## 생성기와 산출물

`sed-template.sh`는 조직·서비스 이름을 입력받아 토큰을 치환하고 두 종류의 산출물을 만듭니다. JavaScript 웹 템플릿은 공개 hostname도 입력받습니다.

| 산출물 | 이동할 곳 | 이후 소유자 |
|---|---|---|
| `svc-<name>/` | 새 서비스 저장소 루트 | 개발자 |
| `gitops-<name>/manifests/<name>/` | k8s-gitops의 manifests | 배포 설정 담당 |
| `gitops-<name>/argocd/apps/<name>.yaml` | k8s-gitops의 argocd/apps | 배포 설정 담당 |

[Node 생성기][generator]와 [웹 생성기][web-generator]는 파일 복사·토큰 치환·미해결 토큰 검사를 수행합니다. 생성기의 덮어쓰기 정책은 템플릿마다 다르므로 **새 빈 출력 경로에서 생성**하고 기존 서비스 폴더에 재실행하지 않습니다.

생성된 서비스는 독립적으로 관리합니다. 이후 템플릿 변경은 기존 서비스의 호환성을 확인한 뒤 개별 반영합니다.

## 개발자가 제출하는 것과 플랫폼이 준비하는 것

| 개발자 역할 | 플랫폼 역할 |
|---|---|
| 템플릿 규약에 맞는 소스·테스트·Dockerfile | 실행 가능한 언어별 테스트/빌드 환경 |
| 포트·health/readiness·환경변수 정의 | Deployment/Service/probe 설정 연결 |
| 필요한 DB/시크릿/공개 host 요청 | namespace·권한·시크릿·라우트 준비 |
| 공통 CI를 호출하는 Jenkinsfile | Shared Library·서비스 등록·잡 발견 |
| 빌드 실패 수정, 새 버전 제출 | 이미지 검사/서명·GitOps 갱신·배포 상태 관찰 |

생성된 [Jenkinsfile][jenkinsfile]은 공통 `ci(service: ...)`를 호출합니다. 개발자가 서비스마다 파이프라인 전체를 복사해 관리하지 않도록 한 경계입니다.

<a id="아직-남아-있는-온보딩-단계"></a>
## 생성 후 수동 온보딩

1. `services.yaml`에 서비스와 language를 등록합니다. 없으면 공통 CI가 초기 단계에서 실패합니다.
2. 플랫폼에 namespace를 추가하고, apps AppProject의 destination에 연결합니다.
3. 이미지 pull·런타임 시크릿을 해당 namespace에 준비합니다.
4. 필요한 경우 전용 DB 사용자/스키마를 준비하고 앱 배포 설정에서 참조합니다.
5. 생성한 매니페스트와 Application을 GitOps 저장소에 반영합니다.
6. Jenkins가 저장소를 발견했는지, 빌드와 배포 상태가 연결되는지 확인합니다.

Jenkins의 저장소 발견은 [JCasC][jenkins-values]의 `svc-*` 필터·주기 스캔에 기반합니다. webhook을 등록한 뒤에도 저장소 발견과 잡 생성 여부를 확인해야 합니다.

상세 실행 명령은 [템플릿 README][template-guide]와 [DB 온보딩 스크립트][db-guide]에 정리했습니다.

<a id="공통-개선을-어디에-반영하는가"></a>
## 공통 개선의 반영 위치

- 특정 앱의 업무 기능이면 해당 앱에서 끝냅니다.
- 생성되는 모든 같은 언어 서비스에 필요한 수정이면 템플릿에 반영합니다.
- 테스트 명령·검사 정책 같은 공통 절차면 Shared Library를 바꿉니다.
- 이미 생성된 서비스는 호환성을 확인한 뒤 필요한 수정만 별도로 적용합니다.

예를 들어 [Node Dockerfile][dockerfile]은 package.json과 lockfile을 함께 복사하고 `npm ci`로 의존성을 설치합니다. 이 규약을 템플릿에 두어 새 Node 서비스에 같은 설치 방식을 적용합니다.

<a id="검증과-확장-기준"></a>
## 템플릿 생성과 서비스 배포

2026-09-30에 JavaScript 템플릿을 새 임시 폴더에 생성해 기본 테스트 6개·정적 빌드·Kustomize 렌더링을 통과했습니다. 로컬 생성에 사용한 툴체인·명령·결과는 [템플릿 실행 기록](09-evidence/E01-2026-09-30-template-generation.md)에 정리했습니다.

운영 중인 서비스의 배포 흐름은 [batch #23](09-evidence/E02-2026-09-30-onboarding-deployment.md)에서 확인했습니다. 공통 CI가 게시한 이미지 digest를 GitOps 변경·ArgoCD 이력·실행 imageID까지 추적했습니다.

[templates]: https://github.com/GGingGGang/app-templates/tree/dff19b1e2562fd249c05d4f134000e334ee26bd1
[ci-config]: https://github.com/GGingGGang/jenkins-shared-library/blob/f762a6d841dc23bae11ace3628d5fb8d3b7c120d/resources/ci/services.yaml
[generator]: https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/node-app/sed-template.sh
[web-generator]: https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/javascript-app/sed-template.sh
[jenkinsfile]: https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/node-app/Jenkinsfile
[jenkins-values]: https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/kubernetes/platform/jenkins/values.yaml
[template-guide]: https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/README.md
[db-guide]: https://github.com/GGingGGang/oci-always-free-k8s/blob/bd4b1b787955f844a672ae2e369576108b3cec22/scripts/README.md
[dockerfile]: https://github.com/GGingGGang/app-templates/blob/dff19b1e2562fd249c05d4f134000e334ee26bd1/node-app/Dockerfile

---

<!-- reading-nav -->
[← 2. 인프라](02-infrastructure.md) · [4. CI/CD →](04-delivery.md) · [전체 읽기 순서](README.md)
<!-- /reading-nav -->
