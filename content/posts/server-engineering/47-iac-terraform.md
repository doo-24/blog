---
title: "[배포와 CI/CD] 7편 — 환경 관리와 IaC: 인프라를 코드로"
date: 2026-03-17T20:01:00+09:00
draft: false
tags: ["IaC", "Terraform", "불변 인프라", "프로비저닝", "서버"]
series: ["배포와 CICD"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 6
summary: "불변 인프라 원칙과 환경 간 드리프트 방지, Terraform 기초·상태 관리·모듈화, 환경변수 vs Secret 관리와 프로비저닝 자동화까지"
---

서버가 수십 대를 넘어가는 순간, "이 서버는 언제 누가 어떻게 만들었지?"라는 질문이 장애 대응 채널에 올라오기 시작한다.

SSH로 접속해 패키지를 손으로 설치하고, `/etc/nginx/nginx.conf`를 직접 수정하며, 그 변경 사항을 메모장 어딘가에 적어두는 방식은 규모가 커질수록 치명적인 기술 부채가 된다.

인프라를 코드로 관리한다는 것—Infrastructure as Code(IaC)—은 단순한 자동화 트릭이 아니다. 서버의 존재 방식 자체를 바꾸는 패러다임 전환이다.

이 글에서는 불변 인프라의 철학부터 Terraform을 활용한 실전 상태 관리와 모듈화, 그리고 환경변수와 시크릿을 안전하게 다루는 방법까지 구체적인 코드와 함께 살펴본다.

---

## 불변 인프라: 변경이 아닌 교체

### 뮤터블 인프라의 문제

전통적인 인프라 관리 방식은 **뮤터블(mutable) 인프라**다. 서버가 살아있는 채로 그 위에 변경을 가한다. 패키지를 업그레이드하고, 설정 파일을 수정하고, 서비스를 재시작한다.

초기에는 단순하고 직관적으로 느껴지지만, 시간이 지날수록 여러 문제가 축적된다.

**드리프트(drift)**가 대표적이다. production 서버 A는 3개월 전에 Node.js 18.12를 설치했고, 서버 B는 2개월 전에 18.16으로 업그레이드했으며, 서버 C는 누군가 실험적으로 20.x를 올렸다.

세 서버가 동일한 애플리케이션을 실행하지만 환경은 제각각이다. 이 상태를 **구성 드리프트(configuration drift)**라고 부른다.

```bash
# 드리프트가 발생한 서버 군의 현실
$ ansible all -m command -a "node --version"
server-a | SUCCESS | rc=0 >>
v18.12.0
server-b | SUCCESS | rc=0 >>
v18.16.1
server-c | SUCCESS | rc=0 >>
v20.11.0
server-d | FAILED | rc=127 >>
node: command not found
```

이런 상황에서 장애가 발생하면 원인 추적이 극도로 어려워진다.

"server-c에서만 재현되는 버그"는 사실 Node.js 버전 차이에서 비롯된 것일 수 있지만, 그 사실을 파악하는 데 수 시간이 걸린다.

### 불변 인프라의 철학

**불변 인프라(immutable infrastructure)**는 서버를 절대 수정하지 않는다는 원칙이다. 변경이 필요하면 새 서버를 만들고 트래픽을 전환한 뒤 구 서버를 폐기한다.

서버는 일회용 가축(cattle)이지 애지중지 키우는 반려동물(pet)이 아니다.

이 접근법의 핵심 이점은 세 가지다.

1. **일관성**: 모든 서버는 동일한 이미지와 동일한 프로비저닝 스크립트로 생성된다. 드리프트가 구조적으로 불가능하다.
2. **재현 가능성**: 장애 서버를 수리하는 대신 교체한다. 동일한 코드로 동일한 서버를 언제든 재생성할 수 있다.
3. **롤백 단순화**: 이전 버전의 이미지로 교체하면 된다. 복잡한 언인스톨 절차가 없다.

```
뮤터블 인프라:
서버 → 패치 → 패치 → 패치 → ... (상태가 누적됨)

불변 인프라:
이미지 v1 → 서버 생성 → 폐기
이미지 v2 → 서버 생성 → 폐기
이미지 v3 → 서버 생성 → (현재 실행 중)
```

### 환경 간 드리프트 방지

개발(dev), 스테이징(staging), 프로덕션(production) 환경 사이의 드리프트도 심각한 문제다.

"내 로컬에서는 되는데"라는 말이 나오는 대부분의 이유는 환경 불일치다.

드리프트를 방지하려면 모든 환경을 **동일한 코드베이스**에서 생성해야 한다. 환경 간 차이는 변수(variable)로만 표현한다.

```hcl
# 잘못된 접근: 환경별로 별개의 코드를 유지
# prod/main.tf, staging/main.tf, dev/main.tf — 세 파일이 조금씩 달라지기 시작함

# 올바른 접근: 하나의 코드, 환경별 변수 파일
# modules/app-server/main.tf  ← 공통 코드
# environments/prod/terraform.tfvars
# environments/staging/terraform.tfvars
# environments/dev/terraform.tfvars
```

---

## Terraform 기초

### 왜 Terraform인가

IaC 도구는 여러 가지가 있다. AWS CloudFormation은 AWS 전용이고, Pulumi는 범용 프로그래밍 언어를 사용한다. Ansible은 설정 관리에 더 가깝다.

Terraform은 멀티 클라우드를 지원하면서도 선언적(declarative) 문법으로 인프라의 **원하는 상태(desired state)**를 기술한다는 점에서 가장 널리 채택된 IaC 도구다.

선언적이라는 것은 "이런 상태여야 한다"고 기술하면 Terraform이 현재 상태와 비교해 필요한 변경만 수행한다는 의미다.

"EC2 인스턴스를 만들어라"가 아니라 "EC2 인스턴스가 존재해야 한다"고 기술한다.

### HCL 기본 문법

Terraform은 HashiCorp Configuration Language(HCL)를 사용한다.

```hcl
# main.tf

# 프로바이더 설정: Terraform이 어떤 클라우드와 통신할지
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# 변수 선언
variable "aws_region" {
  description = "AWS 리전"
  type        = string
  default     = "ap-northeast-2"
}

variable "instance_type" {
  description = "EC2 인스턴스 타입"
  type        = string
}

variable "environment" {
  description = "배포 환경 (dev/staging/prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment는 dev, staging, prod 중 하나여야 합니다."
  }
}

# 데이터 소스: 기존 리소스를 참조
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

# 리소스 선언
resource "aws_instance" "app_server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  tags = {
    Name        = "app-server-${var.environment}"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# 출력값
output "instance_public_ip" {
  description = "EC2 인스턴스 퍼블릭 IP"
  value       = aws_instance.app_server.public_ip
}
```

```hcl
# terraform.tfvars (개발 환경)
aws_region    = "ap-northeast-2"
instance_type = "t3.micro"
environment   = "dev"
```

### 기본 워크플로

```bash
# 1. 초기화: 프로바이더 플러그인 다운로드
terraform init

# 2. 계획: 어떤 변경이 일어날지 미리 확인
terraform plan

# 3. 적용: 실제로 인프라 변경 수행
terraform apply

# 4. 상태 확인
terraform show

# 5. 삭제 (주의: 실제 리소스가 삭제됨)
terraform destroy
```

`terraform plan` 출력을 읽는 법을 익히는 것이 중요하다.

```
Terraform will perform the following actions:

  # aws_instance.app_server will be created
  + resource "aws_instance" "app_server" {
      + ami                          = "ami-0c9c942bd7bf113a2"
      + instance_type                = "t3.micro"
      + tags                         = {
          + "Environment" = "dev"
          + "ManagedBy"   = "terraform"
          + "Name"        = "app-server-dev"
        }
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

`+`는 생성, `~`는 수정, `-`는 삭제를 의미한다. `-/+`는 리소스를 삭제하고 새로 만드는 교체(replacement)다.

교체는 다운타임을 유발할 수 있으므로 특히 주의해야 한다.

---

## 상태 관리

### Terraform 상태 파일이란

Terraform은 관리하는 인프라의 현재 상태를 `terraform.tfstate` 파일에 JSON 형식으로 저장한다.

이 파일은 Terraform이 "현실"을 이해하는 단일 진실 공급원(single source of truth)이다.

```json
{
  "version": 4,
  "terraform_version": "1.6.4",
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "app_server",
      "instances": [
        {
          "attributes": {
            "id": "i-0a1b2c3d4e5f67890",
            "ami": "ami-0c9c942bd7bf113a2",
            "instance_type": "t3.micro",
            "public_ip": "13.124.x.x"
          }
        }
      ]
    }
  ]
}
```

### 원격 백엔드: 팀 협업의 필수 조건

`terraform.tfstate`를 로컬에만 두면 두 가지 문제가 발생한다. 첫째, 다른 팀원이 동시에 `terraform apply`를 실행하면 상태가 충돌한다. 둘째, 상태 파일을 잃어버리면 Terraform이 어떤 리소스를 관리하는지 모르게 된다.

해결책은 **원격 백엔드(remote backend)**다.

S3 + DynamoDB 조합이 AWS 환경에서 표준적으로 사용된다.

```hcl
# backend.tf

