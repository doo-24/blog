---
title: "[AWS 실전 구축] 8편 — 실전 아키텍처 조합: 서비스 전체를 AWS 위에 올리기"
date: 2026-03-17T11:01:00+09:00
draft: false
tags: ["아키텍처", "3-Tier", "서버리스", "Terraform", "AWS", "서버"]
series: ["AWS 실전 구축"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "3-Tier 웹 서비스 레퍼런스 아키텍처(CloudFront+ALB+ECS+RDS), 서버리스 API 아키텍처(API Gateway+Lambda+DynamoDB+Cognito), 환경 분리 전략과 Terraform 모듈 구성, 비용 추정 실전까지"
---

지금까지 VPC, EC2, RDS, ECS, Lambda, API Gateway, CloudFront 등 AWS의 핵심 서비스를 하나씩 살펴봤다. 이번 편에서는 그 모든 조각을 하나로 합친다. 실제 서비스를 AWS 위에 올릴 때 어떤 아키텍처를 선택하고, 어떻게 구성하며, 비용은 얼마나 나오는지를 처음부터 끝까지 정리한다.

---

## 1. 3-Tier 웹 서비스 레퍼런스 아키텍처

### 전체 구성도

전통적인 웹 서비스는 프레젠테이션, 애플리케이션, 데이터 계층으로 나뉜다. AWS에서는 각 계층을 다음 서비스로 매핑한다.

```
사용자 브라우저
     |
[CloudFront] ← S3 (정적 파일: JS/CSS/이미지)
     |
[ALB (Application Load Balancer)]
     |
[ECS Fargate] × N 컨테이너 (앱 서버)
     |         \
[RDS Aurora]  [ElastiCache Redis]
(쓰기 엔드포인트) (세션/캐시)
     |
[RDS Aurora Read Replica] (읽기 엔드포인트)
```

각 계층은 독립된 서브넷에 위치하고, 보안 그룹으로 통신을 제한한다.

### 네트워크 레이아웃

VPC를 3개 계층으로 나눈다.

```
VPC (10.0.0.0/16)
├── Public Subnet (10.0.1.0/24, 10.0.2.0/24)  ← ALB, NAT Gateway
├── Private Subnet (10.0.11.0/24, 10.0.12.0/24) ← ECS 컨테이너
└── Data Subnet (10.0.21.0/24, 10.0.22.0/24)   ← RDS, ElastiCache
```

Public Subnet에는 인터넷 게이트웨이를 통한 인바운드가 허용된다. Private Subnet의 ECS는 NAT Gateway를 통해서만 아웃바운드가 가능하다. Data Subnet은 Private Subnet에서 오는 트래픽만 수신한다.

### CloudFront 설정 포인트

CloudFront는 두 가지 Origin을 갖는다.

첫째, `/api/*` 경로는 ALB로 포워딩한다. 둘째, 그 외 모든 경로는 S3 버킷(정적 빌드 결과물)을 Origin으로 삼는다. 이렇게 하면 React/Vue SPA를 S3에 배포하고, API 요청만 ALB로 흘릴 수 있다.

```hcl
# CloudFront origins 설정 (Terraform)
ordered_cache_behavior {
  path_pattern     = "/api/*"
  target_origin_id = "alb-origin"

  forwarded_values {
    query_string = true
    headers      = ["Authorization", "Content-Type"]
    cookies { forward = "none" }
  }

  viewer_protocol_policy = "redirect-to-https"
  cache_policy_id        = aws_cloudfront_cache_policy.no_cache.id
}

default_cache_behavior {
  target_origin_id       = "s3-origin"
  viewer_protocol_policy = "redirect-to-https"
  cache_policy_id        = data.aws_cloudfront_cache_policy.caching_optimized.id
}
```

ALB Origin에는 `Cache-Control: no-store`를 강제해 API 응답이 CloudFront에 캐시되지 않도록 주의한다.

### ECS Fargate 서비스 구성

ECS는 태스크 정의와 서비스 두 부분으로 구성된다.

```hcl
resource "aws_ecs_task_definition" "app" {
  family                   = "my-app"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 512
  memory                   = 1024

  container_definitions = jsonencode([{
    name  = "app"
    image = "${aws_ecr_repository.app.repository_url}:latest"
    portMappings = [{ containerPort = 8080 }]

    environment = [
      { name = "DB_HOST",    value = aws_rds_cluster.main.endpoint },
      { name = "REDIS_HOST", value = aws_elasticache_replication_group.main.primary_endpoint_address }
    ]

    secrets = [
      { name = "DB_PASSWORD", valueFrom = aws_ssm_parameter.db_password.arn }
    ]

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = "/ecs/my-app"
        awslogs-region        = "ap-northeast-2"
        awslogs-stream-prefix = "ecs"
      }
    }
  }])
}

resource "aws_ecs_service" "app" {
  name            = "my-app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 8080
  }
}
```

`desired_count = 2`는 최소 가용성 보장을 위한 기본값이다. Auto Scaling은 별도 `aws_appautoscaling_target`으로 정의한다.

### RDS Aurora + ElastiCache

Aurora MySQL은 Writer와 Reader 엔드포인트를 자동으로 제공한다.

```hcl
resource "aws_rds_cluster" "main" {
  cluster_identifier     = "my-app-cluster"
  engine                 = "aurora-mysql"
  engine_version         = "8.0.mysql_aurora.3.04.0"
  database_name          = "myapp"
  master_username        = "admin"
  manage_master_user_password = true  # Secrets Manager 자동 연동
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  backup_retention_period = 7
  deletion_protection     = true
}

resource "aws_rds_cluster_instance" "writer" {
  identifier         = "my-app-writer"
  cluster_identifier = aws_rds_cluster.main.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.main.engine
}

resource "aws_rds_cluster_instance" "reader" {
  identifier         = "my-app-reader"
  cluster_identifier = aws_rds_cluster.main.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.main.engine
}
```

`manage_master_user_password = true`를 사용하면 비밀번호가 Secrets Manager에 자동 저장되고, 90일마다 자동 교체된다.

ElastiCache Redis는 세션 저장과 API 응답 캐싱에 사용한다.

```hcl
resource "aws_elasticache_replication_group" "main" {
  replication_group_id       = "my-app-redis"
  description                = "Session and cache store"
  node_type                  = "cache.r6g.large"
  num_cache_clusters         = 2
  automatic_failover_enabled = true
  subnet_group_name          = aws_elasticache_subnet_group.main.name
  security_group_ids         = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
}
```

`num_cache_clusters = 2`로 Primary + Replica 구성을 만들고, `automatic_failover_enabled`로 장애 자동 복구를 활성화한다.

---

## 2. 서버리스 API 아키텍처

### 전체 구성도

서버를 관리하지 않고 API를 운영하는 구성이다.

```
클라이언트 (모바일/브라우저)
     |
[CloudFront]
     |
[API Gateway (HTTP API)]
     |           \
[Cognito]    [Lambda Functions]
(인증/인가)       |          \
            [DynamoDB]   [S3 (파일 업로드)]
            (기본 DB)
                |
           [DynamoDB Streams]
                |
           [Lambda (이벤트 처리)]
                |
           [SQS / SNS / EventBridge]
```

서버리스 아키텍처는 트래픽이 불규칙하거나, 초기 스타트업이 서버 운영 부담을 줄이고 싶을 때 적합하다.

### API Gateway + Lambda 기본 구성

HTTP API는 REST API보다 저렴하고 지연시간이 낮다.

```hcl
resource "aws_apigatewayv2_api" "main" {
  name          = "my-serverless-api"
  protocol_type = "HTTP"

  cors_configuration {
    allow_origins = ["https://myapp.com"]
    allow_methods = ["GET", "POST", "PUT", "DELETE"]
    allow_headers = ["Authorization", "Content-Type"]
    max_age       = 300
  }
}

resource "aws_apigatewayv2_stage" "prod" {
  api_id      = aws_apigatewayv2_api.main.id
  name        = "prod"
  auto_deploy = true

  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_gw.arn
  }
}

resource "aws_apigatewayv2_integration" "users" {
  api_id             = aws_apigatewayv2_api.main.id
  integration_type   = "AWS_PROXY"
  integration_uri    = aws_lambda_function.users.invoke_arn
  payload_format_version = "2.0"
}

resource "aws_apigatewayv2_route" "get_users" {
  api_id    = aws_apigatewayv2_api.main.id
  route_key = "GET /users"
  target    = "integrations/${aws_apigatewayv2_integration.users.id}"

  authorization_type = "JWT"
  authorizer_id      = aws_apigatewayv2_authorizer.cognito.id
}
```

`payload_format_version = "2.0"`을 사용하면 이벤트 구조가 단순해지고 Lambda 응답 형식도 간결해진다.

### Cognito 인증 연동

Cognito User Pool은 사용자 관리와 JWT 발급을 담당한다.

```hcl
resource "aws_cognito_user_pool" "main" {
  name = "my-app-users"

  password_policy {
    minimum_length    = 8
    require_uppercase = true
    require_numbers   = true
  }

  auto_verified_attributes = ["email"]

  account_recovery_setting {
    recovery_mechanism {
      name     = "verified_email"
      priority = 1
    }
  }
}

resource "aws_cognito_user_pool_client" "web" {
  name         = "web-client"
  user_pool_id = aws_cognito_user_pool.main.id

  explicit_auth_flows = [
    "ALLOW_USER_SRP_AUTH",
    "ALLOW_REFRESH_TOKEN_AUTH"
  ]

  access_token_validity  = 1   # 1시간
  refresh_token_validity = 30  # 30일
  token_validity_units {
    access_token  = "hours"
    refresh_token = "days"
  }
}

resource "aws_apigatewayv2_authorizer" "cognito" {
  api_id           = aws_apigatewayv2_api.main.id
  authorizer_type  = "JWT"
  identity_sources = ["$request.header.Authorization"]
  name             = "cognito-authorizer"

  jwt_configuration {
    audience = [aws_cognito_user_pool_client.web.id]
    issuer   = "https://cognito-idp.ap-northeast-2.amazonaws.com/${aws_cognito_user_pool.main.id}"
  }
}
```

Lambda 함수에서는 별도 인증 로직 없이 `event.requestContext.authorizer.jwt.claims`에서 사용자 정보를 꺼낼 수 있다.

### DynamoDB 테이블 설계

서버리스에서 DynamoDB는 자연스러운 선택이다. 하지만 스키마 설계가 핵심이다.

```hcl
resource "aws_dynamodb_table" "users" {
  name         = "Users"
  billing_mode = "PAY_PER_REQUEST"  # On-Demand
  hash_key     = "PK"
  range_key    = "SK"

  attribute {
    name = "PK"
    type = "S"
  }
  attribute {
    name = "SK"
    type = "S"
  }
  attribute {
    name = "GSI1PK"
    type = "S"
  }
  attribute {
    name = "GSI1SK"
    type = "S"
  }

  global_secondary_index {
    name            = "GSI1"
    hash_key        = "GSI1PK"
    range_key       = "GSI1SK"
    projection_type = "ALL"
  }

  point_in_time_recovery { enabled = true }
  server_side_encryption { enabled = true }

  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"
}
```

Single Table Design을 기반으로 PK/SK에 `USER#<id>`, `POST#<date>` 형식을 사용하면 하나의 테이블로 여러 엔티티를 관리할 수 있다.

### Lambda 함수 예시 (Node.js)

```javascript
// src/handlers/getUser.js
const { DynamoDBClient } = require("@aws-sdk/client-dynamodb");
const { DynamoDBDocumentClient, GetCommand } = require("@aws-sdk/lib-dynamodb");

const client = DynamoDBDocumentClient.from(new DynamoDBClient({}));

exports.handler = async (event) => {
  const userId = event.pathParameters.userId;
  const claims = event.requestContext.authorizer.jwt.claims;

  // 본인 정보만 조회 가능
  if (claims.sub !== userId) {
    return { statusCode: 403, body: JSON.stringify({ error: "Forbidden" }) };
  }

  const result = await client.send(new GetCommand({
    TableName: process.env.TABLE_NAME,
    Key: { PK: `USER#${userId}`, SK: "PROFILE" }
  }));

  if (!result.Item) {
    return { statusCode: 404, body: JSON.stringify({ error: "Not found" }) };
  }

  return {
    statusCode: 200,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(result.Item)
  };
};
```

DynamoDB 클라이언트는 Lambda 초기화 단계(핸들러 바깥)에서 한 번만 생성한다. 이렇게 하면 Warm Start 시 재사용되어 응답 시간이 빨라진다.

---

## 3. 환경 분리 전략과 Terraform 모듈 구성

### 멀티 계정 vs 단일 계정 멀티 VPC

두 방식의 차이를 정리하면 다음과 같다.

| 항목 | 멀티 계정 | 단일 계정 멀티 VPC |
|------|---------|-----------------|
| 격리 수준 | 완전 격리 (IAM, 네트워크, 비용) | 논리적 격리 (VPC 분리) |
| 비용 가시성 | 계정별 청구서 | 태그 기반 분류 필요 |
| 복잡도 | AWS Organizations 필요 | 상대적으로 단순 |
| 권장 규모 | 팀 5명 이상, 규정 준수 필요 | 소규모 스타트업 |

소규모 팀이라면 단일 계정에 `dev`, `staging`, `prod` VPC를 분리하는 방식으로 시작하는 것이 현실적이다.

```text
[멀티 계정 구조]                    [단일 계정 멀티 VPC]
AWS Organizations                   AWS Account
  ├── Prod Account                    ├── prod-vpc  (10.0.0.0/16)
  │     └── prod-vpc                  ├── staging-vpc (10.1.0.0/16)
  ├── Staging Account                 └── dev-vpc  (10.2.0.0/16)
  │     └── staging-vpc
  └── Dev Account                  VPC Peering으로 필요 시 연결
        └── dev-vpc                (단, 완전한 IAM 격리 불가)

