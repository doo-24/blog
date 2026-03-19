---
title: "[배포와 CI/CD] 5편 — CI/CD 파이프라인: 자동화된 품질 게이트"
date: 2026-03-17T20:03:00+09:00
draft: false
tags: ["CI/CD", "GitHub Actions", "GitOps", "ArgoCD", "서버"]
series: ["배포와 CICD"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 6
summary: "CI 파이프라인 설계(빌드→테스트→정적 분석→보안 스캔), GitHub Actions/GitLab CI 실전 구성, CD 전략과 환경별 승인 게이트, GitOps 개념(ArgoCD, Flux)과 선언적 배포까지"
---

배포는 코드가 세상과 만나는 순간이다.

그 순간이 두려운 팀은 배포를 미루고, 미룬 배포는 더 큰 변경 덩어리가 되어 더 큰 공포를 낳는다.

반대로 배포가 버튼 하나 혹은 코드 푸시 한 번으로 끝나는 팀은 하루에도 수십 번 배포한다.

그 차이를 만드는 것이 CI/CD 파이프라인이다.

파이프라인은 단순한 자동화 스크립트가 아니다. 팀이 합의한 품질 기준을 코드로 표현한 것이며, 사람이 놓칠 수 있는 실수를 기계가 반드시 잡아내도록 설계한 안전망이다.

이 글에서는 CI 파이프라인의 각 단계를 설계하는 원칙부터, GitHub Actions와 GitLab CI의 실전 구성, 환경별 CD 전략, 그리고 선언적 배포의 완성형인 GitOps까지 깊게 다룬다.

---

## CI 파이프라인 설계: 빌드 → 테스트 → 정적 분석 → 보안 스캔

CI(Continuous Integration)의 본질은 변경사항이 메인 브랜치에 합류하기 전에 품질을 검증하는 것이다.

파이프라인의 각 단계는 순서에 의미가 있다. 빠르게 실패할 수 있는 단계를 앞에 두어 개발자 피드백 루프를 최소화한다.

### 단계별 설계 원칙

**빌드 단계**는 가장 먼저 실행된다. 코드가 컴파일되지 않는다면 이후 단계는 의미가 없다. 의존성 캐싱이 핵심이다. `node_modules`, Maven의 `.m2`, Go의 모듈 캐시를 파이프라인 캐시로 저장해 두면 빌드 시간이 수 분에서 수십 초로 줄어든다.

```
[빌드] → [단위 테스트] → [통합 테스트] → [정적 분석] → [보안 스캔] → [아티팩트 배포]
  30초        1분            3분              2분            2분              30초
```

각 단계의 목표 실행 시간을 정해 두는 것이 중요하다.

전체 CI 파이프라인이 10분을 넘기 시작하면 개발자들은 피드백을 기다리지 않고 다른 작업으로 전환한다.

컨텍스트 스위칭 비용이 발생하고, 파이프라인 실패 시 이미 다른 일을 하고 있어 수정이 늦어진다.

**테스트 단계**는 단위 테스트와 통합 테스트를 분리하는 것이 좋다. 단위 테스트는 외부 의존성 없이 수십 초 안에 끝나야 한다.

통합 테스트는 데이터베이스, 메시지 큐 등 실제 인프라가 필요하므로 Docker Compose나 Testcontainers를 활용한다.

**정적 분석(Static Analysis)**은 코드를 실행하지 않고 문제를 찾는다. 린터(ESLint, golangci-lint, Checkstyle)는 코드 스타일과 잠재적 버그를, SonarQube나 CodeClimate는 복잡도와 중복을 측정한다. 정적 분석은 코드 리뷰어가 놓치는 패턴을 잡는다.

**보안 스캔**은 최근 가장 중요성이 높아진 단계다. 의존성 취약점 스캔(Dependabot, Snyk, Trivy), 시크릿 누출 탐지(TruffleHog, Gitleaks), SAST(Static Application Security Testing)를 포함한다.

### 안티패턴: 모든 것을 직렬로 실행하기

가장 흔한 실수는 모든 단계를 순서대로만 실행하는 것이다. 단위 테스트와 정적 분석은 서로 의존하지 않으므로 병렬로 실행할 수 있다.

빌드가 완료된 후 단위 테스트와 정적 분석을 동시에 실행하면 전체 파이프라인 시간이 크게 줄어든다.

---

## GitHub Actions 실전 구성

GitHub Actions는 현재 가장 널리 사용되는 CI/CD 플랫폼이다.

YAML로 워크플로를 정의하고, 마켓플레이스의 수천 가지 액션을 재사용할 수 있다.

### 기본 CI 워크플로

다음은 Node.js 백엔드 서비스의 실전 CI 파이프라인이다.

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: "20"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # 빌드와 단위 테스트
  build-and-test:
    name: Build & Unit Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Unit tests
        run: npm run test:unit -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true

  # 정적 분석 (빌드와 병렬 실행)
  lint-and-analyze:
    name: Lint & Static Analysis
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: ESLint
        run: npm run lint -- --format=@microsoft/eslint-formatter-sarif --output-file=eslint-results.sarif
        continue-on-error: true

      - name: Upload ESLint results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: eslint-results.sarif
          wait-for-processing: true

      - name: TypeScript type check
        run: npm run type-check

  # 보안 스캔 (빌드와 병렬 실행)
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: "fs"
          scan-ref: "."
          format: "sarif"
          output: "trivy-results.sarif"
          severity: "CRITICAL,HIGH"

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: "trivy-results.sarif"

      - name: Secret scanning with Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # 통합 테스트 (빌드 성공 후 실행)
  integration-test:
    name: Integration Test
    runs-on: ubuntu-latest
    needs: build-and-test
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run database migrations
        run: npm run db:migrate
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb

      - name: Integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
          NODE_ENV: test

  # 컨테이너 이미지 빌드 및 푸시 (main 브랜치에만)
  build-image:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: [build-and-test, lint-and-analyze, security-scan, integration-test]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    permissions:
      contents: read
      packages: write
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Sign image with Cosign
        uses: sigstore/cosign-installer@v3
        with:
          cosign-release: "v2.2.3"

      - name: Sign the container image
        run: |
          cosign sign --yes ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
        env:
          COSIGN_EXPERIMENTAL: 1
```

### 재사용 가능한 워크플로 (Reusable Workflows)

여러 서비스가 동일한 CI 패턴을 공유할 때 재사용 가능한 워크플로를 활용한다.

```yaml
# .github/workflows/reusable-node-ci.yml
name: Reusable Node.js CI

on:
  workflow_call:
    inputs:
      node-version:
        required: false
        type: string
        default: "20"
      run-integration-tests:
        required: false
        type: boolean
        default: true
    secrets:
      CODECOV_TOKEN:
        required: true

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
      - run: npm ci
      - run: npm run build
      - run: npm test
```

```yaml
# 각 서비스의 워크플로에서 호출
# .github/workflows/ci.yml (서비스 A)
name: CI
on: [push, pull_request]
jobs:
  ci:
    uses: ./.github/workflows/reusable-node-ci.yml
    with:
      node-version: "20"
      run-integration-tests: true
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

---

## GitLab CI 실전 구성

GitLab CI는 `.gitlab-ci.yml` 하나로 파이프라인 전체를 정의한다.

특히 온프레미스 환경이나 GitLab.com을 사용하는 팀에 적합하다.

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - analyze
  - package
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

# 캐시 전략 정의
.node-cache: &node-cache
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/
    policy: pull-push

# 빌드
build:
  stage: build
  image: node:20-alpine
  <<: *node-cache
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

# 단위 테스트 (병렬)
unit-test:
  stage: test
  image: node:20-alpine
  <<: *node-cache
  needs: [build]
  script:
    - npm ci
    - npm run test:unit -- --coverage --reporter=junit --output-file=junit.xml
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    when: always
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

# 정적 분석 (병렬)
lint:
  stage: test
  image: node:20-alpine
  <<: *node-cache
  needs: [build]
  script:
    - npm ci
    - npm run lint
    - npm run type-check

# 보안 스캔 (GitLab 내장 기능 활용)
dependency-scan:
  stage: analyze
  image: registry.gitlab.com/gitlab-org/security-products/analyzers/gemnasium:latest
  script:
    - /analyzer run
  artifacts:
    reports:
      dependency_scanning: gl-dependency-scanning-report.json

sast:
  stage: analyze
  include:
    - template: Security/SAST.gitlab-ci.yml

secret-detection:
  stage: analyze
  include:
    - template: Security/Secret-Detection.gitlab-ci.yml

# 컨테이너 빌드
build-image:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  needs:
    - job: unit-test
    - job: lint
    - job: dependency-scan
  only:
    - main
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
    - docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest

# 스테이징 배포 (자동)
deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: [build-image]
  only:
    - main
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - kubectl set image deployment/api api=$IMAGE_TAG -n staging
    - kubectl rollout status deployment/api -n staging --timeout=5m

# 프로덕션 배포 (수동 승인)
deploy-production:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: [deploy-staging]
  only:
    - main
  when: manual
  environment:
    name: production
    url: https://example.com
  script:
    - kubectl set image deployment/api api=$IMAGE_TAG -n production
    - kubectl rollout status deployment/api -n production --timeout=10m
```

---

## CD 전략과 환경별 승인 게이트

CD(Continuous Delivery/Deployment)는 CI가 성공한 아티팩트를 각 환경에 배포하는 과정이다.

환경 수와 배포 전략에 따라 설계가 크게 달라진다.

### 환경 계층 설계

일반적인 환경 계층은 다음과 같다.

```
[개발자 로컬] → [dev] → [staging] → [production]
                 자동      자동        수동 승인
```

각 환경의 목적과 승인 정책을 명확히 해야 한다.

| 환경 | 목적 | 배포 트리거 | 승인 |
|------|------|-------------|------|
| dev | 기능 통합 검증 | feature 브랜치 머지 | 자동 |
| staging | 릴리스 후보 검증 | main 브랜치 푸시 | 자동 |
| production | 실제 서비스 | main 브랜치 태그 | 수동 |

### GitHub Actions로 환경별 승인 게이트 구현

```yaml
# .github/workflows/cd.yml
name: CD Pipeline

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: "배포 환경"
        required: true
        type: choice
        options: [staging, production]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_STAGING_ROLE_ARN }}
          aws-region: ap-northeast-2

      - name: Deploy to EKS
        run: |
          aws eks update-kubeconfig --name staging-cluster --region ap-northeast-2
          kubectl set image deployment/api \
            api=${{ secrets.REGISTRY }}/api:${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/api -n staging --timeout=5m

      - name: Run smoke tests
        run: |
          sleep 30
          curl -f https://staging.example.com/health || exit 1

  # 스테이징 성공 후 프로덕션은 수동 승인
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production          # GitHub 환경 보호 규칙 적용
      url: https://example.com  # 필요한 리뷰어 지정 가능
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_ROLE_ARN }}
          aws-region: ap-northeast-2

      - name: Create deployment record
        id: deployment
        uses: chrnorm/deployment-action@v2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          environment: production

      - name: Deploy to EKS (Blue-Green)
        run: |
          aws eks update-kubeconfig --name prod-cluster --region ap-northeast-2

          # 새 버전을 green에 배포
          kubectl set image deployment/api-green \
            api=${{ secrets.REGISTRY }}/api:${{ github.sha }} \
            -n production
          kubectl rollout status deployment/api-green -n production --timeout=10m

          # 헬스체크 통과 후 트래픽 전환
          kubectl patch service api-service -n production \
            -p '{"spec":{"selector":{"slot":"green"}}}'

          # 이전 버전(blue) 정리
          kubectl scale deployment/api-blue --replicas=0 -n production

      - name: Update deployment status (success)
        if: success()
        uses: chrnorm/deployment-status@v2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          state: success
          deployment-id: ${{ steps.deployment.outputs.deployment_id }}

      - name: Update deployment status (failure)
        if: failure()
        uses: chrnorm/deployment-status@v2
        with:
          token: ${{ secrets.REGISTRY }}/api:${{ github.sha }}
          state: failure
          deployment-id: ${{ steps.deployment.outputs.deployment_id }}
