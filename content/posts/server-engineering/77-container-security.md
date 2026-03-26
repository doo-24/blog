---
title: "[Docker & 컨테이너] 8편 — 컨테이너 보안과 운영 패턴: 프로덕션에서 살아남기"
date: 2026-03-20T14:53:00+09:00
draft: false
tags: ["컨테이너 보안", "Trivy", "RBAC", "PDB", "Kubernetes", "서버"]
series: ["Docker와 컨테이너"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 11
summary: "이미지 취약점 스캔(Trivy, Snyk)과 이미지 서명(cosign), Pod Security Standards와 SecurityContext·OPA/Gatekeeper, RBAC 설계(ServiceAccount, Role, ClusterRole), 노드 관리(cordon/drain)와 PDB, DaemonSet 패턴 로그/메트릭 수집까지"
---

## 들어가며

컨테이너를 클러스터에 올리는 것은 시작일 뿐이다. 프로덕션에서 오래 살아남으려면 **보안**과 **운영 패턴**이 함께 갖춰져야 한다.

이번 편에서는 이미지 취약점 스캔부터 서명, 파드 보안 정책, RBAC 설계, 노드 유지보수 패턴, 그리고 로그·메트릭 수집 DaemonSet까지 한 번에 정리한다.

---

## 목차

1. [이미지 취약점 스캔 — Trivy](#1-이미지-취약점-스캔--trivy)
2. [Snyk 연동과 CI 파이프라인](#2-snyk-연동과-ci-파이프라인)
3. [이미지 서명 — cosign](#3-이미지-서명--cosign)
4. [Pod Security Standards](#4-pod-security-standards)
5. [SecurityContext 설정](#5-securitycontext-설정)
6. [OPA / Gatekeeper](#6-opa--gatekeeper)
7. [RBAC 설계](#7-rbac-설계)
8. [ServiceAccount 분리](#8-serviceaccount-분리)
9. [노드 관리 — cordon과 drain](#9-노드-관리--cordon과-drain)
10. [PDB — Pod Disruption Budget](#10-pdb--pod-disruption-budget)
11. [DaemonSet 패턴 — 로그 수집](#11-daemonset-패턴--로그-수집)
12. [DaemonSet 패턴 — 메트릭 수집](#12-daemonset-패턴--메트릭-수집)
13. [참고 자료](#참고-자료)

---

## 1. 이미지 취약점 스캔 — Trivy

컨테이너 이미지는 OS 패키지, 언어 런타임, 서드파티 라이브러리가 층층이 쌓인 구조다. 그 어느 레이어에든 알려진 CVE(Common Vulnerabilities and Exposures)가 숨어 있을 수 있다.

**Trivy**는 Aqua Security가 만든 오픈소스 취약점 스캐너로, 설치가 간단하고 CI 통합이 쉬워 사실상 표준 도구로 자리 잡았다.

```bash
# 설치 (macOS)
brew install trivy

# 설치 (Linux)
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# 이미지 스캔
trivy image myapp:v1.0.0

# 심각도 필터 — CRITICAL, HIGH만 표시
trivy image --severity CRITICAL,HIGH myapp:v1.0.0

# JSON 출력 (CI 파이프라인에서 활용)
trivy image --format json --output trivy-report.json myapp:v1.0.0
```

스캔 결과에는 패키지명, CVE ID, 심각도, 수정 버전이 함께 표시된다.

```
myapp:v1.0.0 (alpine 3.17.0)
================================
Total: 3 (CRITICAL: 1, HIGH: 2)

┌──────────────────┬────────────────┬──────────┬────────────────────┬───────────────┐
│    Library       │ Vulnerability  │ Severity │ Installed Version  │ Fixed Version │
├──────────────────┼────────────────┼──────────┼────────────────────┼───────────────┤
│ openssl          │ CVE-2023-0286  │ CRITICAL │ 3.0.7-r0           │ 3.0.8-r0      │
│ libcurl          │ CVE-2023-23914 │ HIGH     │ 7.87.0-r0          │ 7.88.1-r0     │
│ zlib             │ CVE-2022-37434 │ HIGH     │ 1.2.12-r3          │ 1.2.13-r0     │
└──────────────────┴────────────────┴──────────┴────────────────────┴───────────────┘
```

베이스 이미지를 최신 패치 버전으로 올리는 것만으로도 대부분의 CRITICAL 취약점이 해소된다.

---

## 2. Snyk 연동과 CI 파이프라인

**Snyk**은 SaaS 형태의 보안 플랫폼으로, 컨테이너 이미지뿐 아니라 애플리케이션 의존성(npm, Maven, Go modules 등)까지 통합 스캔한다.

GitHub Actions에 Trivy를 통합하는 예시다.

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1       # CRITICAL/HIGH 발견 시 파이프라인 실패

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

`exit-code: 1` 설정으로 CRITICAL 취약점이 있는 이미지는 레지스트리에 푸시되지 않도록 막는 것이 핵심이다.

Snyk을 사용할 경우 `snyk/actions/docker` 액션으로 동일하게 통합할 수 있다.

```yaml
- name: Snyk container scan
  uses: snyk/actions/docker@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    image: myapp:${{ github.sha }}
    args: --severity-threshold=high
```

---

## 3. 이미지 서명 — cosign

취약점이 없더라도 **공급망 공격(Supply Chain Attack)**은 별개의 위협이다. 이미지가 레지스트리에 업로드된 뒤 누군가 교체하더라도 클러스터는 이를 모른다.

**cosign**은 Sigstore 프로젝트의 컨테이너 이미지 서명 도구다. 서명된 이미지만 클러스터에서 실행하도록 강제할 수 있다.

```bash
# cosign 설치
brew install cosign

# 키 쌍 생성
cosign generate-key-pair
# cosign.key (비공개키), cosign.pub (공개키) 생성됨

# 이미지 서명
cosign sign --key cosign.key registry.example.com/myapp:v1.0.0

# 서명 검증
cosign verify --key cosign.pub registry.example.com/myapp:v1.0.0
```

키리스(keyless) 서명은 OIDC 토큰을 활용해 별도의 키 관리 없이 서명할 수 있다.

```bash
# GitHub Actions 환경에서 키리스 서명
cosign sign \
  --oidc-issuer https://token.actions.githubusercontent.com \
  registry.example.com/myapp:${{ github.sha }}
```

서명 검증을 클러스터 수준에서 강제하려면 **Cosign Admission Webhook** 또는 **Sigstore Policy Controller**를 배포한다.

```yaml
# ClusterImagePolicy — 서명 없는 이미지 거부
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
    - glob: "registry.example.com/**"
  authorities:
    - key:
        data: |
          -----BEGIN PUBLIC KEY-----
          MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
          -----END PUBLIC KEY-----
```

---

## 4. Pod Security Standards

Kubernetes 1.25부터 PodSecurityPolicy(PSP)가 제거되고 **Pod Security Standards(PSS)**가 기본 메커니즘이 됐다. PSP는 적용 순서와 상호작용이 복잡해 의도치 않은 권한 허용이 발생하기 쉽고, 운영자도 어떤 정책이 실제로 적용되는지 파악하기 어려웠다. PSS는 네임스페이스 레이블 한 줄로 보안 프로필을 선언하는 방식으로 이 복잡도를 대폭 줄였다.

PSS는 네임스페이스 레이블로 세 가지 프로필을 적용한다.

| 프로필 | 설명 | 사용 대상 |
|--------|------|-----------|
| `privileged` | 제한 없음 | 시스템 컴포넌트 |
| `baseline` | 알려진 위험 방지 | 일반 워크로드 |
| `restricted` | 최소 권한 강제 | 보안 민감 워크로드 |

각 프로필은 `enforce`, `audit`, `warn` 세 가지 모드로 설정할 수 있다.

```bash
# 네임스페이스에 restricted 프로필 강제 적용
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.28 \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

`warn` 모드는 파드 생성을 막지 않고 경고만 출력하므로, 기존 클러스터에 단계적으로 도입할 때 유용하다.

---

## 5. SecurityContext 설정

**SecurityContext**는 파드와 컨테이너 수준에서 보안 옵션을 직접 제어하는 필드다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  template:
    spec:
      # 파드 수준 SecurityContext
      securityContext:
        runAsNonRoot: true          # root로 실행 금지
        runAsUser: 1000             # 실행 유저 UID
        runAsGroup: 3000            # 실행 그룹 GID
        fsGroup: 2000               # 볼륨 파일시스템 그룹
        seccompProfile:
          type: RuntimeDefault      # 기본 seccomp 프로필 적용

      containers:
        - name: api
          image: myapp:v1.0.0
          # 컨테이너 수준 SecurityContext
          securityContext:
            allowPrivilegeEscalation: false   # 권한 상승 금지
            readOnlyRootFilesystem: true       # 루트 파일시스템 읽기 전용
            capabilities:
              drop:
                - ALL                          # 모든 Linux capability 제거
              add:
                - NET_BIND_SERVICE             # 1024 미만 포트 바인딩만 허용
          volumeMounts:
            - name: tmp
              mountPath: /tmp                  # 쓰기 필요한 디렉토리만 별도 마운트

      volumes:
        - name: tmp
          emptyDir: {}
```

`readOnlyRootFilesystem: true`는 컨테이너 탈취 후 악성 바이너리를 심는 공격을 차단하는 효과적인 방어다.

`capabilities.drop: [ALL]`은 컨테이너가 기본으로 갖는 Linux 권한을 모두 제거한다. 필요한 것만 `add`로 명시하는 최소 권한 원칙을 따른다.

---

## 6. OPA / Gatekeeper

**OPA(Open Policy Agent)**는 정책을 코드로 표현하는 범용 정책 엔진이다. **Gatekeeper**는 OPA를 Kubernetes Admission Webhook으로 통합한 프로젝트다.

Gatekeeper를 사용하면 PSS로 표현하기 어려운 세밀한 정책을 Rego 언어로 작성하고 클러스터 전체에 강제할 수 있다.

```bash
# Gatekeeper 설치
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml
```

**ConstraintTemplate** — 정책 템플릿을 정의한다.

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items: { type: string }
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing  := required - provided
          count(missing) > 0
          msg := sprintf("필수 레이블 누락: %v", [missing])
        }
```

**Constraint** — 템플릿을 인스턴스화해 실제 정책을 적용한다.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces: ["production", "staging"]
  parameters:
    labels: ["team", "app", "env"]
```

이 설정을 적용하면 `production`, `staging` 네임스페이스의 모든 Deployment는 `team`, `app`, `env` 레이블 없이 배포가 거부된다.

---

## 7. RBAC 설계

**RBAC(Role-Based Access Control)**는 Kubernetes 리소스에 대한 접근 권한을 역할 기반으로 제어한다.

핵심 오브젝트는 네 가지다.

- **Role**: 특정 네임스페이스 내 권한 정의
- **ClusterRole**: 클러스터 전체 또는 네임스페이스 비종속 리소스 권한 정의
- **RoleBinding**: Role을 특정 주체(Subject)에 연결
- **ClusterRoleBinding**: ClusterRole을 클러스터 전체 주체에 연결

애플리케이션 배포 담당자를 위한 Role 예시다.

```yaml
# 특정 네임스페이스에서 Deployment 관리 권한
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
---
# Role을 사용자에게 연결
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: production
subjects:
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

클러스터 전체 읽기 권한이 필요한 모니터링 도구에는 ClusterRole을 사용한다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-reader
rules:
  - apiGroups: [""]
    resources: ["nodes", "namespaces", "pods", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["nodes", "pods"]
    verbs: ["get", "list"]
```

RBAC 설계 원칙은 **최소 권한**이다. `*` 와일드카드나 `cluster-admin` 바인딩은 꼭 필요한 경우가 아니면 피해야 한다.

---

## 8. ServiceAccount 분리

파드는 기본적으로 `default` ServiceAccount를 사용하고, 이 계정의 토큰이 컨테이너 내부에 자동 마운트된다. 공격자가 컨테이너를 탈취하면 이 토큰으로 API 서버에 접근할 수 있다.

애플리케이션별로 전용 ServiceAccount를 만들고 필요한 권한만 부여하는 것이 바람직하다.

```yaml
# 전용 ServiceAccount 생성
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-server-sa
  namespace: production
automountServiceAccountToken: false   # 기본 토큰 자동 마운트 비활성화
---
# 필요한 권한만 부여
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: api-server-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config"]   # 특정 ConfigMap만 읽기 가능
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-server-rolebinding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: api-server-sa
    namespace: production
roleRef:
  kind: Role
  name: api-server-role
  apiGroup: rbac.authorization.k8s.io
---
# Deployment에 ServiceAccount 연결
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: api-server-sa
      automountServiceAccountToken: false   # Deployment 수준에서도 비활성화
      containers:
        - name: api
          image: myapp:v1.0.0
```

기본적으로 모든 Pod에 ServiceAccount 토큰이 자동 마운트되는 것은 모든 직원에게 서버실 마스터키를 나눠주는 것과 같다. 컨테이너가 탈취되면 공격자가 그 토큰으로 K8s API에 접근해 클러스터 전체를 장악할 수 있다. API 서버와 통신이 필요한 경우에만 `automountServiceAccountToken: true`로 설정하고, 해당 ServiceAccount에 최소한의 권한만 부여한다.

현재 설정된 권한을 검증할 때는 `kubectl auth can-i`를 활용한다.

```bash
# api-server-sa가 configmaps를 get할 수 있는지 확인
kubectl auth can-i get configmaps \
  --as=system:serviceaccount:production:api-server-sa \
  -n production

# 전체 권한 목록 확인
kubectl auth can-i --list \
  --as=system:serviceaccount:production:api-server-sa \
  -n production
```

---

## 9. 노드 관리 — cordon과 drain

노드 OS 패치, 커널 업그레이드, 하드웨어 교체 등의 유지보수 작업을 할 때 해당 노드의 파드를 안전하게 처리해야 한다.

**cordon** — 노드를 스케줄링 불가 상태로 표시한다. 기존 파드는 유지되지만 새 파드는 배치되지 않는다.

```bash
# 노드 cordon (스케줄링 중지)
kubectl cordon node-01

# 상태 확인
kubectl get nodes
# NAME      STATUS                     ROLES    AGE
# node-01   Ready,SchedulingDisabled   worker   30d
# node-02   Ready                      worker   30d

# cordon 해제
kubectl uncordon node-01
```

**drain** — 노드의 파드를 다른 노드로 이동시키고 cordon 상태로 전환한다.

```bash
# 노드 drain
kubectl drain node-01 \
  --ignore-daemonsets \     # DaemonSet 파드는 무시 (어차피 모든 노드에 있음)
  --delete-emptydir-data \  # emptyDir 볼륨 데이터 삭제 허용
  --grace-period=60 \       # 파드 종료 대기 시간(초)
  --timeout=300s            # 전체 drain 타임아웃

# 유지보수 작업 수행 후 복구
kubectl uncordon node-01
```

drain 중 파드가 Evict되지 않으면 `--force` 플래그로 강제 종료할 수 있다. 단, StatefulSet 파드에는 주의가 필요하다.

---

## 10. PDB — Pod Disruption Budget

drain을 실행하면 파드가 Evict(축출)된다. 모든 파드가 동시에 축출되면 서비스가 중단될 수 있다.

**PDB(Pod Disruption Budget)**는 자발적 중단(Voluntary Disruption) 시 최소 가용 파드 수를 보장하는 정책이다.

```yaml
# 최소 가용 파드 수 보장
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  minAvailable: 2       # 항상 최소 2개 파드 유지
  selector:
    matchLabels:
      app: api-server
```

또는 `maxUnavailable`로 최대 중단 수를 제한할 수 있다.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  maxUnavailable: 1     # 동시에 최대 1개 파드만 중단 허용
  selector:
    matchLabels:
      app: api-server
```

PDB가 설정된 상태에서 drain을 실행하면, K8s는 PDB 조건을 만족하는 범위에서만 파드를 축출한다.

```bash
# PDB 상태 확인
kubectl get pdb -n production
# NAME             MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# api-server-pdb   2               N/A               1                     5d
```

`ALLOWED DISRUPTIONS`가 0이면 drain이 블록된다. 이 경우 레플리카 수를 늘리거나 PDB 조건을 조정해야 한다.

PDB는 drain 외에도 클러스터 오토스케일러의 노드 축소 작업에도 적용된다. 고가용성 워크로드에는 항상 PDB를 설정하는 것이 좋다.

---

## 11. DaemonSet 패턴 — 로그 수집

**DaemonSet**은 클러스터의 모든 노드(또는 선택된 노드)에 파드를 하나씩 배포하는 워크로드다. 로그 수집, 메트릭 수집, 네트워크 플러그인처럼 노드마다 에이전트가 필요한 용도에 쓰인다.

Fluentd를 사용한 로그 수집 DaemonSet 예시다.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd-sa
      tolerations:
        # 마스터 노드에도 배포되도록 허용
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch.logging.svc.cluster.local"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
          resources:
            requests:
              cpu: 100m
              memory: 200Mi
            limits:
              cpu: 500m
              memory: 500Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluentd/etc
      terminationGracePeriodSeconds: 30
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluentd-config
```

`hostPath` 볼륨으로 노드의 로그 디렉토리를 컨테이너에 마운트해 파드 로그를 수집한다.

`tolerations`을 설정하면 컨트롤 플레인 노드에도 DaemonSet 파드를 배포할 수 있다.

특정 노드에만 배포하려면 `nodeSelector` 또는 `nodeAffinity`를 사용한다.

```yaml
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: linux
        logging: "enabled"
```

---

## 12. DaemonSet 패턴 — 메트릭 수집

**Node Exporter**는 노드의 CPU, 메모리, 디스크, 네트워크 메트릭을 Prometheus 형식으로 수집하는 에이전트다. DaemonSet으로 배포하는 것이 표준 패턴이다.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
      annotations:
        # Prometheus 자동 스크랩 설정
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"
    spec:
      hostNetwork: true      # 노드 네트워크 네임스페이스 사용
      hostPID: true          # 노드 프로세스 네임스페이스 사용
      hostIPC: true
      tolerations:
        - operator: Exists   # 모든 taint를 허용 — 모든 노드에 배포
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534     # nobody 유저
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
            - --path.rootfs=/host/root
            - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker)
          ports:
            - containerPort: 9100
              hostPort: 9100   # 노드 포트에 직접 바인딩
          resources:
            requests:
              cpu: 50m
              memory: 30Mi
            limits:
              cpu: 200m
              memory: 100Mi
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
            - name: root
              mountPath: /host/root
              mountPropagation: HostToContainer
              readOnly: true
      volumes:
        - name: proc
          hostPath: { path: /proc }
        - name: sys
          hostPath: { path: /sys }
        - name: root
          hostPath: { path: / }
```

`hostNetwork: true`와 `hostPID: true`는 노드 수준 메트릭 수집에 필요하지만 보안 민감도가 높다. SecurityContext로 최소 권한을 유지하면서 사용한다.

Prometheus는 `prometheus.io/scrape: "true"` 어노테이션을 통해 이 파드를 자동으로 스크랩 대상으로 인식한다.

DaemonSet 상태는 다음 명령으로 확인한다.

```bash
# DaemonSet 현황 확인
kubectl get daemonset -n monitoring
# NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
# node-exporter   3         3         3       3            3           <none>          7d

# 특정 노드에서 파드가 실행 중인지 확인
kubectl get pods -n monitoring -o wide | grep node-exporter
```

---

## 보안 체크리스트

프로덕션 클러스터를 구성할 때 다음 항목을 기준으로 점검하자.

| 항목 | 확인 방법 |
|------|-----------|
| 이미지 취약점 스캔 (CI 통합) | `trivy image` + `exit-code: 1` |
| 이미지 서명 검증 | Sigstore Policy Controller |
| 네임스페이스 PSS 프로필 적용 | `kubectl get ns --show-labels` |
| 컨테이너 root 실행 금지 | `runAsNonRoot: true` |
| 루트 파일시스템 읽기 전용 | `readOnlyRootFilesystem: true` |
| capabilities 최소화 | `drop: [ALL]` + 필요 시 `add` |
| ServiceAccount 분리 | `automountServiceAccountToken: false` |
| RBAC 최소 권한 | `kubectl auth can-i --list` |
| PDB 설정 (고가용성 서비스) | `kubectl get pdb` |
| 로그/메트릭 DaemonSet 배포 | `kubectl get ds -A` |

---

## 참고 자료

- [Trivy 공식 문서](https://aquasecurity.github.io/trivy/)
- [Snyk Container 문서](https://docs.snyk.io/products/snyk-container)
- [Sigstore cosign](https://docs.sigstore.dev/cosign/overview/)
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)
- [Kubernetes RBAC 공식 문서](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Disruption Budget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [DaemonSet 공식 문서](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Prometheus Node Exporter](https://github.com/prometheus/node_exporter)