완전 격리, 비용 추적 명확          구성 단순, 소규모 팀에 적합
```

### Terraform 디렉토리 구조

```
infrastructure/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── ecs-service/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── rds-aurora/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── cloudfront-alb/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       └── terraform.tfvars
└── backend.tf
```

`modules/` 아래에 재사용 가능한 인프라 단위를 정의하고, `environments/` 아래에서 모듈을 조합한다.

### 모듈 예시: vpc

```hcl
# modules/vpc/variables.tf
variable "env" {
  type        = string
  description = "환경 이름 (dev, staging, prod)"
}

variable "cidr_block" {
  type    = string
  default = "10.0.0.0/16"
}

variable "azs" {
  type    = list(string)
  default = ["ap-northeast-2a", "ap-northeast-2c"]
}
```

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = { Name = "${var.env}-vpc", Environment = var.env }
}

resource "aws_subnet" "public" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index + 1)
  availability_zone = var.azs[count.index]

  tags = { Name = "${var.env}-public-${count.index + 1}", Tier = "public" }
}

resource "aws_subnet" "private" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index + 11)
  availability_zone = var.azs[count.index]

  tags = { Name = "${var.env}-private-${count.index + 1}", Tier = "private" }
}
```

`cidrsubnet` 함수를 사용하면 CIDR 블록을 자동으로 분배해 서브넷을 생성할 수 있다.

