---
title: "[AWS 실전 구축] 2편 — 도메인과 HTTPS: Route 53, ACM, CloudFront 연결"
date: 2026-03-17T11:07:00+09:00
draft: false
tags: ["Route 53", "ACM", "CloudFront", "HTTPS", "AWS", "서버"]
series: ["AWS 실전 구축"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "Route 53 호스팅 존과 레코드 타입(A, CNAME, Alias), ACM 인증서 발급과 DNS 검증, CloudFront 배포 구성(S3/ALB 오리진, OAC, 캐시 정책), 정적 사이트 배포 파이프라인까지"
---

EC2 인스턴스를 띄우고 애플리케이션을 배포했다면, 다음 단계는 도메인을 연결하고 HTTPS를 적용하는 것이다. AWS는 Route 53, ACM, CloudFront라는 세 서비스를 조합해 이 작업을 깔끔하게 처리한다. 각 서비스의 역할과 연결 방식을 순서대로 살펴본다.

---

## Route 53: DNS를 AWS에서 관리하기

Route 53은 AWS의 DNS 서비스다. 도메인 등록부터 레코드 관리, 상태 검사까지 담당한다.

```text
사용자 브라우저
  |  DNS 조회 (example.com)
  ▼
[Route 53]  →  Alias 레코드  →  CloudFront 배포
                                      |
                         ┌────────────┴────────────┐
                         ▼                         ▼
               /api/* 경로                  그 외 경로
                [ALB]                         [S3]
                  |                        (정적 파일)
               [ECS / EC2]

ACM 인증서 (us-east-1)  →  CloudFront 에 연결 (HTTPS)
ACM 인증서 (ap-northeast-2)  →  ALB 에 연결 (HTTPS)
```

### 호스팅 존(Hosted Zone)

호스팅 존은 특정 도메인에 대한 DNS 레코드 묶음이다. `example.com`을 Route 53에서 관리하려면 먼저 호스팅 존을 생성해야 한다.

```bash
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference "$(date +%s)"
```

호스팅 존을 만들면 NS(Name Server) 레코드 4개가 자동으로 생성된다. 이 NS 레코드를 도메인 등록 기관(가비아, Namecheap 등)의 네임서버 설정에 입력해야 Route 53이 DNS 권한을 갖는다.

퍼블릭 호스팅 존은 인터넷에서 접근 가능한 DNS를 관리한다. 프라이빗 호스팅 존은 VPC 내부 전용 DNS를 관리한다. 대부분의 경우 퍼블릭 호스팅 존을 사용한다.

### 레코드 타입

Route 53에서 자주 쓰는 레코드 타입은 다음과 같다.

**A 레코드**: 도메인을 IPv4 주소로 연결한다. `example.com → 1.2.3.4` 형태다.

**CNAME 레코드**: 도메인을 다른 도메인으로 연결한다. `www.example.com → example.com` 형태다. 루트 도메인(`example.com` 자체)에는 사용할 수 없다는 제약이 있다.

**AAAA 레코드**: 도메인을 IPv6 주소로 연결한다. A 레코드의 IPv6 버전이다.

**MX 레코드**: 메일 서버를 지정한다. 이메일 전송 경로를 정의한다.

**TXT 레코드**: 텍스트 값을 저장한다. 도메인 소유권 인증이나 SPF 설정에 쓴다.

### Alias 레코드

Route 53 고유의 레코드 타입이다. AWS 리소스(CloudFront, ALB, S3 웹사이트 엔드포인트 등)를 직접 가리킬 수 있다.

CNAME과 달리 루트 도메인에도 사용할 수 있다. 또한 AWS 리소스의 IP가 변경되어도 자동으로 따라간다. Route 53 쿼리 비용도 발생하지 않는다.

```bash
# Alias 레코드 생성 예시 (CloudFront 배포 연결)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "d1234abcd.cloudfront.net",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```

CloudFront의 호스팅 존 ID는 항상 `Z2FDTNDATAQYW2`다. 모든 리전에서 동일하다.

### 라우팅 정책

Route 53은 단순 DNS 응답을 넘어 다양한 라우팅 정책을 지원한다.

**단순 라우팅(Simple)**: 하나의 레코드에 하나의 값. 기본 설정이다.

**가중치 라우팅(Weighted)**: 여러 리소스에 트래픽을 비율로 분산한다. 블루/그린 배포나 A/B 테스트에 유용하다.

```
example.com → 서버A (가중치 70)
example.com → 서버B (가중치 30)
```

전체 가중치 합계가 기준이 아니라 각 레코드의 가중치 비율이 기준이다. 서버A가 70%, 서버B가 30%의 트래픽을 받는다.

**지연 시간 라우팅(Latency)**: 클라이언트와 가장 낮은 지연 시간을 가진 리전으로 연결한다. 멀티 리전 배포 시 성능을 최적화한다.

**장애 조치 라우팅(Failover)**: Primary 리소스가 정상이면 그쪽으로, 장애 감지 시 Secondary로 자동 전환한다. 상태 검사(Health Check)와 함께 사용한다.

**지리적 라우팅(Geolocation)**: 클라이언트의 지리적 위치에 따라 다른 리소스로 연결한다. 국가별로 다른 서버를 운영할 때 사용한다.

**IP 기반 라우팅(IP-based)**: 클라이언트 IP CIDR 범위에 따라 라우팅한다. 사내망 사용자를 별도 서버로 연결하는 식으로 활용한다.

---

## ACM: SSL/TLS 인증서 무료로 관리하기

AWS Certificate Manager(ACM)는 SSL/TLS 인증서를 무료로 발급하고 관리한다. 인증서 갱신도 자동으로 처리한다.

### 인증서 발급

CloudFront에 연결할 인증서는 반드시 **us-east-1(버지니아 북부)** 리전에서 발급해야 한다. CloudFront는 전 세계 엣지 로케이션에서 동작하는 글로벌 서비스이며, AWS 내부적으로 인증서를 각 엣지 로케이션에 배포할 때 us-east-1을 단일 소스로 사용한다. 서울 리전에서 발급한 인증서는 CloudFront에서 선택 자체가 불가능하다.

```bash
# us-east-1 리전에서 인증서 요청
aws acm request-certificate \
  --domain-name example.com \
  --subject-alternative-names "*.example.com" \
  --validation-method DNS \
  --region us-east-1
```

`*.example.com`을 추가하면 모든 서브도메인(`www`, `api`, `blog` 등)을 하나의 인증서로 커버한다.

ALB에 연결할 인증서는 ALB가 위치한 리전에서 발급한다. 예를 들어 ALB가 ap-northeast-2(서울)에 있다면 서울 리전에서 발급한다.

### DNS 검증

인증서 발급 시 도메인 소유권을 검증해야 한다. DNS 검증과 이메일 검증 두 가지 방법이 있다. DNS 검증이 훨씬 편리하다.

ACM이 CNAME 레코드 하나를 제공한다. 그 레코드를 Route 53에 등록하면 ACM이 자동으로 검증하고 인증서를 발급한다.

```bash
# 검증에 필요한 CNAME 정보 확인
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:us-east-1:123456789:certificate/abc123 \
  --region us-east-1 \
  --query 'Certificate.DomainValidationOptions'
```

출력에서 `ResourceRecord` 필드를 확인한다. `Name`과 `Value`가 CNAME 레코드의 키-값이다.

Route 53에 해당 도메인의 호스팅 존이 있다면 콘솔에서 "Route 53에서 레코드 생성" 버튼 하나로 자동 등록된다. DNS 전파 후 수 분 내에 검증이 완료된다.

검증이 완료된 ACM 인증서는 자동 갱신된다. 만료 60일 전부터 갱신을 시도하며, CNAME 레코드가 유지되는 한 사람이 개입할 필요가 없다.

### 인증서 상태 확인

```bash
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:us-east-1:123456789:certificate/abc123 \
  --region us-east-1 \
  --query 'Certificate.Status'
```

`PENDING_VALIDATION` → `ISSUED` 순서로 상태가 바뀐다. `ISSUED` 상태가 되어야 CloudFront나 ALB에 연결할 수 있다.

---

## CloudFront: CDN과 HTTPS 적용

CloudFront는 AWS의 CDN(Content Delivery Network) 서비스다. 전 세계 엣지 로케이션에서 콘텐츠를 캐시하고, HTTPS 종료 지점 역할도 한다.

### 배포(Distribution) 생성

CloudFront 배포는 하나의 CDN 설정 단위다. 오리진(원본 서버), 캐시 동작, HTTPS 설정, 커스텀 도메인 등을 구성한다.

콘솔에서 "배포 생성"을 클릭하거나 CLI로 생성한다.

```bash
aws cloudfront create-distribution \
  --distribution-config file://distribution-config.json
```

`distribution-config.json`에 전체 설정을 담는다.

### S3 오리진 구성

정적 파일(HTML, CSS, JS, 이미지)을 S3에 저장하고 CloudFront로 제공하는 패턴이다. 정적 사이트 호스팅의 표준 방식이다.

S3 오리진을 설정할 때 두 가지 방식이 있다.

**S3 웹사이트 엔드포인트**: S3 버킷의 정적 웹사이트 호스팅 기능을 활성화하고 엔드포인트를 오리진으로 지정한다. HTTP만 지원하고 버킷을 퍼블릭으로 설정해야 한다.

**S3 REST API 엔드포인트 + OAC**: 버킷을 비공개로 유지하면서 CloudFront만 접근 가능하게 한다. 현재 권장 방식이다.

### OAC(Origin Access Control)

OAC는 CloudFront만 S3 버킷에 접근할 수 있도록 제한하는 설정이다. 기존의 OAI(Origin Access Identity)를 대체하는 최신 방식이다.

```bash
# OAC 생성
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "my-oac",
    "Description": "OAC for S3 bucket",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }'
```

OAC를 생성한 후 CloudFront 배포의 오리진 설정에 연결한다. 그리고 S3 버킷 정책에 CloudFront 서비스 프린시플이 접근할 수 있도록 허용 구문을 추가한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789:distribution/EDFDVBD6EXAMPLE"
        }
      }
    }
  ]
}
```

`AWS:SourceArn`에 실제 CloudFront 배포 ARN을 넣는다. 특정 배포만 허용하므로 다른 CloudFront 배포가 이 버킷에 접근하는 것을 막는다.

### ALB 오리진 구성

동적 애플리케이션(API 서버, 서버사이드 렌더링 등)은 ALB를 오리진으로 설정한다. CloudFront가 HTTPS 종료 지점이 되고 ALB까지는 HTTP로 통신하는 구성도 가능하다. 하지만 ALB에도 HTTPS를 적용하는 것이 보안상 권장된다.

ALB 오리진을 설정할 때 커스텀 헤더를 추가할 수 있다. ALB는 해당 헤더가 있는 요청만 처리하도록 설정해 CloudFront를 우회한 직접 접근을 차단한다.

```json
{
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "alb-origin",
      "DomainName": "my-alb-1234567890.ap-northeast-2.elb.amazonaws.com",
      "CustomOriginConfig": {
        "HTTPSPort": 443,
        "OriginProtocolPolicy": "https-only"
      },
      "CustomHeaders": {
        "Quantity": 1,
        "Items": [{
          "HeaderName": "X-Custom-Header",
          "HeaderValue": "my-secret-value"
        }]
      }
    }]
  }
}
```

### 캐시 정책(Cache Policy)

캐시 정책은 CloudFront가 응답을 캐시하는 방식을 정의한다. TTL(Time to Live), 캐시 키 구성 요소(헤더, 쿼리 문자열, 쿠키)를 설정한다.

AWS 관리형 캐시 정책을 활용하면 편리하다.

| 정책 이름 | 용도 |
|---|---|
| `CachingOptimized` | 정적 파일 최적화 (기본 권장) |
| `CachingDisabled` | 캐시 비활성화 (동적 API) |
| `CachingOptimizedForUncompressedObjects` | 압축 안 하는 파일 |

정적 사이트라면 `CachingOptimized`를 쓰고, API 응답이라면 `CachingDisabled`를 쓰는 식으로 경로(path pattern)별로 다른 캐시 정책을 적용할 수 있다.

```bash
# 관리형 캐시 정책 ID 확인
aws cloudfront list-cache-policies \
  --type managed \
  --query 'CachePolicyList.Items[].{Name:CachePolicy.CachePolicyConfig.Name,Id:CachePolicy.Id}'