terraform {
  backend "s3" {
    bucket         = "my-terraform-state-prod"
    key            = "services/app-server/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true

    # DynamoDB 테이블로 동시 실행 잠금
    dynamodb_table = "terraform-state-lock"
  }
}
```

```bash
# 백엔드용 S3 버킷 생성 (버저닝 활성화)
aws s3api create-bucket \
  --bucket my-terraform-state-prod \
  --region ap-northeast-2 \
  --create-bucket-configuration LocationConstraint=ap-northeast-2

aws s3api put-bucket-versioning \
  --bucket my-terraform-state-prod \
  --versioning-configuration Status=Enabled

# 잠금용 DynamoDB 테이블 생성
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-northeast-2
```

두 명이 동시에 `terraform apply`를 실행하면 DynamoDB가 잠금을 제공해 두 번째 실행이 대기하거나 실패한다.

```
│ Error: Error acquiring the state lock
│
│ Error message: ConditionalCheckFailedException: The conditional request failed
│
│ Terraform acquires a state lock to protect the state from being written
│ by multiple users at the same time.
```

### 상태 조작 명령어

상태 파일을 직접 수정해야 하는 상황이 생긴다.

예를 들어 수동으로 만든 리소스를 Terraform 관리 하에 들여오거나, Terraform 코드 내에서 리소스를 이름 변경했을 때다.

```bash
# 현재 상태의 모든 리소스 목록
terraform state list

