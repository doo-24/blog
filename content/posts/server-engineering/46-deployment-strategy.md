---
title: "[배포와 CI/CD] 6편 — 배포 전략: 무중단 배포의 기술"
date: 2026-03-17T20:02:00+09:00
draft: false
tags: ["블루-그린", "카나리", "롤링 배포", "Feature Flag", "서버"]
series: ["배포와 CI/CD"]
summary: "블루-그린 배포 원리와 롤백, 카나리 배포와 가중치 라우팅·자동 롤백, 롤링 배포의 maxUnavailable/maxSurge 설정, Feature Flag 기반 점진적 릴리즈까지"
---

서비스가 커질수록 배포는 단순한 코드 업로드가 아니라 리스크 관리 행위가 된다. 새 버전이 운영 환경에 올라가는 그 순간, 수만 명의 사용자 요청이 살아있고 DB 스키마는 이미 마이그레이션 중이며 외부 API 연동 코드가 동시에 동작하고 있다. 다운타임 5분이 수천만 원 매출 손실로 이어지는 이커머스, 실시간성이 생명인 게임 서버, SLA 99.9%를 약속한 B2B SaaS — 이런 환경에서 무중단 배포는 선택이 아니라 생존 조건이다. 블루-그린, 카나리, 롤링, Feature Flag. 각 전략은 다운타임 최소화와 리스크 통제 사이의 트레이드오프를 다르게 해결한다. 어떤 전략이 언제 맞는지, 각 전략의 내부 동작과 실제 설정 예시를 코드 레벨로 파헤쳐 본다.

---

## 1. 블루-그린 배포 (Blue-Green Deployment)

### 원리: 두 개의 동일한 환경

블루-그린 배포의 핵심 아이디어는 단순하다. 운영 환경을 두 벌 유지하고, 한 번에 하나만 트래픽을 받게 한다. 현재 운영 중인 버전이 **블루**, 새 버전이 배포될 환경이 **그린**이다.

```
[사용자 트래픽]
       |
       v
  [로드 밸런서]
   /         \
[블루: v1]  [그린: v2] ← 배포 준비 중
  (활성)     (대기)
```

배포 순서는 다음과 같다.

1. 그린 환경에 새 버전 배포 및 워밍업
2. 헬스체크, 스모크 테스트 통과 확인
3. 로드 밸런서 타겟 그룹을 블루 → 그린으로 전환 (원자적 전환)
4. 트래픽이 모두 그린으로 이동하면 블루는 대기 상태 유지

### AWS에서의 구현: ALB 가중치 기반 전환

AWS Application Load Balancer는 타겟 그룹 가중치를 지원한다. 완전한 블루-그린 전환은 가중치를 0/100으로 설정하는 방식으로 구현한다.

```bash
# 그린 타겟 그룹 생성
aws elbv2 create-target-group \
  --name myapp-green-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-xxxxxxxx \
  --health-check-path /health \
  --health-check-interval-seconds 10 \
  --healthy-threshold-count 2

# 현재 리스너 규칙 확인
aws elbv2 describe-rules \
  --listener-arn arn:aws:elasticloadbalancing:...

# 블루 → 그린 전환: 가중치 스왑
aws elbv2 modify-rule \
  --rule-arn arn:aws:elasticloadbalancing:...:rule/xxx \
  --actions '[
    {
      "Type": "forward",
      "ForwardConfig": {
        "TargetGroups": [
          {"TargetGroupArn": "arn:...:blue-tg", "Weight": 0},
          {"TargetGroupArn": "arn:...:green-tg", "Weight": 100}
        ]
      }
    }
  ]'
```

### Kubernetes에서의 구현: Service 셀렉터 전환

Kubernetes 환경에서는 Service의 셀렉터를 변경하는 방식으로 블루-그린을 구현한다.

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  labels:
    app: myapp
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:v1.2.3
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
```

```yaml
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
  labels:
    app: myapp
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:v1.3.0   # 새 버전
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
```

```yaml
# service.yaml — 셀렉터를 바꾸는 것만으로 전환 완료
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp
    version: blue   # ← 이 값을 green으로 변경하면 전환
  ports:
  - port: 80
    targetPort: 8080
