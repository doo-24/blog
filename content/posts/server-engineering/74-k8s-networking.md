---
title: "[Docker & 컨테이너] 5편 — K8s 네트워킹과 서비스 노출: 트래픽이 Pod에 도달하기까지"
date: 2026-03-20T14:56:00+09:00
draft: false
tags: ["Kubernetes", "Service", "Ingress", "CNI", "NetworkPolicy", "서버"]
series: ["Docker와 컨테이너"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 11
summary: "Pod 네트워킹 모델(CNI)과 Service 유형(ClusterIP, NodePort, LoadBalancer), Ingress 컨트롤러(Nginx Ingress, ALB Ingress)와 TLS 종단, DNS 기반 서비스 디스커버리와 Headless Service, NetworkPolicy로 트래픽 제어까지"
---

## 들어가며

Pod는 언제든 사라지고 새로 생긴다. IP도 매번 바뀐다.

그렇다면 클라이언트는 어떻게 Pod를 찾아 통신할까? 외부 트래픽은 어떤 경로를 통해 클러스터 안으로 들어올까? 이 질문에 답하는 것이 K8s 네트워킹의 핵심이다.

---

## 목차

1. [Pod 네트워킹 모델과 CNI](#pod-네트워킹-모델과-cni)
2. [Service: 안정적인 엔드포인트](#service-안정적인-엔드포인트)
   - ClusterIP
   - NodePort
   - LoadBalancer
   - ExternalName
3. [Ingress: L7 트래픽 라우팅](#ingress-l7-트래픽-라우팅)
   - Nginx Ingress Controller
   - AWS ALB Ingress Controller
   - TLS 종단
4. [DNS 기반 서비스 디스커버리](#dns-기반-서비스-디스커버리)
5. [Headless Service](#headless-service)
6. [NetworkPolicy: 트래픽 제어](#networkpolicy-트래픽-제어)
7. [실전 패턴 요약](#실전-패턴-요약)
8. [참고 자료](#참고-자료)

---

## Pod 네트워킹 모델과 CNI

K8s의 네트워킹은 세 가지 기본 원칙을 따른다.

- 모든 Pod는 고유한 IP를 갖는다.
- 같은 클러스터 안의 Pod는 NAT 없이 서로 직접 통신할 수 있다.
- Node와 Pod 사이도 NAT 없이 통신한다.

NAT를 배제한 이유는 IP 변환이 개입하면 출발지 IP가 바뀌어 로깅·방화벽 정책·분산 트레이싱이 복잡해지기 때문이다. 각 Pod가 고유한 IP를 그대로 유지하면 네트워크 디버깅이 훨씬 단순해진다.

이 원칙을 구현하는 것이 **CNI(Container Network Interface)**다. CNI는 네트워크 플러그인 표준 인터페이스로, 실제 구현체는 여러 종류가 있다.

| CNI 플러그인 | 특징 |
|---|---|
| Flannel | 간단한 오버레이 네트워크, 소규모에 적합 |
| Calico | L3 기반 라우팅, NetworkPolicy 지원 강력 |
| Weave Net | 암호화 지원, 멀티클라우드 환경에 적합 |
| Cilium | eBPF 기반, 고성능 및 관측 기능 우수 |

```
┌───────────────────────────────────────────────────────┐
│                     K8s 클러스터                       │
│                                                       │
│  Node A (10.0.0.1)          Node B (10.0.0.2)        │
│  ┌──────────────────┐       ┌──────────────────┐     │
│  │ Pod A            │       │ Pod B            │     │
│  │ IP: 10.244.1.10  │◄─────►│ IP: 10.244.2.20  │     │
│  └──────────────────┘       └──────────────────┘     │
│        │   CNI (Calico/Flannel/Cilium...)              │
│        └───────────────────────────────────────       │
└───────────────────────────────────────────────────────┘
```

CNI 플러그인은 클러스터 설치 시 선택한다. 이후 변경하려면 클러스터 재구성이 필요하므로 처음에 신중하게 골라야 한다.

> **핵심 포인트**: Pod IP는 임시다. Pod가 재시작되면 IP가 바뀐다. 따라서 Pod IP로 직접 통신하는 방식은 운영 환경에서 사용하면 안 된다.

---

## Service: 안정적인 엔드포인트

**Service**는 동일한 역할을 하는 Pod들 앞에 위치하는 안정적인 가상 IP(ClusterIP)와 DNS 이름을 제공한다.

Pod가 교체되더라도 Service의 IP와 이름은 변하지 않는다. Service는 `selector`로 대상 Pod를 선택하고, kube-proxy가 실제 트래픽을 해당 Pod로 분산한다.

### ClusterIP

기본 Service 유형이다. 클러스터 내부에서만 접근 가능한 가상 IP를 부여한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 80        # Service가 노출하는 포트
      targetPort: 8080 # Pod가 실제로 수신하는 포트
```

`kubectl get svc backend-svc` 를 실행하면 `CLUSTER-IP` 열에 내부 IP가 표시된다. 이 IP는 클러스터 외부에서는 접근할 수 없다.

### NodePort

각 Node의 특정 포트를 열어 외부에서 `<NodeIP>:<NodePort>` 형태로 접근할 수 있게 한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
      nodePort: 30080  # 30000-32767 범위
```

NodePort는 간단하지만 한계가 있다. Node IP가 변경되면 접근 경로도 바뀌고, 포트 번호가 사용자에게 노출된다. 주로 개발·테스트 환경에서 쓴다.

### LoadBalancer

클라우드 환경(AWS, GCP, Azure)에서 외부 로드밸런서를 자동으로 프로비저닝한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

`kubectl get svc web-svc` 를 실행하면 `EXTERNAL-IP` 열에 클라우드 로드밸런서의 공인 IP 또는 DNS 이름이 표시된다.

Service 하나당 로드밸런서 하나가 생성되므로 비용이 발생한다. 서비스가 많아지면 Ingress를 사용하는 것이 더 효율적이다.

### ExternalName

클러스터 내부 DNS 이름을 외부 DNS 이름으로 매핑한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: prod-db.example.com
```

Pod 안에서 `external-db`라는 이름으로 접근하면 `prod-db.example.com`으로 CNAME 리다이렉션된다. 외부 데이터베이스나 서드파티 API를 클러스터 내부 이름으로 추상화할 때 유용하다.

> **핵심 포인트**: Service 유형 선택 기준 — 클러스터 내부 통신이면 ClusterIP, 외부 노출이 필요하면 LoadBalancer, HTTP/HTTPS 라우팅이 복잡하면 Ingress를 쓴다.

---

## Ingress: L7 트래픽 라우팅

LoadBalancer Service는 L4(TCP/UDP) 레벨에서 동작한다. HTTP 경로나 호스트 기반 라우팅은 불가능하다.

**Ingress**는 L7(HTTP/HTTPS) 수준의 라우팅 규칙을 정의하는 리소스다. 하나의 엔드포인트로 여러 Service에 트래픽을 분배할 수 있다.

```
인터넷
  │
  ▼
Ingress Controller (단일 LoadBalancer)
  │
  ├── /api/*      → api-service:80
  ├── /admin/*    → admin-service:80
  └── (default)   → frontend-service:80
```

Ingress 리소스 자체는 규칙 명세일 뿐이다. 실제로 트래픽을 처리하는 것은 **Ingress Controller**다.

### Nginx Ingress Controller

가장 널리 쓰이는 Ingress Controller다. Nginx를 기반으로 동작하며, 다양한 어노테이션으로 세밀한 설정이 가능하다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

Nginx Ingress Controller는 Helm으로 간단히 설치할 수 있다.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### AWS ALB Ingress Controller

AWS 환경에서는 **AWS Load Balancer Controller**를 사용해 ALB(Application Load Balancer)를 Ingress로 활용한다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: alb-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:123456789:certificate/xxxx
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

ALB Ingress는 AWS WAF, ACM 인증서, 타겟 그룹 등 AWS 네이티브 서비스와 긴밀하게 통합된다.

### TLS 종단

Nginx Ingress에서 TLS를 구성하려면 Secret에 인증서를 저장하고 Ingress에 참조한다.

```yaml
# TLS Secret 생성
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

```yaml
# Ingress에 TLS 적용
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: tls-secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

인증서 자동 갱신이 필요하다면 **cert-manager**와 Let's Encrypt를 연동하는 것이 일반적인 패턴이다.

```bash
# cert-manager 설치
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

> **핵심 포인트**: Ingress는 하나의 로드밸런서로 여러 Service에 L7 라우팅을 제공한다. Service마다 LoadBalancer를 만드는 것보다 비용 효율적이다.

---

## DNS 기반 서비스 디스커버리

K8s는 클러스터 내부 DNS를 자동으로 관리한다. Service를 생성하면 **CoreDNS**가 자동으로 DNS 레코드를 등록한다.

DNS 이름 형식은 다음과 같다.

```
<service-name>.<namespace>.svc.cluster.local
```

같은 Namespace 안에서는 Service 이름만으로도 접근할 수 있다.

```bash
# 같은 Namespace 안에서
curl http://backend-svc

# 다른 Namespace에서
curl http://backend-svc.production.svc.cluster.local

# 포트까지 명시
curl http://backend-svc.production.svc.cluster.local:80
```

Pod도 DNS 이름을 갖는다. 형식은 다음과 같다.

```
<pod-ip-with-dashes>.<namespace>.pod.cluster.local
# 예: 10-244-1-10.default.pod.cluster.local
```

Pod IP 기반 DNS는 잘 쓰이지 않는다. 일반적으로 Service 이름을 통해 접근한다.

CoreDNS 설정은 ConfigMap으로 관리된다.

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

> **핵심 포인트**: 서비스 간 통신 시 IP가 아닌 DNS 이름(`http://backend-svc`)을 사용해야 한다. Pod 재시작으로 IP가 바뀌어도 DNS 이름은 유지된다.

---

## Headless Service

일반 Service는 ClusterIP를 통해 여러 Pod 중 하나에 트래픽을 전달한다. 로드밸런싱이 자동으로 된다.

그런데 **StatefulSet**처럼 각 Pod를 개별적으로 식별해야 하는 경우가 있다. 예를 들어 데이터베이스 클러스터에서 Primary Pod와 Replica Pod를 구분해야 할 때다.

이 경우 **Headless Service**를 사용한다. `clusterIP: None`으로 설정하면 ClusterIP를 부여하지 않고 각 Pod의 IP를 DNS로 직접 노출한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None     # Headless 핵심 설정
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

Headless Service에 DNS 쿼리를 하면 ClusterIP 대신 각 Pod의 IP 목록이 반환된다.

```bash
# 일반 Service: ClusterIP 하나 반환
nslookup backend-svc

# Headless Service: 모든 Pod IP 반환
nslookup db-headless
```

StatefulSet과 Headless Service를 함께 쓰면 각 Pod에 안정적인 DNS 이름이 부여된다.

```
<pod-name>.<headless-service-name>.<namespace>.svc.cluster.local
# 예: postgres-0.db-headless.default.svc.cluster.local
#     postgres-1.db-headless.default.svc.cluster.local
```

이 이름은 Pod가 재시작되더라도 변하지 않는다. 데이터베이스 클러스터처럼 멤버 각각을 직접 지정해야 하는 상황에서 필수적이다.

> **핵심 포인트**: Headless Service는 StatefulSet 운영의 핵심 요소다. PostgreSQL, MySQL, Kafka, Zookeeper 같은 스테이트풀 워크로드에서 거의 반드시 사용한다.

---

## NetworkPolicy: 트래픽 제어

기본적으로 K8s 클러스터의 모든 Pod는 서로 자유롭게 통신할 수 있다. 이는 개발 환경에서는 편리하지만, 운영 환경에서는 보안 위협이 된다.

**NetworkPolicy**는 Pod 간 트래픽을 허용하거나 차단하는 방화벽 규칙이다. `podSelector`로 정책을 적용할 Pod를 선택하고, `ingress`·`egress` 규칙으로 허용할 트래픽을 지정한다.

> NetworkPolicy는 CNI 플러그인이 지원해야 동작한다. Flannel은 기본적으로 NetworkPolicy를 지원하지 않는다. Calico, Cilium을 사용해야 한다.

### 기본 차단 정책

먼저 모든 인바운드 트래픽을 차단하는 Default Deny 정책을 적용한다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}     # 모든 Pod에 적용
  policyTypes:
    - Ingress
```

이 정책이 적용되면 `production` Namespace의 모든 Pod는 어떤 인바운드 트래픽도 받지 않는다.

### 특정 트래픽만 허용

이제 필요한 통신만 명시적으로 허용한다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend        # 이 정책이 적용될 Pod
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend  # frontend Pod에서 오는 트래픽만 허용
      ports:
        - protocol: TCP
          port: 8080
```

`frontend` 레이블을 가진 Pod에서 `backend` Pod의 8080 포트로 오는 트래픽만 허용한다. 다른 Pod에서 오는 트래픽은 모두 차단된다.

### Namespace 간 트래픽 제어

모니터링 도구처럼 다른 Namespace의 Pod가 접근해야 하는 경우, `namespaceSelector`를 사용한다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090
```

`monitoring` Namespace의 `prometheus` Pod가 `production` Namespace의 `backend` Pod 9090 포트에 접근할 수 있다.

### Egress 제어

외부로 나가는 트래픽도 제한할 수 있다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - ports:
        - protocol: UDP
          port: 53     # DNS 쿼리는 허용
```

`backend` Pod는 `postgres` Pod로 나가는 트래픽과 DNS 쿼리(UDP 53)만 허용된다. 다른 외부 통신은 모두 차단된다.

> **핵심 포인트**: NetworkPolicy는 `ingress`와 `egress`를 독립적으로 설정한다. Egress를 제한할 때는 DNS(UDP 53) 트래픽을 반드시 허용해야 한다. 그렇지 않으면 Pod가 다른 Service 이름을 resolve하지 못한다.

---

## 실전 패턴 요약

```
외부 사용자
    │
    ▼
┌─────────────────────────────────────┐
│  Ingress Controller (LoadBalancer)  │
│  - TLS 종단                          │
│  - 호스트/경로 기반 라우팅             │
└──────────────┬──────────────────────┘
               │
       ┌───────┴──────────┐
       ▼                  ▼
 ┌──────────┐       ┌──────────┐
 │ClusterIP │       │ClusterIP │
 │(frontend)│       │  (api)   │
 └────┬─────┘       └────┬─────┘
      │                  │
   ┌──┴──┐           ┌───┴──┐
   │ Pod │           │ Pod  │  ← NetworkPolicy로 보호
   │ Pod │           │ Pod  │
   └─────┘           └──┬───┘
                        │
                 ┌──────▼───────┐
                 │ Headless Svc │
                 │  (postgres)  │
                 └──────┬───────┘
                        │
               ┌────────┴────────┐
               ▼                 ▼
          postgres-0         postgres-1
         (StatefulSet)      (StatefulSet)
```

| 상황 | 사용할 리소스 |
|---|---|
| 클러스터 내부 통신 | ClusterIP Service |
| HTTP/HTTPS 외부 노출 (다수 서비스) | Ingress + ClusterIP |
| 단일 서비스 외부 노출 | LoadBalancer Service |
| Pod 개별 식별 필요 (DB 클러스터) | Headless Service + StatefulSet |
| 외부 서비스 추상화 | ExternalName Service |
| Pod 간 통신 격리 | NetworkPolicy |

네트워킹 구성은 한 번에 완성되지 않는다. ClusterIP로 내부 통신부터 검증하고, Ingress로 외부 노출을 추가하며, NetworkPolicy로 점진적으로 격리 수준을 높이는 순서로 접근하는 것이 안전하다.

---

## 참고 자료

- [Kubernetes 공식 문서 - Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes 공식 문서 - Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes 공식 문서 - Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes 공식 문서 - DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [CNI (Container Network Interface) GitHub](https://github.com/containernetworking/cni)
- [Nginx Ingress Controller 공식 문서](https://kubernetes.github.io/ingress-nginx/)
- [AWS Load Balancer Controller 공식 문서](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [cert-manager 공식 문서](https://cert-manager.io/docs/)
- [Calico NetworkPolicy 가이드](https://docs.tigera.io/calico/latest/network-policy/)
