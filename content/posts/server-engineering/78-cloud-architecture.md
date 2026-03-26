---
title: "[AWS / 클라우드] 1편 — 클라우드 아키텍처 기초: 온프레미스와 무엇이 다른가"
date: 2026-03-20T14:08:00+09:00
draft: false
tags: ["AWS", "클라우드", "Well-Architected", "IaaS", "서버"]
series: ["AWS / 클라우드"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "IaaS/PaaS/SaaS 경계와 공유 책임 모델, 리전·가용 영역·엣지 로케이션 개념, AWS Well-Architected Framework 6대 핵심 원칙, 클라우드 네이티브 설계 원칙(Design for Failure, Loosely Coupled)까지"
---

## 들어가며

서버 엔지니어링을 공부하다 보면 어느 순간 "그래서 이걸 어디에 올리지?"라는 질문에 부딪힌다. 온프레미스 서버를 직접 관리하는 시대는 점점 멀어지고, 대부분의 서비스는 클라우드 위에서 돌아간다.

이번 편에서는 클라우드가 온프레미스와 근본적으로 무엇이 다른지, AWS가 제안하는 Well-Architected Framework는 어떤 원칙을 담고 있는지, 클라우드 네이티브 설계란 무엇인지를 처음부터 정리한다.

---

## 목차

1. [온프레미스 vs 클라우드 — 무엇이 달라졌나](#1-온프레미스-vs-클라우드--무엇이-달라졌나)
2. [서비스 모델 — IaaS / PaaS / SaaS](#2-서비스-모델--iaas--paas--saas)
3. [공유 책임 모델](#3-공유-책임-모델)
4. [AWS 글로벌 인프라 — 리전, 가용 영역, 엣지 로케이션](#4-aws-글로벌-인프라--리전-가용-영역-엣지-로케이션)
5. [Well-Architected Framework — 6대 핵심 원칙](#5-well-architected-framework--6대-핵심-원칙)
6. [클라우드 네이티브 설계 원칙](#6-클라우드-네이티브-설계-원칙)
7. [온프레미스에서 클라우드로 전환할 때 흔히 하는 실수](#7-온프레미스에서-클라우드로-전환할-때-흔히-하는-실수)
8. [참고 자료](#참고-자료)

---

## 1. 온프레미스 vs 클라우드 — 무엇이 달라졌나

**온프레미스(On-Premises)**는 자사 데이터센터에 물리 서버를 직접 두고 운영하는 방식이다. 하드웨어 구매, 네트워크 설계, 전력·냉각, OS 패치, 보안 설정까지 모두 직접 책임진다.

클라우드는 이 인프라를 인터넷을 통해 **필요한 만큼, 필요한 때**에 빌려 쓰는 모델이다. AWS, GCP, Azure 같은 클라우드 제공자(CSP, Cloud Service Provider)가 물리 인프라를 소유하고 운영한다.

핵심 차이를 표로 정리하면 다음과 같다.

| 항목 | 온프레미스 | 클라우드 |
|------|-----------|---------|
| 초기 비용 | 높음 (서버·네트워크 구매) | 낮음 (종량제) |
| 확장 속도 | 느림 (장비 도입에 주 단위) | 빠름 (분 단위) |
| 운영 책임 | 전체 (HW~App) | 서비스 모델에 따라 분담 |
| 비용 구조 | CapEx (자본 지출) 중심 | OpEx (운영 지출) 중심 |
| 장애 내성 | 직접 설계 | 제공자 인프라 활용 가능 |
| 글로벌 배포 | 별도 데이터센터 필요 | 리전 선택만으로 가능 |

클라우드가 온프레미스보다 언제나 싸다는 건 오해다. 트래픽이 극히 안정적이고 예측 가능한 대규모 서비스는 온프레미스가 총소유비용(TCO) 면에서 유리할 수 있다. 클라우드의 핵심 가치는 **비용 절감**이 아니라 **민첩성(Agility)**과 **탄력성(Elasticity)**이다. 예를 들어 새 서비스를 테스트할 때 서버를 수 분 안에 올리고, 반응이 없으면 즉시 내려서 비용을 낭비하지 않는 것이 클라우드가 가져다주는 실질적인 이점이다.

---

## 2. 서비스 모델 — IaaS / PaaS / SaaS

클라우드 서비스는 제공자가 관리하는 범위에 따라 세 가지 모델로 나뉜다.

### IaaS — Infrastructure as a Service

**IaaS**는 가상 머신(VM), 스토리지, 네트워킹 같은 인프라 자원을 제공한다. OS 이상의 모든 것은 사용자가 책임진다.

- AWS EC2, Azure VM, Google Compute Engine
- 직접 OS를 설치하고 미들웨어와 애플리케이션을 구성한다.
- 온프레미스에서 마이그레이션할 때 가장 먼저 닿는 모델이다.

### PaaS — Platform as a Service

**PaaS**는 실행 환경(런타임, 미들웨어, 데이터베이스)을 제공한다. 사용자는 애플리케이션 코드만 올리면 된다. OS 패치나 서버 설정은 제공자가 담당한다.

- AWS Elastic Beanstalk, Google App Engine, Heroku
- AWS RDS는 DB 엔진(MySQL, PostgreSQL 등)을 PaaS 방식으로 제공한다.
- OS 관리 부담이 없고 배포가 빠르지만, 플랫폼 제약에 따라야 한다.

### SaaS — Software as a Service

**SaaS**는 완성된 애플리케이션을 인터넷으로 제공한다. 사용자는 소프트웨어를 직접 설치하지 않고 브라우저나 API로 접근한다.

- Gmail, Slack, Salesforce, GitHub
- 인프라와 애플리케이션 모두 제공자가 관리한다.
- 서버 엔지니어링 관점에서는 SaaS를 **만드는** 쪽이다.

### 서비스 모델별 책임 비교

```
사용자 책임 범위 (많음 → 적음)
┌─────────────────┬──────────────────┬──────────────────┐
│      IaaS       │      PaaS        │      SaaS        │
├─────────────────┼──────────────────┼──────────────────┤
│ Application     │ Application      │ ████ 제공자 관리  │
│ Runtime         │ ████ 제공자 관리  │ ████ 제공자 관리  │
│ OS              │ ████ 제공자 관리  │ ████ 제공자 관리  │
│ Virtualization  │ ████ 제공자 관리  │ ████ 제공자 관리  │
│ Hardware        │ ████ 제공자 관리  │ ████ 제공자 관리  │
└─────────────────┴──────────────────┴──────────────────┘
```

현실에서는 이 세 모델의 경계가 흐릿하다. AWS Lambda는 코드만 올리면 되니 PaaS에 가깝지만, 서버리스(Serverless)라는 독립 범주로 부르기도 한다.

---

## 3. 공유 책임 모델

클라우드를 쓴다고 보안 책임이 사라지는 것이 아니다. AWS는 **공유 책임 모델(Shared Responsibility Model)**을 통해 제공자와 사용자의 보안 책임 경계를 명확히 정의한다.

### AWS의 책임 — "클라우드의 보안(Security OF the Cloud)"

AWS가 책임지는 영역이다.

- 데이터센터 물리 보안 (출입 통제, 화재·홍수 방지)
- 하이퍼바이저, 네트워크 장비, 스토리지 하드웨어
- AWS 서비스 자체의 소프트웨어 (예: RDS 엔진, S3 인프라)
- 글로벌 네트워크 인프라

### 사용자의 책임 — "클라우드에서의 보안(Security IN the Cloud)"

사용자가 책임지는 영역이다.

- EC2 위에 올린 OS 패치 및 설정
- 애플리케이션 코드의 취약점
- 데이터 암호화 (전송 중, 저장 중)
- IAM 사용자 및 권한 관리
- 보안 그룹(Security Group) 및 네트워크 ACL 설정
- S3 버킷 퍼블릭 공개 여부

### 서비스 모델별 책임 이동

서비스 모델이 IaaS → PaaS → SaaS로 올라갈수록 사용자 책임 범위는 줄어든다.

```
IaaS (EC2):   사용자 → [OS, 미들웨어, 앱, 데이터, 네트워크 설정]
PaaS (RDS):   사용자 → [앱 설정, 데이터, DB 접근 제어]
SaaS:         사용자 → [데이터, 사용자 계정 관리]
```

공유 책임 모델을 이해하지 못하면 "AWS가 다 알아서 해주겠지"라는 착각에 빠진다. S3 버킷을 잘못 공개하거나 RDS를 퍼블릭 서브넷에 두거나 IAM 키를 코드에 하드코딩하는 실수가 모두 **사용자 책임 영역**의 실수다.

---

## 4. AWS 글로벌 인프라 — 리전, 가용 영역, 엣지 로케이션

AWS 인프라는 세 계층의 물리적 단위로 구성된다.

### 리전 (Region)

**리전**은 지리적으로 독립된 데이터센터 클러스터다. 각 리전은 서로 완전히 독립적이며, 한 리전의 장애가 다른 리전에 영향을 주지 않는다.

- 예: `ap-northeast-2` (서울), `us-east-1` (버지니아 북부), `eu-west-1` (아일랜드)
- 2024년 기준 33개 이상의 리전이 운영 중이다.
- 데이터 주권(Data Sovereignty) 법률 때문에 특정 리전을 선택해야 하는 경우가 있다. 국내 금융·의료 서비스는 서울 리전(`ap-northeast-2`)을 사용하는 경우가 많다.
- 서비스에 따라 일부 리전에서만 제공된다. 신규 서비스는 보통 `us-east-1`에 먼저 출시된다.

### 가용 영역 (Availability Zone, AZ)

**가용 영역(AZ)**은 리전 내에 위치한 독립된 데이터센터(또는 데이터센터 클러스터)다. 각 AZ는 별도의 전력·냉각·네트워크 인프라를 갖춘다.

- 서울 리전(`ap-northeast-2`)은 4개의 AZ(`ap-northeast-2a`, `2b`, `2c`, `2d`)를 제공한다.
- AZ 간에는 전용 고속 광 케이블로 연결되어 있고, 일반 인터넷보다 낮은 지연(single-digit ms)을 제공한다.
- **고가용성(HA) 설계의 기본 단위**는 AZ 분산이다. 서비스를 2개 이상의 AZ에 배포하면 하나의 AZ가 장애를 겪어도 서비스가 유지된다.

```
ap-northeast-2 (서울 리전)
├── ap-northeast-2a  ← AZ 1 (독립 데이터센터)
├── ap-northeast-2b  ← AZ 2
├── ap-northeast-2c  ← AZ 3
└── ap-northeast-2d  ← AZ 4
```

### 엣지 로케이션 (Edge Location)

**엣지 로케이션**은 CDN(CloudFront)과 DNS(Route 53) 서비스를 위한 캐시 노드다. 리전보다 훨씬 많은 수(400개 이상)가 전 세계 주요 도시에 분산되어 있다.

사용자와 지리적으로 가까운 엣지 로케이션에서 정적 콘텐츠를 제공하면 레이턴시를 크게 줄일 수 있다. 서울에서 버지니아 리전 S3의 이미지를 직접 받으면 100ms 이상 걸리지만, 서울 엣지 로케이션에서 캐시된 이미지를 받으면 수 ms로 줄어든다.

### 세 계층의 역할 요약

| 단위 | 개수(AWS 전체) | 주요 역할 | 설계 포인트 |
|------|--------------|----------|------------|
| 리전 | 33+ | 지리적 격리, 데이터 주권 | 재해 복구(DR) 설계 |
| 가용 영역 | 105+ | 단일 AZ 장애 격리 | 고가용성(HA) 설계 |
| 엣지 로케이션 | 400+ | 콘텐츠 캐시, 낮은 레이턴시 | CDN, DNS 설계 |

---

## 5. Well-Architected Framework — 6대 핵심 원칙

AWS는 **Well-Architected Framework**를 통해 클라우드에서 좋은 아키텍처를 만드는 방법을 6개의 원칙(Pillar)으로 정리한다. 단순히 AWS를 잘 쓰는 방법이 아니라, 클라우드 아키텍처 전반에 통용되는 원칙이다.

### 1. 운영 우수성 (Operational Excellence)

운영을 코드처럼 다루는 원칙이다. 수동 작업을 자동화하고, 실패에서 배우며, 운영 절차를 지속적으로 개선한다.

- **핵심 질문**: "이 운영 작업을 자동화할 수 있는가?"
- 인프라를 코드로 관리(IaC)하고, 변경 사항을 코드 리뷰와 자동화된 배포 파이프라인을 통해 적용한다.
- 장애 후 블레임리스(Blameless) 포스트모템을 진행해 시스템 개선에 집중한다.

```
Before: 개발자가 SSH 접속 → 수동으로 설정 파일 편집 → "어떻게 변경했지?"
After:  코드 변경 → PR 리뷰 → CI/CD → Terraform apply → 변경 이력 추적 가능
```

### 2. 보안 (Security)

데이터, 시스템, 자산을 보호하면서 비즈니스 가치를 제공하는 원칙이다.

- **최소 권한 원칙(Principle of Least Privilege)**: IAM 정책에서 필요한 권한만 부여한다.
- **전 계층 보안(Defense in Depth)**: 네트워크, OS, 애플리케이션, 데이터 계층 각각에 보안 통제를 둔다.
- 보안 이벤트에 자동으로 대응하는 메커니즘을 구축한다. AWS GuardDuty + Lambda로 의심스러운 API 호출 시 자동 차단하는 패턴이 대표적이다.
- 저장 데이터(at rest)와 전송 데이터(in transit) 모두 암호화한다.

### 3. 신뢰성 (Reliability)

의도한 기능을 일관되게 수행하고, 장애를 빠르게 복구하는 원칙이다.

- **핵심 질문**: "이 컴포넌트가 장애를 겪어도 서비스가 유지되는가?"
- 단일 장애 지점(Single Point of Failure, SPOF)을 없애는 것이 첫 번째 목표다.
- 자동 복구(Auto Recovery)를 설계한다. EC2 헬스 체크 실패 시 자동 교체, RDS 장애 시 자동 페일오버가 예시다.
- 요청량이 급증할 때 자동으로 스케일 아웃한다(Auto Scaling).

```
나쁜 예: 단일 EC2 + 단일 RDS → 인스턴스 장애 시 서비스 다운
좋은 예: ALB → EC2 (Multi-AZ) + RDS Multi-AZ → AZ 장애에도 서비스 유지
```

### 4. 성능 효율성 (Performance Efficiency)

컴퓨팅 자원을 효율적으로 사용하고, 요구사항이 바뀌어도 효율성을 유지하는 원칙이다.

- 워크로드 특성에 맞는 인스턴스 유형을 선택한다. 메모리 집약적 작업에는 `r` 계열, 컴퓨팅 집약적 작업에는 `c` 계열이 적합하다.
- 서버리스와 매니지드 서비스를 적극 활용해 운영 부담을 줄이면서 성능을 확보한다.
- 글로벌 사용자를 위해 CloudFront로 레이턴시를 줄인다.
- 성능 실험을 자주 한다. 클라우드에서는 새 인스턴스 유형을 테스트하는 비용이 온프레미스에 비해 극히 낮다.

### 5. 비용 최적화 (Cost Optimization)

비즈니스 가치를 제공하면서 지출을 최소화하는 원칙이다.

- **가장 흔한 낭비**: 사용하지 않는 인스턴스 실행, 과도한 인스턴스 사이즈, 오래된 스냅샷 방치.
- Reserved Instance(예약 인스턴스)나 Savings Plans로 온디맨드 대비 최대 72% 할인을 받는다.
- Spot Instance는 중단 가능한 배치 작업에 사용하면 온디맨드 대비 최대 90% 저렴하다.
- AWS Cost Explorer와 Trusted Advisor로 비용 이상을 모니터링한다.

```bash
# AWS CLI로 미사용 EBS 볼륨 찾기 (비용 절감 포인트)
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].[VolumeId,Size,CreateTime]' \
  --output table
```

### 6. 지속 가능성 (Sustainability)

2021년에 추가된 여섯 번째 원칙이다. 클라우드 워크로드가 환경에 미치는 영향을 최소화하는 데 집중한다.

- 자원 활용률을 높여 불필요한 인프라를 줄인다. 서버리스와 컨테이너는 유휴 자원을 최소화한다.
- 관리형 서비스를 사용하면 AWS가 규모의 경제로 에너지 효율을 높인다.
- 데이터 생명주기 정책을 설정해 불필요한 데이터 저장을 줄인다.

### 6대 원칙 요약

| 원칙 | 핵심 질문 | 대표 AWS 서비스 |
|------|----------|---------------|
| 운영 우수성 | 자동화할 수 있는가? | CloudFormation, CodePipeline |
| 보안 | 최소 권한인가? | IAM, GuardDuty, KMS |
| 신뢰성 | 장애가 나도 동작하는가? | Multi-AZ, Auto Scaling, Route 53 |
| 성능 효율성 | 올바른 리소스를 쓰는가? | CloudFront, ElastiCache, Lambda |
| 비용 최적화 | 낭비가 없는가? | Cost Explorer, Reserved Instances |
| 지속 가능성 | 환경 영향을 줄이는가? | Serverless, 컨테이너, 수명 주기 정책 |

---

## 6. 클라우드 네이티브 설계 원칙

클라우드에서 애플리케이션을 잘 운영하려면 온프레미스 방식의 사고방식에서 벗어나야 한다. 클라우드 네이티브(Cloud Native) 설계에서 자주 언급되는 핵심 원칙들을 정리한다.

### Design for Failure — 실패를 가정하고 설계하라

온프레미스에서는 서버를 오래 안정적으로 유지하는 것이 목표였다. 클라우드에서는 인스턴스가 언제든 죽을 수 있다고 가정하고 설계한다.

AWS EC2 인스턴스는 하드웨어 장애, 유지보수 이벤트, Spot 중단 등 다양한 이유로 예고 없이 종료될 수 있다. 이를 받아들이고 **자동 복구**와 **무상태(Stateless) 설계**를 기본으로 한다.

```
나쁜 설계: 인스턴스 1개 → 장애 시 수동 복구 필요, 복구까지 수십 분
좋은 설계: Auto Scaling Group (최소 2개 AZ) → 인스턴스 장애 감지 후 자동 교체, 수 분 내 복구
```

**Netflix의 Chaos Engineering**이 이 원칙의 극단적 실천이다. 프로덕션 환경에서 의도적으로 인스턴스를 종료해 시스템이 실패를 견디는지 검증한다.

### Loosely Coupled — 느슨하게 연결하라

컴포넌트 간 직접 의존성을 줄이고, 실패가 전파되지 않도록 격리한다. 동기 직접 호출보다 **메시지 큐**나 **이벤트**를 활용한 비동기 통신을 선호한다.

```
강하게 연결된 예:
주문 서비스 → HTTP → 재고 서비스
재고 서비스 응답 지연 → 주문 서비스까지 지연

느슨하게 연결된 예:
주문 서비스 → SQS → 재고 서비스
재고 서비스 지연 → SQS에 메시지 쌓임, 주문 서비스는 정상 응답
```

### Elasticity — 수요에 맞게 늘리고 줄여라

온프레미스에서는 최대 트래픽을 감당할 만큼 서버를 미리 확보한다. 평소에는 자원이 유휴 상태로 낭비된다. 클라우드에서는 트래픽에 따라 자동으로 확장/축소(Elastic Scale)한다.

```yaml
# EC2 Auto Scaling Group 예시
AutoScalingGroup:
  MinSize: 2           # 최소 2개 유지 (HA 보장)
  MaxSize: 20          # 최대 20개까지 확장
  DesiredCapacity: 2
  TargetTrackingScaling:
    MetricType: ASGAverageCPUUtilization
    TargetValue: 60    # CPU 60% 기준으로 자동 증감
```

### Immutable Infrastructure — 인프라를 교체하라, 수정하지 말라

온프레미스에서는 서버에 SSH로 접속해 패치를 적용하고 설정을 바꾸는 것이 일반적이었다. 이 방식은 서버마다 설정이 달라지는 **설정 드리프트(Configuration Drift)** 문제를 낳는다.

클라우드 네이티브 접근에서는 서버를 직접 수정하지 않는다. 새 AMI(Amazon Machine Image)나 컨테이너 이미지를 빌드하고, 기존 인스턴스를 새 이미지로 교체한다.

```
Mutable 방식: SSH → apt update → 설정 파일 수정 → 재시작
Immutable 방식: 코드 변경 → CI로 새 AMI 빌드 → ASG 롤링 업데이트 → 구 인스턴스 자동 폐기
```

### Infrastructure as Code — 인프라를 코드로

클릭으로 인프라를 만들면 재현이 불가능하다. Terraform, AWS CDK, CloudFormation 같은 IaC 도구로 인프라를 코드로 정의하고 버전 관리한다.

```hcl
# Terraform으로 EC2 인스턴스 선언
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id

  tags = {
    Name = "web-server"
    Env  = "production"
  }
}
```

코드로 정의된 인프라는 코드 리뷰, 자동화 테스트, 변경 이력 추적이 모두 가능하다.

### Managed Services First — 관리형 서비스를 먼저 검토하라

EC2에 직접 MySQL을 설치할 수도 있지만, AWS RDS를 쓰면 자동 백업, 장애 복구, 패치 적용, 모니터링을 AWS가 담당한다. 관리형 서비스를 먼저 검토하고, 기능 제약이나 비용이 문제일 때만 직접 운영을 고려한다.

| 직접 운영 | 관리형 서비스 |
|---------|------------|
| EC2에 MySQL 설치 | Amazon RDS |
| EC2에 Redis 설치 | Amazon ElastiCache |
| EC2에 Elasticsearch 설치 | Amazon OpenSearch Service |
| EC2에 Kafka 설치 | Amazon MSK |
| EC2에 Nginx 로드 밸런서 | AWS ALB |

---

## 7. 온프레미스에서 클라우드로 전환할 때 흔히 하는 실수

클라우드를 처음 도입할 때 온프레미스 방식을 그대로 가져오는 경향이 있다. 이를 **Lift & Shift**라고 부르는데, 단기적으로는 빠르지만 클라우드의 장점을 거의 활용하지 못한다.

### 실수 1: 단일 AZ 배포

가장 흔한 실수다. 온프레미스에서 서버 한 대를 쓰던 습관으로 EC2를 하나만 띄운다. 해당 AZ에 장애가 발생하면 서비스가 다운된다.

**해결책**: 최소 2개 AZ에 리소스를 분산하고 로드 밸런서를 앞에 둔다.

### 실수 2: 보안 그룹을 전체 오픈

포트 `0.0.0.0/0`으로 열어두고 방화벽은 OS 수준에서만 관리한다. AWS 보안 그룹(Security Group)은 화이트리스트 방식의 스테이트풀 방화벽으로, 필요한 포트와 소스 IP만 허용해야 한다.

```
나쁜 예: Inbound 0.0.0.0/0:0-65535 (전체 허용)
좋은 예: Inbound 0.0.0.0/0:443 (HTTPS만 허용)
         Inbound [ALB Security Group]:8080 (ALB에서만 앱 포트 허용)
         Inbound [Bastion Security Group]:22 (배스천 호스트에서만 SSH 허용)
```

### 실수 3: 로그를 인스턴스 로컬에 저장

EC2가 종료되면 로컬 로그도 사라진다. 로그는 항상 CloudWatch Logs나 S3 같은 외부 저장소로 보내야 한다.

### 실수 4: 하드코딩된 자격 증명

코드나 설정 파일에 AWS Access Key를 직접 쓰는 것은 심각한 보안 위협이다. EC2에서 실행되는 코드는 **IAM Role**을 통해 자격 증명을 얻어야 한다.

```python
# 나쁜 예 — Access Key 하드코딩
import boto3
client = boto3.client(
    's3',
    aws_access_key_id='AKIAIOSFODNN7EXAMPLE',      # 절대 하지 말 것
    aws_secret_access_key='wJalrXUtnFEMI/K7MDENG'  # 절대 하지 말 것
)

# 좋은 예 — IAM Role 활용 (EC2에 Role 연결 후 자격 증명 자동 주입)
import boto3
client = boto3.client('s3')  # IAM Role에서 자동으로 자격 증명을 가져옴
```

### 실수 5: 리소스 정리 없이 방치

테스트용으로 만든 EC2, 스냅샷, ELB 등을 정리하지 않으면 비용이 쌓인다. AWS Trusted Advisor와 Cost Explorer를 주기적으로 확인하고 불필요한 리소스는 삭제한다.

---

## 클라우드 전환 체크리스트

클라우드 아키텍처를 설계하거나 점검할 때 다음 항목을 기준으로 확인한다.

| 항목 | 확인 방법 |
|------|---------|
| 2개 이상의 AZ에 배포 | AWS Console → 리소스 AZ 분산 확인 |
| IAM 최소 권한 원칙 | IAM Access Analyzer 활용 |
| 보안 그룹 최소 오픈 | Security Group 인바운드 규칙 검토 |
| 데이터 암호화 (저장/전송) | KMS, ACM, S3 버킷 정책 확인 |
| 자동 복구 설정 | Auto Scaling Group, RDS Multi-AZ |
| 로그 외부 저장 | CloudWatch Logs 또는 S3 연동 |
| IaC로 인프라 관리 | Terraform / CloudFormation 사용 여부 |
| 비용 모니터링 | AWS Budgets 알람 설정 |
| 백업 및 복구 테스트 | RDS 스냅샷 복원 테스트 주기적 실시 |

---

## 참고 자료

- [AWS Well-Architected Framework 공식 문서](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [AWS 공유 책임 모델](https://aws.amazon.com/compliance/shared-responsibility-model/)
- [AWS 글로벌 인프라](https://aws.amazon.com/about-aws/global-infrastructure/)
- [AWS Well-Architected Tool](https://aws.amazon.com/well-architected-tool/)
- [Cloud Native Computing Foundation (CNCF) — Cloud Native Definition](https://github.com/cncf/toc/blob/main/DEFINITION.md)
- [The Twelve-Factor App](https://12factor.net/ko/)
- [Netflix Tech Blog — Chaos Engineering](https://netflixtechblog.com/tagged/chaos-engineering)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