```

전환 명령어 한 줄로 무중단 배포가 완료된다.

```bash
kubectl patch service myapp-svc \
  -p '{"spec":{"selector":{"version":"green"}}}'
```

### 롤백: 블루-그린의 가장 강력한 장점

문제가 발생하면 셀렉터를 다시 `blue`로 패치하면 된다. 롤백 시간은 수십 초 이내다. 블루 환경이 그대로 살아있기 때문에 새 컨테이너를 시작하고 워밍업하는 시간이 전혀 필요 없다.

```bash
# 롤백: 셀렉터를 blue로 되돌림
kubectl patch service myapp-svc \
  -p '{"spec":{"selector":{"version":"blue"}}}'

# 또는 Argo Rollouts 사용 시
kubectl argo rollouts abort myapp-rollout
kubectl argo rollouts undo myapp-rollout
```

### 블루-그린의 비용과 한계

블루-그린의 치명적인 단점은 **인프라 비용이 두 배**라는 점이다. 운영 환경 규모가 서버 100대라면 배포 기간 동안 200대를 유지해야 한다. 이를 완화하는 방법은 두 가지다.

**옵션 1: 배포 시점에만 그린 환경 프로비저닝**
평소에는 블루만 운영하다가 배포 직전에 그린 환경을 Terraform 또는 CloudFormation으로 동적으로 생성한다. 배포 검증 후 블루를 파기한다.

**옵션 2: 클라우드 Auto Scaling 그룹 활용**
AWS의 경우 Launch Template을 새 버전으로 업데이트한 뒤 새 ASG를 생성하고, 검증 완료 후 구 ASG를 삭제한다.

또한 블루-그린은 **DB 스키마 마이그레이션**이 어렵다. 블루와 그린이 동시에 같은 DB를 보는 상황에서 스키마 변경이 있으면 양쪽 버전과 호환되는 방식으로 마이그레이션을 단계적으로 진행해야 한다 (Expand-Contract 패턴).

---

## 2. 카나리 배포 (Canary Deployment)

### 원리: 일부 트래픽으로 먼저 검증

카나리 배포는 새 버전을 전체 사용자가 아닌 소수(예: 1~5%)에게만 먼저 노출해서 문제 여부를 검증한 뒤 점진적으로 확대하는 방식이다. 광산에서 유독 가스를 먼저 감지하기 위해 카나리아 새를 들여보낸 역사에서 유래했다.

```
[사용자 트래픽 100%]
         |
         v
   [로드 밸런서]
    /          \
[stable: v1]  [canary: v2]
  95% 트래픽    5% 트래픽
```

### Nginx 가중치 기반 카나리

Nginx upstream의 `weight` 지시자로 간단한 카나리를 구현할 수 있다.

```nginx
upstream myapp_backend {
    server stable-backend:8080  weight=95;
    server canary-backend:8080  weight=5;

    keepalive 32;
}

server {
    listen 80;

    location / {
        proxy_pass http://myapp_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # 카나리 요청 헤더 추가 (로그 분석용)
        proxy_set_header X-Canary $upstream_addr;
    }
}
```

### Istio VirtualService로 정교한 카나리

서비스 메시 Istio를 사용하면 HTTP 헤더, 쿠키, 사용자 세그먼트 기반의 정교한 트래픽 분할이 가능하다.

```yaml
# virtual-service.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
  - myapp-svc
  http:
  # 내부 테스터 그룹은 카나리로 100% 라우팅
  - match:
    - headers:
        x-user-group:
          exact: internal-tester
    route:
    - destination:
        host: myapp-svc
        subset: canary
      weight: 100

  # 일반 트래픽: 5%만 카나리
  - route:
    - destination:
        host: myapp-svc
        subset: stable
      weight: 95
    - destination:
        host: myapp-svc
        subset: canary
      weight: 5