### 환경별 진입점

```hcl
# environments/prod/main.tf
module "vpc" {
  source     = "../../modules/vpc"
  env        = "prod"
  cidr_block = "10.0.0.0/16"
}

module "ecs_service" {
  source         = "../../modules/ecs-service"
  env            = "prod"
  vpc_id         = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnet_ids
  desired_count  = 3
  cpu            = 1024
  memory         = 2048
  container_image = var.container_image
}

module "rds" {
  source           = "../../modules/rds-aurora"
  env              = "prod"
  vpc_id           = module.vpc.vpc_id
  subnet_ids       = module.vpc.data_subnet_ids
  instance_class   = "db.r6g.large"
  instance_count   = 2
}
```

```hcl
# environments/dev/main.tf
module "vpc" {
  source     = "../../modules/vpc"
  env        = "dev"
  cidr_block = "10.1.0.0/16"
}

module "ecs_service" {
  source         = "../../modules/ecs-service"
  env            = "dev"
  vpc_id         = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnet_ids
  desired_count  = 1      # dev는 1개로 충분
  cpu            = 256
  memory         = 512
  container_image = var.container_image
}

module "rds" {
  source         = "../../modules/rds-aurora"
  env            = "dev"
  vpc_id         = module.vpc.vpc_id
  subnet_ids     = module.vpc.data_subnet_ids
  instance_class = "db.t3.medium"  # dev는 저렴한 인스턴스
  instance_count = 1
}
```