# 특정 리소스의 상태 상세 조회
terraform state show aws_instance.app_server

# 기존 리소스를 Terraform 상태로 가져오기
# (수동 생성된 리소스를 IaC 관리 하에 편입)
terraform import aws_instance.app_server i-0a1b2c3d4e5f67890

# 리소스를 상태에서 제거 (실제 리소스는 삭제되지 않음)
terraform state rm aws_instance.old_server

# 리소스 이름 변경 (코드에서 이름을 바꿨을 때)
terraform state mv aws_instance.app_server aws_instance.web_server
```

### 안티패턴: 상태 파일을 Git에 커밋하기

```bash
# 절대 하지 말 것
git add terraform.tfstate
git commit -m "상태 파일 추가"
```

`terraform.tfstate`에는 데이터베이스 비밀번호, API 키 등 민감한 정보가 평문으로 포함될 수 있다.

항상 `.gitignore`에 추가하고 원격 백엔드를 사용해야 한다.

```gitignore
# .gitignore
*.tfstate
*.tfstate.backup
.terraform/
.terraform.lock.hcl  # 이것은 커밋해야 함 — 주의
```

`.terraform.lock.hcl`은 프로바이더 버전을 고정하므로 Git에 커밋해야 한다.

`terraform.tfstate`와 혼동하지 말자.

---

## 모듈화

### 모듈의 개념

Terraform 모듈은 재사용 가능한 인프라 컴포넌트다. 함수처럼 입력(변수)을 받아 출력(output)을 제공한다.

잘 설계된 모듈은 한 번 작성하고 여러 환경, 여러 팀에서 재사용한다.

```
project/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── rds/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── ecs-service/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    ├── dev/
    │   └── main.tf
    ├── staging/
    │   └── main.tf
    └── prod/
        └── main.tf