```

### GitHub Environments 보호 규칙 설정

GitHub Repository Settings → Environments에서 다음을 설정한다.

- **Required reviewers**: 프로덕션 배포 전 반드시 승인해야 할 팀원 지정
- **Wait timer**: 스테이징 배포 후 일정 시간 대기 (예: 10분)
- **Deployment branches**: 특정 브랜치에서만 배포 허용

### 배포 전략 선택

**롤링 배포(Rolling Update)**는 가장 단순하다. 파드를 하나씩 교체하므로 다운타임이 없지만 배포 중에 두 버전이 동시에 서비스된다.

```yaml
# kubernetes rolling update strategy
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 추가로 생성할 최대 파드 수
      maxUnavailable: 0  # 배포 중 사용 불가 파드 수 (0 = 무중단)
```

**카나리 배포(Canary)**는 새 버전을 전체 트래픽의 일부(5~10%)에만 먼저 노출한다. 문제 발생 시 영향 범위가 작다.

```yaml
# Argo Rollouts 카나리 설정
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10   # 10% 트래픽을 새 버전으로
        - pause: {duration: 5m}
        - setWeight: 30
        - pause: {duration: 5m}
        - setWeight: 60
        - pause: {duration: 5m}
        - setWeight: 100
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1
        args:
          - name: service-name
            value: api-canary
