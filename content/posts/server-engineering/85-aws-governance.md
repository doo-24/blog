---
title: "[AWS / 클라우드] 8편 — 멀티 계정과 거버넌스: 조직 단위의 클라우드 관리"
date: 2026-03-20T14:01:00+09:00
draft: false
tags: ["AWS Organizations", "IAM", "Landing Zone", "재해 복구", "AWS", "서버"]
series: ["AWS / 클라우드"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "AWS Organizations와 멀티 계정 전략, IAM 설계 원칙(최소 권한, 역할 기반)과 크로스 계정 접근, Landing Zone·Control Tower 개요, 재해 복구 전략(Backup & Restore~Multi-Site)과 클라우드 락인 완화까지"
---

## 목차

1. [왜 멀티 계정인가](#왜-멀티-계정인가)
2. [AWS Organizations 구조](#aws-organizations-구조)
3. [SCP로 계정 정책 강제하기](#scp로-계정-정책-강제하기)
4. [IAM 설계 원칙](#iam-설계-원칙)
5. [크로스 계정 접근 패턴](#크로스-계정-접근-패턴)
6. [Landing Zone과 Control Tower](#landing-zone과-control-tower)
7. [재해 복구 전략](#재해-복구-전략)
8. [클라우드 락인 완화](#클라우드-락인-완화)
9. [참고 자료](#참고-자료)

---

## 왜 멀티 계정인가

초기에는 하나의 AWS 계정에서 모든 것을 운영하는 방식이 단순해 보인다.
하지만 팀이 커지고 서비스가 늘어날수록 단일 계정은 빠르게 한계를 드러낸다.

**격리(Isolation)** 가 핵심 이유다.
개발 환경에서 실수로 실행한 명령이 프로덕션 데이터베이스에 영향을 주는 상황은 멀티 계정 구조로 원천 차단할 수 있다.

**비용 가시성** 도 중요하다.
계정 단위로 청구를 분리하면 팀별, 서비스별, 환경별 비용을 정확하게 추적할 수 있다.

**보안 경계** 측면에서도 유리하다.
하나의 계정이 침해되더라도 다른 계정으로의 피해 확산을 IAM 경계로 제한할 수 있다.

AWS는 이 멀티 계정 구조를 관리하기 위해 **AWS Organizations** 라는 서비스를 제공한다.

---

## AWS Organizations 구조

AWS Organizations는 여러 AWS 계정을 계층적으로 묶어 중앙에서 관리하는 서비스다.

### 핵심 개념

| 개념 | 설명 |
|------|------|
| Management Account | 조직의 루트 계정. 결제 및 정책 관리를 담당한다. |
| Member Account | 조직에 속한 일반 계정. 각 팀/서비스/환경에 할당된다. |
| OU (Organizational Unit) | 계정을 묶는 논리적 컨테이너. 정책 적용 단위다. |
| SCP (Service Control Policy) | OU 또는 계정에 적용하는 권한 경계 정책. |

### 전형적인 OU 구조

```
Root
├── Security OU
│   ├── Log Archive Account
│   └── Audit Account
├── Infrastructure OU
│   └── Shared Services Account
├── Workloads OU
│   ├── Production OU
│   │   ├── Service-A Prod Account
│   │   └── Service-B Prod Account
│   └── Non-Production OU
│       ├── Service-A Dev Account
│       └── Service-A Staging Account
└── Sandbox OU
    └── Developer Sandbox Accounts
```

Security OU의 Log Archive 계정은 CloudTrail, Config, VPC Flow Logs 등 모든 계정의 로그를 중앙 집중식으로 수집한다.
Audit 계정에는 보안 팀만 접근 권한을 가진다.

### Organizations 활성화

```bash
# Management Account에서 실행
aws organizations create-organization --feature-set ALL

# OU 생성
aws organizations create-organizational-unit \
  --parent-id r-xxxx \
  --name "Workloads"

# 계정을 OU로 이동
aws organizations move-account \
  --account-id 123456789012 \
  --source-parent-id r-xxxx \
  --destination-parent-id ou-xxxx-yyyyyyyy
```

---

## SCP로 계정 정책 강제하기

SCP(Service Control Policy)는 계정 또는 OU에서 사용 가능한 최대 권한을 정의한다.
IAM 정책이 허용하더라도 SCP가 거부하면 해당 작업은 실행되지 않는다.

### SCP의 동작 원리

SCP는 **허용 목록(Allow List)** 방식과 **거부 목록(Deny List)** 방식 두 가지로 운용할 수 있다.

AWS 기본값은 `FullAWSAccess` 정책으로 모든 것을 허용한 상태다.
여기에 특정 작업을 명시적으로 거부하는 Deny List 방식이 가장 일반적이다.

### 실용적인 SCP 예시

**리전 제한 SCP** — 서울 리전과 버지니아 리전 외에는 사용 금지:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "organizations:*",
        "support:*",
        "budgets:*",
        "cloudfront:*",
        "route53:*",
        "waf:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "ap-northeast-2",
            "us-east-1"
          ]
        }
      }
    }
  ]
}
```

**루트 계정 보호 SCP** — 루트 사용자 행동 방지:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProtectRootActions",
      "Effect": "Deny",
      "Action": [
        "organizations:LeaveOrganization",
        "account:EnableRegion",
        "account:DisableRegion"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
```

**S3 퍼블릭 액세스 차단 해제 금지**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyS3PublicAccess",
      "Effect": "Deny",
      "Action": [
        "s3:PutBucketPublicAccessBlock",
        "s3:DeletePublicAccessBlock"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "s3:PublicAccessBlockConfiguration/BlockPublicAcls": "false"
        }
      }
    }
  ]
}
```

### SCP 연결

```bash
# SCP 생성
aws organizations create-policy \
  --content file://deny-non-approved-regions.json \
  --description "Deny actions outside approved regions" \
  --name "DenyNonApprovedRegions" \
  --type SERVICE_CONTROL_POLICY