환경별로 `desired_count`, `instance_class`만 달리해도 상당한 비용 차이가 발생한다.

### 원격 State 백엔드

팀 협업을 위해 Terraform state를 S3에 저장하고 DynamoDB로 잠금을 관리한다.

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "environments/prod/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

S3 버킷은 버전 관리를 활성화해 이전 state로 롤백할 수 있도록 한다.

---

## 4. 비용 추정 실전

### AWS Pricing Calculator 활용

[AWS Pricing Calculator](https://calculator.aws/pricing/2/home) 에서 서비스별 예상 비용을 산출할 수 있다. 하지만 계산기 결과에 의존하기 전에 몇 가지 기준을 잡는 것이 좋다.

### 3-Tier 구성 월 비용 예시 (ap-northeast-2, 소규모 서비스 기준)

| 서비스 | 사양 | 월 비용 (USD) |
|--------|------|-------------|
| ECS Fargate | 0.5 vCPU, 1GB × 2 태스크, 24시간 | ~$30 |
| ALB | 1 LCU 기준 | ~$20 |
| RDS Aurora MySQL | db.t3.medium × 2 (Writer+Reader) | ~$130 |
| ElastiCache Redis | cache.t3.micro × 2 | ~$30 |
| CloudFront | 1TB 전송, 1천만 요청 | ~$25 |
| NAT Gateway | 100GB 처리 | ~$50 |
| **소계** | | **~$285** |

실제 프로덕션 환경은 트래픽 규모에 따라 2~5배 증가할 수 있다. NAT Gateway 비용이 의외로 크므로 PrivateLink나 VPC Endpoint 도입도 고려한다.

### 서버리스 API 구성 월 비용 예시 (소규모 서비스 기준)

| 서비스 | 사양 | 월 비용 (USD) |
|--------|------|-------------|
| API Gateway HTTP API | 100만 요청 | ~$1 |
| Lambda | 100만 호출, 평균 500ms, 512MB | ~$1 |
| DynamoDB On-Demand | 100만 읽기/쓰기 | ~$1.5 |
| Cognito | MAU 10,000 (무료 구간 포함) | $0 |
| CloudFront | 100GB 전송 | ~$8 |
| **소계** | | **~$12** |

서버리스는 초기 트래픽이 낮을 때 압도적으로 유리하다. 하지만 월 요청이 수억 건을 넘어가면 3-Tier 고정 비용이 더 저렴해지는 역전 현상이 발생한다. Lambda와 API Gateway는 호출 건수에 비례해 비용이 선형으로 증가하는 반면, EC2와 ECS는 인스턴스를 한 번 띄워두면 요청이 아무리 많아도 추가 비용이 거의 없다. 트래픽이 충분히 높아지면 고정비용 구조가 종량제보다 유리해지는 시점이 온다.

### 스테이징 비용 절감 전략

프로덕션과 동일한 구성으로 스테이징을 운영하면 비용이 두 배가 된다. 다음 방법으로 스테이징 비용을 줄인다.

**ECS 스케줄 기반 스케일 다운.** 업무 시간 외에는 스테이징 ECS를 0으로 줄인다.

```hcl
resource "aws_appautoscaling_scheduled_action" "staging_scale_down" {
  name               = "staging-scale-down"
  service_namespace  = "ecs"
  resource_id        = "service/${aws_ecs_cluster.staging.name}/${aws_ecs_service.staging_app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  schedule           = "cron(0 21 * * ? *)"  # 매일 21:00 KST (12:00 UTC)

  scalable_target_action {
    min_capacity = 0
    max_capacity = 0
  }
}

resource "aws_appautoscaling_scheduled_action" "staging_scale_up" {
  name               = "staging-scale-up"
  service_namespace  = "ecs"
  resource_id        = "service/${aws_ecs_cluster.staging.name}/${aws_ecs_service.staging_app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  schedule           = "cron(0 0 * * ? *)"  # 매일 09:00 KST (00:00 UTC)

  scalable_target_action {
    min_capacity = 1
    max_capacity = 1
  }
}
```

야간 15시간을 꺼두면 ECS Fargate 비용이 약 40% 절감된다.

**RDS 스테이징 인스턴스 다운사이징.** 스테이징은 단일 `db.t3.small`로 충분하다. Read Replica도 제거한다.

**NAT Gateway 대신 Public Subnet 직접 배치.** 스테이징 ECS는 Public Subnet에 두고 Public IP를 할당하면 NAT Gateway 비용이 없어진다. 보안 그룹으로 인바운드를 제한하면 보안상 문제가 없다.

**RDS 자동 시작/중지.** 개발 DB는 RDS 인스턴스 자동 중지 기능을 활성화한다. 7일간 연결이 없으면 자동으로 중지되고, 연결 시도 시 자동으로 시작된다.

### 비용 모니터링 설정

예산 초과를 방지하기 위해 AWS Budgets를 설정한다.

```hcl
resource "aws_budgets_budget" "monthly" {
  name         = "monthly-budget"
  budget_type  = "COST"
  limit_amount = "500"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["team@myapp.com"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["team@myapp.com"]
  }
}
```

예상 비용이 100%를 초과할 것으로 예측되면 미리 알림을 받을 수 있다.

---

## 마무리: 아키텍처 선택 기준

결국 아키텍처 선택은 트레이드오프다. 서비스 초기에는 서버리스로 빠르게 시작하고, 트래픽이 안정화되면 3-Tier로 전환하는 방식이 현실적인 접근이다.

핵심만 정리하면 다음과 같다.

- 월 활성 사용자 10만 미만: 서버리스 API + CloudFront + S3 조합으로 충분하다.
- 월 활성 사용자 10만~100만: ECS Fargate + RDS Aurora + ElastiCache 3-Tier가 비용 대비 효율적이다.
- 그 이상: 서비스 특성에 따라 아키텍처를 세분화해야 한다.

Terraform 모듈화는 처음부터 해두면 환경 추가와 스펙 변경이 훨씬 편해진다. 나중에 리팩토링하려면 state 관리가 복잡해지므로, 처음 설계에 투자하는 것이 장기적으로 이득이다.

---

## 참고 자료

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Reference Architecture Diagrams](https://aws.amazon.com/architecture/reference-architecture-diagrams/)
- [Terraform AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS Pricing Calculator](https://calculator.aws/pricing/2/home)
- [Amazon ECS Best Practices Guide](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/intro.html)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [AWS Cost Optimization Pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