```

---

## GitOps: 선언적 배포의 완성형

GitOps는 인프라와 애플리케이션의 원하는 상태(desired state)를 Git 저장소에 선언하고, 자동화된 에이전트가 실제 상태를 선언된 상태와 일치시키는 패턴이다.

핵심 원칙은 Git이 단일 진실 원천(Single Source of Truth)이라는 것이다.

### GitOps의 두 가지 접근

**Push 방식**은 CI가 완료되면 CD 시스템이 클러스터에 직접 배포를 밀어 넣는다. 전통적인 CI/CD 방식이다.

클러스터 접근 권한이 CI 시스템에 있어야 하므로 보안 측면에서 취약하다.

**Pull 방식**은 클러스터 내부의 에이전트(ArgoCD, Flux)가 Git 저장소를 주기적으로 폴링하고, 변경이 감지되면 스스로 동기화한다.

CI 시스템이 클러스터에 직접 접근할 필요가 없어 보안이 강화된다.

```
[Push 방식]
CI Server ──credentials──▶ Kubernetes Cluster

[Pull 방식]
CI Server ──push──▶ Git Repo ◀──pull── ArgoCD Agent (in cluster)
                                              │
                                              ▼
                                       Kubernetes Cluster
```

### ArgoCD 실전 구성

ArgoCD는 Kubernetes 네이티브 GitOps 도구다.

Git 저장소에 정의된 Kubernetes 매니페스트나 Helm 차트를 감지해 클러스터에 자동 동기화한다.

**저장소 구조 (App of Apps 패턴)**

```
gitops-repo/
├── apps/                          # ArgoCD Application 정의
│   ├── base/
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── staging/
│       │   └── kustomization.yaml
│       └── production/
│           └── kustomization.yaml
└── services/                      # 실제 서비스 매니페스트
    ├── api/
    │   ├── base/
    │   │   ├── deployment.yaml
    │   │   ├── service.yaml
    │   │   └── kustomization.yaml
    │   └── overlays/
    │       ├── staging/
    │       │   ├── kustomization.yaml
    │       │   └── patch-replicas.yaml
    │       └── production/
    │           ├── kustomization.yaml
    │           └── patch-replicas.yaml
    └── worker/
        └── ...