# OU에 SCP 연결
aws organizations attach-policy \
  --policy-id p-xxxxxxxxxxxx \
  --target-id ou-xxxx-yyyyyyyy
```

---

## IAM 설계 원칙

IAM은 AWS 보안의 핵심이다.
잘못 설계된 IAM은 가장 큰 보안 취약점이 된다.

### 최소 권한 원칙 (Least Privilege)

필요한 작업만 허용하고 나머지는 모두 거부하는 원칙이다.
`*` 와일드카드 남용은 가장 흔한 실수다.

나쁜 예:

```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

좋은 예:

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Resource": "arn:aws:s3:::my-app-bucket/*"
}
```

### 역할 기반 접근 제어 (RBAC)

장기 자격증명(Access Key)을 발급하는 대신 IAM Role을 사용하는 것이 원칙이다.
IAM Role은 임시 자격증명을 발급하므로 유출 위험이 훨씬 낮다.

EC2 인스턴스에서 S3에 접근하는 경우:

```bash
# 역할 생성
aws iam create-role \
  --role-name EC2-S3-ReadOnly \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# 정책 연결
aws iam attach-role-policy \
  --role-name EC2-S3-ReadOnly \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 인스턴스 프로파일 생성 및 연결
aws iam create-instance-profile \
  --instance-profile-name EC2-S3-ReadOnly-Profile

aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-ReadOnly-Profile \
  --role-name EC2-S3-ReadOnly
```

### 권한 경계 (Permission Boundary)

개발자가 직접 IAM 역할을 생성할 수 있도록 허용하되, 그 역할의 최대 권한을 제한하는 메커니즘이다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowIAMWithBoundary",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DevBoundary"
        }
      }
    }
  ]
}
```

### MFA 강제

콘솔 접근 시 MFA를 강제하는 IAM 정책:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyWithoutMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

---

## 크로스 계정 접근 패턴

멀티 계정 환경에서 계정 간 리소스 접근은 필수 패턴이다.

### AssumeRole 기반 크로스 계정 접근

가장 기본적인 패턴이다.
계정 A의 사용자가 계정 B의 역할을 assume하여 계정 B 리소스에 접근한다.

계정 B에서 신뢰 정책 설정 (역할 생성):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_A_ID:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

계정 A에서 역할 assume:

```bash
# 임시 자격증명 획득
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT_B_ID:role/CrossAccountRole \
  --role-session-name my-session \
  --duration-seconds 3600

# 환경 변수 설정 후 사용
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...
```

### 중앙 집중식 역할 관리 패턴

Identity Account(또는 Management Account)에 사용자를 생성하고 각 워크로드 계정의 역할을 assume하는 Hub-and-Spoke 패턴이다.

```
Identity Account (사용자 관리)
    |
    |-- AssumeRole --> Dev Account Role
    |-- AssumeRole --> Staging Account Role
    |-- AssumeRole --> Prod Account Role
```

AWS CLI 프로파일로 관리하면 편리하다:

```ini
# ~/.aws/config

[profile identity]
region = ap-northeast-2

[profile dev]
role_arn = arn:aws:iam::DEV_ACCOUNT_ID:role/Developer
source_profile = identity
region = ap-northeast-2

[profile prod]
role_arn = arn:aws:iam::PROD_ACCOUNT_ID:role/ReadOnly
source_profile = identity
mfa_serial = arn:aws:iam::IDENTITY_ACCOUNT_ID:mfa/my-device
region = ap-northeast-2
```

```bash
# dev 계정에서 작업
aws --profile dev s3 ls

# prod 계정에서 읽기 작업
aws --profile prod ec2 describe-instances
```

### 리소스 기반 정책으로 크로스 계정 허용

S3 버킷, KMS 키, SQS 큐 등은 리소스 자체에 정책을 붙여 크로스 계정 접근을 허용할 수 있다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CrossAccountLogDelivery",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::SOURCE_ACCOUNT_ID:root"
      },
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::central-log-bucket/AWSLogs/SOURCE_ACCOUNT_ID/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    }
  ]
}
```

---

## Landing Zone과 Control Tower

### Landing Zone이란

Landing Zone은 멀티 계정 AWS 환경을 안전하고 일관되게 시작하기 위한 기반 구조(baseline)다.
AWS Well-Architected Framework 기반의 계정 구조, 네트워크, 보안 서비스를 자동으로 구성한다.

Landing Zone이 설정하는 주요 요소:

- Management Account, Log Archive Account, Audit Account 기본 구성
- CloudTrail 조직 전체 활성화 및 중앙 집중 로깅
- AWS Config 전체 계정 활성화
- GuardDuty, Security Hub 위임 관리자 설정
- VPC 기본 설정 및 네트워크 허브 구성
- IAM Identity Center(SSO) 설정

### AWS Control Tower

AWS Control Tower는 Landing Zone을 자동화된 방식으로 설정하고 운영하는 관리형 서비스다.
콘솔에서 몇 가지 설정만으로 모범 사례 기반의 멀티 계정 환경을 구축할 수 있다.

**Control Tower의 핵심 구성요소:**

| 구성요소 | 역할 |
|----------|------|
| Landing Zone | 기반 계정 및 OU 구조 자동 설정 |
| Guardrails (Controls) | SCP 및 Config 규칙 기반의 거버넌스 정책 |
| Account Factory | 신규 계정을 표준 템플릿으로 자동 프로비저닝 |
| Dashboard | 전체 계정의 컴플라이언스 상태 중앙 모니터링 |

### Guardrails 종류

Guardrail은 예방(Preventive)과 탐지(Detective) 두 종류다.

**예방형 Guardrail** — SCP 기반으로 특정 작업 자체를 차단한다.
예: "루트 사용자 액세스 키 생성 금지", "CloudTrail 비활성화 금지"

**탐지형 Guardrail** — AWS Config 규칙 기반으로 정책 위반을 감지하고 알림을 발송한다.
예: "퍼블릭 S3 버킷 탐지", "암호화되지 않은 EBS 볼륨 탐지"

### Account Factory로 계정 자동 프로비저닝

Control Tower Account Factory는 Service Catalog를 통해 표준화된 계정을 자동 생성한다.

```bash
# AWS CLI로 Account Factory 제품 조회
aws servicecatalog search-products \
  --filters Key=FullTextSearch,Value="Account Vending"

# 신규 계정 프로비저닝
aws servicecatalog provision-product \
  --product-id prod-xxxxxxxxxx \
  --provisioning-artifact-id pa-xxxxxxxxxx \
  --provisioned-product-name "service-b-prod" \
  --provisioning-parameters \
    Key=AccountEmail,Value=service-b-prod@company.com \
    Key=AccountName,Value=service-b-prod \
    Key=ManagedOrganizationalUnit,Value="Workloads/Production"
```

### IAM Identity Center (구 SSO)

IAM Identity Center는 멀티 계정 환경에서 단일 로그인 포인트를 제공한다.
사용자는 하나의 자격증명으로 여러 AWS 계정과 애플리케이션에 접근할 수 있다.

```bash
# Permission Set 생성 (개발자용)
aws sso-admin create-permission-set \
  --instance-arn arn:aws:sso:::instance/ssoins-xxxxxxxxxx \
  --name "DeveloperAccess" \
  --description "Developer access to dev accounts" \
  --session-duration PT8H

# Permission Set에 정책 연결
aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn arn:aws:sso:::instance/ssoins-xxxxxxxxxx \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-xxxxxxxxxx/ps-xxxxxxxxxx \
  --managed-policy-arn arn:aws:iam::aws:policy/PowerUserAccess
```

---

## 재해 복구 전략

클라우드 환경에서도 장애는 발생한다.
중요한 것은 장애 발생 시 얼마나 빠르게, 얼마나 적은 데이터 손실로 복구하는가다.

### 핵심 지표

**RTO (Recovery Time Objective)**: 서비스가 중단된 후 복구 완료까지 허용 가능한 최대 시간.
예: RTO 4시간은 장애 발생 후 4시간 이내에 서비스를 복구해야 한다는 의미다.

**RPO (Recovery Point Objective)**: 복구 시 허용 가능한 최대 데이터 손실 시간.
예: RPO 1시간은 최대 1시간치 데이터 손실을 허용한다는 의미다.

### 4가지 DR 전략

AWS는 RTO/RPO와 비용의 균형에 따라 4가지 DR 전략을 제시한다.

#### 1. Backup & Restore (백업 및 복구)

가장 비용이 낮지만 RTO/RPO가 가장 길다.
데이터를 정기적으로 S3나 AWS Backup에 저장하고, 장애 시 복구한다.

```bash
# AWS Backup Plan 생성
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "DailyBackup",
  "Rules": [{
    "RuleName": "DailyRule",
    "TargetBackupVaultName": "Default",
    "ScheduleExpression": "cron(0 3 * * ? *)",
    "StartWindowMinutes": 60,
    "CompletionWindowMinutes": 180,
    "Lifecycle": {
      "DeleteAfterDays": 30
    }
  }]
}'

# 백업 대상 리소스 연결
aws backup create-backup-selection \
  --backup-plan-id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
  --backup-selection '{
    "SelectionName": "AllRDSInstances",
    "IamRoleArn": "arn:aws:iam::123456789012:role/AWSBackupRole",
    "Resources": ["arn:aws:rds:*:*:db:*"]
  }'
```

| 지표 | 범위 |
|------|------|
| RTO | 수 시간 ~ 수십 시간 |
| RPO | 24시간 (일별 백업 기준) |
| 비용 | 매우 낮음 |

#### 2. Pilot Light (파일럿 라이트)

핵심 데이터베이스만 복구 리전에서 실시간으로 복제하고, 나머지 인프라는 최소한으로 유지한다.
장애 발생 시 애플리케이션 서버를 빠르게 확장하여 복구한다.

```bash
# RDS 크로스 리전 읽기 복제본 생성
aws rds create-db-instance-read-replica \
  --db-instance-identifier my-app-db-replica \
  --source-db-instance-identifier arn:aws:rds:ap-northeast-2:123456789012:db:my-app-db \
  --db-instance-class db.t3.medium \
  --source-region ap-northeast-2 \
  --region us-east-1
```

| 지표 | 범위 |
|------|------|
| RTO | 수십 분 ~ 수 시간 |
| RPO | 수 분 ~ 수십 분 |
| 비용 | 낮음 |

#### 3. Warm Standby (웜 스탠바이)

복구 리전에 프로덕션 환경의 축소 버전을 항상 실행 상태로 유지한다.
장애 발생 시 트래픽을 전환하고 스케일 업한다.

```bash
# Route53 장애 조치 레코드 설정
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.myapp.com",
          "Type": "A",
          "SetIdentifier": "Primary",
          "Failover": "PRIMARY",
          "HealthCheckId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "primary-alb.ap-northeast-2.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.myapp.com",
          "Type": "A",
          "SetIdentifier": "Secondary",
          "Failover": "SECONDARY",
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "secondary-alb.us-east-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'
```

| 지표 | 범위 |
|------|------|
| RTO | 수 분 ~ 수십 분 |
| RPO | 수 분 이내 |
| 비용 | 중간 |

#### 4. Multi-Site Active-Active (멀티 사이트)

두 리전 모두 프로덕션 트래픽을 처리하는 완전 이중화 구조다.
하나의 리전이 완전히 중단되어도 나머지 리전이 즉시 전체 트래픽을 처리한다.

```bash
# Global Accelerator로 다중 리전 트래픽 분산
aws globalaccelerator create-accelerator \
  --name my-app-accelerator \
  --ip-address-type IPV4 \
  --enabled

aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::123456789012:accelerator/... \
  --endpoint-group-region ap-northeast-2 \
  --endpoint-configurations EndpointId=arn:aws:elasticloadbalancing:...,Weight=128

aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::123456789012:accelerator/... \
  --endpoint-group-region us-east-1 \
  --endpoint-configurations EndpointId=arn:aws:elasticloadbalancing:...,Weight=128
```

| 지표 | 범위 |
|------|------|
| RTO | 거의 0 (초 단위) |
| RPO | 거의 0 |
| 비용 | 매우 높음 |

### DR 전략 선택 기준

```
비용 낮음 ←────────────────────────────→ 비용 높음
RTO/RPO 길음 ←─────────────────────────→ RTO/RPO 짧음

Backup    Pilot     Warm       Multi-Site
& Restore  Light   Standby   Active-Active
```

내부 관리 시스템이라면 Backup & Restore로도 충분하다.
B2B SaaS 서비스라면 Warm Standby 수준이 현실적인 선택이다.
금융, 의료 등 미션 크리티컬 서비스는 Multi-Site를 고려해야 한다.

---

## 클라우드 락인 완화

특정 클라우드 벤더에 지나치게 의존하면 비용 협상력이 떨어지고 마이그레이션 비용이 기하급수적으로 증가한다.

### 락인의 종류

**서비스 락인**: AWS 전용 서비스(DynamoDB, Kinesis 등)를 깊게 사용하면 타 벤더 이전이 어렵다.

**기술 락인**: AWS SDK를 직접 코드에 포함하면 클라우드 변경 시 코드 전면 수정이 필요하다.

**데이터 락인**: 대용량 데이터를 S3에 저장하면 데이터 전송 비용(Egress Fee)이 이전을 사실상 막는다.

### 완화 전략

**추상화 레이어 도입**

클라우드 서비스를 직접 호출하는 대신 인터페이스로 감싼다.
스토리지 접근을 예로 들면:

```python
# 나쁜 예: AWS SDK 직접 사용
import boto3

s3 = boto3.client('s3')
s3.put_object(Bucket='my-bucket', Key='file.txt', Body=data)

# 좋은 예: 추상화 인터페이스
from storage import StorageBackend

class S3Backend(StorageBackend):
    def put(self, key: str, data: bytes) -> None:
        self._client.put_object(Bucket=self._bucket, Key=key, Body=data)

class GCSBackend(StorageBackend):
    def put(self, key: str, data: bytes) -> None:
        blob = self._bucket.blob(key)
        blob.upload_from_string(data)

# 코드는 StorageBackend 인터페이스만 사용
storage = get_storage_backend()  # 환경 변수로 결정
storage.put("file.txt", data)
```

**오픈 표준 및 이식 가능한 서비스 우선 사용**

컨테이너(Docker + Kubernetes), 오픈소스 데이터베이스(PostgreSQL, MySQL), 표준 메시지 프로토콜(AMQP, MQTT) 등을 선호한다.
AWS EKS에서 운영하던 쿠버네티스 클러스터는 GKE나 온프레미스로 비교적 쉽게 이전할 수 있다.

**멀티 클라우드 전략**

모든 서비스를 하나의 클라우드에서 운영하는 대신 역할을 분산시킨다.
예: 기본 인프라는 AWS, 머신러닝 워크로드는 GCP, 생산성 도구는 Azure.

단, 멀티 클라우드는 운영 복잡도를 크게 높인다.
락인 리스크보다 운영 비용이 더 클 수 있으므로 신중하게 선택해야 한다.

**Terraform으로 인프라 코드화**

AWS CloudFormation 대신 Terraform을 사용하면 멀티 클라우드 이식성이 높아진다.

```hcl
# AWS용 Terraform
resource "aws_s3_bucket" "app_storage" {
  bucket = "my-app-storage"
}

# GCP로 이전 시 Google Cloud Storage로 교체
resource "google_storage_bucket" "app_storage" {
  name = "my-app-storage"
}
```

Terraform은 프로바이더만 교체하면 동일한 코드 구조로 다른 클라우드를 관리할 수 있다.

**데이터 이전 비용 계획**

대규모 데이터 이전은 AWS Egress 비용이 상당하다.
중요 데이터는 멀티 리전 또는 멀티 클라우드 복제를 사전에 구성해두는 것이 현실적이다.

```bash
# S3 크로스 리전 복제 설정
aws s3api put-bucket-replication \
  --bucket source-bucket \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456789012:role/S3ReplicationRole",
    "Rules": [{
      "Status": "Enabled",
      "Destination": {
        "Bucket": "arn:aws:s3:::destination-bucket",
        "StorageClass": "STANDARD_IA"
      }
    }]
  }'
```

### 현실적인 접근법

완전한 락인 제로는 사실상 불가능하고, 추구하다 보면 오히려 아무것도 제대로 못 쓰게 된다.

"합리적인 락인"과 "피해야 할 락인"을 구분하는 것이 핵심이다.
관리형 데이터베이스(RDS), 로드 밸런서(ALB), 오토스케일링 같은 기본 인프라 서비스는 어느 클라우드에나 동등한 대안이 있으므로 락인 리스크가 낮다.
반면 벤더 특화 AI/ML 서비스나 독자적인 데이터 파이프라인 서비스는 이전 비용이 크다.

---

## 참고 자료

- [AWS Organizations 공식 문서](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html)
- [AWS Control Tower 공식 문서](https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html)
- [AWS IAM 모범 사례](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS 재해 복구 Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html)
- [AWS Well-Architected Framework — 신뢰성 원칙](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
- [SCP 정책 예시 모음 (aws-samples)](https://github.com/aws-samples/service-control-policy-examples)
- [Terraform AWS Provider 문서](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS Landing Zone Accelerator](https://aws.amazon.com/solutions/implementations/landing-zone-accelerator-on-aws/)