```

### 모듈 작성 예시: VPC

```hcl
# modules/vpc/variables.tf
variable "environment" {
  type        = string
  description = "배포 환경"
}

variable "vpc_cidr" {
  type        = string
  description = "VPC CIDR 블록"
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  type        = list(string)
  description = "사용할 가용 영역 목록"
}

variable "public_subnet_cidrs" {
  type        = list(string)
  description = "퍼블릭 서브넷 CIDR 목록"
}

variable "private_subnet_cidrs" {
  type        = list(string)
  description = "프라이빗 서브넷 CIDR 목록"
}
```

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "vpc-${var.environment}"
    Environment = var.environment
  }
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name        = "public-subnet-${var.environment}-${count.index + 1}"
    Environment = var.environment
    Tier        = "public"
  }
}

resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name        = "private-subnet-${var.environment}-${count.index + 1}"
    Environment = var.environment
    Tier        = "private"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = "igw-${var.environment}"
    Environment = var.environment
  }
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  description = "생성된 VPC ID"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "퍼블릭 서브넷 ID 목록"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "프라이빗 서브넷 ID 목록"
  value       = aws_subnet.private[*].id
}
```

### 모듈 사용: 환경별 조합

```hcl
# environments/prod/main.tf

terraform {
  required_version = ">= 1.6.0"
  backend "s3" {
    bucket         = "my-terraform-state-prod"
    key            = "environments/prod/terraform.tfstate"
    region         = "ap-northeast-2"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = "ap-northeast-2"
}

module "vpc" {
  source = "../../modules/vpc"

  environment          = "prod"
  vpc_cidr             = "10.0.0.0/16"
  availability_zones   = ["ap-northeast-2a", "ap-northeast-2c"]
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24"]
}

module "database" {
  source = "../../modules/rds"

  environment        = "prod"
  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.private_subnet_ids
  instance_class     = "db.r6g.large"
  multi_az           = true
  deletion_protection = true
}
```

```hcl
# environments/dev/main.tf — 동일한 모듈, 다른 변수값
module "vpc" {
  source = "../../modules/vpc"

  environment          = "dev"
  vpc_cidr             = "10.1.0.0/16"
  availability_zones   = ["ap-northeast-2a"]  # 단일 AZ로 비용 절감
  public_subnet_cidrs  = ["10.1.1.0/24"]
  private_subnet_cidrs = ["10.1.11.0/24"]
}

module "database" {
  source = "../../modules/rds"

  environment        = "dev"
  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.private_subnet_ids
  instance_class     = "db.t3.micro"  # 저렴한 인스턴스
  multi_az           = false
  deletion_protection = false
}
```

개발 환경과 프로덕션 환경이 완전히 동일한 코드 구조를 공유하면서도 비용, 가용성, 보호 수준을 다르게 설정할 수 있다.

---

## 환경변수 vs Secret 관리

### 세 가지 범주의 설정값

애플리케이션 설정값은 민감도에 따라 세 범주로 나뉜다.