```

**ArgoCD Application 매니페스트**

```yaml
# apps/overlays/staging/api-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-staging
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # 삭제 시 리소스 정리
spec:
  project: staging

  source:
    repoURL: https://github.com/myorg/gitops-repo.git
    targetRevision: main
    path: services/api/overlays/staging

  destination:
    server: https://kubernetes.default.svc
    namespace: staging

  syncPolicy:
    automated:
      prune: true      # Git에 없는 리소스 자동 삭제
      selfHeal: true   # 드리프트 발생 시 자동 복구
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - RespectIgnoreDifferences=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # 특정 필드 변경은 무시 (HPA가 레플리카 수를 관리하는 경우)
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

**ArgoCD RBAC 프로젝트 설정**

```yaml
# argocd-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: staging
  namespace: argocd
spec:
  description: Staging environment project

  # 허용된 소스 저장소
  sourceRepos:
    - https://github.com/myorg/gitops-repo.git
    - https://charts.bitnami.com/bitnami

  # 배포 대상 클러스터와 네임스페이스
  destinations:
    - namespace: staging
      server: https://kubernetes.default.svc

  # 허용되지 않는 리소스 종류
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace

  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota
    - group: ""
      kind: LimitRange

  # 동기화 창 (특정 시간대에만 자동 동기화)
  syncWindows:
    - kind: allow
      schedule: "* 9-18 * * 1-5"  # 평일 업무 시간
      duration: 9h
      applications:
        - "*"
      namespaces:
        - staging
```

