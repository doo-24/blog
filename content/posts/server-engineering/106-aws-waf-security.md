---
title: "[AWS 실전 구축] 7편 — 웹 방화벽과 보안 계층: WAF, Shield, 보안 자동화"
date: 2026-03-17T11:02:00+09:00
draft: false
tags: ["WAF", "Shield", "GuardDuty", "보안", "AWS", "서버"]
series: ["AWS 실전 구축"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "WAF Web ACL 구성과 관리형 규칙 그룹, Rate-based 규칙과 IP 차단, WAF+CloudFront/ALB 연동과 로그 분석, Shield·GuardDuty·Config Rules 보안 자동화까지"
---

인터넷에 서비스를 노출하는 순간부터 공격은 시작된다. SQL 인젝션, XSS, 대규모 크롤링, DDoS까지 — AWS WAF는 이 모든 위협을 애플리케이션 앞단에서 차단하는 첫 번째 방어선이다. 이번 편에서는 WAF Web ACL을 실전 수준으로 구성하고, Shield와 GuardDuty를 연동해 보안 계층을 완성하는 방법을 다룬다.

---

## WAF Web ACL 구성 기초

AWS WAF는 Web ACL(Access Control List) 단위로 동작한다. Web ACL 안에 규칙을 배치하고, 각 규칙은 요청을 검사해 허용(Allow), 차단(Block), 카운트(Count), 캡차(CAPTCHA) 중 하나를 수행한다.

Web ACL은 리전(Regional) 또는 글로벌(CloudFront) 두 가지 범위로 생성한다. ALB, API Gateway, AppSync에는 리전 WAF를, CloudFront에는 반드시 `us-east-1` 리전의 글로벌 WAF를 연결해야 한다.

**Web ACL 생성 (AWS CLI)**

```bash
aws wafv2 create-web-acl \
  --name "prod-web-acl" \
  --scope REGIONAL \
  --region ap-northeast-2 \
  --default-action Allow={} \
  --visibility-config \
    SampledRequestsEnabled=true,\
    CloudWatchMetricsEnabled=true,\
    MetricName=ProdWebACL \
  --description "Production Web ACL"
```

`--default-action Allow={}`는 어떤 규칙에도 걸리지 않은 요청을 기본으로 허용한다는 뜻이다. 보안이 더 중요한 환경이라면 `Block={}`으로 설정하고 허용 규칙을 명시적으로 추가하는 화이트리스트 방식을 택한다.

---

## 관리형 규칙 그룹 선택과 커스터마이징

AWS가 제공하는 관리형 규칙 그룹(Managed Rule Group)은 즉시 활성화할 수 있는 보안 규칙 묶음이다. 직접 규칙을 작성하지 않아도 알려진 취약점 패턴을 자동으로 차단한다.

**주요 AWS 관리형 규칙 그룹**

| 규칙 그룹 | 용도 |
|---|---|
| `AWSManagedRulesCommonRuleSet` | OWASP Top 10 공통 취약점 |
| `AWSManagedRulesSQLiRuleSet` | SQL 인젝션 전용 |
| `AWSManagedRulesKnownBadInputsRuleSet` | 알려진 악성 입력 패턴 |
| `AWSManagedRulesAmazonIpReputationList` | Amazon이 관리하는 악성 IP 목록 |
| `AWSManagedRulesAnonymousIpList` | VPN, Tor, 프록시 IP |
| `AWSManagedRulesBotControlRuleSet` | 봇 트래픽 감지 및 차단 |

관리형 규칙 그룹을 Web ACL에 추가하는 JSON 구성 예시다.

```json
{
  "Name": "AWS-AWSManagedRulesCommonRuleSet",
  "Priority": 10,
  "Statement": {
    "ManagedRuleGroupStatement": {
      "VendorName": "AWS",
      "Name": "AWSManagedRulesCommonRuleSet",
      "ExcludedRules": [
        { "Name": "SizeRestrictions_BODY" }
      ]
    }
  },
  "OverrideAction": { "None": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "CommonRuleSet"
  }
}
```

`ExcludedRules`로 특정 규칙을 제외할 수 있다. 파일 업로드 엔드포인트처럼 요청 본문 크기가 큰 경우 `SizeRestrictions_BODY` 규칙이 오탐을 발생시키므로 제외하는 것이 일반적이다.

**규칙 우선순위**

Web ACL 안의 규칙은 Priority 숫자가 낮을수록 먼저 평가된다. IP 허용 목록은 가장 낮은 우선순위 번호(0 또는 1)에 배치해 다른 규칙보다 먼저 통과시키는 것이 일반적이다.

---

## Rate-based 규칙으로 과도한 요청 차단

Rate-based 규칙은 특정 IP에서 일정 시간(5분) 안에 정해진 횟수 이상의 요청이 들어오면 자동으로 차단하는 기능이다.

```json
{
  "Name": "RateLimitRule",
  "Priority": 5,
  "Statement": {
    "RateBasedStatement": {
      "Limit": 2000,
      "AggregateKeyType": "IP"
    }
  },
  "Action": { "Block": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "RateLimitRule"
  }
}
```

`Limit: 2000`은 하나의 IP에서 5분에 2000회 이상 요청 시 차단한다는 뜻이다. 로그인 엔드포인트처럼 민감한 경로는 별도의 Rate-based 규칙으로 더 낮은 임계값을 설정한다.

**특정 URI에만 Rate 제한 적용**

```json
{
  "Name": "LoginRateLimit",
  "Priority": 3,
  "Statement": {
    "RateBasedStatement": {
      "Limit": 100,
      "AggregateKeyType": "IP",
      "ScopeDownStatement": {
        "ByteMatchStatement": {
          "SearchString": "/api/login",
          "FieldToMatch": { "UriPath": {} },
          "TextTransformations": [
            { "Priority": 0, "Type": "LOWERCASE" }
          ],
          "PositionalConstraint": "STARTS_WITH"
        }
      }
    }
  },
  "Action": { "Block": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "LoginRateLimit"
  }
}
```

`ScopeDownStatement`로 Rate 제한을 특정 URI에만 좁혀 적용한다. 로그인 경로에 5분 100회 제한을 걸면 무차별 대입 공격(Brute Force)을 효과적으로 막을 수 있다.

---

## IP 세트를 이용한 IP 평판 차단

수동으로 차단할 IP 목록이 있다면 IP 세트(IP Set)를 만들어 WAF 규칙에서 참조한다.

```bash
# IP 세트 생성
aws wafv2 create-ip-set \
  --name "BlockedIPs" \
  --scope REGIONAL \
  --region ap-northeast-2 \
  --ip-address-version IPV4 \
  --addresses "192.168.1.100/32" "10.0.0.0/8"
```

생성 후 반환된 `ARN`을 규칙에서 참조한다.

```json
{
  "Name": "BlockListedIPs",
  "Priority": 1,
  "Statement": {
    "IPSetReferenceStatement": {
      "ARN": "arn:aws:wafv2:ap-northeast-2:123456789012:regional/ipset/BlockedIPs/xxxxxxxx"
    }
  },
  "Action": { "Block": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "BlockListedIPs"
  }
}
```

IP를 동적으로 추가하거나 제거하려면 `update-ip-set` 명령을 사용한다.

```bash
aws wafv2 update-ip-set \
  --name "BlockedIPs" \
  --scope REGIONAL \
  --region ap-northeast-2 \
  --id "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" \
  --lock-token "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy" \
  --addresses "203.0.113.0/24" "198.51.100.55/32"
```

`--lock-token`은 동시 수정 충돌 방지를 위해 `get-ip-set` 응답에서 가져온 값을 사용한다.

---

## SQLi / XSS 탐지 규칙 직접 작성

관리형 규칙 그룹 외에 직접 SQLi, XSS 탐지 규칙을 작성할 수도 있다.

**SQLi 탐지 규칙**

```json
{
  "Name": "CustomSQLiRule",
  "Priority": 20,
  "Statement": {
    "OrStatement": {
      "Statements": [
        {
          "SqliMatchStatement": {
            "FieldToMatch": { "QueryString": {} },
            "TextTransformations": [
              { "Priority": 1, "Type": "URL_DECODE" },
              { "Priority": 2, "Type": "HTML_ENTITY_DECODE" }
            ]
          }
        },
        {
          "SqliMatchStatement": {
            "FieldToMatch": { "Body": { "OversizeHandling": "MATCH" } },
            "TextTransformations": [
              { "Priority": 1, "Type": "URL_DECODE" }
            ]
          }
        }
      ]
    }
  },
  "Action": { "Block": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "CustomSQLi"
  }
}
```

`TextTransformations`로 URL 인코딩이나 HTML 엔티티 인코딩으로 우회하는 시도를 미리 디코딩한 뒤 검사한다. 중요한 필드는 복수의 변환을 체이닝해 적용한다.

**XSS 탐지 규칙**

```json
{
  "Name": "CustomXSSRule",
  "Priority": 21,
  "Statement": {
    "XssMatchStatement": {
      "FieldToMatch": { "AllQueryArguments": {} },
      "TextTransformations": [
        { "Priority": 1, "Type": "URL_DECODE" },
        { "Priority": 2, "Type": "HTML_ENTITY_DECODE" },
        { "Priority": 3, "Type": "LOWERCASE" }
      ]
    }
  },
  "Action": { "Block": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "CustomXSS"
  }
}
```

---

## WAF + ALB 연동

WAF Web ACL을 Application Load Balancer에 연결하면 EC2, ECS, EKS로 향하는 모든 HTTP(S) 트래픽을 사전 검사할 수 있다.

```bash
# ALB ARN 확인
aws elbv2 describe-load-balancers \
  --names "prod-alb" \
  --query "LoadBalancers[0].LoadBalancerArn" \
  --output text

# WAF Web ACL을 ALB에 연결
aws wafv2 associate-web-acl \
  --web-acl-arn "arn:aws:wafv2:ap-northeast-2:123456789012:regional/webacl/prod-web-acl/xxxxxxxx" \
  --resource-arn "arn:aws:elasticloadbalancing:ap-northeast-2:123456789012:loadbalancer/app/prod-alb/xxxxxxxx" \
  --region ap-northeast-2
```

연결 확인은 `get-web-acl-for-resource` 명령으로 검증한다.

```bash
aws wafv2 get-web-acl-for-resource \
  --resource-arn "arn:aws:elasticloadbalancing:ap-northeast-2:123456789012:loadbalancer/app/prod-alb/xxxxxxxx" \
  --region ap-northeast-2
```

---

## WAF + CloudFront 연동

CloudFront에 WAF를 연결하면 엣지 로케이션 단계에서 차단이 이루어진다. 악성 트래픽이 오리진 서버까지 도달하지 않으므로 오리진 부하를 줄이는 효과도 있다.

CloudFront용 WAF는 반드시 `us-east-1` 리전에서 `--scope CLOUDFRONT`로 생성해야 한다.

```bash
aws wafv2 create-web-acl \
  --name "prod-cf-web-acl" \
  --scope CLOUDFRONT \
  --region us-east-1 \
  --default-action Allow={} \
  --visibility-config \
    SampledRequestsEnabled=true,\
    CloudWatchMetricsEnabled=true,\
    MetricName=ProdCFWebACL
```

CloudFront 배포에 WAF 연결은 콘솔의 배포 설정 또는 Terraform `aws_cloudfront_distribution`의 `web_acl_id` 속성으로 적용한다.

```hcl
resource "aws_cloudfront_distribution" "main" {
  web_acl_id = aws_wafv2_web_acl.cloudfront_acl.arn

  # ... 나머지 설정
}
```

---

## WAF 로그 수집과 S3 + Athena 분석

WAF 로그를 S3에 저장하고 Athena로 쿼리하면 공격 패턴을 체계적으로 분석할 수 있다.

**로깅 구성 활성화**

```bash
# Kinesis Firehose 또는 S3 직접 전송
aws wafv2 put-logging-configuration \
  --logging-configuration '{
    "ResourceArn": "arn:aws:wafv2:ap-northeast-2:123456789012:regional/webacl/prod-web-acl/xxxxxxxx",
    "LogDestinationConfigs": [
      "arn:aws:firehose:ap-northeast-2:123456789012:deliverystream/aws-waf-logs-prod"
    ],
    "LoggingFilter": {
      "DefaultBehavior": "DROP",
      "Filters": [
        {
          "Behavior": "KEEP",
          "Conditions": [
            { "ActionCondition": { "Action": "BLOCK" } }
          ],
          "Requirement": "MEETS_ANY"
        }
      ]
    }
  }' \
  --region ap-northeast-2
```

`LoggingFilter`의 `DefaultBehavior: DROP`과 `BLOCK` 액션만 `KEEP`하면 차단된 요청만 로그로 남겨 저장 비용을 줄일 수 있다. 분석 목적이라면 `DefaultBehavior: KEEP`으로 전체 로그를 수집한다.

**Athena 테이블 생성**

S3에 쌓인 WAF 로그를 Athena로 조회하기 위한 테이블 DDL이다.

```sql
CREATE EXTERNAL TABLE waf_logs (
  timestamp       BIGINT,
  formatversion   INT,
  webaclid        STRING,
  terminatingruleid STRING,
  terminatingruletype STRING,
  action          STRING,
  httpsourcename  STRING,
  httpsourceid    STRING,
  rulegrouplist   ARRAY<STRUCT<
    ruleGroupId: STRING,
    terminatingRule: STRUCT<ruleId: STRING, action: STRING>,
    nonTerminatingMatchingRules: ARRAY<STRUCT<ruleId: STRING, action: STRING>>
  >>,
  ratebasedrulelist ARRAY<STRUCT<rateBasedRuleId: STRING, limitKey: STRING, maxRateAllowed: INT>>,
  nonterminatingmatchingrules ARRAY<STRUCT<ruleid: STRING, action: STRING>>,
  httprequest STRUCT<
    clientip: STRING,
    country: STRING,
    headers: ARRAY<STRUCT<name: STRING, value: STRING>>,
    uri: STRING,
    args: STRING,
    httpversion: STRING,
    httpmethod: STRING,
    requestid: STRING
  >
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://your-waf-logs-bucket/AWSLogs/123456789012/WAFLogs/'
TBLPROPERTIES ('has_encrypted_data' = 'false');
```

**자주 쓰는 Athena 쿼리**

차단된 요청 IP Top 10 조회:

```sql
SELECT httprequest.clientip, COUNT(*) AS block_count
FROM waf_logs
WHERE action = 'BLOCK'
  AND from_unixtime(timestamp / 1000) >= NOW() - INTERVAL '24' HOUR
GROUP BY httprequest.clientip
ORDER BY block_count DESC
LIMIT 10;
```

URI 경로별 차단 현황:

```sql
SELECT httprequest.uri, terminatingruleid, COUNT(*) AS cnt
FROM waf_logs
WHERE action = 'BLOCK'
GROUP BY httprequest.uri, terminatingruleid
ORDER BY cnt DESC
LIMIT 20;
```

---

## Shield Standard vs Advanced

AWS Shield는 DDoS 방어를 담당하는 서비스다. 모든 AWS 계정에 Shield Standard가 무료로 활성화되어 있다.

**Shield Standard**

- L3/L4 DDoS 공격(SYN Flood, UDP Reflection 등) 자동 탐지 및 완화
- CloudFront, Route 53에 적용
- 별도 활성화 없이 자동 적용

**Shield Advanced**

- Shield Standard의 모든 기능 포함
- L7 DDoS 탐지 및 WAF 자동 응답 규칙 생성
- DDoS 공격으로 인한 Auto Scaling, CloudFront 비용 크레딧 지원
- AWS DDoS Response Team(DRT) 24/7 지원
- 월 $3,000 + 데이터 전송 요금

Shield Advanced 활성화 및 리소스 보호:

```bash
# Shield Advanced 구독
aws shield create-subscription

# 보호 대상 리소스 등록
aws shield create-protection \
  --name "ProdALBProtection" \
  --resource-arn "arn:aws:elasticloadbalancing:ap-northeast-2:123456789012:loadbalancer/app/prod-alb/xxxxxxxx"
```

보호 대상으로 등록 가능한 리소스 유형: CloudFront, Route 53 Hosted Zone, ALB, ELB Classic, Elastic IP, Global Accelerator

---

## GuardDuty: 위협 탐지 자동화

GuardDuty는 VPC Flow Logs, CloudTrail, DNS 로그를 분석해 이상 행동을 자동으로 탐지한다. WAF가 HTTP 레이어 공격을 막는다면, GuardDuty는 계정 수준의 위협과 내부 이상 징후를 감지한다.

**GuardDuty 활성화**

```bash
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --region ap-northeast-2
```

`--finding-publishing-frequency`는 `FIFTEEN_MINUTES`, `ONE_HOUR`, `SIX_HOURS` 중 선택한다. 보안이 중요한 환경에서는 15분 단위를 권장한다.

**주요 탐지 유형**

| 카테고리 | 예시 Finding |
|---|---|
| EC2 위협 | `UnauthorizedAccess:EC2/SSHBruteForce` |
| IAM 이상 | `UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B` |
| 크립토마이닝 | `CryptoCurrency:EC2/BitcoinTool.B!DNS` |
| 데이터 유출 | `Exfiltration:S3/MaliciousIPCaller` |
| 악성 IP 통신 | `Backdoor:EC2/C&CActivity.B!DNS` |

**EventBridge + Lambda로 자동 대응**

GuardDuty Finding이 발생하면 EventBridge를 통해 Lambda를 호출해 자동 대응할 수 있다.

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{ "numeric": [">=", 7] }]
  }
}
```

EventBridge 규칙에서 심각도 7 이상(High)의 Finding만 필터링해 Lambda로 전달한다. Lambda 함수에서는 해당 EC2를 격리(Security Group 교체), 스냅샷 생성, Slack 알림 등을 자동화한다.

---

## Config Rules로 보안 컴플라이언스 자동화

AWS Config는 리소스 구성 변경을 추적하고, Config Rules는 구성이 보안 정책에 맞는지 자동으로 평가한다.

**주요 관리형 Config Rules**

```bash
# S3 버킷 퍼블릭 접근 차단 확인
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-access-prohibited",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    }
  }'