| 범주 | 예시 | 저장 위치 |
|------|------|-----------|
| 일반 설정 | 서버 포트, 타임존, 로그 레벨 | 환경변수, 설정 파일 |
| 환경별 설정 | DB 호스트, API 엔드포인트 | 환경변수, Terraform 변수 |
| 시크릿 | DB 비밀번호, API 키, JWT 시크릿 | 전용 시크릿 관리 서비스 |

### 환경변수 관리

Terraform으로 환경변수를 ECS Task Definition이나 Lambda 환경에 주입하는 패턴이다.

```hcl
# modules/ecs-service/main.tf

resource "aws_ecs_task_definition" "app" {
  family                   = "${var.service_name}-${var.environment}"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.cpu
  memory                   = var.memory
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = var.service_name
      image = var.container_image

      # 민감하지 않은 환경변수는 직접 주입
      environment = [
        {
          name  = "NODE_ENV"
          value = var.environment == "prod" ? "production" : "development"
        },
        {
          name  = "PORT"
          value = tostring(var.container_port)
        },
        {
          name  = "LOG_LEVEL"
          value = var.environment == "prod" ? "warn" : "debug"
        },
        {
          name  = "DB_HOST"
          value = var.db_host
        }
      ]

      # 민감한 값은 Secrets Manager에서 주입
      secrets = [
        {
          name      = "DB_PASSWORD"
          valueFrom = "arn:aws:secretsmanager:ap-northeast-2:123456789:secret:prod/app/db-password"
        },
        {
          name      = "JWT_SECRET"
          valueFrom = "arn:aws:secretsmanager:ap-northeast-2:123456789:secret:prod/app/jwt-secret"
        }
      ]

      portMappings = [
        {
          containerPort = var.container_port
          protocol      = "tcp"
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/${var.service_name}-${var.environment}"
          "awslogs-region"        = "ap-northeast-2"
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ])
}
```

### AWS Secrets Manager 연동

```hcl
# 시크릿 생성 (초기값은 더미, 실제 값은 별도로 설정)
resource "aws_secretsmanager_secret" "db_password" {
  name        = "${var.environment}/app/db-password"
  description = "애플리케이션 DB 비밀번호"

  # 삭제 전 복구 대기 기간 (prod: 30일, 그 외: 즉시 삭제)
  recovery_window_in_days = var.environment == "prod" ? 30 : 0

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ECS 태스크 롤에 Secrets Manager 접근 권한 부여
resource "aws_iam_role_policy" "ecs_secrets" {
  name = "ecs-secrets-access"
  role = aws_iam_role.ecs_task.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = [
          aws_secretsmanager_secret.db_password.arn,
          aws_secretsmanager_secret.jwt_secret.arn
        ]
      }
    ]
  })
}
```

실제 시크릿 값은 Terraform이 아닌 별도 프로세스로 설정한다.

```bash
# CI/CD 파이프라인이나 초기 설정 스크립트에서 실행
aws secretsmanager put-secret-value \
  --secret-id "prod/app/db-password" \
  --secret-string "$(openssl rand -base64 32)"

# 또는 AWS 콘솔에서 수동 설정 (최초 1회)
```

### Terraform 변수에서 시크릿 다루기

```hcl
# 시크릿을 Terraform 변수로 받아야 하는 경우
variable "db_master_password" {
  description = "RDS 마스터 비밀번호"
  type        = string
  sensitive   = true  # plan/apply 출력에서 마스킹됨
}
```

```bash
# 환경변수로 전달 (CI/CD에서 안전하게 주입)
export TF_VAR_db_master_password="$(aws secretsmanager get-secret-value \
  --secret-id prod/rds/master-password \
  --query SecretString \
  --output text)"

terraform apply
```

`sensitive = true`로 선언된 변수는 `terraform plan` 출력에서 `(sensitive value)`로 마스킹된다.

### 안티패턴: 하드코딩과 `.tfvars` 커밋