**CI와 ArgoCD 연동 (Image Updater)**

ArgoCD Image Updater를 사용하면 컨테이너 레지스트리에 새 이미지가 푸시될 때 자동으로 GitOps 저장소를 업데이트한다.

```yaml
# services/api/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

images:
  - name: ghcr.io/myorg/api
    newTag: latest  # ArgoCD Image Updater가 이 값을 자동 업데이트

# ArgoCD Image Updater 어노테이션은 Application에 추가
```

```yaml
# Application에 Image Updater 설정 추가
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: api=ghcr.io/myorg/api
    argocd-image-updater.argoproj.io/api.update-strategy: newest-build
    argocd-image-updater.argoproj.io/api.allow-tags: regexp:^sha-[a-f0-9]+$
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
```

### Flux CD와의 비교

Flux는 CNCF 졸업 프로젝트로 ArgoCD와 함께 GitOps의 양대 산맥이다.

Flux는 더 CRD 기반으로 구성되고, Helm, Kustomize, OCI 아티팩트를 지원한다.

```yaml
# Flux GitRepository 소스 정의
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gitops-repo
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/gitops-repo.git
  ref:
    branch: main
  secretRef:
    name: github-credentials
---
# Flux Kustomization (ArgoCD Application에 해당)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: api-staging
  namespace: flux-system
spec:
  interval: 5m
  path: ./services/api/overlays/staging
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-repo
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: api
      namespace: staging
  timeout: 5m
  retryInterval: 30s
```

### 안티패턴: 시크릿을 Git에 저장하기

GitOps의 가장 위험한 함정은 시크릿을 평문으로 Git에 저장하는 것이다.

반드시 암호화 도구를 사용해야 한다.

**Sealed Secrets**는 kubeseal CLI로 시크릿을 암호화한 SealedSecret 리소스를 Git에 저장한다.

클러스터의 sealed-secrets-controller만 복호화할 수 있다.

```bash
# 시크릿 암호화
kubectl create secret generic db-credentials \
  --from-literal=password=mysecretpassword \
  --dry-run=client -o yaml | \
  kubeseal --controller-namespace=kube-system \
  --controller-name=sealed-secrets \
  --format=yaml > sealed-db-credentials.yaml

# sealed-db-credentials.yaml을 Git에 커밋하면 안전
```

