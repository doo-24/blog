---
title: "[AWS 실전 구축] 1편 — VPC 실전 설계: 프로덕션 네트워크를 직접 구성하기"
date: 2026-03-17T11:08:00+09:00
draft: false
tags: ["VPC", "서브넷", "NAT Gateway", "Security Group", "AWS", "서버"]
series: ["AWS 실전 구축"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "멀티 AZ VPC 설계(퍼블릭/프라이빗/격리 서브넷 3-Tier), NAT Gateway 이중화와 VPC Endpoints, ALB+Target Group 실전 구성, Security Group 설계 패턴(체이닝, 자기 참조)까지"
---

AWS에서 서비스를 운영한다는 것은 결국 네트워크를 설계한다는 것이다. VPC(Virtual Private Cloud)는 AWS 인프라의 기반이 되는 격리된 네트워크 공간으로, 여기서 내리는 설계 결정이 이후 보안, 가용성, 비용 전반에 영향을 미친다. 이번 편에서는 프로덕션 수준의 VPC를 처음부터 직접 구성하면서 핵심 개념을 익힌다.

---

## 1. 멀티 AZ VPC 설계 — 3-Tier 서브넷 구조

### 왜 3-Tier인가

프로덕션 VPC는 단순히 서브넷 하나에 모든 리소스를 넣지 않는다. 계층(Tier)을 나누는 이유는 트래픽 흐름을 제어하고, 리소스별 접근 범위를 명확히 하기 위해서다.

| 계층 | 서브넷 유형 | 대표 리소스 |
|------|-----------|------------|
| 1 | 퍼블릭 서브넷 | ALB, Bastion Host, NAT Gateway |
| 2 | 프라이빗 서브넷 | EC2 애플리케이션 서버, EKS 노드 |
| 3 | 격리 서브넷 | RDS, ElastiCache, 내부 DB |

격리 서브넷(Isolated Subnet)은 인터넷 게이트웨이도, NAT Gateway도 연결되지 않는다. 데이터베이스처럼 외부 통신이 불필요한 리소스를 여기에 배치한다.

```text
인터넷
  |
[Internet Gateway]
  |
  ┌────────────────────────────────────────────────┐
  │ VPC (10.0.0.0/16)                              │
  │                                                 │
  │  [퍼블릭 서브넷]  ALB, NAT GW, Bastion          │
  │       |                                         │
  │  [NAT Gateway] (아웃바운드 전용)                 │
  │       |                                         │
  │  [프라이빗 서브넷]  EC2, ECS 앱 서버             │
  │       |                                         │
  │  [격리 서브넷]  RDS, ElastiCache (인터넷 차단)   │
  └────────────────────────────────────────────────┘
  ※ 각 계층은 AZ-A / AZ-C 에 각각 배치 (Multi-AZ)
```

### CIDR 설계 원칙

VPC CIDR은 나중에 변경이 매우 어렵다. VPC에 배포된 리소스(EC2, RDS, Lambda 등)는 모두 해당 CIDR 범위의 IP를 사용하며, CIDR을 바꾸려면 VPC를 삭제하고 재구성해야 한다. 처음부터 충분한 IP 공간을 확보해야 한다.

```
VPC: 10.0.0.0/16  (65,536개 IP)

AZ-A (ap-northeast-2a):
  퍼블릭:   10.0.0.0/24   (256개)
  프라이빗: 10.0.10.0/23  (512개)
  격리:     10.0.20.0/24  (256개)

AZ-C (ap-northeast-2c):
  퍼블릭:   10.0.1.0/24   (256개)
  프라이빗: 10.0.12.0/23  (512개)
  격리:     10.0.21.0/24  (256개)
```

프라이빗 서브넷에 `/23`을 할당한 이유는 애플리케이션 서버 수가 늘어날 경우를 고려했기 때문이다. 퍼블릭 서브넷은 로드밸런서와 NAT Gateway만 올라가므로 `/24`면 충분하다.

### Terraform으로 VPC 생성

```hcl
# vpc.tf

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "prod-vpc"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "prod-igw"
  }
}

# 퍼블릭 서브넷 (AZ-A)
resource "aws_subnet" "public_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.0.0/24"
  availability_zone       = "ap-northeast-2a"
  map_public_ip_on_launch = true

  tags = {
    Name = "prod-public-a"
    Tier = "public"
  }
}

# 퍼블릭 서브넷 (AZ-C)
resource "aws_subnet" "public_c" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-northeast-2c"
  map_public_ip_on_launch = true

  tags = {
    Name = "prod-public-c"
    Tier = "public"
  }
}

# 프라이빗 서브넷 (AZ-A)
resource "aws_subnet" "private_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.10.0/23"
  availability_zone = "ap-northeast-2a"

  tags = {
    Name = "prod-private-a"
    Tier = "private"
  }
}

# 프라이빗 서브넷 (AZ-C)
resource "aws_subnet" "private_c" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.12.0/23"
  availability_zone = "ap-northeast-2c"

  tags = {
    Name = "prod-private-c"
    Tier = "private"
  }
}

# 격리 서브넷 (AZ-A)
resource "aws_subnet" "isolated_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.20.0/24"
  availability_zone = "ap-northeast-2a"

  tags = {
    Name = "prod-isolated-a"
    Tier = "isolated"
  }
}

# 격리 서브넷 (AZ-C)
resource "aws_subnet" "isolated_c" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.21.0/24"
  availability_zone = "ap-northeast-2c"

  tags = {
    Name = "prod-isolated-c"
    Tier = "isolated"
  }
}
```

`enable_dns_hostnames = true`는 EC2 인스턴스에 퍼블릭 DNS 이름을 자동으로 부여한다. ALB와 RDS가 도메인으로 서로를 참조할 때 필요하다.

### 라우팅 테이블 설정

퍼블릭 서브넷은 Internet Gateway로 기본 경로를 설정하고, 프라이빗 서브넷은 NAT Gateway로 연결한다.

```hcl
# 퍼블릭 라우팅 테이블
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "prod-public-rt"
  }
}

resource "aws_route_table_association" "public_a" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_c" {
  subnet_id      = aws_subnet.public_c.id
  route_table_id = aws_route_table.public.id
}
```

격리 서브넷에는 별도 라우팅 테이블을 만들되, 외부 경로를 추가하지 않는다. VPC 내부 통신(로컬 경로)만 허용된다.

---

## 2. NAT Gateway 이중화와 VPC Endpoints

### NAT Gateway를 AZ별로 분리해야 하는 이유

NAT Gateway를 하나만 두면 해당 AZ가 장애를 일으켰을 때 다른 AZ의 프라이빗 서브넷도 인터넷 연결이 끊긴다. 비용이 추가되지만 프로덕션에서는 AZ별로 하나씩 배치하는 것이 원칙이다.

```hcl
# NAT Gateway용 EIP (AZ-A)
resource "aws_eip" "nat_a" {
  domain = "vpc"
  tags = {
    Name = "prod-nat-eip-a"
  }
}

# NAT Gateway용 EIP (AZ-C)
resource "aws_eip" "nat_c" {
  domain = "vpc"
  tags = {
    Name = "prod-nat-eip-c"
  }
}

# NAT Gateway (AZ-A) — 퍼블릭 서브넷에 위치
resource "aws_nat_gateway" "a" {
  allocation_id = aws_eip.nat_a.id
  subnet_id     = aws_subnet.public_a.id

  tags = {
    Name = "prod-nat-a"
  }

  depends_on = [aws_internet_gateway.main]
}

# NAT Gateway (AZ-C)
resource "aws_nat_gateway" "c" {
  allocation_id = aws_eip.nat_c.id
  subnet_id     = aws_subnet.public_c.id

  tags = {
    Name = "prod-nat-c"
  }

  depends_on = [aws_internet_gateway.main]
}

# 프라이빗 라우팅 테이블 (AZ-A) — NAT-A로 연결
resource "aws_route_table" "private_a" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.a.id
  }

  tags = {
    Name = "prod-private-rt-a"
  }
}

# 프라이빗 라우팅 테이블 (AZ-C) — NAT-C로 연결
resource "aws_route_table" "private_c" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.c.id
  }

  tags = {
    Name = "prod-private-rt-c"
  }
}

resource "aws_route_table_association" "private_a" {
  subnet_id      = aws_subnet.private_a.id
  route_table_id = aws_route_table.private_a.id
}

resource "aws_route_table_association" "private_c" {
  subnet_id      = aws_subnet.private_c.id
  route_table_id = aws_route_table.private_c.id
}
```

### VPC Endpoints — NAT Gateway 없이 AWS 서비스 접근

프라이빗 서브넷의 EC2가 S3나 DynamoDB를 사용할 때 NAT Gateway를 거치면 데이터 전송 비용이 발생한다. VPC Endpoint를 사용하면 트래픽이 AWS 내부 네트워크를 통해 전달되므로 비용을 절감할 수 있다.

두 가지 유형이 있다.

- **Gateway Endpoint**: S3, DynamoDB 전용. 라우팅 테이블에 경로를 추가하는 방식. 무료.
- **Interface Endpoint**: 대부분의 AWS 서비스. ENI를 서브넷에 생성. 시간당 요금 발생.

```hcl
# S3 Gateway Endpoint
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.ap-northeast-2.s3"
  vpc_endpoint_type = "Gateway"

  route_table_ids = [
    aws_route_table.private_a.id,
    aws_route_table.private_c.id,
  ]

  tags = {
    Name = "prod-s3-endpoint"
  }
}

# DynamoDB Gateway Endpoint
resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.ap-northeast-2.dynamodb"
  vpc_endpoint_type = "Gateway"

  route_table_ids = [
    aws_route_table.private_a.id,
    aws_route_table.private_c.id,
  ]

  tags = {
    Name = "prod-dynamodb-endpoint"
  }
}

# SSM Interface Endpoint (Bastion 없이 EC2 접속 시 필요)
resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-northeast-2.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = [aws_subnet.private_a.id, aws_subnet.private_c.id]
  security_group_ids  = [aws_security_group.endpoint.id]
  private_dns_enabled = true

  tags = {
    Name = "prod-ssm-endpoint"
  }
}
```

`private_dns_enabled = true`를 설정하면 기존 SDK 코드 변경 없이 엔드포인트로 트래픽이 자동 라우팅된다. SSM 사용을 위해서는 `ssm`, `ssmmessages`, `ec2messages` 세 개의 Interface Endpoint가 모두 필요하다.

### AWS CLI로 Endpoint 연결 상태 확인

```bash
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=vpc-xxxxxxxxx" \
  --query "VpcEndpoints[*].{Service:ServiceName,State:State,Type:VpcEndpointType}" \
  --output table
```

---

## 3. ALB + Target Group 실전 구성

### ALB 기본 구성

ALB(Application Load Balancer)는 퍼블릭 서브넷에 배치하고, 뒤에 붙는 EC2는 프라이빗 서브넷에 둔다. 인터넷에서 ALB로 들어오고, ALB가 프라이빗 서버로 전달하는 구조다.

```hcl
# ALB용 Security Group
resource "aws_security_group" "alb" {
  name        = "prod-alb-sg"
  description = "ALB Security Group"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "prod-alb-sg"
  }
}

# ALB
resource "aws_lb" "main" {
  name               = "prod-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = [aws_subnet.public_a.id, aws_subnet.public_c.id]

  enable_deletion_protection = true

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "prod-alb"
    enabled = true
  }

  tags = {
    Name = "prod-alb"
  }
}
```

`enable_deletion_protection = true`는 실수로 ALB를 삭제하는 것을 방지한다. 프로덕션에서는 반드시 활성화한다.

### Target Group과 헬스체크

```hcl
# API 서버용 Target Group
resource "aws_lb_target_group" "api" {
  name        = "prod-api-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "instance"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }

  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400
    enabled         = false
  }

  tags = {
    Name = "prod-api-tg"
  }
}

# 웹 프론트 서버용 Target Group
resource "aws_lb_target_group" "web" {
  name        = "prod-web-tg"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "instance"

  health_check {
    path                = "/"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    timeout             = 5
    matcher             = "200-299"
  }

  tags = {
    Name = "prod-web-tg"
  }
}
```

헬스체크 `interval`과 `timeout`의 차이를 혼동하기 쉽다. `interval`은 헬스체크 시도 사이의 간격이고, `timeout`은 응답을 기다리는 최대 시간이다. `timeout`은 반드시 `interval`보다 작아야 한다.

### HTTPS 리스너와 경로 기반 라우팅

```hcl
# HTTP -> HTTPS 리다이렉트 리스너
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# HTTPS 리스너
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

# 경로 기반 라우팅 규칙 — /api/* 는 API Target Group으로
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}

# 호스트 기반 라우팅 규칙 — admin.example.com 은 별도 처리
resource "aws_lb_listener_rule" "admin" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 50

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.admin.arn
  }

  condition {
    host_header {
      values = ["admin.example.com"]
    }
  }
}
```

`ssl_policy`는 TLS 1.3을 지원하는 `ELBSecurityPolicy-TLS13-1-2-2021-06`을 사용한다. 이 정책은 TLS 1.2와 1.3을 모두 허용하면서 취약한 암호화 스위트를 제거한다.

`priority` 값이 낮을수록 먼저 평가된다. 호스트 기반 규칙(50)이 경로 기반 규칙(100)보다 먼저 체크되도록 설정했다.

### AWS CLI로 Target Group 헬스 확인

```bash
# Target 등록 상태 확인
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:ap-northeast-2:123456789012:targetgroup/prod-api-tg/xxxxx \
  --query "TargetHealthDescriptions[*].{Id:Target.Id,Port:Target.Port,State:TargetHealth.State}" \
  --output table
```

---

## 4. Security Group 설계 패턴

### 체이닝 패턴 — SG가 SG를 참조한다

CIDR 범위 대신 Security Group ID를 소스로 지정하면 IP 주소가 바뀌어도 규칙을 수정할 필요가 없다. 이것이 체이닝(Chaining) 패턴의 핵심이다.

```
인터넷 -> ALB SG -> App SG -> DB SG
```

각 계층은 이전 계층의 SG만 허용한다.

```hcl
# 애플리케이션 서버 SG — ALB SG로부터만 인바운드 허용
resource "aws_security_group" "app" {
  name        = "prod-app-sg"
  description = "Application Server Security Group"
  vpc_id      = aws_vpc.main.id

  # ALB에서 오는 트래픽만 허용
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "Allow traffic from ALB"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "prod-app-sg"
  }
}

# DB SG — App SG로부터만 인바운드 허용
resource "aws_security_group" "db" {
  name        = "prod-db-sg"
  description = "Database Security Group"
  vpc_id      = aws_vpc.main.id

  # 애플리케이션 서버에서 오는 트래픽만 허용
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    description     = "Allow PostgreSQL from App servers"
  }

  # 아웃바운드는 최소화 (DB는 외부 호출 불필요)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "prod-db-sg"
  }
}
```

이 패턴의 장점은 명시성이다. DB SG 규칙만 보면 "App 서버에서만 접근 가능"이라는 사실이 바로 보인다.

### 자기 참조(Self-Reference) 패턴

같은 SG에 속한 인스턴스끼리 통신을 허용할 때 사용한다. 클러스터링이나 피어 간 통신이 필요한 경우에 유용하다.

```hcl
resource "aws_security_group" "cluster" {
  name        = "prod-cluster-sg"
  description = "Cluster nodes - allow intra-cluster traffic"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "prod-cluster-sg"
  }
}

# 자기 참조 규칙 — 같은 SG 멤버끼리 모든 포트 허용
resource "aws_security_group_rule" "cluster_self_ingress" {
  type                     = "ingress"
  from_port                = 0
  to_port                  = 65535
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.cluster.id
  security_group_id        = aws_security_group.cluster.id
  description              = "Allow intra-cluster communication"
}
```

`aws_security_group`과 `aws_security_group_rule`을 분리한 이유는 자기 참조 규칙에서 순환 의존성이 생기기 때문이다. Terraform은 SG를 먼저 생성한 뒤 별도 리소스로 규칙을 추가하는 방식으로 이를 해결한다.

### VPC Endpoint SG

Interface Endpoint는 전용 SG를 만들어 프라이빗 서브넷에서만 접근하도록 제한한다.

```hcl
resource "aws_security_group" "endpoint" {
  name        = "prod-endpoint-sg"
  description = "VPC Endpoint Security Group"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.10.0/23", "10.0.12.0/23"]
    description = "Allow HTTPS from private subnets"
  }

  tags = {
    Name = "prod-endpoint-sg"
  }
}
```

### NACL과 Security Group의 역할 분담

Security Group과 NACL(Network ACL)은 모두 네트워크 접근을 제어하지만 작동 방식이 다르다.

| 구분 | Security Group | NACL |
|------|---------------|------|
| 적용 단위 | ENI(인스턴스 수준) | 서브넷 수준 |
| 상태 추적 | Stateful (응답 자동 허용) | Stateless (인바운드/아웃바운드 별도 설정) |
| 규칙 평가 | 모든 규칙 평가 후 허용 | 번호 순서대로 평가, 첫 매칭 적용 |
| 기본 동작 | 모두 차단 | 모두 허용 |

NACL은 서브넷 전체에 적용되는 추가 방어선이다. 일반적으로 Security Group으로 세밀한 접근 제어를 하고, NACL은 서브넷 단위의 명시적 차단에만 사용한다.

예를 들어 특정 IP 대역을 서브넷 수준에서 완전히 차단해야 할 때는 NACL이 적합하다.

```hcl
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = [aws_subnet.public_a.id, aws_subnet.public_c.id]

  # HTTPS 허용
  ingress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  # HTTP 허용 (리다이렉트 용도)
  ingress {
    rule_no    = 110
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  # 임시 포트(Ephemeral Port) 허용 — 응답 트래픽
  ingress {
    rule_no    = 120
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }

  # 알려진 악성 IP 차단 (예시)
  ingress {
    rule_no    = 10
    protocol   = "-1"
    action     = "deny"
    cidr_block = "1.2.3.0/24"
    from_port  = 0
    to_port    = 0
  }

  # 모든 아웃바운드 허용
  egress {
    rule_no    = 100
    protocol   = "-1"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  tags = {
    Name = "prod-public-nacl"
  }
}
```

NACL은 Stateless이므로 임시 포트(1024-65535) 인바운드 규칙을 반드시 추가해야 한다. Security Group은 Stateful이라 요청 흐름을 기억하고 응답을 자동 허용하지만, NACL은 각 패킷을 독립적으로 판단한다. 클라이언트가 80번 포트로 요청하면 서버는 클라이언트의 임시 포트(OS가 무작위로 배정한 1024-65535 범위)로 응답을 보내는데, 이 포트를 인바운드에서 허용하지 않으면 응답 패킷이 NACL에서 차단된다. 이 규칙이 없으면 외부에서 들어온 HTTP 요청의 응답이 클라이언트에게 전달되지 않는다.

### Security Group 설계 시 주의할 점

인바운드 0.0.0.0/0을 모든 포트에 열어두는 것은 프로덕션에서 절대 금지다. 특히 22번 포트(SSH)를 퍼블릭에 개방하는 패턴은 자동화된 브루트포스 공격의 표적이 된다.

SSH 접근이 필요하다면 두 가지 방법 중 하나를 선택한다.

- Bastion Host: 제한된 IP에서만 Bastion에 SSH 허용, Bastion에서 내부 서버로 점프
- SSM Session Manager: SSH 포트 없이 AWS 콘솔/CLI로 EC2 접근 (Interface Endpoint 필요)

SSM Session Manager가 보안상 더 낫다. 키 관리가 불필요하고 모든 세션이 CloudTrail에 기록된다.

```bash
# SSM으로 프라이빗 EC2 접속
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --region ap-northeast-2
```

---

## 마무리

VPC 설계에서 가장 중요한 원칙은 최소 권한이다. 각 리소스가 필요한 통신만 허용하고, 나머지는 모두 차단하는 것이 기본이다. 3-Tier 서브넷 구조는 이 원칙을 네트워크 레이어에서 구조적으로 강제하는 방법이다.

NAT Gateway를 AZ별로 이중화하고, S3/DynamoDB에는 Gateway Endpoint를 사용하면 가용성과 비용 두 가지를 동시에 챙길 수 있다. ALB의 경로 기반 라우팅과 Security Group 체이닝 패턴을 함께 사용하면 애플리케이션 레이어까지 일관된 접근 제어가 가능하다.

다음 편에서는 이 VPC 위에 EC2 Auto Scaling Group을 구성하고, Launch Template과 인스턴스 타입 선택 전략을 다룬다.

---

## 참고 자료

- [AWS VPC 사용 설명서](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/what-is-amazon-vpc.html)
- [VPC Endpoints 개요](https://docs.aws.amazon.com/ko_kr/vpc/latest/privatelink/vpc-endpoints.html)
- [ALB 리스너 규칙](https://docs.aws.amazon.com/ko_kr/elasticloadbalancing/latest/application/listener-update-rules.html)
- [Security Groups for Your VPC](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/VPC_SecurityGroups.html)
- [Terraform AWS Provider — aws_vpc](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)
- [AWS Well-Architected Framework — Security Pillar](https://docs.aws.amazon.com/ko_kr/wellarchitected/latest/security-pillar/welcome.html)
