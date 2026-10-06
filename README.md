# nebula-ci-templates

Nebula 서비스들이 공통으로 사용하는 CI/CD 파이프라인 템플릿.
서비스 레포는 아래처럼 inputs 몇 줄만 정의하면 테스트부터 배포까지 같은 보안 게이트를 통과한다.

```
Commit → Secrets Scan → Test → Build → Vuln Scan → Push → SBOM → Sign → GitOps(ArgoCD)
```

## 사용법

서비스 레포에 `.github/workflows/<service>.yml` 하나만 추가한다.
job id를 서비스 이름으로 두면 check 이름이 `service-order / build`처럼 서비스별로 구분된다.

```yaml
name: service-order
on:
  push: { branches: [main], paths: ['service-order/**', '.github/workflows/service-order.yml'] }
  pull_request:
permissions: { contents: read, packages: write, id-token: write }
jobs:
  service-order:
    uses: shashax42/nebula-ci-templates/.github/workflows/service-pipeline.yml@v1
    with: { service: service-order }
    secrets: inherit
```

| input | 기본값 | 설명 |
|---|---|---|
| `service` | (필수) | 서비스 디렉터리 이름이자 이미지 이름 |
| `java-version` | `21` | JDK 버전 |
| `run-tests` | `true` | `false`면 테스트 없이 컴파일만 검증 |
| `fail-on-severity` | `CRITICAL` | 이 등급 취약점이 있으면 실패 (예 `CRITICAL,HIGH`) |
| `gitops-repo` | `shashax42/nebula-gitops` | 이미지 태그를 갱신할 레포 |
| `deploy-branch` | `main` | 이 브랜치 push일 때만 푸시·서명·배포 |

| secret | 설명 |
|---|---|
| `GITOPS_DEPLOY_KEY` | `gitops-repo`의 **write deploy key** (SSH private key). 브랜치 보호를 우회할 수 있는 유일한 경로 |
| `GITOPS_TOKEN` | (deprecated) deploy key가 없을 때만 쓰는 PAT. 브랜치 보호가 켜지면 거부된다 |

## 단계별 동작

| 단계 | 도구 | PR | main push |
|---|---|---|---|
| Changes | git diff | 서비스가 안 바뀌었으면 이후 단계 skip | 항상 실행 |
| Secrets Scan | gitleaks | ✅ 실패 시 차단 | ✅ |
| Test | Gradle | ✅ | ✅ |
| Build + Vuln Scan | Buildx, Trivy | ✅ 실패 시 차단 | ✅ |
| Push | GHCR (`sha-<commit>` 태그) | – | ✅ |
| SBOM | Syft (SPDX) → cosign attestation | – | ✅ |
| Sign | cosign keyless (GitHub OIDC) | – | ✅ |
| GitOps | gitops 레포의 이미지를 `tag@digest`로 갱신 | – | ✅ |

서명은 키 없이 GitHub Actions OIDC로 한다. 클러스터에서는 Kyverno `verifyImages` 정책으로
이 워크플로가 서명한 이미지만 배포되도록 검증한다.

## 버전 관리

호출하는 쪽은 `@v1`처럼 태그를 고정해서 사용한다.
호환되는 변경은 `v1` 태그를 옮기고, 깨지는 변경은 `v2`로 올린다.

## 브랜치 보호 (Forced Review)

모든 Nebula 레포의 main은 같은 규칙을 따른다. 규칙 정의는 각 레포의 `.github/rulesets/main-protection.json`.

| 규칙 | 내용 |
|---|---|
| PR 필수 | main 직접 push 금지. 모든 변경은 PR로 들어오고 CODEOWNERS에게 리뷰 요청 |
| CI 필수 | 레포별 required check가 통과해야 머지 가능 |
| 이력 보호 | force push, 브랜치 삭제 금지 |
| 예외 | `nebula-gitops`만 deploy key(CI 봇)의 직접 push 허용 — 사람은 예외 없음 |

PR에서 서비스가 바뀌지 않으면 `changes` job이 이후 단계를 skip한다.
GitHub는 skip된 required check를 통과로 보기 때문에, 관련 없는 PR이 Pending에 묶이지 않는다.