```

```yaml
# destination-rule.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: myapp-dr
spec:
  host: myapp-svc
  subsets:
  - name: stable
    labels:
      version: stable
  - name: canary
    labels:
      version: canary
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutiveErrors: 5
      interval: 10s
      baseEjectionTime: 30s
```

### Argo Rollouts: 자동화된 카나리 파이프라인

Argo Rollouts는 Kubernetes에서 카나리 배포를 자동화하는 도구다. 메트릭 분석을 기반으로 배포 진행/중단 여부를 자동으로 결정한다.

```yaml
# rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-rollout
spec:
  replicas: 10
  strategy:
    canary:
      canaryService: myapp-canary-svc
      stableService: myapp-stable-svc
      trafficRouting:
        istio:
          virtualService:
            name: myapp-vs
            routes:
            - primary
      steps:
      - setWeight: 5       # 5% 트래픽으로 시작
      - pause: {duration: 5m}
      - analysis:          # 메트릭 분석 실행
          templates:
          - templateName: success-rate-analysis
      - setWeight: 20      # 20%로 확대
      - pause: {duration: 10m}
      - analysis:
          templates:
          - templateName: success-rate-analysis
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100     # 전체 전환
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
        image: myapp:v1.3.0
        ports:
        - containerPort: 8080
```

```yaml
# analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate-analysis
spec:
  metrics:
  - name: success-rate
    interval: 1m
    count: 5
    successCondition: result[0] >= 0.95   # 성공률 95% 이상
    failureLimit: 1
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{
            app="myapp",
            version="canary",
            status!~"5.."
          }[5m]))
          /
          sum(rate(http_requests_total{
            app="myapp",
            version="canary"
          }[5m]))

  - name: latency-p99
    interval: 1m
    count: 5
    successCondition: result[0] <= 0.5    # P99 레이턴시 500ms 이하
    failureLimit: 1
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket{
              app="myapp",
              version="canary"
            }[5m])) by (le)
          )
```

분석 결과 임계값을 넘으면 Argo Rollouts가 자동으로 롤백을 실행한다.

```bash
# 현재 롤아웃 상태 확인
kubectl argo rollouts get rollout myapp-rollout --watch

# 수동으로 다음 단계로 진행
kubectl argo rollouts promote myapp-rollout

# 수동 롤백
kubectl argo rollouts undo myapp-rollout
```

### 카나리의 안티패턴

**안티패턴 1: 메트릭 없이 시간만 기다리기**
단순히 5분 대기 후 자동 진행하는 설정은 카나리가 아니다. 성공률, 레이턴시, 에러율을 측정하지 않으면 카나리의 의미가 없다.

**안티패턴 2: 세션 스티키니스 미처리**
같은 사용자가 요청할 때마다 stable/canary를 오가면 불일치 경험이 발생한다. 쿠키 기반 세션 고정이나 Consistent Hash 라우팅을 사용해야 한다.

```nginx
# 쿠키 기반 카나리 고정 (Nginx)
upstream myapp_backend {
    hash $cookie_session_id consistent;
    server stable-backend:8080;
    server canary-backend:8080;
}
```

**안티패턴 3: 카나리와 stable이 같은 DB를 다른 스키마로 접근**
카나리 버전이 새 컬럼을 쓰고 stable이 구 스키마를 읽으면 데이터 손상이 생긴다. 항상 하위 호환 스키마 변경을 먼저 배포한 뒤 카나리를 진행해야 한다.

---

## 3. 롤링 배포 (Rolling Deployment)

### 원리: 인스턴스를 순차적으로 교체

롤링 배포는 인스턴스를 하나씩(또는 소수씩) 새 버전으로 교체하는 방식이다. 블루-그린처럼 인프라를 두 배로 늘릴 필요가 없고, 카나리처럼 복잡한 트래픽 분할 설정도 필요 없다. Kubernetes의 기본 배포 전략이기도 하다.

```
배포 진행 중:
Pod1: v1 (운영중)
Pod2: v2 (새 버전, 시작중)  ← 교체 중
Pod3: v1 (운영중)
Pod4: v1 (운영중)

