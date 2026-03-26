---
title: "[AWS 실전 구축] 3편 — 인증 인프라: Cognito로 사용자 관리와 인증 구축"
date: 2026-03-17T11:06:00+09:00
draft: false
tags: ["Cognito", "User Pool", "Identity Pool", "OAuth", "AWS", "서버"]
series: ["AWS 실전 구축"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "User Pool 설정(회원가입, MFA, 이메일 검증), App Client와 OAuth 2.0 플로우 연동, Identity Pool과 임시 자격 증명, Lambda 트리거 커스터마이징까지"
---

서비스를 만들면 가장 먼저 부딪히는 문제가 인증이다. 직접 구현하면 비밀번호 해싱, 세션 관리, 토큰 갱신, MFA 연동까지 수백 줄의 코드와 보안 리스크를 껴안아야 한다. AWS Cognito는 이 복잡성을 관리형 서비스로 위임할 수 있게 해준다.

---

## Cognito 구성 요소 개요

Cognito는 크게 두 가지 개념으로 나뉜다.

**User Pool**은 사용자 디렉터리다. 회원가입, 로그인, 비밀번호 정책, MFA, 이메일/SMS 검증을 담당한다.

**Identity Pool**은 AWS 리소스 접근용 임시 자격 증명을 발급한다. 인증된 사용자가 S3 버킷이나 DynamoDB에 직접 접근할 때 사용한다.

대부분의 웹/앱 서비스는 User Pool만으로 충분하다. Identity Pool은 AWS 리소스에 직접 접근이 필요할 때 추가한다.

---

## 1. User Pool 생성

### AWS CLI로 User Pool 생성

```bash
aws cognito-idp create-user-pool \
  --pool-name "MyAppUserPool" \
  --policies '{
    "PasswordPolicy": {
      "MinimumLength": 8,
      "RequireUppercase": true,
      "RequireLowercase": true,
      "RequireNumbers": true,
      "RequireSymbols": false,
      "TemporaryPasswordValidityDays": 7
    }
  }' \
  --auto-verified-attributes email \
  --username-attributes email \
  --mfa-configuration "OPTIONAL" \
  --user-pool-tags '{"Environment": "production"}' \
  --region ap-northeast-2
```

`--username-attributes email`을 지정하면 사용자가 이메일로 로그인한다. 기본값은 username이다.

`--auto-verified-attributes email`은 가입 시 이메일 인증 코드를 자동 발송한다.

### 응답에서 Pool ID 확인

```bash
# 생성 후 Pool ID 저장
USER_POOL_ID=$(aws cognito-idp list-user-pools \
  --max-results 10 \
  --query "UserPools[?Name=='MyAppUserPool'].Id" \
  --output text \
  --region ap-northeast-2)

echo $USER_POOL_ID
# ap-northeast-2_AbcDef123
```

---

## 2. 회원가입 흐름과 이메일 검증

### 사용자 속성(Attribute) 설계

Cognito는 기본 속성(email, phone_number, name 등)과 커스텀 속성을 지원한다.

커스텀 속성 추가 예시다.

```bash
aws cognito-idp add-custom-attributes \
  --user-pool-id $USER_POOL_ID \
  --custom-attributes '[
    {
      "Name": "role",
      "AttributeDataType": "String",
      "Mutable": true,
      "Required": false,
      "StringAttributeConstraints": {
        "MinLength": "1",
        "MaxLength": "50"
      }
    },
    {
      "Name": "company_id",
      "AttributeDataType": "String",
      "Mutable": true,
      "Required": false
    }
  ]' \
  --region ap-northeast-2
```

커스텀 속성은 `custom:` 접두사가 붙는다. 토큰에서 `custom:role`로 참조한다.

### Python SDK로 회원가입

```python
import boto3
from botocore.exceptions import ClientError

cognito = boto3.client('cognito-idp', region_name='ap-northeast-2')

def sign_up(email: str, password: str, name: str) -> dict:
    try:
        response = cognito.sign_up(
            ClientId='YOUR_APP_CLIENT_ID',
            Username=email,
            Password=password,
            UserAttributes=[
                {'Name': 'email', 'Value': email},
                {'Name': 'name', 'Value': name},
                {'Name': 'custom:role', 'Value': 'user'},
            ]
        )
        return {
            'user_sub': response['UserSub'],
            'confirmed': response['UserConfirmed'],
        }
    except ClientError as e:
        error_code = e.response['Error']['Code']
        if error_code == 'UsernameExistsException':
            raise ValueError('이미 등록된 이메일입니다.')
        elif error_code == 'InvalidPasswordException':
            raise ValueError('비밀번호 정책을 충족하지 않습니다.')
        raise
```

### 이메일 인증 코드 확인

```python
def confirm_sign_up(email: str, code: str) -> bool:
    try:
        cognito.confirm_sign_up(
            ClientId='YOUR_APP_CLIENT_ID',
            Username=email,
            ConfirmationCode=code,
        )
        return True
    except ClientError as e:
        error_code = e.response['Error']['Code']
        if error_code == 'CodeMismatchException':
            raise ValueError('인증 코드가 올바르지 않습니다.')
        elif error_code == 'ExpiredCodeException':
            raise ValueError('인증 코드가 만료되었습니다. 재발송을 요청하세요.')
        raise

def resend_confirmation_code(email: str):
    cognito.resend_confirmation_code(
        ClientId='YOUR_APP_CLIENT_ID',
        Username=email,
    )
```

---

## 3. MFA 설정

### TOTP(소프트웨어 MFA) 활성화

User Pool 수준에서 MFA를 선택적으로 허용하고, 사용자가 직접 활성화하는 흐름이 일반적이다.

```python
def setup_totp(access_token: str) -> str:
    """TOTP 설정 시작 — QR 코드용 시크릿 반환"""
    response = cognito.associate_software_token(
        AccessToken=access_token,
    )
    return response['SecretCode']  # Google Authenticator에 등록할 시크릿

def verify_totp_setup(access_token: str, totp_code: str, device_name: str = 'MyPhone'):
    """TOTP 코드 검증 후 활성화"""
    cognito.verify_software_token(
        AccessToken=access_token,
        UserCode=totp_code,
        FriendlyDeviceName=device_name,
    )
    # MFA를 기본 방식으로 설정
    cognito.set_user_mfa_preference(
        AccessToken=access_token,
        SoftwareTokenMfaSettings={
            'Enabled': True,
            'PreferredMfa': True,
        }
    )
```

### MFA 챌린지 처리 로그인 흐름

MFA가 활성화된 사용자는 로그인 시 2단계 챌린지를 거친다.

```python
def login_with_mfa(email: str, password: str, totp_code: str | None = None) -> dict:
    # 1단계: 이메일/비밀번호 제출
    response = cognito.initiate_auth(
        ClientId='YOUR_APP_CLIENT_ID',
        AuthFlow='USER_PASSWORD_AUTH',
        AuthParameters={
            'USERNAME': email,
            'PASSWORD': password,
        }
    )

    # MFA 챌린지가 있는 경우
    if response.get('ChallengeName') == 'SOFTWARE_TOKEN_MFA':
        if not totp_code:
            return {'challenge': 'SOFTWARE_TOKEN_MFA', 'session': response['Session']}

        # 2단계: TOTP 코드 제출
        mfa_response = cognito.respond_to_auth_challenge(
            ClientId='YOUR_APP_CLIENT_ID',
            ChallengeName='SOFTWARE_TOKEN_MFA',
            Session=response['Session'],
            ChallengeResponses={
                'USERNAME': email,
                'SOFTWARE_TOKEN_MFA_CODE': totp_code,
            }
        )
        return mfa_response['AuthenticationResult']

    # MFA 없는 경우 바로 토큰 반환
    return response['AuthenticationResult']
```

반환된 `AuthenticationResult`에는 `IdToken`, `AccessToken`, `RefreshToken`이 포함된다.

---

## 4. App Client 구성

App Client는 Cognito에 접근하는 애플리케이션을 식별하는 단위다. 프론트엔드와 백엔드용을 분리하는 것이 일반적이다.

### App Client 생성

```bash
# 프론트엔드용 (Client Secret 없음 — SPA/모바일은 시크릿 저장 불가)
aws cognito-idp create-user-pool-client \
  --user-pool-id $USER_POOL_ID \
  --client-name "frontend-client" \
  --no-generate-secret \
  --explicit-auth-flows \
    ALLOW_USER_PASSWORD_AUTH \
    ALLOW_REFRESH_TOKEN_AUTH \
    ALLOW_USER_SRP_AUTH \
  --supported-identity-providers COGNITO \
  --callback-urls "https://myapp.com/callback" \
  --logout-urls "https://myapp.com/logout" \
  --allowed-o-auth-flows code \
  --allowed-o-auth-scopes openid email profile \
  --allowed-o-auth-flows-user-pool-client \
  --region ap-northeast-2

# 백엔드 서버용 (Client Secret 있음)
aws cognito-idp create-user-pool-client \
  --user-pool-id $USER_POOL_ID \
  --client-name "backend-client" \
  --generate-secret \
  --explicit-auth-flows \
    ALLOW_USER_PASSWORD_AUTH \
    ALLOW_REFRESH_TOKEN_AUTH \
  --region ap-northeast-2
```

`ALLOW_USER_SRP_AUTH`는 SRP(Secure Remote Password) 프로토콜로, 비밀번호를 네트워크에 직접 전송하지 않는다.

### 토큰 유효 기간 설정

```bash
aws cognito-idp update-user-pool-client \
  --user-pool-id $USER_POOL_ID \
  --client-id YOUR_CLIENT_ID \
  --token-validity-units '{
    "AccessToken": "hours",
    "IdToken": "hours",
    "RefreshToken": "days"
  }' \
  --access-token-validity 1 \
  --id-token-validity 1 \
  --refresh-token-validity 30 \
  --region ap-northeast-2
```

---

## 5. OAuth 2.0 플로우 연동

### Hosted UI vs 커스텀 UI

**Hosted UI**는 Cognito가 제공하는 기본 로그인 페이지다. 빠르게 시작할 수 있지만 디자인 커스터마이징이 제한적이다.

**커스텀 UI**는 Cognito SDK를 직접 호출한다. 디자인 자유도가 높지만 구현 공수가 더 필요하다.

### Authorization Code 플로우 (권장)

```
1. 클라이언트 → Cognito Hosted UI로 리디렉션
   GET https://<domain>.auth.ap-northeast-2.amazoncognito.com/oauth2/authorize
     ?response_type=code
     &client_id=YOUR_CLIENT_ID
     &redirect_uri=https://myapp.com/callback
     &scope=openid+email+profile
     &state=RANDOM_STATE_VALUE

2. 사용자 로그인 완료 후 → 콜백 URL로 리디렉션
   GET https://myapp.com/callback?code=AUTH_CODE&state=RANDOM_STATE_VALUE

3. 서버에서 코드를 토큰으로 교환
   POST https://<domain>.auth.ap-northeast-2.amazoncognito.com/oauth2/token
```

### Node.js로 토큰 교환 구현

```javascript
const axios = require('axios');
const qs = require('querystring');

async function exchangeCodeForTokens(code, redirectUri) {
  const domain = process.env.COGNITO_DOMAIN;
  const clientId = process.env.COGNITO_CLIENT_ID;
  const clientSecret = process.env.COGNITO_CLIENT_SECRET;

  const credentials = Buffer.from(`${clientId}:${clientSecret}`).toString('base64');

  const response = await axios.post(
    `https://${domain}/oauth2/token`,
    qs.stringify({
      grant_type: 'authorization_code',
      code,
      redirect_uri: redirectUri,
      client_id: clientId,
    }),
    {
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Authorization': `Basic ${credentials}`,
      },
    }
  );

  return response.data;
  // { access_token, id_token, refresh_token, token_type, expires_in }
}
```

### 소셜 로그인(Google) 연동

```bash
# Google IdP 추가
aws cognito-idp create-identity-provider \
  --user-pool-id $USER_POOL_ID \
  --provider-name Google \
  --provider-type Google \
  --provider-details '{
    "client_id": "YOUR_GOOGLE_CLIENT_ID",
    "client_secret": "YOUR_GOOGLE_CLIENT_SECRET",
    "authorize_scopes": "email openid profile"
  }' \
  --attribute-mapping '{
    "email": "email",
    "name": "name",
    "picture": "picture"
  }' \
  --region ap-northeast-2