**External Secrets Operator**는 AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager 등 외부 시크릿 저장소와 연동한다.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: production/db/credentials
        property: password
```

---

## 파이프라인 관찰 가능성

파이프라인 자체도 모니터링해야 한다.

배포 빈도, 변경 실패율, 평균 복구 시간(MTTR), 변경 리드 타임은 DORA 지표로 알려진 DevOps 성과 측정 기준이다.

```yaml
# GitHub Actions에서 배포 지표 수집
- name: Record deployment metrics
  run: |
    # Datadog에 배포 이벤트 전송
    curl -X POST "https://api.datadoghq.com/api/v1/events" \
      -H "Content-Type: application/json" \
      -H "DD-API-KEY: ${{ secrets.DATADOG_API_KEY }}" \
      -d @- <<EOF
    {
      "title": "Deployment to ${{ github.event.inputs.environment }}",
      "text": "Deployed ${{ github.sha }} to production",
      "tags": [
        "env:production",
        "service:api",
        "deployer:${{ github.actor }}"
      ],
      "alert_type": "info"
    }
    EOF
```

### 파이프라인 실패 알림

```yaml
# Slack 알림 추가
- name: Notify Slack on failure
  if: failure()
  uses: slackapi/slack-github-action@v1.25.0
  with:
    payload: |
      {
        "text": ":x: 배포 실패",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "*배포 실패* :x:\n*환경:* ${{ github.event.inputs.environment }}\n*커밋:* `${{ github.sha }}`\n*작성자:* ${{ github.actor }}\n*링크:* ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
          }
        ]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

---

## 정리

CI/CD 파이프라인은 팀의 엔지니어링 문화를 코드로 표현한 것이다.

잘 설계된 파이프라인은 개발자가 더 자주, 더 작은 단위로, 더 자신 있게 배포할 수 있게 만든다.

핵심 원칙을 다시 정리하면 이렇다.

첫째, CI는 빠르게 실패해야 한다. 빌드와 단위 테스트가 5분 안에 끝나지 않으면 개발자 경험이 나빠진다. 병렬화와 캐싱을 적극 활용하라.

둘째, CD는 환경별로 다른 승인 정책을 가져야 한다. 스테이징은 자동으로, 프로덕션은 수동 승인이 기본이다.

셋째, GitOps는 인프라 상태를 Git으로 관리해 감사 추적과 롤백을 쉽게 만든다. 특히 프로덕션 환경에서 ArgoCD나 Flux의 자동 동기화와 셀프힐링은 드리프트 문제를 원천 차단한다.

넷째, 시크릿은 절대 Git에 평문으로 저장하지 않는다. Sealed Secrets나 External Secrets Operator를 사용하라.

파이프라인은 완성이 없다. 팀이 성장하고 시스템이 복잡해질수록 파이프라인도 진화해야 한다.

DORA 지표를 정기적으로 측정하고, 느린 단계를 개선하고, 새로운 품질 게이트를 추가하는 작업을 멈추지 마라.

---

## 참고 자료

- [GitHub Actions 공식 문서](https://docs.github.com/en/actions) — 워크플로 문법, 재사용 가능한 워크플로, 환경 보호 규칙
- [ArgoCD 공식 문서](https://argo-cd.readthedocs.io/en/stable/) — GitOps 설치, Application 설정, RBAC, Image Updater
- [Flux CD 공식 문서](https://fluxcd.io/flux/) — GitOps Toolkit, Source Controller, Kustomize Controller
- [Google의 DORA 리서치](https://dora.dev/) — DevOps 성과 지표(배포 빈도, MTTR, 변경 실패율, 리드 타임)
- [Sealed Secrets by Bitnami](https://github.com/bitnami-labs/sealed-secrets) — Kubernetes 시크릿 암호화 도구
- [External Secrets Operator](https://external-secrets.io/latest/) — AWS/GCP/HashiCorp Vault와 Kubernetes 시크릿 연동