배포 완료:
Pod1: v2
Pod2: v2
Pod3: v2
Pod4: v2
```

### maxUnavailable과 maxSurge 설정

Kubernetes Deployment의 롤링 업데이트는 두 가지 핵심 파라미터로 제어된다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2     # 동시에 사용 불가능한 Pod 최대 수 (절댓값 또는 %)
      maxSurge: 2           # 목표 레플리카 수를 초과해서 생성할 수 있는 최대 수
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
        image: myapp:v1.3.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
          failureThreshold: 3
          successThreshold: 1
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 20
          failureThreshold: 3
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
      terminationGracePeriodSeconds: 60   # Graceful shutdown 대기 시간
```

**maxUnavailable = 2, maxSurge = 2**인 경우 (replicas=10):
- 최대 8개 Pod는 항상 운영 가능 상태 유지
- 최대 12개 Pod까지 동시에 존재 가능
- 한 번에 2개씩 롤링하므로 교체 속도가 빠름

**보수적 설정 (고가용성 우선)**:
```yaml
rollingUpdate:
  maxUnavailable: 0   # 무중단 보장
  maxSurge: 1         # 느리지만 리소스 낭비 최소화
```

**공격적 설정 (속도 우선)**:
```yaml
rollingUpdate:
  maxUnavailable: 25%
  maxSurge: 25%
```

### Graceful Shutdown: 롤링 배포의 필수 조건

롤링 배포에서 가장 자주 실수하는 부분이 Graceful Shutdown이다. Kubernetes가 Pod를 종료할 때의 시퀀스를 이해해야 한다.

```
1. Pod에 SIGTERM 전송
2. 서비스 엔드포인트에서 Pod IP 제거 (비동기 처리)
3. terminationGracePeriodSeconds 대기
4. 여전히 실행 중이면 SIGKILL 전송
```

문제는 엔드포인트 제거와 SIGTERM이 거의 동시에 발생하지만, 실제 로드 밸런서가 해당 Pod로의 라우팅을 멈추는 데는 수 초가 걸린다는 점이다. 이 간격에 들어오는 요청이 503을 받게 된다. 해결책은 SIGTERM 수신 시 `preStop` 훅으로 짧게 대기하는 것이다.

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]   # 로드 밸런서가 제거하기를 기다림
```

애플리케이션 코드에서도 SIGTERM을 처리해야 한다.

```go
// Go 서버 Graceful Shutdown 예시
func main() {
    srv := &http.Server{
        Addr:    ":8080",
        Handler: router,
    }

    // 별도 고루틴에서 서버 실행
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %s\n", err)
        }
    }()

    // SIGTERM / SIGINT 수신 대기
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("Shutdown signal received, waiting for in-flight requests...")

    // 30초 이내에 처리 중인 요청 완료 대기
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }

    log.Println("Server exiting")
}
```

### 롤링 배포의 한계: 버전 공존 문제

롤링 배포 도중에는 v1과 v2가 동시에 서비스된다. 이 시점에 API 응답 형식이 달라지거나 DB 스키마가 바뀌면 문제가 생긴다.

**문제 시나리오**: v1은 `user_name` 필드를, v2는 `username` 필드를 쓰는 경우.
프론트엔드가 두 응답 형식을 동시에 받게 되어 UI가 깨진다.

**해결책**: 배포 기간 동안 두 버전 모두와 호환되는 API를 제공하거나, 새 엔드포인트를 추가하는 방식으로 하위 호환성을 유지해야 한다.

---

## 4. Feature Flag 기반 배포

### 원리: 배포와 릴리즈를 분리

Feature Flag(기능 플래그, 또는 Feature Toggle)는 코드 배포와 기능 공개를 분리하는 기법이다. 새 기능의 코드는 이미 운영 서버에 배포되어 있지만, 플래그가 꺼져 있는 동안은 사용자에게 노출되지 않는다. 플래그를 켜는 것만으로 릴리즈가 완료된다.

```
기존 방식:
코드 작성 → 배포 → 사용자에게 노출 (동시 발생)