```hcl
# 절대 하지 말 것 1: 코드에 시크릿 하드코딩
resource "aws_db_instance" "main" {
  password = "super_secret_password_123"  # 위험!
}

# 절대 하지 말 것 2: 시크릿이 담긴 .tfvars를 Git에 커밋
# secrets.tfvars에 db_master_password = "..." 를 기록하고 커밋
```

```gitignore
# .gitignore에 반드시 추가
*.tfvars
!example.tfvars  # 예시 파일은 커밋 가능
```

---

## 프로비저닝 자동화

### User Data로 초기 설정 자동화

EC2 인스턴스가 처음 시작될 때 실행되는 User Data 스크립트로 기본 설정을 자동화한다.

```hcl
resource "aws_launch_template" "app" {
  name_prefix   = "app-server-${var.environment}-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  user_data = base64encode(templatefile("${path.module}/scripts/user-data.sh", {
    environment    = var.environment
    app_version    = var.app_version
    s3_bucket      = var.artifact_bucket
    log_group_name = "/ec2/app-server-${var.environment}"
  }))

  iam_instance_profile {
    name = aws_iam_instance_profile.app.name
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "app-server-${var.environment}"
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

```bash
#!/bin/bash
# scripts/user-data.sh

set -euo pipefail

ENVIRONMENT="${environment}"
APP_VERSION="${app_version}"
S3_BUCKET="${s3_bucket}"

# CloudWatch Agent 설치
yum install -y amazon-cloudwatch-agent

# 애플리케이션 아티팩트 다운로드
aws s3 cp "s3://$${S3_BUCKET}/releases/$${APP_VERSION}/app.tar.gz" /tmp/app.tar.gz
tar -xzf /tmp/app.tar.gz -C /opt/app

# 시스템 서비스 등록
cat > /etc/systemd/system/app.service << 'SYSTEMD'
[Unit]
Description=Application Server
After=network.target

[Service]
Type=simple
User=app
WorkingDirectory=/opt/app
ExecStart=/opt/app/bin/server
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
SYSTEMD

systemctl daemon-reload
systemctl enable app
systemctl start app

echo "프로비저닝 완료: $${ENVIRONMENT} 환경, 버전 $${APP_VERSION}"
```

### Auto Scaling Group으로 가용성 확보

```hcl
resource "aws_autoscaling_group" "app" {
  name                = "asg-app-${var.environment}"
  vpc_zone_identifier = var.private_subnet_ids
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"

  min_size         = var.environment == "prod" ? 2 : 1
  max_size         = var.environment == "prod" ? 10 : 3
  desired_capacity = var.environment == "prod" ? 2 : 1

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  # 인스턴스 교체 시 새 인스턴스가 먼저 올라온 후 구 인스턴스 제거
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
    }
  }

  tag {
    key                 = "Name"
    value               = "app-server-${var.environment}"
    propagate_at_launch = true
  }
}

# CPU 기반 자동 스케일링
resource "aws_autoscaling_policy" "scale_out" {
  name                   = "scale-out-${var.environment}"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 70.0
  }
}
```

### CI/CD 파이프라인과 Terraform 연동

```yaml
# .github/workflows/terraform.yml

name: Terraform

on:
  push:
    branches: [main]
    paths: ['infrastructure/**']
  pull_request:
    paths: ['infrastructure/**']

