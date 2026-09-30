# 2026-09-30 JavaScript 템플릿 생성 검증

| 순서 | 문서 | 이동 |
|---|---|---|
| 0 | [프로젝트 개요](../../README.md) | 이전 |
| **E01** | **템플릿 생성 검증** | **현재 부록** |
| E02 | [배포 추적](E02-2026-09-30-onboarding-deployment.md) | 다음 |

본문으로 돌아가기: [3. 서비스 온보딩](../03-service-onboarding.md) · [부록 목록](../README.md#실행-기록과-원자료) · [0. 프로젝트 개요](../../README.md)

JavaScript 템플릿을 새 임시 폴더에 생성해 기본 테스트·정적 빌드·Kustomize 렌더링을 통과했습니다. **테스트 6개 통과, 실패 0개, skip 0개**이며, 실행 범위는 로컬 생성과 빌드입니다.

## 실행 조건

| 항목 | 값 |
|---|---|
| 템플릿 / SHA | `javascript-app` / `dff19b1e2562fd249c05d4f134000e334ee26bd1` |
| 환경 | Windows PowerShell + Git for Windows Bash |
| 로컬 도구 | Node `v24.21.0`, npm `11.19.0` |
| 표준 CI | `node:22` — 로컬 검증과 Node 버전이 다름 |
| 입력 | `PortfolioExample` / `portfolio-web` / `portfolio.example.com` |
| 출력 | OS 임시 폴더의 새 `ggang-portfolio-20260930-<run-id>` |

## 명령과 결과

`<RUN>`은 생성물을 저장한 임시 폴더입니다.

```bash
bash --noprofile --norc app-templates/javascript-app/sed-template.sh \
  PortfolioExample portfolio-web <RUN> portfolio.example.com
cd <RUN>/svc-portfolio-web
npm ci --ignore-scripts --no-audit --no-fund
npm test
npm run build
kubectl kustomize <RUN>/gitops-portfolio-web/manifests/portfolio-web
```

로컬 설치는 `npm ci --ignore-scripts`로 실행하고 test와 build를 별도로 수행했습니다. 이 템플릿에는 외부 dependency와 install hook이 없습니다. 표준 Jenkins는 `npm ci && npm test`를 사용하므로 Node 버전과 설치 옵션이 다릅니다.

| 확인 | 결과 |
|---|---|
| 생성 | exit 0, 서비스·GitOps 폴더 생성 |
| 설치 | exit 0, `up to date in 16s` |
| 테스트 | exit 0, pass 6 / fail 0 / skipped 0 |
| 정적 빌드 | exit 0 |
| 잔여 토큰 | 생성기 내부 전체 파일 검사 통과; 별도 rg도 일치 없음 |
| Kustomize | exit 0, Service·Deployment·HTTPRoute |
| 이미지 | `ghcr.io/portfolioexample/svc-portfolio-web:bootstrap` |
| HTTPRoute | host `portfolio.example.com`, namespace `portfolio-web` |

테스트 출력입니다. 개별 테스트 시간은 생략했습니다.

```text
✔ static shell keeps runtime configuration external
✔ nginx exposes its own probes
✔ entrypoint renders runtime files under tmp
✔ build script emits browser assets
✔ browser helpers read config and format service status
✔ dev server serves probes and runtime browser assets
tests 6
pass 6
fail 0
cancelled 0
skipped 0
todo 0
duration_ms 433.5799
```

총 시간은 위 로컬 환경의 테스트 실행 시간입니다. 개발 서버 테스트는 루프백 임의 포트를 사용합니다.

## 검증 범위

JavaScript 소스와 배포 매니페스트 생성, 기본 테스트, 정적 빌드, Kustomize 렌더링까지의 결과입니다. 컨테이너 이미지 빌드와 운영 배포는 별도 단계입니다.

렌더링된 이미지 태그 `bootstrap`은 첫 CI의 Bump에서 소스 SHA로 갱신됩니다. 이 단계의 산출물은 임시 폴더에 생성된 코드와 매니페스트입니다.

---

<!-- reading-nav -->
[← 0. 프로젝트 개요](../../README.md) · [E02. 배포 추적 →](E02-2026-09-30-onboarding-deployment.md) · [전체 읽기 순서](../README.md)
<!-- /reading-nav -->