Feature Flag 방식:
코드 작성 → 배포 (플래그 OFF) → 검증 → 플래그 ON → 사용자에게 노출 (분리)
```

### 기본 구현: 환경변수 기반 플래그

가장 단순한 형태는 환경변수로 기능을 켜고 끄는 것이다.

```go
// feature_flags.go
package flags

import "os"

type FeatureFlags struct {
    NewCheckoutFlow bool
    RecommendEngine bool
    DarkMode        bool
}

func Load() FeatureFlags {
    return FeatureFlags{
        NewCheckoutFlow: os.Getenv("FF_NEW_CHECKOUT_FLOW") == "true",
        RecommendEngine: os.Getenv("FF_RECOMMEND_ENGINE") == "true",
        DarkMode:        os.Getenv("FF_DARK_MODE") == "true",
    }
}
```

```go
// handler.go
func CheckoutHandler(flags FeatureFlags) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if flags.NewCheckoutFlow {
            newCheckoutFlow(w, r)
        } else {
            legacyCheckoutFlow(w, r)
        }
    }
}
```

환경변수 방식은 단순하지만, 플래그를 변경할 때마다 재배포가 필요하다는 단점이 있다.

### 동적 플래그: LaunchDarkly / Unleash

운영 중 실시간으로 플래그를 변경하려면 외부 플래그 관리 서비스가 필요하다. 오픈소스 자체 호스팅으로는 **Unleash**가 많이 쓰인다.

```yaml
# docker-compose.yml (Unleash 자체 호스팅)
version: "3.9"
services:
  unleash-db:
    image: postgres:15
    environment:
      POSTGRES_DB: unleash
      POSTGRES_USER: unleash
      POSTGRES_PASSWORD: secret

  unleash:
    image: unleashorg/unleash-server:latest
    ports:
    - "4242:4242"
    environment:
      DATABASE_URL: postgres://unleash:secret@unleash-db/unleash
      INIT_FRONTEND_API_TOKENS: "default:development.unleash-insecure-frontend-api-token"
    depends_on:
    - unleash-db
```

```go
// Unleash Go SDK 사용
import unleash "github.com/Unleash/unleash-client-go/v3"

func InitFeatureFlags() {
    err := unleash.Initialize(
        unleash.WithUrl("http://unleash:4242/api"),
        unleash.WithAppName("myapp"),
        unleash.WithInstanceId("myapp-pod-1"),
        unleash.WithCustomHeaders(http.Header{
            "Authorization": []string{"*:default.unleash-insecure-api-token"},
        }),
    )
    if err != nil {
        log.Fatal("failed to initialize unleash:", err)
    }
}