```

Google OAuth 콘솔에서 승인된 리디렉션 URI에 `https://<domain>.auth.ap-northeast-2.amazoncognito.com/oauth2/idpresponse`를 추가해야 한다.

### JWT 검증

Cognito가 발급한 토큰을 백엔드에서 검증하는 코드다.

```python
import requests
import jwt
from jwt import PyJWKClient

REGION = 'ap-northeast-2'
USER_POOL_ID = 'ap-northeast-2_AbcDef123'
CLIENT_ID = 'YOUR_CLIENT_ID'

JWKS_URL = f'https://cognito-idp.{REGION}.amazonaws.com/{USER_POOL_ID}/.well-known/jwks.json'

jwks_client = PyJWKClient(JWKS_URL)

def verify_token(token: str) -> dict:
    signing_key = jwks_client.get_signing_key_from_jwt(token)

    payload = jwt.decode(
        token,
        signing_key.key,
        algorithms=['RS256'],
        audience=CLIENT_ID,
        options={'verify_exp': True},
    )

    # token_use 검증: id 토큰인지 access 토큰인지 확인
    if payload.get('token_use') not in ('id', 'access'):
        raise ValueError('잘못된 token_use 값입니다.')

    return payload
```

---

## 6. Identity Pool과 임시 자격 증명