# 루트 계정 MFA 활성화 확인
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "root-account-mfa-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "ROOT_ACCOUNT_MFA_ENABLED"
    }
  }'

# 보안 그룹 0.0.0.0/0 SSH 허용 금지
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "restricted-ssh",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "INCOMING_SSH_DISABLED"
    }
  }'
```

**Config Rules 비준수 자동 수정 (Auto Remediation)**

비준수 리소스를 감지하면 SSM Automation으로 자동 수정할 수 있다.

```bash
aws configservice put-remediation-configurations \
  --remediation-configurations '[
    {
      "ConfigRuleName": "s3-bucket-public-access-prohibited",
      "TargetType": "SSM_DOCUMENT",
      "TargetId": "AWS-DisableS3BucketPublicReadWrite",
      "Parameters": {
        "AutomationAssumeRole": {
          "StaticValue": {
            "Values": ["arn:aws:iam::123456789012:role/ConfigRemediationRole"]
          }
        },
        "S3BucketName": {
          "ResourceValue": { "Value": "RESOURCE_ID" }
        }
      },
      "Automatic": true,
      "MaximumAutomaticAttempts": 3,
      "RetryAttemptSeconds": 60
    }
  ]'
```

`"Automatic": true`로 설정하면 Config가 비준수를 탐지하는 즉시 SSM Automation이 실행된다.

---

## Security Hub로 보안 상태 통합 관리

Shield, GuardDuty, Config, Inspector, Macie에서 발생하는 Finding을 한 곳에서 집계하려면 Security Hub를 활성화한다.

```bash
aws securityhub enable-security-hub \
  --enable-default-standards \
  --region ap-northeast-2