permissions:
  id-token: write  # OIDC 인증용
  contents: read
  pull-requests: write

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: AWS 인증 (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-terraform
          aws-region: ap-northeast-2

      - name: Terraform 설치
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.4"

      - name: Terraform 초기화
        working-directory: infrastructure/environments/prod
        run: terraform init

      - name: Terraform 형식 검사
        run: terraform fmt -check -recursive

      - name: Terraform 유효성 검사
        working-directory: infrastructure/environments/prod
        run: terraform validate

      - name: Terraform Plan
        working-directory: infrastructure/environments/prod
        run: terraform plan -out=tfplan
        env:
          TF_VAR_environment: prod

      - name: PR에 Plan 결과 게시
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 결과
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\``;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production  # GitHub 환경 보호 규칙 적용
    steps:
      - uses: actions/checkout@v4
      # ... (위와 동일한 초기화 단계)

      - name: Terraform Apply
        working-directory: infrastructure/environments/prod
        run: terraform apply -auto-approve
```

---

## 실전 운영 팁

### `terraform plan`을 PR 리뷰 필수 단계로

인프라 변경은 애플리케이션 코드 변경보다 영향 범위가 크다.

`terraform plan` 출력을 PR 코멘트에 자동으로 첨부하고, 팀원이 리뷰한 후 머지하는 프로세스를 필수화해야 한다.

### `prevent_destroy`로 데이터 손실 방지

```hcl
resource "aws_db_instance" "main" {
  # ...

  lifecycle {
    prevent_destroy = true  # terraform destroy 명령으로 삭제 불가
  }
}
```

데이터베이스, S3 버킷, 중요한 보안 그룹 등 실수로 삭제하면 치명적인 리소스에는 `prevent_destroy`를 설정한다.

### Helm Chart와 K8s 패키지 관리

쿠버네티스 환경에서의 애플리케이션 패키징과 Helm Chart 관리는 이 시리즈의 별도 챕터([11편] Docker & 컨테이너)에서 다룬다.

Terraform은 클러스터 자체(EKS, GKE)와 네트워크 인프라를 관리하고, 그 위에서 동작하는 애플리케이션 워크로드는 Helm이 관리하는 구조가 일반적이다.

### 상태 파일 백업과 복구

```bash
# 상태 파일 수동 백업
terraform state pull > backup-$(date +%Y%m%d-%H%M%S).tfstate

# 손상된 상태 복구
terraform state push backup-20240315-143022.tfstate

# S3 버저닝을 통한 이전 상태 복원
aws s3api list-object-versions \
  --bucket my-terraform-state-prod \
  --prefix environments/prod/terraform.tfstate \
  --query 'Versions[*].{Key:Key,VersionId:VersionId,LastModified:LastModified}'
```

---

## 요약

불변 인프라는 서버를 교체 가능한 단위로 만들어 드리프트 문제를 구조적으로 해결한다.

Terraform은 HCL로 인프라의 원하는 상태를 선언하고, 상태 파일로 현실과의 차이를 추적하며, 모듈로 재사용 가능한 컴포넌트를 만든다.

시크릿은 코드가 아닌 전용 관리 서비스(Secrets Manager, Vault)에 보관하고, CI/CD 파이프라인에서 `plan → review → apply` 흐름을 자동화하면 인프라 변경이 애플리케이션 배포만큼 안전하고 예측 가능해진다.

핵심은 하나다. **인프라도 코드다. 코드처럼 리뷰하고, 버전 관리하고, 테스트하라.**

---

## 참고 자료

- [Terraform 공식 문서](https://developer.hashicorp.com/terraform/docs) — HCL 문법, 프로바이더 레퍼런스, 모범 사례
- [AWS Provider for Terraform](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) — AWS 리소스 전체 레퍼런스
- [Terraform: Up & Running, 3rd Edition](https://www.terraformupandrunning.com/) — Yevgeniy Brikman 저, IaC 실전 교과서
- [Immutable Infrastructure by HashiCorp](https://www.hashicorp.com/resources/what-is-mutable-vs-immutable-infrastructure) — 뮤터블 vs 불변 인프라 개념 정리
- [AWS Secrets Manager 모범 사례](https://docs.aws.amazon.com/secretsmanager/latest/userguide/best-practices.html) — 시크릿 로테이션과 접근 제어
- [Atlantis](https://www.runatlantis.io/) — PR 기반 Terraform 워크플로 자동화 오픈소스