Identity Pool은 인증된 사용자에게 AWS 서비스에 직접 접근할 수 있는 임시 IAM 자격 증명을 발급한다.

### Identity Pool 생성

```bash
aws cognito-identity create-identity-pool \
  --identity-pool-name "MyAppIdentityPool" \
  --no-allow-unauthenticated-identities \
  --cognito-identity-providers '[
    {
      "ProviderName": "cognito-idp.ap-northeast-2.amazonaws.com/ap-northeast-2_AbcDef123",
      "ClientId": "YOUR_APP_CLIENT_ID",
      "ServerSideTokenCheck": true
    }
  ]' \
  --region ap-northeast-2
```

### IAM 역할 설정

인증된 사용자용 IAM 역할을 생성하고 S3 접근 권한을 부여한다.

```json
// 신뢰 정책 (trust-policy.json)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "cognito-identity.amazonaws.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "cognito-identity.amazonaws.com:aud": "ap-northeast-2:IDENTITY_POOL_ID"
        },
        "ForAnyValue:StringLike": {
          "cognito-identity.amazonaws.com:amr": "authenticated"
        }
      }
    }
  ]
}
```

```bash
# 역할 생성
aws iam create-role \
  --role-name CognitoAuthenticatedRole \
  --assume-role-policy-document file://trust-policy.json

# S3 접근 권한 추가 (사용자별 폴더 격리)
aws iam put-role-policy \
  --role-name CognitoAuthenticatedRole \
  --policy-name S3UserUploadPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
        "Resource": "arn:aws:s3:::my-uploads-bucket/${cognito-identity.amazonaws.com:sub}/*"
      }
    ]
  }'
```

