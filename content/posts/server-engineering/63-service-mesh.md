---
title: "[분산 시스템과 MSA] 5편 — 서비스 메시와 인프라: MSA 운영의 현실"
date: 2026-03-17T17:03:00+09:00
draft: false
tags: ["서비스 메시", "Istio", "Envoy", "Sidecar", "서버"]
series: ["분산 시스템과 MSA"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 9
summary: "서비스 메시 개념과 Sidecar 프록시 패턴, Istio 아키텍처(Envoy, Pilot, Citadel)와 트래픽 관리, mTLS 자동화와 관측 가능성 통합, 도입 판단 기준과 오버헤드 비용까지"
---

마이크로서비스가 수십 개를 넘어서면, 개발자는 코드보다 서비스 간 연결을 더 많이 걱정하게 된다.

서비스 A가 서비스 B를 제대로 호출하고 있는지, 그 트래픽이 암호화되어 있는지, 장애 시 어디에서 터진 건지 — 이 모든 문제가 애플리케이션 코드 바깥에서 발생한다.

서비스 메시는 이 문제를 인프라 레이어에서 해결하는 접근법이다. 코드 한 줄 없이 mTLS, 트래픽 제어, 분산 추적을 적용할 수 있다고 약속한다.

그 약속이 얼마나 현실적인지, 그리고 얼마나 비싼지 지금부터 살펴본다.

---

## 서비스 메시란 무엇인가

### 문제의 출발점: 분산 시스템의 횡단 관심사

마이크로서비스 아키텍처에서 각 서비스는 독립적으로 배포되고 독립적으로 실행된다.

하지만 서비스들은 서로 통신해야 하고, 그 통신에는 공통적으로 필요한 기능들이 있다.

- **재시도(Retry)**: 일시적 장애에 대한 자동 재시도
- **타임아웃(Timeout)**: 응답 지연 시 연결 끊기
- **회로 차단기(Circuit Breaker)**: 연속 장애 시 호출 중단
- **로드 밸런싱**: 여러 인스턴스 간 부하 분산
- **mTLS**: 서비스 간 암호화 통신
- **관측 가능성**: 메트릭, 트레이싱, 로깅

이런 기능들을 각 서비스 팀이 각자 구현하면 중복과 불일치가 생긴다.

Go 팀은 Go용 라이브러리를 쓰고, Java 팀은 Spring Cloud를 쓰고, Python 팀은 또 다른 방식을 쓴다. 정책 변경이 필요하면 모든 팀이 동시에 배포해야 한다.

서비스 메시는 이 횡단 관심사를 애플리케이션 코드에서 꺼내어 인프라 레이어로 옮긴다.

---

## Sidecar 프록시 패턴

### 핵심 아이디어

서비스 메시의 핵심은 **Sidecar 패턴**이다.

각 서비스 인스턴스 옆에 프록시 컨테이너를 하나씩 붙인다. 모든 네트워크 트래픽은 이 프록시를 통해 들어오고 나간다.

```
┌─────────────────────────────────┐
│         Kubernetes Pod          │
│                                 │
│  ┌──────────────┐  ┌─────────┐ │
│  │  App 컨테이너  │  │ Envoy   │ │
│  │  (포트 8080) │◄─►│ Sidecar │ │
│  └──────────────┘  └────┬────┘ │
│                         │      │
└─────────────────────────┼──────┘
                          │ 네트워크
```

애플리케이션은 여전히 `localhost:8080`으로 통신한다.

하지만 실제 패킷은 Envoy를 통해 라우팅되고, Envoy가 mTLS, 재시도, 메트릭 수집을 대신 처리한다.

### iptables 트릭

Sidecar가 트래픽을 가로채는 방법은 **iptables 규칙**이다.

Kubernetes Init Container가 Pod 시작 시점에 iptables 규칙을 삽입한다. 이 규칙은 모든 인바운드/아웃바운드 트래픽을 Envoy 포트(보통 15001)로 리다이렉트한다.

```bash
# Istio가 주입하는 iptables 규칙 (단순화)
iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-port 15001
iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-port 15006
```

애플리케이션 코드는 아무것도 바꾸지 않아도 된다.

### Data Plane vs Control Plane

서비스 메시는 두 레이어로 구성된다.

**Data Plane**: 실제 트래픽을 처리하는 Sidecar 프록시들의 집합이다. Istio에서는 Envoy가 담당한다.

**Control Plane**: 모든 Sidecar에 설정을 배포하고 정책을 관리하는 중앙 컴포넌트다. Istio에서는 `istiod`가 담당한다.

```
┌─────────────────────────────────────────────┐
│              Control Plane (istiod)          │
│  ┌─────────┐  ┌──────────┐  ┌───────────┐  │
│  │  Pilot  │  │ Citadel  │  │  Galley   │  │
│  │(트래픽)  │  │ (인증서)  │  │(설정검증) │  │
│  └────┬────┘  └────┬─────┘  └───────────┘  │
└───────┼────────────┼────────────────────────┘
        │ xDS API    │ 인증서
        ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Envoy A │  │ Envoy B │  │ Envoy C │
   └─────────┘  └─────────┘  └─────────┘
```

---

## Istio 아키텍처 심층 분석

### istiod: 통합 Control Plane

초기 Istio는 Pilot, Citadel, Galley, Mixer를 별도 컴포넌트로 운영했다.

1.5 버전부터 이들을 `istiod` 단일 프로세스로 통합했다. 배포 복잡도와 성능 오버헤드가 대폭 줄었다.

**Pilot의 역할**: 서비스 디스커버리와 트래픽 관리 설정을 Envoy에 배포한다.

Kubernetes API를 모니터링하여 서비스, 엔드포인트 변경을 감지하고, xDS(Envoy Discovery Service) 프로토콜로 Envoy에 전달한다.

**Citadel의 역할**: 인증서 발급과 갱신을 담당한다.

각 Pod에 SPIFFE 표준에 따른 X.509 인증서를 발급한다. mTLS 통신에 사용되며 자동으로 갱신된다.

### xDS 프로토콜: Envoy의 언어

Envoy는 정적 설정 파일보다 **xDS API**를 통한 동적 설정을 선호한다.

xDS는 여러 Discovery Service의 집합이다.

- **LDS (Listener Discovery Service)**: 어떤 포트에서 트래픽을 수신할지
- **RDS (Route Discovery Service)**: 들어온 요청을 어디로 라우팅할지
- **CDS (Cluster Discovery Service)**: 업스트림 서비스 집합 정의
- **EDS (Endpoint Discovery Service)**: 각 클러스터의 실제 엔드포인트 목록

istiod가 이 API를 구현하고, Envoy가 구독(subscribe)하여 설정을 실시간으로 수신한다.

```yaml
# Envoy가 istiod에 연결하는 bootstrap 설정
node:
  id: sidecar~10.0.0.1~my-pod.default~default.svc.cluster.local
  cluster: my-service

dynamic_resources:
  ads_config:
    api_type: GRPC
    transport_api_version: V3
    grpc_services:
    - envoy_grpc:
        cluster_name: xds-grpc
  cds_config:
    resource_api_version: V3
    ads: {}
  lds_config:
    resource_api_version: V3
    ads: {}
```

---

## 트래픽 관리 실전

### VirtualService: 라우팅 규칙 정의

Istio의 트래픽 관리는 `VirtualService`와 `DestinationRule` 두 CRD를 중심으로 한다.

`VirtualService`는 요청이 어떻게 라우팅될지 규칙을 정의한다.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts:
  - product-service
  http:
  # 헤더 기반 카나리 라우팅
  - match:
    - headers:
        x-canary-user:
          exact: "true"
    route:
    - destination:
        host: product-service
        subset: v2
      weight: 100
  # 기본 트래픽: 90% v1, 10% v2
  - route:
    - destination:
        host: product-service
        subset: v1
      weight: 90
    - destination:
        host: product-service
        subset: v2
      weight: 10
    # 재시도 설정
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,retriable-4xx
    # 타임아웃
    timeout: 10s
```

### DestinationRule: 연결 정책 정의

`DestinationRule`은 트래픽이 목적지에 도달한 후의 동작을 정의한다.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service
spec:
  host: product-service
  # 기본 로드 밸런싱: LEAST_CONN
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    # 연결 풀 설정
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
    # 회로 차단기
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  # 버전별 서브셋 정의
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    # v2에만 mTLS 강제
    trafficPolicy:
      tls:
        mode: ISTIO_MUTUAL
```

### 회로 차단기 동작 방식

`outlierDetection` 설정이 회로 차단기 역할을 한다.

- `consecutive5xxErrors: 5`: 5번 연속 5xx 응답 시 해당 엔드포인트 추방
- `interval: 30s`: 30초마다 에러 카운트 집계
- `baseEjectionTime: 30s`: 첫 추방 시 30초간 트래픽 차단
- `maxEjectionPercent: 50`: 최대 50% 엔드포인트만 추방 (서비스 자체는 유지)

추방 시간은 연속 추방될수록 지수적으로 증가한다 (`baseEjectionTime * 추방 횟수`).

---

## mTLS 자동화

### SPIFFE와 인증서 체계

Istio는 **SPIFFE(Secure Production Identity Framework For Everyone)** 표준을 사용한다.

각 워크로드는 고유한 SPIFFE ID를 가진다.

```
spiffe://cluster.local/ns/default/sa/product-service
```

이 ID가 X.509 인증서의 SAN(Subject Alternative Name) 필드에 삽입된다.

서비스 간 mTLS 연결 시 양쪽이 서로의 인증서를 검증한다. 인증서가 같은 신뢰 루트(istiod CA)에서 발급되었는지, SPIFFE ID가 올바른지 확인한다.

### PeerAuthentication: mTLS 정책

```yaml
# 네임스페이스 전체에 mTLS 강제
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # PERMISSIVE(선택적), STRICT(강제), DISABLE
```

`STRICT` 모드에서는 mTLS 없는 연결이 완전히 차단된다.

`PERMISSIVE` 모드는 마이그레이션 중 유용하다. mTLS와 평문 트래픽을 모두 허용하므로, 점진적으로 서비스들을 메시에 온보딩할 수 있다.

### AuthorizationPolicy: 서비스 간 접근 제어

mTLS로 신원이 확인되면, 정책으로 접근을 제어할 수 있다.

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: product-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  rules:
  # order-service만 GET /products/** 허용
  - from:
    - source:
        principals:
        - cluster.local/ns/production/sa/order-service
    to:
    - operation:
        methods: ["GET"]
        paths: ["/products/*"]
  # admin 네임스페이스는 모든 요청 허용
  - from:
    - source:
        namespaces: ["admin"]
```

코드 변경 없이 서비스 간 Zero Trust 네트워크를 구현한다.

### 인증서 자동 갱신

Citadel은 기본적으로 인증서를 24시간 유효기간으로 발급하고, 만료 전 자동 갱신한다.

갱신 주기가 짧을수록 유출된 인증서의 영향이 줄어든다. 운영자 개입 없이 이 모든 과정이 자동화된다.

---

## 관측 가능성 통합

### 메트릭: Prometheus + Grafana

Envoy는 기본적으로 수백 개의 메트릭을 노출한다.

Istio는 이를 표준화하여 `istio_requests_total`, `istio_request_duration_milliseconds` 같은 메트릭을 자동 생성한다.

```yaml
# Prometheus ServiceMonitor 설정
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-component-monitor
spec:
  selector:
    matchLabels:
      istio: pilot
  endpoints:
  - port: http-monitoring
    interval: 15s
```

Grafana 대시보드에서 서비스별 골든 시그널(요청률, 에러율, 지연시간)을 별도 코드 없이 확인할 수 있다.

### 분산 추적: Jaeger/Zipkin

Envoy는 OpenTelemetry 형식의 트레이스 스팬을 자동 생성한다.

단, 트레이스 전파(propagation)를 위해 애플리케이션이 B3 헤더를 다음 서비스 호출에 전달해야 한다.

```python
# 트레이스 헤더 전파 예시 (FastAPI)
TRACE_HEADERS = [
    "x-request-id",
    "x-b3-traceid",
    "x-b3-spanid",
    "x-b3-parentspanid",
    "x-b3-sampled",
    "x-b3-flags",
]

@app.get("/orders/{order_id}")
async def get_order(order_id: str, request: Request):
    # 헤더를 다음 서비스로 전달
    headers = {h: request.headers[h] for h in TRACE_HEADERS if h in request.headers}

    async with httpx.AsyncClient() as client:
        product = await client.get(
            f"http://product-service/products/{order_id}",
            headers=headers
        )
    return {"order_id": order_id, "product": product.json()}
```

이것이 서비스 메시의 한계다. 헤더 전파는 여전히 애플리케이션의 책임이다.

### Kiali: 서비스 메시 시각화

Kiali는 Istio 전용 관측 대시보드다.

서비스 간 트래픽 흐름을 실시간 그래프로 시각화하고, 에러율과 지연시간을 엣지에 표시한다.

장애 발생 시 어느 서비스 간 통신에서 에러가 발생하는지 한눈에 파악할 수 있다.

---

## 장애 주입 (Fault Injection)

### 카오스 엔지니어링을 코드 없이

Istio의 장애 주입 기능은 특정 서비스에 인위적으로 지연이나 에러를 발생시킨다.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service-chaos
spec:
  hosts:
  - payment-service
  http:
  - match:
    - headers:
        x-chaos-test:
          exact: "true"
    fault:
      # 50% 요청에 5초 지연
      delay:
        percentage:
          value: 50
        fixedDelay: 5s
      # 10% 요청에 503 에러
      abort:
        percentage:
          value: 10
        httpStatus: 503
    route:
    - destination:
        host: payment-service
  - route:
    - destination:
        host: payment-service
```

특정 헤더를 가진 요청에만 장애를 주입하므로, 프로덕션에서도 특정 테스터만 영향을 받도록 제어할 수 있다.

---

## 도입 판단 기준

### 서비스 메시가 필요한 시점

서비스 메시는 강력하지만, 모든 시스템에 필요한 것은 아니다.

다음 조건 중 대부분이 해당될 때 도입을 고려한다.

- **서비스가 20개 이상**: 서비스 수가 적으면 복잡도 비용이 이득보다 크다
- **다중 언어/프레임워크**: 각 팀이 개별 라이브러리를 관리하는 비용이 높을 때
- **강력한 보안 요구사항**: Zero Trust 네트워크, 서비스 간 암호화가 필수일 때
- **운영 성숙도**: Kubernetes를 이미 잘 운영하고 있을 때
- **플랫폼 팀 존재**: 메시를 유지보수할 전담 팀이 있을 때

### 피해야 할 안티패턴

**안티패턴 1: 너무 이른 도입**

서비스 5~10개 규모에 Istio를 도입하면, 기능 개발보다 메시 운영에 더 많은 시간을 쓰게 된다.

초기에는 서비스 라이브러리(resilience4j, go-kit 등)로 충분하다.

**안티패턴 2: 점진적 도입 없이 전체 전환**

모든 서비스를 한 번에 메시로 전환하면, 문제 발생 시 원인 파악이 어렵다.

`PERMISSIVE` 모드로 시작하여 서비스별로 점진 전환한다.

**안티패턴 3: Envoy 설정을 직접 수정**

Istio가 관리하는 Envoy 설정을 `EnvoyFilter`로 직접 수정하면, Istio 업그레이드 시 호환성 문제가 생긴다.

가능하면 표준 Istio CRD만 사용한다.

**안티패턴 4: 관측 가능성 스택 없이 도입**

메시 없이 디버깅하는 것보다 메시 있는 상태에서 관측 도구 없이 디버깅하는 것이 더 어렵다.

Prometheus + Grafana + Jaeger를 먼저 구축하고 메시를 도입한다.

---

## 오버헤드와 복잡도 비용

### 성능 오버헤드

Sidecar 프록시는 모든 요청에 추가 홉(hop)을 삽입한다.

Istio 공식 벤치마크에서 추가 지연시간은 P50 기준 약 **0.5~2ms**다. 대부분의 워크로드에서 무시할 수 있는 수준이다.

더 큰 문제는 리소스 사용량이다.

```
# 실측치 (서비스당 Sidecar 오버헤드)
CPU:    50~100m (0.05~0.1 vCPU)
Memory: 50~100Mi

# 50개 서비스 기준
총 추가 CPU:    2500~5000m (2.5~5 vCPU)
총 추가 메모리: 2500~5000Mi (2.5~5Gi)
```

서비스 수가 늘수록 클러스터 리소스 비용이 선형으로 증가한다.

### 운영 복잡도

Istio 도입은 학습 곡선이 가파르다.

새로운 CRD만 10개 이상이고, Envoy의 내부 동작을 이해해야 트러블슈팅이 가능하다.

Istio 업그레이드도 주의가 필요하다. 마이너 버전 업그레이드에서도 하위 호환성이 깨지는 경우가 있다.

### Ambient Mesh: Sidecar 없는 미래

Istio 1.21부터 **Ambient Mesh** 모드가 안정화되었다.

Sidecar 대신 노드 레벨의 `ztunnel` 데몬셋이 트래픽을 처리한다. L4 기능(mTLS, 기본 라우팅)은 ztunnel이, L7 기능(HTTP 라우팅, 정책)은 선택적 waypoint proxy가 담당한다.

```
┌────────────────────────────────┐
│           Kubernetes Node      │
│  ┌────────┐  ┌────────┐       │
│  │ Pod A  │  │ Pod B  │       │
│  └───┬────┘  └───┬────┘       │
│      │           │            │
│  ┌───▼───────────▼──────────┐ │
│  │   ztunnel (L4 mTLS)      │ │
│  └───────────────┬──────────┘ │
└──────────────────┼────────────┘
                   │
           ┌───────▼──────┐
           │ Waypoint Proxy│ (L7, 선택적)
           │ (네임스페이스) │
           └──────────────┘
```

Sidecar당 오버헤드가 사라지므로 리소스 사용량이 크게 줄어든다.

### Linkerd vs Istio

Istio만이 선택지는 아니다.

**Linkerd**는 Rust로 작성된 경량 프록시를 사용한다. 오버헤드가 Istio보다 낮지만, 기능 범위도 좁다.

L7 기능이 풍부하게 필요하다면 Istio, 단순한 mTLS와 관측성이 목표라면 Linkerd가 더 적합할 수 있다.

---

## 실전 배포 전략

### 점진적 온보딩

```bash
# 1. Istio 설치 (PERMISSIVE 모드)
istioctl install --set profile=default \
  --set meshConfig.defaultConfig.holdApplicationUntilProxyStarts=true

# 2. 네임스페이스에 자동 주입 활성화
kubectl label namespace production istio-injection=enabled

# 3. 기존 Pod 재시작 (Sidecar 주입)
kubectl rollout restart deployment -n production

# 4. mTLS 상태 확인
istioctl x describe pod <pod-name> -n production

# 5. STRICT 모드로 전환 (모든 서비스 준비 후)
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
EOF
```

### 트러블슈팅 명령어

```bash
# Envoy 설정 덤프
istioctl proxy-config all <pod-name> -n production

# 특정 클러스터 라우팅 확인
istioctl proxy-config clusters <pod-name> --fqdn product-service.production.svc.cluster.local

# mTLS 연결 상태 확인
istioctl authn tls-check <pod-name> product-service.production.svc.cluster.local

# Envoy 로그 레벨 동적 변경
istioctl proxy-config log <pod-name> --level debug

# 실시간 액세스 로그 확인
kubectl logs <pod-name> -c istio-proxy -f
```

---

## 정리: 서비스 메시는 무료 점심이 아니다

서비스 메시는 분산 시스템의 진짜 문제를 해결한다.

mTLS, 트래픽 제어, 관측 가능성을 코드 없이 제공하는 것은 혁신적이다.

하지만 그 대가로 운영 복잡도, 리소스 비용, 가파른 학습 곡선을 요구한다.

20개 미만 서비스 규모, 플랫폼 팀 부재, Kubernetes 운영 경험이 얕은 환경에서의 도입은 득보다 실이 크다.

서비스 메시를 도입하기 전에 먼저 물어야 할 질문은 "무엇을 할 수 있느냐"가 아니라 "우리 팀이 이것을 운영할 수 있느냐"다.

그 답이 "예스"일 때, 서비스 메시는 MSA 운영을 진정한 플랫폼 엔지니어링 수준으로 끌어올린다.

---

## 참고 자료

1. **Istio 공식 문서** — [istio.io/latest/docs](https://istio.io/latest/docs/) — VirtualService, DestinationRule, PeerAuthentication CRD 레퍼런스
2. **Envoy Proxy 문서** — [envoyproxy.io/docs](https://www.envoyproxy.io/docs/envoy/latest/) — xDS API, 필터 체인, 클러스터 설정
3. **William Morgan, "Service Mesh: What Every Engineer Needs to Know"** (Buoyant, 2022) — 서비스 메시 개념 정립
4. **Istio Ambient Mesh 설계 문서** — [github.com/istio/istio/tree/master/architecture/ambient](https://github.com/istio/istio) — ztunnel과 waypoint proxy 아키텍처
5. **CNCF, "ServiceMesh Performance Benchmark"** (2023) — Istio vs Linkerd 성능 비교 데이터
6. **Liz Rice, "Security Observability with eBPF"** (O'Reilly, 2022) — eBPF 기반 서비스 메시 미래 방향