func CheckoutHandler(w http.ResponseWriter, r *http.Request) {
    userID := getUserID(r)

    // 사용자 컨텍스트 기반 플래그 평가
    ctx := context.Background()
    uc := unleash.WithContext(unleash.Context{
        UserId: userID,
        Properties: map[string]string{
            "plan": getUserPlan(userID),
            "country": getCountry(r),
        },
    })

    if unleash.IsEnabled("new-checkout-flow", uc) {
        newCheckoutFlow(w, r)
    } else {
        legacyCheckoutFlow(w, r)
    }
}
```

### 점진적 릴리즈: 사용자 세그먼트 기반 롤아웃

Unleash의 **Gradual Rollout** 전략을 사용하면 사용자 ID 해시를 기반으로 점진적으로 기능을 확대할 수 있다.

```json
// Unleash 전략 설정 (API 또는 UI로 설정)
{
  "name": "new-checkout-flow",
  "enabled": true,
  "strategies": [
    {
      "name": "gradualRolloutUserId",
      "parameters": {
        "percentage": "10",
        "groupId": "new-checkout-flow"
      }
    }
  ]
}
```

10%로 시작해서 에러율, 전환율 지표를 보면서 25% → 50% → 100%로 단계적으로 확대한다. 문제가 생기면 UI에서 즉시 플래그를 끄면 재배포 없이 롤백이 완료된다.

### Feature Flag 기술 부채 관리

Feature Flag는 편리하지만 방치하면 코드베이스에 조건 분기가 넘쳐나 가독성을 해친다. 반드시 **만료 정책**을 가져야 한다.

```go
// 잘못된 예: 오래된 플래그가 코드에 잔재
func ProcessOrder(order Order) error {
    if flags.IsEnabled("new-payment-flow") {       // 이미 6개월 전에 100% 롤아웃됨
        if flags.IsEnabled("payment-v2-retry") {   // 이미 전체 배포됨
            return newPaymentWithRetry(order)
        }
        return newPayment(order)
    }
    return legacyPayment(order)  // 실제로 실행되지 않는 코드
}

// 올바른 예: 플래그 제거 후 정리된 코드
func ProcessOrder(order Order) error {
    return newPaymentWithRetry(order)
}
```

플래그 생성 시 JIRA 티켓에 "만료 날짜" 필드를 필수로 추가하고, 정기적으로 오래된 플래그를 제거하는 PR을 만드는 것이 좋은 관행이다.

### Feature Flag와 DB 마이그레이션 조합

Feature Flag는 DB 스키마 변경과 결합할 때 특히 강력하다.

**단계 1**: 새 컬럼 추가 (NULL 허용), 플래그 OFF 상태로 배포

```sql
ALTER TABLE orders ADD COLUMN new_payment_method VARCHAR(50);
```

**단계 2**: 플래그 ON, 새 코드가 새 컬럼에 쓰기 시작

**단계 3**: 구 컬럼 데이터를 새 컬럼으로 백필, 검증 완료

**단계 4**: 구 컬럼 제거 마이그레이션 + 플래그 코드 정리

---

## 5. 전략 선택 가이드

각 전략은 서로 다른 트레이드오프를 가진다. 상황에 따라 적절히 선택하거나 조합해야 한다.

| 전략 | 롤백 속도 | 인프라 비용 | 복잡도 | 버전 공존 | 적합한 상황 |
|------|-----------|-------------|--------|-----------|-------------|
| 블루-그린 | 수십 초 | 2배 | 낮음 | 없음 | 중요 배포, 빠른 롤백 필요 |
| 카나리 | 수분 | +소수 % | 높음 | 있음 | 대규모 트래픽, 점진적 검증 |
| 롤링 | 수분~수십분 | 최소 | 낮음 | 있음 | 일반적인 무중단 배포 |
| Feature Flag | 수초 | 없음 | 중간 | 제어 가능 | 기능 단위 제어, A/B 테스트 |

**대부분의 실제 환경에서는 조합을 사용한다.** 예를 들어 롤링 배포로 새 버전을 전체 배포하되, 새 기능은 Feature Flag로 점진적으로 노출하는 방식이다. 또는 카나리 배포로 새 버전을 검증하면서 특정 기능은 Feature Flag로 추가 제어층을 두기도 한다.

---

## 참고 자료

- [Kubernetes Deployments — Rolling Update Strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment): maxUnavailable, maxSurge 공식 문서
- [Argo Rollouts Documentation](https://argoproj.github.io/argo-rollouts/): 카나리, 블루-그린 자동화 도구
- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/): VirtualService 기반 가중치 라우팅
- [Unleash Feature Toggle Service](https://docs.getunleash.io/): 오픈소스 Feature Flag 서버
- [Martin Fowler — Feature Toggles](https://martinfowler.com/articles/feature-toggles.html): Feature Flag 패턴의 체계적 분류와 관리 전략
- [The Twelve-Factor App — Config](https://12factor.net/config): 환경별 설정 관리 원칙