`${cognito-identity.amazonaws.com:sub}`는 IAM 정책 변수로, 사용자 고유 Identity ID로 자동 치환된다. 각 사용자가 자신의 폴더에만 접근할 수 있다.

### 프론트엔드에서 임시 자격 증명으로 S3 업로드

```javascript
import { CognitoIdentityClient } from "@aws-sdk/client-cognito-identity";
import { fromCognitoIdentityPool } from "@aws-sdk/credential-provider-cognito-identity";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

const REGION = 'ap-northeast-2';
const IDENTITY_POOL_ID = 'ap-northeast-2:YOUR_IDENTITY_POOL_ID';
const USER_POOL_ID = 'ap-northeast-2_AbcDef123';

async function uploadFileToS3(file, idToken) {
  // Identity Pool에서 임시 자격 증명 획득
  const s3Client = new S3Client({
    region: REGION,
    credentials: fromCognitoIdentityPool({
      client: new CognitoIdentityClient({ region: REGION }),
      identityPoolId: IDENTITY_POOL_ID,
      logins: {
        [`cognito-idp.${REGION}.amazonaws.com/${USER_POOL_ID}`]: idToken,
      },
    }),
  });

  const command = new PutObjectCommand({
    Bucket: 'my-uploads-bucket',
    Key: `uploads/${file.name}`,
    Body: file,
    ContentType: file.type,
  });

  await s3Client.send(command);
  console.log('업로드 완료');
}
```

