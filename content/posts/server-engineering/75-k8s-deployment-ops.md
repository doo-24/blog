---
title: "[Docker & 컨테이너] 6편 — K8s 배포 전략과 운영: 프로덕션 워크로드 관리"
date: 2026-03-20T14:55:00+09:00
draft: false
tags: ["Kubernetes", "HPA", "Probe", "QoS", "Rolling Update", "서버"]
series: ["Docker와 컨테이너"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 11
summary: "Rolling Update·Recreate·카나리·블루-그린 배포의 K8s 구현, Readiness/Liveness/Startup Probe 설계와 Graceful Shutdown 연동, Resource Request/Limit과 QoS 클래스, HPA·VPA·KEDA 이벤트 기반 스케일링까지"
---

## 목차

1. [배포 전략 개요](#1-배포-전략-개요)
2. [Rolling Update](#2-rolling-update)
3. [Recreate 배포](#3-recreate-배포)
4. [카나리 배포](#4-카나리-배포)
5. [블루-그린 배포](#5-블루-그린-배포)
6. [Probe 설계](#6-probe-설계)
7. [Graceful Shutdown 연동](#7-graceful-shutdown-연동)
8. [Resource Request와 Limit](#8-resource-request와-limit)
9. [QoS 클래스](#9-qos-클래스)
10. [HPA — 수평 파드 자동 스케일링](#10-hpa--수평-파드-자동-스케일링)
11. [VPA — 수직 파드 자동 스케일링](#11-vpa--수직-파드-자동-스케일링)
12. [KEDA — 이벤트 기반 스케일링](#12-keda--이벤트-기반-스케일링)
13. [참고 자료](#참고-자료)

---

## 1. 배포 전략 개요

프로덕션 환경에서 새 버전을 배포할 때 가장 중요한 목표는 **다운타임을 최소화**하면서 **장애 전파를 차단**하는 것이다.

Kubernetes는 Deployment 오브젝트를 통해 여러 배포 전략을 지원하며, 각각 트레이드오프가 다르다.

| 전략 | 다운타임 | 롤백 속도 | 리소스 추가 비용 |
|------|----------|-----------|-----------------|
| Recreate | 있음 | 빠름 | 없음 |
| Rolling Update | 없음 | 보통 | 낮음 |
| 카나리 | 없음 | 빠름 | 낮음 |
| 블루-그린 | 없음 | 즉시 | 2배 |

전략 선택은 서비스의 **SLA 요구사항**, **롤백 민감도**, **인프라 예산**을 함께 고려해야 한다.

---

## 2. Rolling Update

Rolling Update는 K8s Deployment의 **기본 전략**이다. 구버전 파드를 점진적으로 교체하면서 가용성을 유지한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # 동시에 추가로 띄울 수 있는 파드 수
      maxUnavailable: 1  # 동시에 종료 가능한 파드 수
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: myapp:v2.1.0
          ports:
            - containerPort: 8080
```

`maxSurge`와 `maxUnavailable`을 조합해 배포 속도와 안정성을 균형 있게 조절한다.

예를 들어 `maxSurge: 0, maxUnavailable: 1`은 추가 리소스 없이 한 번에 하나씩 교체하는 보수적인 방식이다.

반대로 `maxSurge: 50%, maxUnavailable: 0`은 절반을 먼저 새 버전으로 올린 뒤 구버전을 내리는 안전한 방식이다.

### 배포 진행 상황 확인

```bash
kubectl rollout status deployment/api-server
kubectl rollout history deployment/api-server
```

### 즉시 롤백

```bash
kubectl rollout undo deployment/api-server
kubectl rollout undo deployment/api-server --to-revision=3
```

`--to-revision` 플래그로 특정 리비전으로 되돌릴 수 있다.

---

## 3. Recreate 배포

Recreate는 모든 구버전 파드를 **일괄 종료**한 뒤 새 버전을 시작하는 전략이다.

```yaml
spec:
  strategy:
    type: Recreate
```

다운타임이 발생하지만, 데이터베이스 마이그레이션처럼 **구버전과 신버전이 동시에 실행되면 안 되는 경우**에 적합하다.

스테이트풀 워크로드나 단일 인스턴스 레거시 앱에서도 주로 사용된다.

---

## 4. 카나리 배포

카나리 배포는 **일부 트래픽만 신버전으로 라우팅**해 위험을 점진적으로 검증하는 전략이다.

K8s에서는 레이블과 Service 셀렉터를 활용해 구현한다.

### 구버전 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: api-server
      track: stable
  template:
    metadata:
      labels:
        app: api-server
        track: stable
    spec:
      containers:
        - name: api
          image: myapp:v2.0.0
```

### 카나리 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server-canary
spec:
  replicas: 1   # 전체 10개 중 1개 = 10% 트래픽
  selector:
    matchLabels:
      app: api-server
      track: canary
  template:
    metadata:
      labels:
        app: api-server
        track: canary
    spec:
      containers:
        - name: api
          image: myapp:v2.1.0
```

### 공통 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-server
spec:
  selector:
    app: api-server   # track 레이블 없이 양쪽 모두 선택
  ports:
    - port: 80
      targetPort: 8080
```

Service는 `app: api-server` 레이블만 셀렉트하므로, **파드 수 비율대로 트래픽이 분산**된다.

카나리 파드가 1개, 스테이블이 9개이면 자동으로 10%/90%로 라우팅된다.

검증이 완료되면 카나리 Deployment의 `replicas`를 점진적으로 늘리고 스테이블은 줄인다.

---

## 5. 블루-그린 배포

블루-그린은 **구버전(블루)과 신버전(그린)을 동시에 운영**하다가 Service 셀렉터를 한 번에 전환하는 전략이다.

```yaml
# 블루 Deployment (현재 운영 중)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server-blue
spec:
  replicas: 4
  selector:
    matchLabels:
      app: api-server
      version: blue
  template:
    metadata:
      labels:
        app: api-server
        version: blue
    spec:
      containers:
        - name: api
          image: myapp:v2.0.0
---
# 그린 Deployment (신버전 준비)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server-green
spec:
  replicas: 4
  selector:
    matchLabels:
      app: api-server
      version: green
  template:
    metadata:
      labels:
        app: api-server
        version: green
    spec:
      containers:
        - name: api
          image: myapp:v2.1.0
```

### Service 전환

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-server
spec:
  selector:
    app: api-server
    version: green   # blue에서 green으로 변경
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# 한 줄로 즉시 전환
kubectl patch service api-server \
  -p '{"spec":{"selector":{"version":"green"}}}'
```

문제가 발생하면 `version: blue`로 되돌려 **즉시 롤백**이 가능하다.

단점은 그린 환경을 사전에 구성하는 만큼 **리소스가 2배** 필요하다는 것이다.

---

## 6. Probe 설계

K8s는 세 가지 Probe로 파드의 상태를 확인한다.

| Probe | 목적 | 실패 시 동작 |
|-------|------|-------------|
| Startup | 앱 초기화 완료 확인 | 파드 재시작 |
| Liveness | 앱이 살아있는지 확인 | 파드 재시작 |
| Readiness | 트래픽 수신 가능 여부 | Service 엔드포인트에서 제거 |

### Startup Probe

느리게 시작하는 앱(JVM 워밍업, 대용량 데이터 로딩 등)에 필수다.

Startup Probe가 성공하기 전까지 Liveness Probe는 실행되지 않는다.

```yaml
startupProbe:
  httpGet:
    path: /healthz/startup
    port: 8080
  failureThreshold: 30   # 최대 30 * 10s = 5분 대기
  periodSeconds: 10
```

### Liveness Probe

앱이 데드락 등으로 응답 불능 상태가 됐을 때 자동 복구를 위해 사용한다.

```yaml
livenessProbe:
  httpGet:
    path: /healthz/live
    port: 8080
  initialDelaySeconds: 0   # Startup Probe 이후 즉시 시작
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3
```

Liveness 엔드포인트는 **외부 의존성(DB, 캐시 등)을 체크하지 않아야 한다.** 외부 서비스 장애 시 파드가 무한 재시작되는 사태를 막기 위함이다.

### Readiness Probe

파드가 실제로 트래픽을 처리할 준비가 됐는지 확인한다.

```yaml
readinessProbe:
  httpGet:
    path: /healthz/ready
    port: 8080
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 2
  successThreshold: 1
```

Readiness는 **외부 의존성을 포함**해 체크해도 된다. 실패해도 파드가 재시작되지 않고, 단지 트래픽만 차단된다.

### 종합 예시

```yaml
containers:
  - name: api
    image: myapp:v2.1.0
    ports:
      - containerPort: 8080
    startupProbe:
      httpGet:
        path: /healthz/startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz/live
        port: 8080
      periodSeconds: 15
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /healthz/ready
        port: 8080
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 2
```

---

## 7. Graceful Shutdown 연동

K8s가 파드를 종료할 때의 흐름은 다음과 같다.

```
1. SIGTERM 신호 전송
2. Service 엔드포인트에서 파드 제거 (Readiness 실패 처리)
3. terminationGracePeriodSeconds 대기 (기본 30초)
4. 기간 내 종료 안 되면 SIGKILL 강제 종료
```

문제는 1번과 2번이 **동시에 진행**된다는 것이다. SIGTERM을 받은 직후에도 잠시간 트래픽이 들어올 수 있다.

이를 해결하기 위해 앱에서 **preStop 훅으로 짧은 대기**를 추가한다.

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: api
      image: myapp:v2.1.0
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]
```

`sleep 5`는 엔드포인트 제거 전파가 완료될 시간을 벌어준다.

앱 코드에서도 SIGTERM을 받으면 **새 연결 수락을 중단**하고 **진행 중인 요청을 완료**한 뒤 종료해야 한다.

### Go 예시

```go
srv := &http.Server{Addr: ":8080"}

go func() {
    if err := srv.ListenAndServe(); err != http.ErrServerClosed {
        log.Fatal(err)
    }
}()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
    log.Fatal("강제 종료:", err)
}
```

---

## 8. Resource Request와 Limit

K8s 스케줄러는 `requests` 값을 기준으로 노드에 파드를 배치한다.

`limits`는 파드가 실제로 사용할 수 있는 **최대치**를 제한한다.

```yaml
resources:
  requests:
    cpu: "250m"      # 0.25 코어
    memory: "256Mi"
  limits:
    cpu: "1000m"     # 1 코어
    memory: "512Mi"
```

### CPU 동작

CPU는 **압축 가능(compressible)** 리소스다. Limit을 초과하면 스로틀링(throttling)이 발생하지만 파드는 종료되지 않는다.

### Memory 동작

메모리는 **비압축(incompressible)** 리소스다. Limit을 초과하면 OOMKilled로 즉시 종료된다.

따라서 메모리 Limit은 애플리케이션의 실제 최대 사용량에 여유를 두고 설정해야 한다.

### Request와 Limit 비율

Request를 너무 낮게 설정하면 노드에 파드가 과밀 배치되어 실제 부하 시 성능이 저하된다.

일반적으로 Request는 **평균 사용량**, Limit은 **피크 사용량의 1.5~2배** 정도로 설정하는 것이 권장된다.

---

## 9. QoS 클래스

K8s는 Resource 설정에 따라 파드에 QoS(Quality of Service) 클래스를 자동 부여한다.

### Guaranteed

모든 컨테이너의 `requests == limits`일 때 부여된다.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

노드 메모리 부족 시 **가장 마지막에 종료**된다. 중요한 프로덕션 워크로드에 적합하다.

### Burstable

최소 하나의 컨테이너에 `requests` 또는 `limits`가 설정됐고 Guaranteed 조건을 충족하지 않을 때 부여된다.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "1000m"
    memory: "512Mi"
```

평소에는 유연하게 동작하지만, 리소스 압박 시 BestEffort 다음으로 종료 대상이 된다.

### BestEffort

`requests`와 `limits`를 **전혀 설정하지 않은** 파드에 부여된다.

노드 리소스 부족 시 **가장 먼저 종료**된다. 개발/테스트 환경이 아니면 사용을 피해야 한다.

### QoS 확인

```bash
kubectl get pod api-server-xxx -o jsonpath='{.status.qosClass}'
```

---

## 10. HPA — 수평 파드 자동 스케일링

HPA(Horizontal Pod Autoscaler)는 CPU, 메모리, 커스텀 메트릭에 따라 **파드 수를 자동 조정**한다.

### 기본 HPA (CPU 기반)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60   # CPU 사용률 60% 목표
```

HPA는 15초마다 메트릭을 평가해 파드 수를 조정한다.

`averageUtilization: 60`은 전체 파드의 평균 CPU 사용률이 60%가 되도록 유지한다는 의미다.

### 다중 메트릭 HPA

```yaml
spec:
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 400Mi
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
```

여러 메트릭 중 **가장 많은 파드 수를 요구하는 메트릭**이 우선 적용된다.

### 스케일 동작 제어

과도한 스케일링을 방지하기 위해 `behavior`를 설정할 수 있다.

```yaml
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # 5분간 안정화 후 스케일 다운
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60   # 1분에 최대 10%씩 감소
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60   # 1분에 최대 4개씩 증가
```

스케일 다운은 보수적으로, 스케일 업은 빠르게 설정하는 것이 일반적인 패턴이다.

### HPA 상태 확인

```bash
kubectl get hpa
kubectl describe hpa api-server-hpa
```

---

## 11. VPA — 수직 파드 자동 스케일링

VPA(Vertical Pod Autoscaler)는 파드의 **Resource Request/Limit을 자동으로 조정**한다.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-server-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  updatePolicy:
    updateMode: "Auto"   # Off | Initial | Recreate | Auto
  resourcePolicy:
    containerPolicies:
      - containerName: api
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "4000m"
          memory: "4Gi"
        controlledResources: ["cpu", "memory"]
```

`updateMode`에 따라 동작이 달라진다.

- `Off`: 권장값 계산만 하고 적용 안 함 (모니터링 용도)
- `Initial`: 파드 생성 시에만 적용
- `Recreate`: 권장값 변경 시 파드 재시작
- `Auto`: 현재 Recreate와 동일하게 동작

### VPA 권장값 확인

```bash
kubectl describe vpa api-server-vpa
```

출력 예시:

```
Recommendation:
  Container Recommendations:
    Container Name: api
    Lower Bound:
      Cpu:     100m
      Memory:  262144k
    Target:
      Cpu:     587m
      Memory:  939524k
    Upper Bound:
      Cpu:     1500m
      Memory:  2Gi
```

VPA와 HPA를 CPU/메모리 메트릭에 동시 적용하면 충돌이 발생한다. VPA는 `Off` 모드로 권장값만 참고하고, HPA로 스케일링하는 방식이 일반적이다.

---

## 12. KEDA — 이벤트 기반 스케일링

KEDA(Kubernetes Event-driven Autoscaling)는 **외부 이벤트 소스(메시지 큐, DB, 메트릭 등)를 기반**으로 파드를 스케일링한다.

HPA와 달리 **0개까지 스케일 다운**이 가능하다. 유휴 시 완전히 종료되고 메시지가 들어오면 다시 시작된다.

### KEDA 설치

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

### Kafka 기반 스케일링

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
spec:
  scaleTargetRef:
    name: kafka-consumer
  minReplicaCount: 0    # 0개까지 스케일 다운 가능
  maxReplicaCount: 30
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka-broker:9092
        consumerGroup: order-processor
        topic: orders
        lagThreshold: "100"   # 파드당 100개의 메시지 백로그
        offsetResetPolicy: latest
```

토픽의 컨슈머 랙(lag)이 100이 되면 파드 1개 추가, 0이면 파드 감소하는 방식이다.

### Redis 리스트 기반 스케일링

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-worker-scaler
spec:
  scaleTargetRef:
    name: redis-worker
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
    - type: redis
      metadata:
        address: redis-master:6379
        listName: job-queue
        listLength: "5"   # 파드당 5개의 항목
      authenticationRef:
        name: redis-trigger-auth
```

### Prometheus 메트릭 기반 스케일링

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prometheus-scaler
spec:
  scaleTargetRef:
    name: api-server
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: http_requests_total
        query: sum(rate(http_requests_total{job="api-server"}[2m]))
        threshold: "100"   # 초당 100 RPS 기준
```

KEDA는 내부적으로 HPA를 생성해 동작하므로 K8s 표준 스케일링 메커니즘과 완전히 호환된다.

### 스케일링 전략 비교

```
일반 웹 API     → HPA (CPU/RPS 기반)
배치 처리 워커   → KEDA (메시지 큐 랙 기반)
이벤트 드리븐   → KEDA (0개 스케일 다운으로 비용 절감)
메모리 집약형   → VPA (Request 최적화) + HPA
```

---

## 핵심 포인트

> **배포 전략**: 다운타임 허용 여부와 롤백 요구사항에 따라 전략을 선택한다. 블루-그린은 즉시 롤백이 필요할 때, 카나리는 점진적 검증이 필요할 때 사용한다.

> **Probe 분리**: Liveness는 외부 의존성 없이 앱 자체 상태만 체크, Readiness는 외부 의존성까지 포함해 트래픽 수신 가능 여부를 체크한다.

> **QoS 보장**: 중요 워크로드는 `requests == limits`로 설정해 Guaranteed 클래스를 확보하고 우선순위를 높인다.

> **스케일링 조합**: CPU/메모리 기반은 HPA, 유휴 상태가 자주 발생하는 이벤트 드리븐 워크로드는 KEDA의 0→N 스케일링을 활용한다.

---

## 참고 자료

- [Kubernetes 공식 문서 — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes 공식 문서 — Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes 공식 문서 — Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes 공식 문서 — Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [KEDA 공식 문서](https://keda.sh/docs/latest/)
- [VPA GitHub Repository](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Kubernetes 공식 문서 — Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
