---
title: "[AWS / 클라우드] 2편 — 네트워크와 컴퓨팅: VPC 위에 서버 올리기"
date: 2026-03-20T14:07:00+09:00
draft: false
tags: ["VPC", "EC2", "ALB", "Auto Scaling", "AWS", "서버"]
series: ["AWS / 클라우드"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "VPC 설계(서브넷, 라우팅 테이블, IGW, NAT Gateway), Security Group vs NACL 보안 계층, EC2 인스턴스 유형과 Spot/Reserved/On-Demand 전략, ALB/NLB 설정과 오토스케일링 그룹까지"
---

## 목차

1. [VPC란 무엇인가](#1-vpc란-무엇인가)
2. [서브넷 설계: Public vs Private](#2-서브넷-설계-public-vs-private)
3. [인터넷 연결: IGW와 NAT Gateway](#3-인터넷-연결-igw와-nat-gateway)
4. [라우팅 테이블: 패킷의 길을 정하다](#4-라우팅-테이블-패킷의-길을-정하다)
5. [보안 계층: Security Group vs NACL](#5-보안-계층-security-group-vs-nacl)
6. [EC2 인스턴스 유형 선택 전략](#6-ec2-인스턴스-유형-선택-전략)
7. [구매 옵션: On-Demand, Reserved, Spot](#7-구매-옵션-on-demand-reserved-spot)
8. [로드 밸런서: ALB와 NLB](#8-로드-밸런서-alb와-nlb)
9. [Auto Scaling Group 구성](#9-auto-scaling-group-구성)
10. [실전 아키텍처 구성 예시](#10-실전-아키텍처-구성-예시)
11. [참고 자료](#참고-자료)

---

## 1. VPC란 무엇인가

VPC(Virtual Private Cloud)는 AWS 클라우드 안에 격리된 가상 네트워크 공간이다.
쉽게 말해, 인터넷에서 직접 접근할 수 없는 나만의 사설 데이터센터를 AWS 위에 만드는 것이다.

VPC를 생성하면 CIDR 블록(IP 주소 범위)을 지정한다.
예를 들어 `10.0.0.0/16`으로 설정하면 `10.0.0.0 ~ 10.0.255.255`까지 65,536개의 IP 주소를 사용할 수 있다.

```bash
# VPC 생성
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-prod-vpc}]'
```

VPC는 하나의 AWS 리전(region)에 속하며, 여러 가용 영역(AZ)에 걸쳐 사용할 수 있다.
기본적으로 다른 VPC나 인터넷과는 완전히 격리된 상태로 생성된다.

> **핵심 포인트**: VPC는 AWS 네트워크 설계의 시작점이다. 모든 리소스(EC2, RDS, Lambda 등)는 VPC 안에 배치된다.

---

## 2. 서브넷 설계: Public vs Private

서브넷은 VPC를 더 작은 네트워크 단위로 나눈 것이다.
서브넷은 특정 가용 영역(AZ)에 속하며, 인터넷 접근 여부에 따라 Public과 Private으로 구분한다.

### Public 서브넷

인터넷과 직접 통신이 가능한 서브넷이다.
웹 서버, 로드 밸런서, 배스천 호스트(Bastion Host) 등 외부 접근이 필요한 리소스를 배치한다.

```bash
# Public 서브넷 생성 (ap-northeast-2a)
aws ec2 create-subnet \
  --vpc-id vpc-0abc1234 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-northeast-2a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-2a}]'
```

### Private 서브넷

인터넷에서 직접 접근할 수 없는 서브넷이다.
애플리케이션 서버, 데이터베이스, 캐시 서버 등 내부 리소스를 배치한다.

```bash
# Private 서브넷 생성 (ap-northeast-2a)
aws ec2 create-subnet \
  --vpc-id vpc-0abc1234 \
  --cidr-block 10.0.11.0/24 \
  --availability-zone ap-northeast-2a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-2a}]'
```

### 다중 AZ 설계

고가용성을 위해 최소 2개 이상의 AZ에 서브넷을 배치하는 것이 권장된다.

```
VPC: 10.0.0.0/16
├── Public  Subnet AZ-a: 10.0.1.0/24
├── Public  Subnet AZ-c: 10.0.2.0/24
├── Private Subnet AZ-a: 10.0.11.0/24
└── Private Subnet AZ-c: 10.0.12.0/24
```

한 AZ에 장애가 발생해도 다른 AZ에서 서비스를 지속할 수 있다.

> **핵심 포인트**: 서브넷 CIDR을 설계할 때 미래 확장을 고려해 충분한 IP 주소를 확보해야 한다. `/24`는 251개, `/23`은 507개의 사용 가능한 IP를 제공한다(AWS가 5개 예약).

---

## 3. 인터넷 연결: IGW와 NAT Gateway

### Internet Gateway (IGW)

IGW는 VPC와 인터넷을 연결하는 게이트웨이다.
Public 서브넷의 리소스가 인터넷과 양방향 통신을 할 수 있게 해준다.

```bash
# IGW 생성 및 VPC에 연결
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=my-igw}]'

aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0abc1234 \
  --vpc-id vpc-0abc1234
```

IGW는 VPC당 하나만 연결할 수 있으며, 고가용성이 자동으로 보장된다.

### NAT Gateway

Private 서브넷의 리소스가 인터넷으로 아웃바운드 통신을 할 수 있게 해준다.
중요한 점은 단방향이라는 것이다. 외부에서 Private 서브넷으로 직접 접근은 불가능하다.

```bash
# Elastic IP 할당 (NAT Gateway에 필요)
aws ec2 allocate-address --domain vpc

# NAT Gateway 생성 (Public 서브넷에 배치)
aws ec2 create-nat-gateway \
  --subnet-id subnet-public-2a \
  --allocation-id eipalloc-0abc1234 \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-gw-2a}]'
```

NAT Gateway는 시간당 요금($0.059)과 처리 데이터 요금($0.059/GB)이 부과된다.
비용 절감을 위해 각 AZ마다 하나씩 배치하는 것이 권장된다(AZ 간 데이터 전송 비용 방지).

> **핵심 포인트**: NAT Gateway는 Private 서브넷에서 패키지 업데이트, 외부 API 호출 등에 필수적이다. IGW와 달리 단방향 통신만 지원하므로 보안 측면에서 유리하다.

---

## 4. 라우팅 테이블: 패킷의 길을 정하다

라우팅 테이블은 네트워크 패킷이 어디로 이동할지 결정하는 규칙 집합이다.
각 서브넷은 하나의 라우팅 테이블과 연결되며, 기본적으로 VPC의 메인 라우팅 테이블을 사용한다.

### Public 서브넷용 라우팅 테이블

```bash
# 라우팅 테이블 생성
aws ec2 create-route-table \
  --vpc-id vpc-0abc1234 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]'

# 인터넷 게이트웨이로 가는 기본 경로 추가
aws ec2 create-route \
  --route-table-id rtb-0abc1234 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0abc1234

# Public 서브넷에 라우팅 테이블 연결
aws ec2 associate-route-table \
  --route-table-id rtb-0abc1234 \
  --subnet-id subnet-public-2a
```

### Private 서브넷용 라우팅 테이블

```bash
# 라우팅 테이블 생성
aws ec2 create-route-table \
  --vpc-id vpc-0abc1234 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt-2a}]'

# NAT Gateway로 가는 기본 경로 추가
aws ec2 create-route \
  --route-table-id rtb-0def5678 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-0abc1234

# Private 서브넷에 라우팅 테이블 연결
aws ec2 associate-route-table \
  --route-table-id rtb-0def5678 \
  --subnet-id subnet-private-2a
```

라우팅 테이블의 규칙은 가장 구체적인 경로(Longest Prefix Match)가 우선 적용된다.
`10.0.0.0/16` (VPC 내부)은 로컬 통신에 사용되고, `0.0.0.0/0` (모든 IP)은 외부 게이트웨이로 향한다.

---

## 5. 보안 계층: Security Group vs NACL

AWS에는 두 가지 방화벽 메커니즘이 있다. 각각 다른 레벨에서 동작하며 상호 보완적이다.

### Security Group

EC2 인스턴스(또는 네트워크 인터페이스) 레벨에서 동작하는 가상 방화벽이다.
**상태 저장(Stateful)** 방식으로, 인바운드 규칙을 통과한 트래픽은 자동으로 아웃바운드가 허용된다.

```bash
# 웹 서버용 Security Group 생성
aws ec2 create-security-group \
  --group-name web-sg \
  --description "Web server security group" \
  --vpc-id vpc-0abc1234

# HTTP, HTTPS 인바운드 허용
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc1234 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc1234 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# SSH는 특정 IP만 허용 (관리자 IP)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc1234 \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.0/32
```

Security Group은 "화이트리스트" 방식이다. 명시적으로 허용하지 않은 모든 트래픽은 차단된다.

### NACL (Network Access Control List)

서브넷 레벨에서 동작하는 방화벽이다.
**상태 비저장(Stateless)** 방식으로, 인바운드와 아웃바운드 규칙을 각각 명시해야 한다.

| 규칙 번호 | 프로토콜 | 포트 | 소스/대상 | 허용/거부 |
|----------|---------|------|----------|---------|
| 100 | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 110 | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 120 | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| * | ALL | ALL | 0.0.0.0/0 | DENY |

규칙 번호가 낮을수록 우선순위가 높다. 규칙은 순서대로 평가되며 첫 번째 일치 규칙이 적용된다.

### Security Group vs NACL 비교

| 구분 | Security Group | NACL |
|------|---------------|------|
| 적용 레벨 | 인스턴스(ENI) | 서브넷 |
| 상태 | Stateful | Stateless |
| 기본 정책 | 모두 거부 | 모두 허용 |
| 규칙 유형 | 허용만 가능 | 허용/거부 모두 가능 |
| 규칙 평가 | 모든 규칙 | 번호 순서대로 |

> **핵심 포인트**: 일반적으로 Security Group으로 충분하다. NACL은 서브넷 수준의 추가 방어선이 필요할 때, 예를 들어 특정 IP를 명시적으로 차단하고 싶을 때 사용한다.

---

## 6. EC2 인스턴스 유형 선택 전략

EC2 인스턴스 유형은 CPU, 메모리, 스토리지, 네트워크 성능에 따라 다양한 패밀리로 구분된다.

### 인스턴스 패밀리

**범용(General Purpose)**
- `t` 계열: 버스터블 성능. 평소에는 낮은 CPU를 사용하다가 필요 시 크레딧을 소모해 높은 성능 발휘. 개발/테스트 환경에 적합.
- `m` 계열: CPU와 메모리가 균형 잡힌 범용 인스턴스. 웹 서버, 애플리케이션 서버에 적합.

**컴퓨팅 최적화(Compute Optimized)**
- `c` 계열: 높은 CPU 성능. 배치 처리, 미디어 인코딩, 게임 서버에 적합.

**메모리 최적화(Memory Optimized)**
- `r` 계열: 대용량 메모리. 인메모리 데이터베이스, 빅데이터 분석에 적합.
- `x` 계열: 초고용량 메모리. SAP HANA 등 대규모 인메모리 워크로드에 적합.

**스토리지 최적화(Storage Optimized)**
- `i` 계열: 고성능 NVMe SSD. 고속 I/O가 필요한 NoSQL 데이터베이스에 적합.
- `d` 계열: 대용량 HDD. 데이터 웨어하우스, Hadoop에 적합.

**가속 컴퓨팅(Accelerated Computing)**
- `p`, `g` 계열: GPU 인스턴스. 머신러닝 학습, 그래픽 렌더링에 적합.

### 인스턴스 크기 표기

```
인스턴스 유형: m6i.xlarge
                ^^^ ^^^
                패밀리.크기

크기: nano < micro < small < medium < large < xlarge < 2xlarge < ...
```

```bash
# 사용 가능한 인스턴스 유형 조회
aws ec2 describe-instance-types \
  --filters "Name=instance-type,Values=m6i.*" \
  --query 'InstanceTypes[*].[InstanceType,VCpuInfo.DefaultVCpus,MemoryInfo.SizeInMiB]' \
  --output table
```

### 최신 세대 선택

동일 패밀리라면 최신 세대가 비용 대비 성능이 더 좋다.
`m5` → `m6i` → `m7i` 순으로 성능이 향상되며 가격 차이는 크지 않다.

> **핵심 포인트**: 인스턴스 유형 선택 시 먼저 `m` 계열 범용으로 시작하고, 실제 모니터링 데이터를 기반으로 최적화하는 것이 바람직하다. AWS Compute Optimizer를 활용하면 사용 패턴에 맞는 추천을 받을 수 있다.

---

## 7. 구매 옵션: On-Demand, Reserved, Spot

EC2는 세 가지 구매 방식을 제공하며, 비용과 유연성 사이의 트레이드오프가 있다.

### On-Demand

사용한 만큼만 비용을 지불하는 방식이다. 초 단위 또는 시간 단위로 과금된다.
예측 불가능한 워크로드나 단기 테스트에 적합하다.

- 장점: 약정 없음, 언제든 시작/중지 가능
- 단점: 세 가지 옵션 중 가장 비싸다

### Reserved Instances (RI)

1년 또는 3년 약정을 통해 On-Demand 대비 최대 72% 할인을 받는 방식이다.
지속적으로 운영되는 서비스의 베이스라인 용량에 적합하다.

```bash
# Reserved Instance 구매 가능한 옵션 조회
aws ec2 describe-reserved-instances-offerings \
  --instance-type m6i.large \
  --product-description "Linux/UNIX" \
  --offering-class standard \
  --offering-type "All Upfront"
```

약정 유형별 특징:

| 유형 | 설명 | 할인율 |
|------|------|--------|
| Standard RI | 인스턴스 유형 고정 | 최대 72% |
| Convertible RI | 인스턴스 유형 변경 가능 | 최대 54% |
| Scheduled RI | 특정 시간대만 예약 | 약 5-10% |

### Spot Instances

AWS의 미사용 EC2 용량을 경매 방식으로 구매하는 방식이다.
On-Demand 대비 최대 90% 저렴하지만, AWS가 용량을 회수할 때 2분 전 알림 후 중단된다.

```bash
# Spot 인스턴스 가격 조회
aws ec2 describe-spot-price-history \
  --instance-types m6i.large \
  --product-descriptions "Linux/UNIX" \
  --start-time 2026-03-20T00:00:00 \
  --query 'SpotPriceHistory[*].[Timestamp,SpotPrice,AvailabilityZone]' \
  --output table

# Spot 인스턴스 요청
aws ec2 request-spot-instances \
  --instance-count 2 \
  --type one-time \
  --launch-specification file://spot-spec.json
```

Spot은 배치 처리, 데이터 분석, CI/CD, 스테이트리스 웹 서버의 추가 용량에 적합하다.

### 비용 최적화 전략

실제 서비스에서는 세 가지를 혼합해서 사용하는 것이 효율적이다.

```
베이스라인 트래픽: Reserved Instances (비용 최소화)
정상 트래픽 처리: On-Demand Instances (유연성 확보)
버스트 트래픽:    Spot Instances (비용 최대 절감)
```

> **핵심 포인트**: Savings Plans는 Reserved Instances보다 유연한 약정 방식이다. EC2 Savings Plans는 특정 인스턴스 패밀리와 리전에 약정하고, Compute Savings Plans는 인스턴스 유형에 무관하게 약정한다.

---

## 8. 로드 밸런서: ALB와 NLB

AWS는 Elastic Load Balancing(ELB) 서비스를 통해 두 가지 주요 로드 밸런서를 제공한다.

### ALB (Application Load Balancer)

HTTP/HTTPS 레벨(Layer 7)에서 동작하는 로드 밸런서다.
요청의 URL 경로, 호스트 헤더, HTTP 메서드 등을 기반으로 세밀한 라우팅이 가능하다.

```bash
# ALB 생성
aws elbv2 create-load-balancer \
  --name my-alb \
  --subnets subnet-public-2a subnet-public-2c \
  --security-groups sg-alb-0abc1234 \
  --scheme internet-facing \
  --type application

# 타겟 그룹 생성
aws elbv2 create-target-group \
  --name web-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-0abc1234 \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# 리스너 생성 (HTTPS)
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:... \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...
```

**경로 기반 라우팅 예시**

```bash
# /api/* 요청은 API 서버 타겟 그룹으로 라우팅
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --priority 10 \
  --conditions Field=path-pattern,Values='/api/*' \
  --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:.../api-tg/...
```

ALB는 WebSocket, HTTP/2, gRPC를 지원하며, AWS WAF와 통합하여 웹 애플리케이션 방화벽 기능을 추가할 수 있다.

### NLB (Network Load Balancer)

TCP/UDP/TLS 레벨(Layer 4)에서 동작하는 로드 밸런서다.
초당 수백만 건의 요청을 처리할 수 있으며 매우 낮은 지연 시간(단일 자릿수 밀리초)을 제공한다.

```bash
# NLB 생성
aws elbv2 create-load-balancer \
  --name my-nlb \
  --subnets subnet-public-2a subnet-public-2c \
  --scheme internet-facing \
  --type network

# TCP 타겟 그룹 생성
aws elbv2 create-target-group \
  --name tcp-tg \
  --protocol TCP \
  --port 443 \
  --vpc-id vpc-0abc1234
```

NLB는 고정 IP 주소를 제공하므로, IP 기반 화이트리스트가 필요한 B2B 서비스에 적합하다.
또한 클라이언트 IP를 그대로 전달하므로 소스 IP 보존이 중요한 경우에 사용한다.

### ALB vs NLB 선택 기준

| 기준 | ALB | NLB |
|------|-----|-----|
| 프로토콜 | HTTP/HTTPS/gRPC | TCP/UDP/TLS |
| 라우팅 방식 | 콘텐츠 기반 | 연결 기반 |
| 고정 IP | 미지원 (DNS) | 지원 (Elastic IP) |
| WebSocket | 지원 | 지원 |
| 지연 시간 | 보통 | 매우 낮음 |
| 주요 용도 | 웹 애플리케이션 | 게임, 금융, IoT |

---

## 9. Auto Scaling Group 구성

Auto Scaling Group(ASG)은 트래픽 변화에 따라 EC2 인스턴스 수를 자동으로 조정하는 서비스다.
로드 밸런서와 결합하면 고가용성과 비용 효율성을 동시에 달성할 수 있다.

### Launch Template 생성

ASG가 인스턴스를 시작할 때 사용하는 설정 템플릿이다.

```bash
# Launch Template 생성
aws ec2 create-launch-template \
  --launch-template-name web-lt \
  --version-description "v1" \
  --launch-template-data '{
    "ImageId": "ami-0c2d3e23e757b5d84",
    "InstanceType": "m6i.large",
    "SecurityGroupIds": ["sg-0abc1234"],
    "UserData": "IyEvYmluL2Jhc2gKeXVtIHVwZGF0ZSAteQo=",
    "IamInstanceProfile": {
      "Arn": "arn:aws:iam::123456789012:instance-profile/web-role"
    },
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [{"Key": "Name", "Value": "web-server"}]
    }]
  }'
```

### Auto Scaling Group 생성

```bash
# ASG 생성
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --launch-template LaunchTemplateName=web-lt,Version='$Latest' \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 2 \
  --vpc-zone-identifier "subnet-private-2a,subnet-private-2c" \
  --target-group-arns arn:aws:elasticloadbalancing:... \
  --health-check-type ELB \
  --health-check-grace-period 300
```

`min-size`는 항상 유지할 최소 인스턴스 수, `max-size`는 스케일 아웃 시 최대 수를 의미한다.

### 스케일링 정책 설정

**Target Tracking 정책** (권장)

CPU 사용률, 요청 수 등 특정 지표를 목표값으로 유지하도록 자동 조정한다.

```bash
# CPU 70% 유지를 목표로 하는 스케일링 정책
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 70.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

**Step Scaling 정책**

지표의 크기에 따라 단계적으로 스케일링 크기를 조절하는 방식이다.

```bash
# CloudWatch 알람 기반 스케일 아웃
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name scale-out \
  --policy-type StepScaling \
  --adjustment-type ChangeInCapacity \
  --step-adjustments '[
    {"MetricIntervalLowerBound": 0, "MetricIntervalUpperBound": 20, "ScalingAdjustment": 2},
    {"MetricIntervalLowerBound": 20, "ScalingAdjustment": 4}
  ]'
```

### 헬스 체크와 인스턴스 갱신

ASG는 EC2 헬스 체크와 ELB 헬스 체크를 통해 비정상 인스턴스를 자동으로 교체한다.

```bash
# 인스턴스 갱신 시작 (Rolling update)
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name web-asg \
  --preferences '{
    "MinHealthyPercentage": 80,
    "InstanceWarmup": 300
  }'
```

> **핵심 포인트**: `ScaleInCooldown`은 스케일 인 후 대기 시간, `ScaleOutCooldown`은 스케일 아웃 후 대기 시간이다. 스케일 아웃은 빠르게(쿨다운 짧게), 스케일 인은 보수적으로(쿨다운 길게) 설정하는 것이 안정적이다.

---

## 10. 실전 아키텍처 구성 예시

지금까지 학습한 개념을 종합해 실제 운영 환경에서 자주 사용되는 아키텍처를 구성해 본다.

### 3계층 웹 애플리케이션 아키텍처

```
인터넷
  |
  v
[ALB] (Public Subnet - 멀티 AZ)
  |
  v
[EC2 Auto Scaling Group] (Private Subnet - 멀티 AZ)
  - 웹/앱 서버
  - Security Group: ALB에서만 인바운드 허용
  |
  v
[RDS Multi-AZ] (Private Subnet - DB 전용)
  - Security Group: 앱 서버에서만 인바운드 허용
```

**네트워크 구성 요약**

```
VPC: 10.0.0.0/16

# 공개 계층 (ALB, NAT Gateway)
Public  AZ-a: 10.0.1.0/24
Public  AZ-c: 10.0.2.0/24

# 앱 계층 (EC2 ASG)
Private AZ-a: 10.0.11.0/24
Private AZ-c: 10.0.12.0/24

# 데이터 계층 (RDS, ElastiCache)
Private AZ-a: 10.0.21.0/24
Private AZ-c: 10.0.22.0/24
```

### Security Group 체인

Security Group은 다른 Security Group을 소스로 참조할 수 있다.
이를 통해 IP 주소 대신 역할 기반으로 접근 제어를 구성할 수 있다.

```bash
# ALB Security Group: 인터넷 → ALB
aws ec2 authorize-security-group-ingress \
  --group-id sg-alb \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# 앱 서버 Security Group: ALB → 앱 서버
aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp --port 8080 \
  --source-group sg-alb

# DB Security Group: 앱 서버 → DB
aws ec2 authorize-security-group-ingress \
  --group-id sg-db \
  --protocol tcp --port 5432 \
  --source-group sg-app
```

이 구성은 각 계층이 바로 위 계층에서만 트래픽을 받도록 강제한다.
앱 서버가 직접 인터넷에 노출되거나, DB에 직접 접근하는 경로를 차단한다.

### EC2 비용 최적화 예시

```
베이스라인 (항상 필요):
  - 2개 On-Demand 또는 Reserved (m6i.large) → 고가용성 보장

피크 타임 확장:
  - ASG Target Tracking으로 자동 스케일 아웃
  - 추가 인스턴스는 Spot 혼합 사용 (60% On-Demand + 40% Spot)

비용 절감 효과:
  - 베이스라인 RI 사용 시 On-Demand 대비 ~40% 절감
  - Spot 혼합 사용 시 추가 인스턴스 비용 ~60-70% 절감
```

```bash
# Spot 혼합 ASG 설정
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name web-asg-mixed \
  --mixed-instances-policy '{
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateName": "web-lt",
        "Version": "$Latest"
      },
      "Overrides": [
        {"InstanceType": "m6i.large"},
        {"InstanceType": "m5.large"},
        {"InstanceType": "m6a.large"}
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 40,
      "SpotAllocationStrategy": "capacity-optimized"
    }
  }' \
  --min-size 2 --max-size 20 --desired-capacity 4 \
  --vpc-zone-identifier "subnet-private-2a,subnet-private-2c"
```

`capacity-optimized` 전략은 가장 용량이 풍부한 Spot 풀에서 인스턴스를 선택해 중단 가능성을 낮춘다.

---

## 마무리

이번 편에서는 AWS 네트워크와 컴퓨팅의 핵심 요소를 살펴봤다.

- **VPC**: 격리된 가상 네트워크 공간. CIDR, 서브넷, 라우팅 테이블로 네트워크 구조를 정의한다.
- **IGW / NAT Gateway**: Public 서브넷은 IGW로 양방향 인터넷 연결, Private 서브넷은 NAT로 아웃바운드 연결.
- **Security Group / NACL**: 인스턴스 레벨과 서브넷 레벨의 이중 보안 계층.
- **EC2 인스턴스 유형**: 워크로드 특성에 맞는 패밀리와 크기를 선택하고, 실제 지표를 기반으로 최적화.
- **구매 옵션**: Reserved(베이스라인) + Spot(버스트)의 혼합 전략으로 비용을 최적화.
- **ALB / NLB**: 웹 애플리케이션에는 ALB, 저지연/고성능 TCP 처리에는 NLB.
- **Auto Scaling Group**: Launch Template + Target Tracking으로 트래픽 변화에 자동 대응.

다음 편에서는 이 인프라 위에서 운영되는 관리형 데이터베이스 서비스(RDS, DynamoDB, ElastiCache)를 다룰 예정이다.

---

## 참고 자료

- [Amazon VPC 사용 설명서](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/what-is-amazon-vpc.html)
- [Amazon EC2 인스턴스 유형](https://aws.amazon.com/ko/ec2/instance-types/)
- [AWS Elastic Load Balancing 문서](https://docs.aws.amazon.com/ko_kr/elasticloadbalancing/latest/userguide/what-is-load-balancing.html)
- [Amazon EC2 Auto Scaling 사용 설명서](https://docs.aws.amazon.com/ko_kr/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
- [AWS 비용 최적화 - EC2 구매 옵션](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/instance-purchasing-options.html)
- [VPC 서브넷 설계 모범 사례](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/vpc-subnet-design.html)
- [Security Group vs NACL](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/infrastructure-security.html)
- [EC2 Spot Instances 모범 사례](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/spot-best-practices.html)