```

### 캐시 무효화(Invalidation)

파일을 S3에 새로 업로드해도 CloudFront는 이전 캐시를 계속 제공한다. 변경 내용을 즉시 반영하려면 캐시를 무효화해야 한다.

```bash
# 전체 무효화
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*"

# 특정 파일만 무효화
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/index.html" "/css/main.css"
```

`/*`는 모든 파일을 무효화한다. 무효화는 요청 수에 따라 비용이 발생한다. 매월 1,000개 경로까지 무료다.

배포 파이프라인에서 매번 `/*`로 전체 무효화하면 비용과 성능 모두 비효율적이다. 변경된 파일만 선택적으로 무효화하거나, 파일명에 해시를 붙여(`main.a1b2c3.css`) 캐시 버스팅을 구현하는 방식이 낫다.

### HTTPS 및 커스텀 도메인 설정

CloudFront 배포에 커스텀 도메인과 ACM 인증서를 연결한다.

```json
{
  "Aliases": {
    "Quantity": 2,
    "Items": ["example.com", "www.example.com"]
  },
  "ViewerCertificate": {
    "ACMCertificateArn": "arn:aws:acm:us-east-1:123456789:certificate/abc123",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  }
}
```

`SSLSupportMethod`는 `sni-only`를 권장한다. `vip` 방식은 전용 IP를 할당해 별도 비용이 발생한다. 현대 브라우저는 모두 SNI를 지원하므로 `sni-only`로 충분하다.

`MinimumProtocolVersion`은 `TLSv1.2_2021`로 설정한다. TLS 1.0, 1.1은 보안 취약점이 있어 지원을 끊는 것이 좋다.

HTTP 요청을 HTTPS로 리다이렉트하려면 뷰어 프로토콜 정책을 설정한다.

```json
{
  "DefaultCacheBehavior": {
    "ViewerProtocolPolicy": "redirect-to-https"
  }
}
```

`allow-all`은 HTTP/HTTPS 모두 허용, `redirect-to-https`는 HTTP를 HTTPS로 301 리다이렉트, `https-only`는 HTTP 접근 자체를 거부한다.

---

## 정적 사이트 배포 파이프라인

S3 + CloudFront + Route 53을 조합해 정적 사이트를 배포하는 전체 흐름을 정리한다.

### 아키텍처 개요

```
사용자 브라우저
    ↓ (HTTPS)
Route 53 (example.com → CloudFront Alias)
    ↓
CloudFront (HTTPS 종료, 캐시)
    ↓ (S3 API 요청)
S3 버킷 (정적 파일 저장, 비공개)
```

브라우저는 Route 53을 통해 CloudFront로 연결된다. CloudFront는 HTTPS를 처리하고 캐시에서 응답하거나 S3에서 파일을 가져온다. S3 버킷은 OAC로 보호되어 직접 접근이 차단된다.

### 단계별 구성

**1단계: S3 버킷 생성**

```bash
aws s3api create-bucket \
  --bucket my-static-site \
  --region ap-northeast-2 \
  --create-bucket-configuration LocationConstraint=ap-northeast-2

# 퍼블릭 액세스 차단 (OAC 사용하므로 비공개 유지)
aws s3api put-public-access-block \
  --bucket my-static-site \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

**2단계: ACM 인증서 발급 (us-east-1)**

```bash
aws acm request-certificate \
  --domain-name example.com \
  --subject-alternative-names "www.example.com" \
  --validation-method DNS \
  --region us-east-1
```

Route 53에서 DNS 검증 레코드를 생성하고 인증서 상태가 `ISSUED`가 될 때까지 기다린다.

**3단계: CloudFront 배포 생성**

OAC를 생성하고, S3 버킷을 오리진으로 설정한 CloudFront 배포를 만든다. ACM 인증서를 연결하고 커스텀 도메인(`example.com`, `www.example.com`)을 추가한다.

**4단계: S3 버킷 정책 업데이트**

CloudFront 배포 ID를 확인한 후 S3 버킷 정책에 CloudFront 접근을 허용하는 구문을 추가한다.

```bash
aws s3api put-bucket-policy \
  --bucket my-static-site \
  --policy file://bucket-policy.json
```

**5단계: Route 53 레코드 등록**

CloudFront 배포 도메인(`d1234abcd.cloudfront.net`)을 가리키는 Alias 레코드를 등록한다.

```bash
# example.com → CloudFront
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "example.com",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z2FDTNDATAQYW2",
            "DNSName": "d1234abcd.cloudfront.net",
            "EvaluateTargetHealth": false
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "www.example.com",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z2FDTNDATAQYW2",
            "DNSName": "d1234abcd.cloudfront.net",
            "EvaluateTargetHealth": false
          }
        }
      }
    ]
  }'
```

**6단계: 파일 업로드 및 무효화**

```bash
# 빌드 결과물 S3에 동기화
aws s3 sync ./dist s3://my-static-site --delete

# CloudFront 캐시 무효화
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*"
```

### CI/CD 파이프라인에서 자동화

GitHub Actions로 배포를 자동화하는 예시다.

```yaml
name: Deploy to S3 + CloudFront

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: npm ci && npm run build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2

      - name: Deploy to S3
        run: aws s3 sync ./dist s3://my-static-site --delete

      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

IAM 사용자를 만들고 필요한 최소 권한만 부여한다. S3 버킷 접근과 CloudFront 무효화 생성 권한만 있으면 된다.

### 주의사항: SPA 라우팅

React, Vue 같은 SPA(Single Page Application)는 클라이언트 사이드 라우팅을 사용한다. `/about` 경로로 직접 접근하면 S3에 해당 파일이 없어 403 또는 404 오류가 발생한다.

두 가지 해결 방법이 있다.

**방법 1**: CloudFront 오류 페이지 설정에서 403, 404 응답을 `/index.html`로 리다이렉트한다. 응답 코드는 200으로 반환한다.

```json
{
  "CustomErrorResponses": {
    "Quantity": 2,
    "Items": [
      {
        "ErrorCode": 403,
        "ResponsePagePath": "/index.html",
        "ResponseCode": "200",
        "ErrorCachingMinTTL": 0
      },
      {
        "ErrorCode": 404,
        "ResponsePagePath": "/index.html",
        "ResponseCode": "200",
        "ErrorCachingMinTTL": 0
      }
    ]
  }
}
```

**방법 2**: CloudFront Functions 또는 Lambda@Edge를 사용해 요청 경로를 `/index.html`로 재작성한다. 더 세밀한 제어가 필요할 때 사용한다.

---

## 정리

Route 53은 DNS 관리와 라우팅 정책을 담당한다. 호스팅 존에 레코드를 등록하고, Alias 레코드로 AWS 리소스를 연결한다.

ACM은 SSL/TLS 인증서를 무료로 발급하고 자동 갱신한다. CloudFront용 인증서는 반드시 us-east-1에서 발급해야 한다.

CloudFront는 CDN 역할과 함께 HTTPS 종료 지점이 된다. OAC로 S3 버킷을 보호하고, 캐시 정책으로 성능을 최적화한다.

세 서비스를 조합하면 비용 효율적이고 확장 가능한 정적 사이트 인프라를 구성할 수 있다. 다음 편에서는 EC2와 ALB를 조합한 동적 애플리케이션 배포를 다룬다.

---

## 참고 자료

- [AWS Route 53 개발자 가이드](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/)
- [AWS Certificate Manager 사용 설명서](https://docs.aws.amazon.com/acm/latest/userguide/)
- [Amazon CloudFront 개발자 가이드](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/)
- [CloudFront OAC 설정 가이드](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- [Route 53 라우팅 정책](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)
- [ACM DNS 검증](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html)
