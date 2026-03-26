---
title: "[Docker & 컨테이너] 4편 — Kubernetes 핵심 개념: 컨테이너 오케스트레이션의 원리"
date: 2026-03-20T14:57:00+09:00
draft: false
tags: ["Kubernetes", "Pod", "Deployment", "Control Plane", "서버"]
series: ["Docker와 컨테이너"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 11
summary: "K8s 아키텍처(API Server, etcd, Scheduler, Controller Manager), 워커 노드 구성(kubelet, kube-proxy, CRI), Pod 생명주기와 ReplicaSet·Deployment 관계, 선언적 관리 원칙과 kubectl 실전 명령어까지"
---

## 들어가며

Docker로 컨테이너를 만들 수 있게 됐다. 그런데 실제 서비스는 컨테이너 하나로 돌아가지 않는다.

수십, 수백 개의 컨테이너가 여러 서버에 분산되어야 하고, 장애가 나면 자동으로 복구되어야 하며, 트래픽이 몰리면 빠르게 확장되어야 한다. 이 복잡한 문제를 해결하는 도구가 **Kubernetes(K8s)**다.

---

## 목차

1. [Kubernetes란 무엇인가](#kubernetes란-무엇인가)
2. [K8s 클러스터 아키텍처 전체 그림](#k8s-클러스터-아키텍처-전체-그림)
3. [컨트롤 플레인 구성 요소](#컨트롤-플레인-구성-요소)
   - API Server
   - etcd
   - Scheduler
   - Controller Manager
4. [워커 노드 구성 요소](#워커-노드-구성-요소)
   - kubelet
   - kube-proxy
   - CRI (Container Runtime Interface)
5. [Pod: K8s의 최소 배포 단위](#pod-k8s의-최소-배포-단위)
6. [ReplicaSet: 복제본 유지](#replicaset-복제본-유지)
7. [Deployment: 선언적 배포 관리](#deployment-선언적-배포-관리)
8. [선언적 관리 원칙](#선언적-관리-원칙)
9. [kubectl 실전 명령어](#kubectl-실전-명령어)
10. [참고 자료](#참고-자료)

---

## Kubernetes란 무엇인가

Kubernetes는 구글이 내부에서 사용하던 컨테이너 관리 시스템 **Borg**의 경험을 바탕으로 오픈소스화한 프로젝트다. 2014년에 공개됐고, 지금은 CNCF(Cloud Native Computing Foundation)가 관리한다.

핵심 기능은 다음과 같다.

- **자동 배치**: 사용 가능한 노드에 컨테이너를 자동으로 스케줄링한다.
- **자가 치유**: 컨테이너가 죽으면 자동으로 재시작하고, 응답 없는 노드의 컨테이너는 다른 노드로 이동시킨다.
- **수평 확장**: CPU/메모리 사용량에 따라 컨테이너 수를 자동으로 늘리거나 줄인다.
- **롤링 업데이트**: 서비스 중단 없이 새 버전으로 점진적으로 교체한다.
- **서비스 디스커버리**: 컨테이너 간 통신을 위한 DNS와 로드밸런싱을 자동 관리한다.

> **핵심 포인트**: Kubernetes는 "원하는 상태(desired state)"를 선언하면, 시스템이 알아서 그 상태를 유지한다. 운영자가 매 순간 직접 명령을 내리는 방식이 아니다.

---

## K8s 클러스터 아키텍처 전체 그림

Kubernetes 클러스터는 크게 두 종류의 노드로 구성된다.

```
┌─────────────────────────────────────────────────────┐
│                   K8s 클러스터                       │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │              Control Plane (Master)           │   │
│  │  ┌──────────┐  ┌──────┐  ┌────────────────┐ │   │
│  │  │API Server│  │ etcd │  │   Scheduler    │ │   │
│  │  └──────────┘  └──────┘  └────────────────┘ │   │
│  │  ┌──────────────────────────────────────┐   │   │
│  │  │        Controller Manager            │   │   │
│  │  └──────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────┘   │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐                 │
│  │  Worker Node │  │  Worker Node │  ...            │
│  │  ┌─────────┐ │  │  ┌─────────┐ │                 │
│  │  │ kubelet │ │  │  │ kubelet │ │                 │
│  │  ├─────────┤ │  │  ├─────────┤ │                 │
│  │  │kube-pro │ │  │  │kube-pro │ │                 │
│  │  ├─────────┤ │  │  ├─────────┤ │                 │
│  │  │   CRI   │ │  │  │   CRI   │ │                 │
│  │  └─────────┘ │  │  └─────────┘ │                 │
│  │  Pod Pod Pod │  │  Pod Pod Pod │                 │
│  └──────────────┘  └──────────────┘                 │
└─────────────────────────────────────────────────────┘
```

**Control Plane**은 클러스터 전체를 관리하는 두뇌다. **워커 노드**는 실제 컨테이너(Pod)가 실행되는 서버다.

---

## 컨트롤 플레인 구성 요소

### API Server

클러스터의 모든 통신이 지나가는 **단일 진입점**이다.

`kubectl` 명령어, 다른 컴포넌트들, 외부 시스템 모두 API Server를 통해서만 클러스터 상태를 읽고 쓴다. REST API를 제공하고, 요청마다 인증(Authentication), 인가(Authorization), 어드미션 컨트롤을 거친다.

```
사용자(kubectl) ──► API Server ──► etcd
워커 노드(kubelet) ──► API Server
외부 시스템 ──► API Server
```

### etcd

클러스터의 모든 상태 정보를 저장하는 **분산 키-값 저장소**다.

어떤 노드에 어떤 Pod가 실행 중인지, 현재 설정이 무엇인지 등 모든 정보가 etcd에 기록된다. etcd가 손상되면 클러스터 복구가 불가능하기 때문에, 운영 환경에서는 반드시 백업과 고가용성 구성이 필요하다.

> **주의**: etcd에는 API Server만 직접 접근한다. 다른 컴포넌트는 API Server를 통해 간접적으로만 상태를 읽는다. 이렇게 설계된 이유는 인증·인가·감사 로깅을 API Server 한 곳에서 일관되게 처리하기 위해서다. etcd에 직접 접근을 허용하면 누가 어떤 데이터를 변경했는지 추적할 수 없게 된다.

### Scheduler

새로 생성된 Pod를 **어느 노드에 배치할지 결정**하는 컴포넌트다.

Pod의 리소스 요구사항(CPU, 메모리), 노드의 현재 사용량, 어피니티(affinity) 규칙, 테인트(taint)와 톨러레이션(toleration) 등 다양한 조건을 고려해서 최적의 노드를 선택한다. Scheduler는 배치만 결정할 뿐, 실제 컨테이너를 실행하지는 않는다.

```
Scheduler 동작 흐름:
1. 아직 노드가 배정되지 않은 Pod 감지
2. 실행 가능한 노드 후보 필터링
3. 후보 노드에 점수 부여
4. 최고 점수 노드 선택 → API Server에 결과 기록
```

### Controller Manager

**원하는 상태와 현재 상태의 차이를 감지하고 조정**하는 컴포넌트다.

여러 개의 컨트롤러가 하나의 프로세스 안에서 실행된다. 대표적인 컨트롤러들은 다음과 같다.

| 컨트롤러 | 역할 |
|---|---|
| Node Controller | 노드 상태 모니터링, 비정상 노드 감지 |
| ReplicaSet Controller | 지정한 수의 Pod 복제본 유지 |
| Deployment Controller | 롤링 업데이트, 롤백 관리 |
| Service Account Controller | 새 네임스페이스에 기본 계정 생성 |

각 컨트롤러는 독립적인 **제어 루프(control loop)**로 동작한다. 목표 상태를 지속적으로 확인하고, 현재 상태가 다르면 API Server를 통해 수정 작업을 요청한다.

---

## 워커 노드 구성 요소

### kubelet

워커 노드에서 실행되는 **에이전트**로, 컨트롤 플레인의 지시를 받아 실제 Pod를 관리한다.

API Server를 주기적으로 폴링하여 자신이 담당해야 할 Pod 스펙을 가져오고, CRI를 통해 컨테이너를 시작하거나 중지한다. 컨테이너의 헬스체크(liveness probe, readiness probe)도 kubelet이 수행한다.

```
kubelet 주요 책임:
- Pod 스펙에 따라 컨테이너 시작/중지
- 컨테이너 상태 모니터링 및 API Server 보고
- 헬스체크 실행 (liveness / readiness probe)
- 볼륨 마운트 관리
```

### kube-proxy

각 노드에서 실행되는 **네트워크 프록시**다.

Kubernetes Service의 가상 IP(ClusterIP)로 오는 트래픽을 실제 Pod로 라우팅하는 규칙을 관리한다. 리눅스의 iptables 또는 IPVS를 이용해서 트래픽 전달 규칙을 구성한다.

> **참고**: kube-proxy는 실제 트래픽을 직접 중계하는 게 아니라, 운영체제 수준의 네트워크 규칙을 설정하는 역할이다.

### CRI (Container Runtime Interface)

kubelet이 컨테이너 런타임과 통신하기 위한 **표준 인터페이스**다.

Kubernetes는 특정 컨테이너 런타임에 종속되지 않는다. CRI 인터페이스를 구현한 런타임이라면 무엇이든 사용할 수 있다.

| 런타임 | 설명 |
|---|---|
| containerd | 현재 가장 널리 쓰이는 기본 런타임 |
| CRI-O | OCI 표준에 특화된 경량 런타임 |
| Docker Engine | dockershim 제거 후 containerd를 내부적으로 사용 |

> **알아두기**: Kubernetes 1.24부터 dockershim이 제거됐다. Docker CLI로 이미지를 빌드하더라도, 클러스터에서 실행되는 런타임은 containerd다.

---

## Pod: K8s의 최소 배포 단위

Pod는 Kubernetes에서 **배포하고 관리하는 가장 작은 단위**다.

하나의 Pod 안에는 하나 이상의 컨테이너가 들어갈 수 있다. 같은 Pod 안의 컨테이너들은 네트워크 네임스페이스와 스토리지 볼륨을 공유한다.

아래는 가장 기본적인 Pod 정의다. `resources` 필드로 CPU와 메모리를 지정하지 않으면 다른 Pod의 자원을 무제한으로 잠식할 수 있으므로, 운영 환경에서는 반드시 설정한다.

```yaml
# 단순한 Pod 정의 예시
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  labels:
    app: my-app
spec:
  containers:
    - name: my-app
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

### Pod 생명주기

Pod는 생성부터 종료까지 여러 단계를 거친다.

| Phase | 설명 |
|---|---|
| Pending | API Server에 등록됐지만 아직 노드에 배치되지 않음 |
| Running | 노드에 배치되어 최소 하나의 컨테이너가 실행 중 |
| Succeeded | 모든 컨테이너가 정상적으로 종료됨 (종료 코드 0) |
| Failed | 하나 이상의 컨테이너가 비정상 종료됨 |
| Unknown | 노드 통신 오류로 상태를 알 수 없음 |

### Pod의 한계

Pod는 직접 생성하면 장애 시 자동으로 복구되지 않는다. 노드가 죽거나 Pod가 종료되면 그냥 사라진다.

실제 운영에서는 Pod를 직접 생성하지 않고, ReplicaSet이나 Deployment 같은 상위 오브젝트를 사용한다.

### 멀티 컨테이너 Pod 패턴

Pod에 여러 컨테이너를 넣는 경우, 보통 다음 패턴을 따른다.

**Sidecar 패턴**: 메인 컨테이너 옆에 보조 컨테이너를 붙인다. 로그 수집, 프록시, 모니터링 에이전트 등에 쓰인다.

**Init Container**: 메인 컨테이너가 시작하기 전에 초기화 작업을 수행하는 컨테이너다. 설정 파일 다운로드, DB 마이그레이션 완료 대기 등에 활용한다.

```yaml
spec:
  initContainers:
    - name: init-db-wait
      image: busybox
      command: ['sh', '-c', 'until nc -z db-service 5432; do echo waiting; sleep 2; done']
  containers:
    - name: my-app
      image: my-app:latest
    - name: log-collector
      image: fluentd:latest
```

---

## ReplicaSet: 복제본 유지

ReplicaSet은 **지정한 수의 Pod 복제본이 항상 실행되도록 유지**하는 오브젝트다.

Pod가 죽으면 새로운 Pod를 생성하고, 지정한 수보다 많으면 초과분을 삭제한다. 셀렉터(selector)를 이용해 자신이 관리할 Pod를 식별한다.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-replicaset
spec:
  replicas: 3           # 항상 3개 유지
  selector:
    matchLabels:
      app: my-app       # 이 레이블을 가진 Pod를 관리
  template:             # 새 Pod를 만들 때 사용할 템플릿
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: nginx:1.25
```

### ReplicaSet의 한계

ReplicaSet은 이미지 버전을 업데이트할 때 수동으로 조작해야 한다. 롤링 업데이트나 롤백 기능이 없다.

실무에서는 ReplicaSet을 직접 사용하지 않고, 이를 감싸는 **Deployment**를 사용한다.

---

## Deployment: 선언적 배포 관리

Deployment는 ReplicaSet을 관리하면서 **롤링 업데이트와 롤백**을 자동으로 처리해 주는 오브젝트다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # 업데이트 중 최대 1개까지 불가 허용
      maxSurge: 1         # 업데이트 중 최대 1개까지 초과 허용
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: nginx:1.25
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
```

### Deployment와 ReplicaSet의 관계

Deployment를 생성하면 내부적으로 ReplicaSet이 만들어진다. 이미지를 업데이트하면 새 ReplicaSet이 생성되고, 이전 ReplicaSet의 Pod 수는 0으로 줄어든다. 이전 ReplicaSet은 삭제되지 않고 남아있어 롤백에 활용된다.

```
Deployment
  └─ ReplicaSet (nginx:1.25, replicas=3)  ← 현재 활성
  └─ ReplicaSet (nginx:1.24, replicas=0)  ← 롤백 대기
  └─ ReplicaSet (nginx:1.23, replicas=0)  ← 이전 히스토리
```

### 롤링 업데이트 과정

```
업데이트 전:  [Pod v1] [Pod v1] [Pod v1]
업데이트 중:  [Pod v1] [Pod v1] [Pod v2]  ← v2 하나 추가, v1 하나 제거
업데이트 중:  [Pod v1] [Pod v2] [Pod v2]
업데이트 후:  [Pod v2] [Pod v2] [Pod v2]
```

`maxUnavailable`과 `maxSurge` 값으로 업데이트 속도와 가용성 트레이드오프를 조정할 수 있다.

### 배포 전략 비교

| 전략 | 설명 | 다운타임 |
|---|---|---|
| RollingUpdate | 점진적으로 교체 (기본값) | 없음 |
| Recreate | 기존 Pod 전체 삭제 후 새로 생성 | 있음 |
| Blue/Green | 새 버전 전체 준비 후 트래픽 전환 (Service로 구현) | 없음 |
| Canary | 일부 트래픽만 새 버전으로 (복수 Deployment로 구현) | 없음 |

---

## 선언적 관리 원칙

Kubernetes의 핵심 철학은 **선언적(declarative) 관리**다.

"지금 당장 Pod를 3개 만들어라"(명령적, imperative)가 아니라 "항상 Pod가 3개 있는 상태여야 한다"(선언적, declarative)고 지정한다. 이후 시스템이 알아서 현재 상태를 목표 상태로 맞춰간다.

```
선언적 방식:
원하는 상태(YAML) ──► API Server ──► etcd 저장
                                          ↓
                              Controller Manager가 감시
                                          ↓
                              현재 상태 != 원하는 상태 감지
                                          ↓
                              Scheduler가 노드 배정
                                          ↓
                              kubelet이 컨테이너 실행
```

### 제어 루프 (Control Loop)

모든 컨트롤러는 다음 루프를 반복한다.

```
observe → diff → act → observe → diff → act → ...
(현재 상태 확인) → (목표와 비교) → (차이 수정) → 반복
```

이 구조 덕분에 일시적인 장애나 네트워크 불안정이 발생해도 시스템이 스스로 회복한다.

### YAML 파일의 중요성

선언적 관리는 YAML 파일을 코드처럼 관리하는 것을 전제로 한다. 이를 **GitOps**라고 부르기도 한다.

- YAML 파일을 Git으로 버전 관리한다.
- 변경사항은 코드 리뷰를 거친다.
- `kubectl apply -f`로 파일을 클러스터에 적용한다.
- 클러스터 상태는 항상 Git 저장소를 기준으로 재현할 수 있다.

---

## kubectl 실전 명령어

### 클러스터 상태 확인

```bash
# 클러스터 정보 확인
kubectl cluster-info

# 모든 노드 목록
kubectl get nodes

# 노드 상세 정보
kubectl describe node <node-name>

# 현재 컨텍스트 확인
kubectl config current-context
```

### Pod 관리

```bash
# 전체 네임스페이스의 Pod 목록
kubectl get pods --all-namespaces

# 특정 네임스페이스의 Pod 목록
kubectl get pods -n default

# Pod 상세 정보 (이벤트 포함)
kubectl describe pod <pod-name>

# Pod 로그 확인
kubectl logs <pod-name>

# 실시간 로그 스트리밍
kubectl logs -f <pod-name>

# 멀티 컨테이너 Pod에서 특정 컨테이너 로그
kubectl logs <pod-name> -c <container-name>

# Pod 내부 셸 접속
kubectl exec -it <pod-name> -- /bin/sh

# Pod 삭제
kubectl delete pod <pod-name>
```

### Deployment 관리

```bash
# Deployment 목록
kubectl get deployments

# Deployment 생성 또는 업데이트 (YAML 적용)
kubectl apply -f deployment.yaml

# 이미지 버전 업데이트
kubectl set image deployment/my-app-deployment my-app=nginx:1.26

# 롤아웃 상태 확인
kubectl rollout status deployment/my-app-deployment

# 롤아웃 히스토리 확인
kubectl rollout history deployment/my-app-deployment

# 이전 버전으로 롤백
kubectl rollout undo deployment/my-app-deployment

# 특정 버전으로 롤백
kubectl rollout undo deployment/my-app-deployment --to-revision=2

# 레플리카 수 조정
kubectl scale deployment my-app-deployment --replicas=5
```

### 디버깅

```bash
# 리소스 사용량 확인 (metrics-server 필요)
kubectl top pods
kubectl top nodes

# 이벤트 확인 (장애 원인 파악에 유용)
kubectl get events --sort-by=.metadata.creationTimestamp

# YAML 형식으로 현재 상태 출력
kubectl get deployment my-app-deployment -o yaml

# 특정 레이블을 가진 리소스 조회
kubectl get pods -l app=my-app

# 리소스 변경 실시간 감시
kubectl get pods -w
```

### 네임스페이스 관리

```bash
# 네임스페이스 목록
kubectl get namespaces

# 네임스페이스 생성
kubectl create namespace my-team

# 기본 네임스페이스 변경
kubectl config set-context --current --namespace=my-team
```

### dry-run으로 안전하게 확인

```bash
# 실제 적용 없이 결과 미리 확인
kubectl apply -f deployment.yaml --dry-run=client

# 서버 측 검증 (더 정확함)
kubectl apply -f deployment.yaml --dry-run=server

# 특정 리소스 삭제 시뮬레이션
kubectl delete deployment my-app --dry-run=client
```

---

## 정리

Kubernetes를 처음 접하면 구성 요소가 많아서 복잡하게 느껴진다. 하지만 핵심은 단순하다.

**컨트롤 플레인**은 클러스터의 두뇌다. API Server가 모든 통신을 중계하고, etcd가 상태를 저장하며, Scheduler가 배치를 결정하고, Controller Manager가 원하는 상태를 유지한다.

**워커 노드**는 실제 일꾼이다. kubelet이 Pod를 실행하고, kube-proxy가 네트워크 규칙을 관리하며, CRI가 컨테이너 런타임과 통신한다.

**Pod**는 컨테이너를 감싸는 가장 작은 단위고, **ReplicaSet**이 복제본을 유지하며, **Deployment**가 업데이트와 롤백을 관리한다.

그리고 이 모든 것을 관통하는 원칙은 **선언적 관리**다. 원하는 상태를 YAML로 선언하면, 시스템이 알아서 그 상태를 향해 움직인다.

다음 편에서는 Pod끼리, 그리고 외부와 통신하는 방법인 **Service와 Ingress**를 다룬다.

---

## 참고 자료

- [Kubernetes 공식 문서 - Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes 공식 문서 - Workloads](https://kubernetes.io/docs/concepts/workloads/)
- [Kubernetes 공식 문서 - kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [etcd 공식 문서](https://etcd.io/docs/)
- [CNCF - Cloud Native Landscape](https://landscape.cncf.io/)
- Brendan Burns et al., *Kubernetes: Up and Running*, O'Reilly Media