```

`--enable-default-standards`는 AWS Foundational Security Best Practices와 CIS AWS Foundations Benchmark를 자동으로 활성화한다.

Security Hub의 Finding은 ASFF(Amazon Security Finding Format)로 정규화된다. EventBridge로 전달해 SIEM 시스템이나 Slack/PagerDuty에 연동하는 패턴이 일반적이다.

---

## 보안 계층 아키텍처 요약

실전 환경에서 각 보안 서비스가 담당하는 레이어를 정리하면 다음과 같다.

```
인터넷 요청
    │
    ▼
[Route 53] ─── Shield Standard (L3/L4 DDoS)
    │
    ▼
[CloudFront] ─── WAF (L7: SQLi, XSS, Rate-limit)
    │                └── Shield Advanced (L7 DDoS)
    ▼
[ALB] ─────────── WAF (내부 트래픽 추가 검사)
    │
    ▼
[EC2 / ECS] ──── Security Group (포트/IP 제어)
    │                VPC Flow Logs → GuardDuty
    ▼
[S3 / RDS] ───── Config Rules (컴플라이언스)
                  CloudTrail → GuardDuty (API 이상 탐지)
```

WAF는 HTTP 레이어 공격을, Shield는 볼륨 DDoS를, GuardDuty는 계정·인프라 수준 이상을, Config Rules는 구성 준수를 각각 담당한다. 네 계층이 모두 작동할 때 비로소 방어 깊이(Defense in Depth)가 완성된다.

---

## 참고 자료

- [AWS WAF 개발자 안내서](https://docs.aws.amazon.com/waf/latest/developerguide/)
- [AWS Managed Rules rule groups list](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-list.html)
- [AWS Shield Advanced features](https://docs.aws.amazon.com/waf/latest/developerguide/shield-advanced-features.html)
- [Amazon GuardDuty Finding types](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-active.html)
- [AWS Config Managed Rules](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html)
- [AWS Security Hub ASFF](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-findings-format.html)
- [WAF + Athena 로그 분석 블로그](https://aws.amazon.com/blogs/security/querying-aws-waf-logs-with-amazon-athena/)
