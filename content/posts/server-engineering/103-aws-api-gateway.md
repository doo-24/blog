---
title: "[AWS 실전 구축] 4편 — API 관리: API Gateway 실전 구성"
date: 2026-03-17T11:05:00+09:00
draft: false
tags: ["API Gateway", "Lambda Authorizer", "스로틀링", "AWS", "서버"]
series: ["AWS 실전 구축"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "REST API vs HTTP API 선택 기준, Cognito/Lambda Authorizer 인증 통합, 요청 검증과 스로틀링·사용량 플랜, API Gateway+Lambda/ALB(ECS) 조합 패턴까지"
---

API Gateway는 AWS에서 HTTP 엔드포인트를 외부에 노출하는 가장 빠른 방법이다. 하지만 "그냥 만들면 되지"라는 생각으로 접근하면 인증 구멍, 스로틀링 미설정, 과도한 비용 같은 문제가 쌓인다. 이번 편에서는 API 유형 선택부터 인증 통합, 트래픽 제어, 백엔드 연결 패턴까지 실전에서 바로 쓸 수 있는 수준으로 정리한다.

---

## REST API vs HTTP API: 무엇을 선택할까

API Gateway에는 세 가지 제품이 있다. REST API, HTTP API, WebSocket API다.

WebSocket은 실시간 양방향 통신 전용이므로 일반 HTTP 서비스라면 REST API와 HTTP API 중 하나를 고른다.

**HTTP API가 더 싼 이유**는 기능을 줄인 덕분이다. REST API 대비 요청당 단가가 약 71% 저렴하고 지연 시간도 낮다.

반면 REST API는 기능이 훨씬 많다.

| 기능 | REST API | HTTP API |
|---|---|---|
| 요청/응답 변환 (Mapping Template) | O | X |
| API 키 + 사용량 플랜 | O | X |
| WAF 통합 | O | X |
| 캐싱 | O | X |
| Private 통합 (VPC Link v1) | O | O (v2) |
| JWT Authorizer | X | O (내장) |
| Lambda Authorizer | O | O |
| Cognito Authorizer | O | Lambda 경유 |

결론은 간단하다.

- **단순 Lambda 프록시 + JWT 인증** → HTTP API
- **Mapping Template, API 키, WAF, 캐싱 중 하나라도 필요** → REST API

### HTTP API 생성 예시 (AWS CLI)

```bash
# HTTP API 생성
aws apigatewayv2 create-api \
  --name "my-http-api" \
  --protocol-type HTTP \
  --cors-configuration \
    AllowOrigins="https://app.example.com",AllowMethods="GET,POST,DELETE",AllowHeaders="Authorization,Content-Type" \
  --region ap-northeast-2
```

응답에서 `ApiId`를 받아 이후 라우트와 통합에 사용한다.

```bash
# Lambda 통합 생성
aws apigatewayv2 create-integration \
  --api-id <API_ID> \
  --integration-type AWS_PROXY \
  --integration-uri arn:aws:lambda:ap-northeast-2:<ACCOUNT_ID>:function:my-function \
  --payload-format-version "2.0"
```

`payload-format-version 2.0`은 HTTP API 전용 포맷이다. Lambda 함수 코드도 v2 포맷에 맞게 작성해야 한다. v1에서는 `event.pathParameters`, `event.queryStringParameters`로 분리되어 있던 요청 정보가 v2에서는 `event.requestContext.http.method`, `event.rawQueryString` 등 구조가 달라지므로, 기존 REST API용 핸들러를 그대로 재사용하면 동작하지 않는다.

---

## 스테이지와 커스텀 도메인

### 스테이지 관리

REST API는 스테이지(stage)를 명시적으로 생성한다. `dev`, `staging`, `prod`를 분리하거나, 단일 API에서 스테이지 변수로 백엔드 엔드포인트를 교체하는 방식 모두 쓰인다.

```bash
# 스테이지 생성 (REST API)
aws apigateway create-stage \
  --rest-api-id <REST_API_ID> \
  --stage-name prod \
  --deployment-id <DEPLOYMENT_ID> \
  --variables "backendUrl=https://internal.example.com"
```

스테이지 변수 `${stageVariables.backendUrl}`을 통합 URI에 참조하면 환경별 백엔드를 스테이지 하나로 관리한다.

### 커스텀 도메인 연결

기본 URL은 `https://<api-id>.execute-api.ap-northeast-2.amazonaws.com/prod` 형태다. 이를 `api.example.com`으로 바꾸려면 커스텀 도메인을 설정한다.

```bash
# ACM 인증서 ARN 확인 (us-east-1이 아닌 리전에서 사용 시 해당 리전 인증서 필요)
CERT_ARN=$(aws acm list-certificates \
  --region ap-northeast-2 \
  --query "CertificateSummaryList[?DomainName=='*.example.com'].CertificateArn" \
  --output text)

# 커스텀 도메인 생성
aws apigateway create-domain-name \
  --domain-name api.example.com \
  --regional-certificate-arn $CERT_ARN \
  --endpoint-configuration types=REGIONAL

# API 매핑
aws apigateway create-base-path-mapping \
  --domain-name api.example.com \
  --rest-api-id <REST_API_ID> \
  --stage prod \
  --base-path v1
```

이후 Route 53에서 `api.example.com` → 도메인 네임의 `regionalDomainName`으로 ALIAS 레코드를 추가한다.

---

## 인증 통합: Cognito Authorizer와 Lambda Authorizer

### Cognito Authorizer (REST API)

Cognito User Pool을 직접 연결해 JWT 토큰을 검증한다. 별도 코드 없이 콘솔 또는 CLI로 설정한다.

```bash
# Cognito Authorizer 생성
aws apigateway create-authorizer \
  --rest-api-id <REST_API_ID> \
  --name "CognitoAuth" \
  --type COGNITO_USER_POOLS \
  --identity-source "method.request.header.Authorization" \
  --provider-arns "arn:aws:cognito-idp:ap-northeast-2:<ACCOUNT_ID>:userpool/<USER_POOL_ID>"
```

메서드에 Authorizer를 연결하면 API Gateway가 토큰 서명, 만료, issuer를 자동으로 검증한다.

클라이언트는 Cognito에서 받은 `id_token` 또는 `access_token`을 `Authorization` 헤더에 Bearer 형식으로 전달하면 된다.

### Lambda Authorizer: 커스텀 인증 로직

서드파티 JWT, API 키 검증, IP 화이트리스트 등 Cognito로 커버되지 않는 인증 로직은 Lambda Authorizer로 처리한다.

Lambda Authorizer는 두 가지 유형이 있다.

- **TOKEN 유형**: `Authorization` 헤더 값 하나만 Lambda에 전달
- **REQUEST 유형**: 헤더, 쿼리 파라미터, 스테이지 변수 모두 전달

다음은 TOKEN 유형의 Python Lambda Authorizer 예시다.

```python
import json
import jwt  # PyJWT 라이브러리

SECRET_KEY = "your-secret-key"  # 실제로는 Secrets Manager에서 조회

def lambda_handler(event, context):
    token = event.get("authorizationToken", "")
    method_arn = event.get("methodArn")

    # "Bearer <token>" 형식 처리
    if token.startswith("Bearer "):
        token = token[7:]

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        principal_id = payload.get("sub", "user")
        effect = "Allow"
    except jwt.ExpiredSignatureError:
        # 만료된 토큰 → 명시적 Deny
        principal_id = "unauthorized"
        effect = "Deny"
    except jwt.InvalidTokenError:
        raise Exception("Unauthorized")  # 401 반환

    return build_policy(principal_id, effect, method_arn)


def build_policy(principal_id: str, effect: str, resource: str) -> dict:
    return {
        "principalId": principal_id,
        "policyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Action": "execute-api:Invoke",
                    "Effect": effect,
                    "Resource": resource,
                }
            ],
        },
        # context 필드는 백엔드 Lambda에 $context.authorizer.* 로 전달됨
        "context": {
            "userId": principal_id,
            "role": "admin",
        },
    }
```

Lambda Authorizer 등록 시 `resultTtlInSeconds`를 설정하면 동일 토큰의 정책을 캐싱한다. 300초(5분)가 일반적이며, 0으로 설정하면 캐싱을 비활성화한다.

```bash
aws apigateway create-authorizer \
  --rest-api-id <REST_API_ID> \
  --name "LambdaTokenAuth" \
  --type TOKEN \
  --identity-source "method.request.header.Authorization" \
  --authorizer-uri "arn:aws:apigateway:ap-northeast-2:lambda:path/2015-03-31/functions/arn:aws:lambda:ap-northeast-2:<ACCOUNT_ID>:function:my-authorizer/invocations" \
  --authorizer-result-ttl-in-seconds 300
```

Lambda Authorizer를 사용하면 API Gateway가 Lambda를 호출할 권한도 부여해야 한다.

```bash
aws lambda add-permission \
  --function-name my-authorizer \
  --statement-id apigw-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:ap-northeast-2:<ACCOUNT_ID>:<REST_API_ID>/authorizers/*"
```

### HTTP API의 JWT Authorizer

HTTP API는 별도 Lambda 없이 JWT를 검증하는 내장 JWT Authorizer를 제공한다.

```bash
aws apigatewayv2 create-authorizer \
  --api-id <API_ID> \
  --authorizer-type JWT \
  --name "JwtAuth" \
  --identity-source '$request.header.Authorization' \
  --jwt-configuration \
    Audience=<CLIENT_ID>,Issuer=https://cognito-idp.ap-northeast-2.amazonaws.com/<USER_POOL_ID>
```

Cognito뿐 아니라 Auth0, Okta 등 표준 OIDC issuer라면 모두 연결 가능하다.

---

## 요청 검증

API Gateway는 백엔드를 호출하기 전에 요청 구조를 검증할 수 있다. 잘못된 요청을 Lambda 호출 전에 차단하면 비용과 지연 시간 모두 줄인다.

### 요청 모델 정의 (REST API)

```bash
# JSON Schema 모델 생성
aws apigateway create-model \
  --rest-api-id <REST_API_ID> \
  --name "CreateUserRequest" \
  --content-type "application/json" \
  --schema '{
    "$schema": "http://json-schema.org/draft-04/schema#",
    "type": "object",
    "required": ["email", "name"],
    "properties": {
      "email": { "type": "string", "format": "email" },
      "name": { "type": "string", "minLength": 1, "maxLength": 100 },
      "age": { "type": "integer", "minimum": 0, "maximum": 150 }
    },
    "additionalProperties": false
  }'
```

메서드에 Request Validator를 설정해 본문 + 파라미터를 동시에 검증한다.

```bash
# Validator 생성 (본문 + 파라미터 모두 검증)
aws apigateway create-request-validator \
  --rest-api-id <REST_API_ID> \
  --name "ValidateAll" \
  --validate-request-body \
  --validate-request-parameters

# 메서드에 적용
aws apigateway update-method \
  --rest-api-id <REST_API_ID> \
  --resource-id <RESOURCE_ID> \
  --http-method POST \
  --patch-operations \
    op=replace,path=/requestValidatorId,value=<VALIDATOR_ID>
```

검증 실패 시 API Gateway가 직접 `400 Bad Request`를 반환한다. Lambda는 호출되지 않는다.

---

## 스로틀링과 사용량 플랜

### 스로틀링 기본 개념

API Gateway의 스로틀링은 두 단계로 동작한다.

1. **계정 레벨**: 리전당 기본 10,000 RPS (버스트 5,000)
2. **스테이지/메서드 레벨**: 개별 API 또는 메서드에 별도 제한 설정

스테이지 레벨 스로틀링 설정은 콘솔 또는 CLI로 한다.

```bash
aws apigateway update-stage \
  --rest-api-id <REST_API_ID> \
  --stage-name prod \
  --patch-operations \
    op=replace,path=/defaultRouteSettings/throttlingRateLimit,value=1000 \
    op=replace,path=/defaultRouteSettings/throttlingBurstLimit,value=500
```

특정 메서드에만 엄격한 제한이 필요하면 메서드 오버라이드를 사용한다.

```bash
# POST /users 메서드만 100 RPS로 제한
aws apigateway update-stage \
  --rest-api-id <REST_API_ID> \
  --stage-name prod \
  --patch-operations \
    op=replace,path=/~1users/POST/throttling/rateLimit,value=100 \
    op=replace,path=/~1users/POST/throttling/burstLimit,value=50
```

### API 키와 사용량 플랜 (REST API)

외부 파트너나 서드파티 개발자에게 API를 제공할 때 API 키로 트래픽을 제어한다.

```bash
# API 키 생성
aws apigateway create-api-key \
  --name "partner-a-key" \
  --enabled

# 사용량 플랜 생성 (월 10,000 요청, 초당 50 RPS)
aws apigateway create-usage-plan \
  --name "PartnerPlan" \
  --api-stages apiId=<REST_API_ID>,stage=prod \
  --throttle rateLimit=50,burstLimit=25 \
  --quota limit=10000,period=MONTH

# API 키를 사용량 플랜에 연결
aws apigateway create-usage-plan-key \
  --usage-plan-id <PLAN_ID> \
  --key-id <KEY_ID> \
  --key-type API_KEY
```

클라이언트는 `x-api-key: <key>` 헤더를 포함해 요청한다. 메서드에서 API 키 필수 옵션을 켜야 적용된다.

```bash
aws apigateway update-method \
  --rest-api-id <REST_API_ID> \
  --resource-id <RESOURCE_ID> \
  --http-method GET \
  --patch-operations op=replace,path=/apiKeyRequired,value=true
```

---

## API Gateway + Lambda 조합 패턴

Lambda 통합은 API Gateway의 가장 일반적인 사용 패턴이다.

### Lambda 프록시 통합

REST API에서 `AWS_PROXY` 통합을 사용하면 요청 전체를 Lambda에 그대로 전달한다.

```python
# Lambda 함수 (Python, REST API v1 포맷)
import json

def lambda_handler(event, context):
    http_method = event["httpMethod"]
    path = event["path"]
    headers = event.get("headers", {})
    body = json.loads(event.get("body") or "{}")

    # 비즈니스 로직
    if http_method == "POST" and path == "/users":
        user = create_user(body)
        return {
            "statusCode": 201,
            "headers": {
                "Content-Type": "application/json",
                "X-Request-Id": context.aws_request_id,
            },
            "body": json.dumps(user),
        }

    return {
        "statusCode": 404,
        "body": json.dumps({"error": "Not found"}),
    }
```

HTTP API는 payload format v2를 사용하므로 `event` 구조가 다르다.

```python
# Lambda 함수 (HTTP API v2 포맷)
import json

def lambda_handler(event, context):
    method = event["requestContext"]["http"]["method"]
    path = event["requestContext"]["http"]["path"]
    body = json.loads(event.get("body") or "{}")

    return {
        "statusCode": 200,
        "body": json.dumps({"message": "ok"}),
    }
```

### 비동기 Lambda 호출

장시간 처리(이미지 변환, 대용량 파일 처리)는 API Gateway 타임아웃(29초) 이내에 끝나지 않는다. 이럴 때는 SQS 또는 EventBridge를 거쳐 비동기로 처리한다.

```python
# API Lambda: 작업을 SQS에 넣고 즉시 202 반환
import boto3, json, uuid

sqs = boto3.client("sqs")
QUEUE_URL = "https://sqs.ap-northeast-2.amazonaws.com/<ACCOUNT_ID>/job-queue"

def lambda_handler(event, context):
    job_id = str(uuid.uuid4())
    payload = json.loads(event.get("body") or "{}")

    sqs.send_message(
        QueueUrl=QUEUE_URL,
        MessageBody=json.dumps({"jobId": job_id, **payload}),
    )

    return {
        "statusCode": 202,
        "body": json.dumps({"jobId": job_id, "status": "queued"}),
    }
```

클라이언트는 `jobId`로 별도 GET 엔드포인트를 폴링하거나 WebSocket으로 완료 알림을 받는다.

---

## API Gateway + ALB(ECS) 조합 패턴

컨테이너 기반 백엔드(ECS, EKS)는 ALB를 앞에 두고 API Gateway를 그 위에 얹는 구조를 자주 사용한다.

### 왜 이 조합을 쓰는가

- API Gateway: 인증, 스로틀링, WAF, 커스텀 도메인 처리
- ALB: ECS 태스크로의 로드 밸런싱, 헬스 체크, 타겟 그룹 관리

API Gateway가 퍼블릭 진입점이 되고, ALB는 VPC 내부에 Private으로 위치한다. ALB에 직접 접근하는 경로를 차단하면 모든 트래픽이 API Gateway를 통과한다.

### VPC Link 설정

REST API는 VPC Link v1을, HTTP API는 VPC Link v2를 사용한다.

```bash
# VPC Link 생성 (HTTP API용 v2)
aws apigatewayv2 create-vpc-link \
  --name "ecs-vpc-link" \
  --subnet-ids subnet-aaa subnet-bbb \
  --security-group-ids sg-vpc-link

# HTTP API 통합 생성 (ALB 연결)
aws apigatewayv2 create-integration \
  --api-id <API_ID> \
  --integration-type HTTP_PROXY \
  --integration-method ANY \
  --integration-uri "arn:aws:elasticloadbalancing:ap-northeast-2:<ACCOUNT_ID>:listener/app/my-alb/<ALB_ID>/<LISTENER_ID>" \
  --connection-type VPC_LINK \
  --connection-id <VPC_LINK_ID> \
  --payload-format-version "1.0"
```

VPC Link를 통하면 트래픽이 인터넷을 경유하지 않고 AWS 내부 네트워크로 전달된다.

### ALB 보안 그룹 설정

ALB의 인바운드 규칙은 VPC Link 보안 그룹에서 오는 트래픽만 허용한다.

```bash
# ALB 보안 그룹: VPC Link SG에서만 80/443 허용
aws ec2 authorize-security-group-ingress \
  --group-id sg-alb \
  --protocol tcp \
  --port 80 \
  --source-group sg-vpc-link

# 퍼블릭 인터넷에서 ALB 직접 접근 차단 확인
aws ec2 describe-security-groups \
  --group-ids sg-alb \
  --query "SecurityGroups[0].IpPermissions"
```

이 구성에서 ALB는 Internet-facing이 아닌 Internal 타입으로 생성해야 한다.

### 헤더 전달과 백엔드 식별

API Gateway를 통과한 요청임을 백엔드에서 확인하려면 커스텀 헤더를 추가한다.

```bash
# API Gateway → ALB로 커스텀 헤더 추가 (Mapping Template, REST API)
# Integration Request → Mapping Template 설정
{
  "method": "$context.httpMethod",
  "path": "$context.resourcePath",
  "sourceIp": "$context.identity.sourceIp",
  "requestId": "$context.requestId"
}
```

백엔드 ECS 서비스는 `X-Forwarded-For` 또는 커스텀 헤더로 원본 IP와 요청 ID를 로깅한다.

---

## CloudWatch 모니터링 설정

API Gateway의 기본 메트릭은 자동 수집된다. 중요한 것들을 CloudWatch 경보로 연결한다.

```bash
# 5XX 오류율 경보
aws cloudwatch put-metric-alarm \
  --alarm-name "apigw-5xx-high" \
  --metric-name 5XXError \
  --namespace AWS/ApiGateway \
  --dimensions Name=ApiName,Value=my-rest-api Name=Stage,Value=prod \
  --statistic Average \
  --period 60 \
  --threshold 0.05 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions <SNS_TOPIC_ARN>

# 지연 시간 P99 경보
aws cloudwatch put-metric-alarm \
  --alarm-name "apigw-latency-p99" \
  --metric-name IntegrationLatency \
  --namespace AWS/ApiGateway \
  --dimensions Name=ApiName,Value=my-rest-api Name=Stage,Value=prod \
  --extended-statistics p99 \
  --period 60 \
  --threshold 3000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions <SNS_TOPIC_ARN>
```

Access Logging을 활성화하면 요청별 상세 정보를 CloudWatch Logs에 저장한다.

```bash
aws apigateway update-stage \
  --rest-api-id <REST_API_ID> \
  --stage-name prod \
  --patch-operations \
    op=replace,path=/accessLogSettings/destinationArn,value=arn:aws:logs:ap-northeast-2:<ACCOUNT_ID>:log-group:/aws/apigateway/prod \
    op=replace,path=/accessLogSettings/format,value='{"requestId":"$context.requestId","ip":"$context.identity.sourceIp","caller":"$context.identity.caller","user":"$context.identity.user","requestTime":"$context.requestTime","httpMethod":"$context.httpMethod","resourcePath":"$context.resourcePath","status":"$context.status","protocol":"$context.protocol","responseLength":"$context.responseLength"}'
```

---

## 정리: 선택 기준 요약

실전에서 API Gateway 설계 결정은 세 가지 질문으로 좁힌다.

첫째, **비용이 최우선인가?** 그렇다면 HTTP API를 선택하고 Lambda Authorizer나 JWT Authorizer로 인증을 처리한다.

둘째, **기존 REST API 기능이 필요한가?** WAF, 사용량 플랜, Mapping Template 중 하나라도 필요하면 REST API를 선택한다.

셋째, **백엔드가 컨테이너인가?** VPC Link + ALB 조합으로 내부 트래픽만 허용하고, API Gateway가 모든 진입 제어를 담당하게 설계한다.

아키텍처는 단순할수록 운영이 쉽다. API Gateway의 기능을 과도하게 쌓기보다는 필요한 기능만 켜고, 나머지는 백엔드에서 처리하는 분업이 장기적으로 유리하다.

---

## 참고 자료

- [AWS API Gateway 공식 문서](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
- [HTTP API vs REST API 비교](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html)
- [Lambda Authorizer 가이드](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html)
- [API Gateway 사용량 플랜 설정](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html)
- [VPC Link를 이용한 프라이빗 통합](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vpc-links.html)
- [API Gateway 스로틀링 제한](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)