임시 자격 증명은 1시간 유효하며, SDK가 자동으로 갱신을 처리한다.

---

## 7. Lambda 트리거

Lambda 트리거는 Cognito 인증 흐름의 특정 시점에 커스텀 로직을 실행한다.

### 주요 트리거 종류

| 트리거 | 실행 시점 | 주요 용도 |
|--------|-----------|-----------|
| Pre Sign-Up | 회원가입 요청 직후 | 도메인 화이트리스트, 자동 승인 |
| Post Confirmation | 이메일 인증 완료 후 | DB 사용자 레코드 생성 |
| Pre Token Generation | 토큰 발급 직전 | 커스텀 클레임 추가 |
| Custom Message | 이메일/SMS 발송 직전 | 메시지 템플릿 커스터마이징 |
| Post Authentication | 로그인 성공 후 | 로그인 이력 기록 |

### Pre Sign-Up 트리거 — 도메인 화이트리스트

특정 이메일 도메인만 가입을 허용하는 예시다.

```python
# lambda_function.py
ALLOWED_DOMAINS = ['mycompany.com', 'partner.com']

def lambda_handler(event, context):
    email = event['request']['userAttributes'].get('email', '')
    domain = email.split('@')[-1].lower()

    if domain not in ALLOWED_DOMAINS:
        raise Exception(f'허용되지 않은 이메일 도메인입니다: {domain}')

    # autoConfirmUser: True로 설정하면 이메일 인증 건너뜀
    # 내부 직원 계정에 유용
    if domain == 'mycompany.com':
        event['response']['autoConfirmUser'] = True
        event['response']['autoVerifyEmail'] = True

    return event
```

Lambda가 예외를 발생시키면 Cognito는 회원가입 요청을 거부한다.

### Post Confirmation 트리거 — DB 사용자 레코드 생성

이메일 인증 완료 직후 애플리케이션 DB에 사용자 레코드를 생성한다.

```python
import boto3
import os
from datetime import datetime, timezone

dynamodb = boto3.resource('dynamodb', region_name='ap-northeast-2')
table = dynamodb.Table(os.environ['USERS_TABLE'])

def lambda_handler(event, context):
    # 이메일 인증 확인 트리거만 처리
    if event['triggerSource'] != 'PostConfirmation_ConfirmSignUp':
        return event

    user_attrs = event['request']['userAttributes']

    table.put_item(
        Item={
            'userId': event['userName'],
            'email': user_attrs.get('email'),
            'name': user_attrs.get('name', ''),
            'sub': user_attrs.get('sub'),
            'createdAt': datetime.now(timezone.utc).isoformat(),
            'role': 'user',
            'status': 'active',
        },
        ConditionExpression='attribute_not_exists(userId)',
    )

    return event
```

### Pre Token Generation 트리거 — 커스텀 클레임 추가

JWT에 커스텀 클레임을 추가해 백엔드에서 추가 조회 없이 권한을 판단할 수 있다.

