---
title: "[Docker & 컨테이너] 7편 — Helm과 K8s 패키지 관리: 복잡한 매니페스트를 다루는 법"
date: 2026-03-20T14:54:00+09:00
draft: false
tags: ["Helm", "Kustomize", "ArgoCD", "GitOps", "Kubernetes", "서버"]
series: ["Docker와 컨테이너"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 11
summary: "Helm 차트 구조와 values.yaml·템플릿 문법, Chart 버저닝과 릴리즈 관리·롤백, Kustomize vs Helm 비교와 환경별 설정 오버레이, GitOps와 Helm(ArgoCD + Helm Chart 배포 플로우)까지"
---

## 들어가며

쿠버네티스 클러스터를 운영하다 보면 YAML 파일이 기하급수적으로 늘어난다. Deployment, Service, ConfigMap, Ingress, Secret... 각 애플리케이션마다 수십 개의 리소스 정의가 필요하다.

이 복잡도를 관리하는 대표적인 도구가 **Helm**과 **Kustomize**다. 이번 편에서는 두 도구의 구조와 사용법, 그리고 GitOps 패턴과의 통합 방식까지 살펴본다.

---

## 1. Helm이란 무엇인가

Helm은 쿠버네티스의 **패키지 매니저**다. npm이 Node.js 패키지를 관리하듯, Helm은 K8s 애플리케이션 패키지(Chart)를 관리한다.

핵심 개념은 세 가지다.

- **Chart**: 쿠버네티스 리소스 템플릿 묶음 (패키지)
- **Release**: Chart를 클러스터에 배포한 인스턴스
- **Repository**: Chart를 저장하고 공유하는 저장소

Helm CLI를 설치하는 방법은 다음과 같다.

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 버전 확인
helm version
```

---

## 2. Helm Chart 구조

`helm create myapp` 명령으로 기본 차트 구조를 생성할 수 있다.

```
myapp/
├── Chart.yaml          # 차트 메타데이터
├── values.yaml         # 기본 설정값
├── charts/             # 의존 차트 디렉토리
└── templates/          # 쿠버네티스 리소스 템플릿
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── _helpers.tpl    # 재사용 가능한 템플릿 헬퍼
    ├── NOTES.txt       # 설치 후 출력되는 안내 메시지
    └── tests/
        └── test-connection.yaml
```

### Chart.yaml

차트의 메타데이터를 정의하는 파일이다.

```yaml
apiVersion: v2
name: myapp
description: A Helm chart for my application
type: application
version: 1.2.0        # 차트 버전
appVersion: "2.5.1"   # 배포하는 앱의 버전

maintainers:
  - name: Hong Gildong
    email: hong@example.com

dependencies:
  - name: postgresql
    version: "12.1.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

`version`은 차트 자체의 버전이고, `appVersion`은 패키징되는 애플리케이션 버전이다. 둘은 독립적으로 관리된다.

---

## 3. values.yaml과 템플릿 문법

### values.yaml 구조

`values.yaml`은 차트의 기본 설정값을 담는다. 배포 시 이 값을 오버라이드할 수 있다.

```yaml
# values.yaml
replicaCount: 2

image:
  repository: myregistry.io/myapp
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  host: myapp.example.com
  tls: false

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
    password: ""   # 실제 환경에서는 Secret으로 관리
```

### 템플릿 문법 기초

Helm 템플릿은 Go 템플릿 엔진을 사용한다. `{{ }}` 안에 표현식을 작성한다.

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### _helpers.tpl 헬퍼 정의

재사용 가능한 템플릿 조각을 `_helpers.tpl`에 정의한다.

```yaml
# templates/_helpers.tpl
{{/*
앱의 풀네임을 생성한다.
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
공통 레이블
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
셀렉터 레이블
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

`{{- ... -}}`에서 `-`는 앞뒤 공백과 개행을 제거한다. YAML은 들여쓰기에 민감하기 때문에 템플릿 로직이 삽입한 빈 줄이 그대로 남으면 파싱 오류가 발생한다. `-`로 공백을 명시적으로 제거함으로써 렌더링된 YAML이 항상 올바른 형식을 유지한다. `nindent`는 지정한 칸만큼 들여쓰기를 추가하는데, `include`로 삽입된 멀티라인 블록이 상위 YAML 구조에 맞게 정렬되도록 하기 위해 사용한다.

### 조건문과 반복문

```yaml
# 조건문
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "myapp.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}

# 반복문: 환경 변수 목록 처리
env:
  {{- range .Values.env }}
  - name: {{ .name }}
    value: {{ .value | quote }}
  {{- end }}

# with: 스코프 축소
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

---

## 4. Chart 버저닝과 릴리즈 관리

### 설치와 업그레이드

```bash
# 로컬 차트 설치
helm install my-release ./myapp

# 커스텀 values로 설치
helm install my-release ./myapp \
  --values ./values-prod.yaml \
  --set image.tag=v2.1.0 \
  --set replicaCount=3

# 네임스페이스 지정
helm install my-release ./myapp \
  --namespace production \
  --create-namespace

# 차트 저장소에서 설치
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-postgres bitnami/postgresql \
  --set auth.postgresPassword=secret123

# 업그레이드 (없으면 설치)
helm upgrade --install my-release ./myapp \
  --set image.tag=v2.2.0
```

`--set`은 단일 값 오버라이드에, `--values`는 파일 단위 오버라이드에 사용한다. 둘을 함께 쓸 경우 `--set`이 우선순위가 높다.

### 릴리즈 상태 확인

```bash
# 설치된 릴리즈 목록
helm list -A

# 특정 릴리즈 상태
helm status my-release

# 렌더링된 매니페스트 확인 (실제 배포 전 검증)
helm template my-release ./myapp --values values-prod.yaml

# 차트 문법 검증
helm lint ./myapp

# dry-run으로 사전 확인
helm install my-release ./myapp --dry-run --debug
```

### 롤백

Helm은 모든 릴리즈 히스토리를 관리한다. 문제가 생기면 이전 버전으로 즉시 롤백할 수 있다.

```bash
# 릴리즈 히스토리 확인
helm history my-release

# REVISION  UPDATED                  STATUS     CHART         APP VERSION  DESCRIPTION
# 1         Mon Mar 20 09:00:00 2026 superseded myapp-1.0.0   2.0.0        Install complete
# 2         Mon Mar 20 11:00:00 2026 superseded myapp-1.1.0   2.1.0        Upgrade complete
# 3         Mon Mar 20 14:00:00 2026 deployed   myapp-1.2.0   2.2.0        Upgrade complete

# 특정 리비전으로 롤백
helm rollback my-release 2

# 바로 이전 버전으로 롤백
helm rollback my-release

# 롤백 후 상태 확인
helm history my-release
```

롤백 자체도 새로운 리비전으로 기록된다. 이력이 남기 때문에 감사(audit)가 용이하다.

### 릴리즈 삭제

```bash
# 릴리즈 삭제 (히스토리도 함께 삭제)
helm uninstall my-release

# 히스토리는 유지하면서 삭제
helm uninstall my-release --keep-history
```

---

## 5. Kustomize

### Kustomize란

Kustomize는 YAML 템플릿 없이 순수 쿠버네티스 매니페스트를 **패치 기반으로** 조합하는 도구다. kubectl에 내장되어 있어 별도 설치가 필요 없다.

핵심 아이디어는 **base + overlay** 구조다. 공통 설정을 base에 두고, 환경별 차이점만 overlay에서 정의한다.

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── development/
    │   ├── kustomization.yaml
    │   └── patch-replicas.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patch-resources.yaml
    └── production/
        ├── kustomization.yaml
        ├── patch-replicas.yaml
        └── patch-resources.yaml
```

### base 정의

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: myapp
  managed-by: kustomize
```

```yaml
# k8s/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myregistry.io/myapp:latest
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
```

### overlay로 환경별 설정

```yaml
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namePrefix: prod-

images:
  - name: myregistry.io/myapp
    newTag: "v2.2.0"

patchesStrategicMerge:
  - patch-replicas.yaml
  - patch-resources.yaml

configMapGenerator:
  - name: myapp-config
    literals:
      - ENV=production
      - LOG_LEVEL=warn
```

```yaml
# k8s/overlays/production/patch-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
```

```yaml
# k8s/overlays/production/patch-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp
          resources:
            limits:
              cpu: 1000m
              memory: 1Gi
            requests:
              cpu: 500m
              memory: 512Mi
```

Kustomize 적용 방법은 다음과 같다.

```bash
# 렌더링 결과 확인
kubectl kustomize k8s/overlays/production

# 실제 적용
kubectl apply -k k8s/overlays/production

# 삭제
kubectl delete -k k8s/overlays/production
```

---

## 6. Helm vs Kustomize 비교

두 도구는 철학이 다르다. 상황에 맞게 선택하거나 함께 사용하면 된다.

| 항목 | Helm | Kustomize |
|------|------|-----------|
| 방식 | 템플릿 기반 렌더링 | 패치 기반 오버레이 |
| 학습 곡선 | 높음 (Go 템플릿 문법) | 낮음 (순수 YAML) |
| 패키지 생태계 | 풍부 (Artifact Hub) | 없음 (직접 관리) |
| 릴리즈 관리 | 내장 (히스토리, 롤백) | 없음 (kubectl 위임) |
| 값 오버라이드 | values.yaml + --set | overlay 파일 |
| 비밀 관리 | helm-secrets 플러그인 | Sealed Secrets 등 외부 도구 |
| kubectl 내장 | 별도 설치 필요 | kubectl -k 내장 |

**Helm이 적합한 경우**: 공개 소프트웨어 배포, 복잡한 조건 분기, 릴리즈 이력 관리가 필요할 때.

**Kustomize가 적합한 경우**: 팀이 YAML에 익숙하고, 단순 환경별 차이만 관리할 때.

**함께 사용**: Helm으로 외부 의존성(PostgreSQL, Redis)을 관리하고, Kustomize로 자체 서비스를 관리하는 방식도 흔하다.

---

## 7. GitOps와 Helm: ArgoCD 통합

### GitOps란

GitOps는 Git 저장소를 단일 진실의 원천(Single Source of Truth)으로 사용하는 운영 방식이다. 클러스터 상태는 항상 Git에 선언된 상태와 일치해야 한다.

```text
개발자          Git 저장소          ArgoCD          K8s 클러스터
  │                 │                 │                 │
  │── git push ────►│                 │                 │
  │                 │                 │                 │
  │                 │◄── 폴링/웹훅 ───│                 │
  │                 │                 │                 │
  │                 │── 변경 감지 ────►│                 │
  │                 │                 │── sync ─────────►│
  │                 │                 │  (Helm 렌더링)   │
  │                 │                 │                 │── 리소스 생성/업데이트
  │                 │                 │◄── 상태 보고 ────│
  │                 │  Git = 클러스터  │                 │
  │                 │  상태 일치 확인  │                 │
```

수동 kubectl 명령을 지양하고, 모든 변경은 Git을 통해서만 이루어진다.

### ArgoCD 설치

```bash
# ArgoCD 네임스페이스 생성 및 설치
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ArgoCD CLI 설치 (macOS)
brew install argocd

# 초기 비밀번호 확인
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode

# 포트 포워딩으로 UI 접근
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### ArgoCD Application 정의 — Helm Chart 배포

```yaml
# argocd-app-helm.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/myapp-charts
    targetRevision: main
    path: charts/myapp

    # Helm 설정
    helm:
      releaseName: myapp
      valueFiles:
        - values-base.yaml
        - values-production.yaml
      parameters:
        - name: image.tag
          value: "v2.2.0"
        - name: replicaCount
          value: "5"

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true      # Git에서 삭제된 리소스를 클러스터에서도 삭제
      selfHeal: true   # 클러스터 상태가 Git과 달라지면 자동 복구
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
```

```bash
# Application 등록
kubectl apply -f argocd-app-helm.yaml

# CLI로 등록하는 방법
argocd app create myapp-production \
  --repo https://github.com/myorg/myapp-charts \
  --path charts/myapp \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --helm-set image.tag=v2.2.0 \
  --sync-policy automated

# 동기화 상태 확인
argocd app get myapp-production

# 수동 동기화
argocd app sync myapp-production

# 롤백 (이전 히스토리 ID로)
argocd app history myapp-production
argocd app rollback myapp-production <ID>
```

### ArgoCD Application 정의 — Kustomize 배포

```yaml
# argocd-app-kustomize.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/myapp-k8s
    targetRevision: main
    path: k8s/overlays/staging

    # Kustomize 설정
    kustomize:
      images:
        - myregistry.io/myapp:v2.2.0

  destination:
    server: https://kubernetes.default.svc
    namespace: staging

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### GitOps 배포 플로우 전체 그림

```
1. 개발자가 코드를 main 브랜치에 Push
         |
         v
2. CI 파이프라인 실행 (GitHub Actions / GitLab CI)
   - 테스트 실행
   - Docker 이미지 빌드 및 푸시
   - 이미지 태그: v2.3.0
         |
         v
3. Config 저장소 업데이트
   - values-production.yaml의 image.tag를 v2.3.0으로 변경
   - PR 생성 → 리뷰 → Merge
         |
         v
4. ArgoCD가 Config 저장소 변경 감지 (폴링 또는 Webhook)
         |
         v
5. ArgoCD가 클러스터 현재 상태와 Git 상태 비교 (Diff)
         |
         v
6. 자동 동기화: Helm upgrade 또는 kubectl apply -k 실행
         |
         v
7. 롤아웃 진행 (RollingUpdate 전략)
         |
         v
8. 헬스 체크 통과 → 배포 완료
   또는 실패 → ArgoCD 알림 발송
```

이 흐름에서 **클러스터에 직접 kubectl 명령을 치는 사람은 없다**. 모든 변경의 근거가 Git 커밋 히스토리에 남는다.

### 이미지 업데이트 자동화 — Argo CD Image Updater

매번 Config 저장소를 수동으로 업데이트하는 수고를 덜어주는 도구다.

```yaml
# Image Updater 어노테이션 추가
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=myregistry.io/myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    argocd-image-updater.argoproj.io/myapp.allow-tags: "^v[0-9]+\\.[0-9]+\\.[0-9]+$"
    argocd-image-updater.argoproj.io/write-back-method: git
spec:
  # ...
```

semver 전략으로 설정하면, `v2.x.x` 형식의 새 이미지가 레지스트리에 푸시될 때 자동으로 Config 저장소를 업데이트하고 배포까지 진행한다.

---

## 8. 실전 팁

### Helm Secret 관리

민감한 정보는 values.yaml에 평문으로 넣으면 안 된다. `helm-secrets` 플러그인과 SOPS를 활용한다.

```bash
# helm-secrets 플러그인 설치
helm plugin install https://github.com/jkroepke/helm-secrets

# secrets.yaml 암호화
sops --encrypt --age <recipient-key> secrets.yaml > secrets.enc.yaml

# 암호화된 파일로 배포
helm secrets upgrade --install my-release ./myapp \
  -f values.yaml \
  -f secrets.enc.yaml
```

### 차트 의존성 관리

```bash
# Chart.yaml의 dependencies를 다운로드
helm dependency update ./myapp

# 의존성 목록 확인
helm dependency list ./myapp
```

### 멀티 환경 values 파일 패턴

```bash
# 공통 → 환경별 → 긴급 오버라이드 순으로 적용
helm upgrade --install my-release ./myapp \
  -f values.yaml \          # 기본값
  -f values-prod.yaml \     # 운영 환경 오버라이드
  --set image.tag=${CI_COMMIT_TAG}  # CI에서 주입
```

---

## 마무리

Helm은 복잡한 쿠버네티스 애플리케이션을 패키지화하고 버전 관리하는 강력한 도구다. Kustomize는 템플릿 없이 순수 YAML로 환경별 설정을 관리하는 간단한 접근법을 제공한다.

ArgoCD와 결합하면 **선언적 GitOps 파이프라인**이 완성된다. 모든 배포 변경이 Git에 기록되고, 클러스터는 항상 Git 상태로 자동 수렴한다.

다음 편에서는 **쿠버네티스 네트워킹 심화**로, Ingress Controller와 Service Mesh(Istio)를 다룬다.

---

## 참고 자료

- [Helm 공식 문서](https://helm.sh/docs/)
- [Kustomize 공식 문서](https://kustomize.io/)
- [ArgoCD 공식 문서](https://argo-cd.readthedocs.io/)
- [Argo CD Image Updater](https://argocd-image-updater.readthedocs.io/)
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [GitOps with ArgoCD — CNCF Blog](https://www.cncf.io/blog/2021/01/06/gitops-with-argo-cd/)
- [Artifact Hub — Helm Chart 저장소](https://artifacthub.io/)