```python
import boto3
import os

dynamodb = boto3.resource('dynamodb', region_name='ap-northeast-2')
table = dynamodb.Table(os.environ['USERS_TABLE'])

def lambda_handler(event, context):
    user_id = event['userName']

    # DB에서 사용자 역할 조회
    response = table.get_item(Key={'userId': user_id})
    user = response.get('Item', {})

    role = user.get('role', 'user')
    permissions = user.get('permissions', [])

    # 토큰에 클레임 추가
    event['response']['claimsOverrideDetails'] = {
        'claimsToAddOrOverride': {
            'custom:role': role,
            'custom:permissions': ','.join(permissions),
        },
        'claimsToSuppress': [],
    }

    return event
```

이후 JWT를 디코딩하면 `custom:role`, `custom:permissions` 클레임을 바로 사용할 수 있다.

### Lambda 트리거 연결

```bash
# 먼저 Lambda 함수에 Cognito 호출 권한 부여
aws lambda add-permission \
  --function-name PostConfirmationTrigger \
  --statement-id cognito-trigger \
  --action lambda:InvokeFunction \
  --principal cognito-idp.amazonaws.com \
  --source-arn arn:aws:cognito-idp:ap-northeast-2:ACCOUNT_ID:userpool/$USER_POOL_ID

# User Pool에 트리거 연결
aws cognito-idp update-user-pool \
  --user-pool-id $USER_POOL_ID \
  --lambda-config '{
    "PreSignUp": "arn:aws:lambda:ap-northeast-2:ACCOUNT_ID:function:PreSignUpTrigger",
    "PostConfirmation": "arn:aws:lambda:ap-northeast-2:ACCOUNT_ID:function:PostConfirmationTrigger",
    "PreTokenGeneration": "arn:aws:lambda:ap-northeast-2:ACCOUNT_ID:function:PreTokenGenerationTrigger"
  }' \
  --region ap-northeast-2
```

---

## 8. 토큰 갱신과 로그아웃

### Refresh Token으로 갱신

```python
def refresh_tokens(refresh_token: str) -> dict:
    response = cognito.initiate_auth(
        ClientId='YOUR_APP_CLIENT_ID',
        AuthFlow='REFRESH_TOKEN_AUTH',
        AuthParameters={
            'REFRESH_TOKEN': refresh_token,
        }
    )
    return response['AuthenticationResult']
    # access_token, id_token 반환 (refresh_token은 갱신 안 됨)
```

### 전체 로그아웃 (모든 기기)

```python
def global_sign_out(access_token: str):
    """현재 사용자의 모든 기기에서 로그아웃 — Refresh Token 무효화"""
    cognito.global_sign_out(AccessToken=access_token)
```

`global_sign_out`은 해당 사용자의 모든 Refresh Token을 즉시 무효화한다. 계정 도용 의심 시 유용하다.

---

## 운영 팁

**User Pool 삭제는 복구 불가능하다.** 프로덕션 User Pool은 반드시 삭제 방지 설정을 적용한다.

```bash
aws cognito-idp update-user-pool \
  --user-pool-id $USER_POOL_ID \
  --deletion-protection ACTIVE \
  --region ap-northeast-2
```

**CloudWatch Logs 활성화**로 인증 이벤트를 추적할 수 있다. 비정상 로그인 시도 알림에 활용한다.

**Advanced Security Features(ASF)**는 위험 기반 인증을 제공한다. 의심스러운 IP, 새 기기 감지 시 추가 인증을 요구한다. 추가 비용이 발생하지만 프로덕션에서는 고려할 만하다.

---

## 참고 자료

- [Amazon Cognito 공식 문서](https://docs.aws.amazon.com/cognito/latest/developerguide/what-is-amazon-cognito.html)
- [Cognito User Pool API 레퍼런스](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/Welcome.html)
- [JWT 검증 — python-jose vs PyJWT](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-jwt.html)
- [Identity Pool 개발자 가이드](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-identity.html)
- [Lambda 트리거 레퍼런스](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools-working-with-aws-lambda-triggers.html)
- [AWS SDK for Python (Boto3) — Cognito IdP](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/cognito-idp.html)
