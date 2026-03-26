---
title: "[AWS 실전 구축] 5편 — 시크릿과 설정 관리: Secrets Manager, Parameter Store, KMS"
date: 2026-03-17T11:04:00+09:00
draft: false
tags: ["Secrets Manager", "Parameter Store", "KMS", "AWS", "서버"]
series: ["AWS 실전 구축"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "Secrets Manager vs Parameter Store 선택 기준, RDS 비밀번호 자동 로테이션, KMS 키 정책과 봉투 암호화, 애플리케이션 통합 패턴(SDK, ECS 태스크 역할, Spring Cloud AWS)까지"
---

애플리케이션을 운영하다 보면 데이터베이스 비밀번호, API 키, OAuth 클라이언트 시크릿 같은 민감한 값들을 어디에 저장할지 반드시 결정해야 한다. 코드에 하드코딩하거나 환경변수 파일을 서버에 올려두는 방식은 보안 감사에서 즉시 지적받는 항목이다. AWS는 이 문제를 해결하기 위해 Secrets Manager와 Parameter Store, 그리고 암호화 기반인 KMS를 제공한다. 세 서비스의 역할과 차이를 이해하면 운영 부담을 줄이면서도 보안 수준을 크게 높일 수 있다.

---

## Secrets Manager vs Parameter Store — 무엇을 골라야 하는가

두 서비스는 비슷해 보이지만 설계 철학이 다르다.

**AWS Secrets Manager**는 시크릿의 생명 주기 관리에 집중한다. 저장, 조회, 자동 로테이션이 하나의 흐름 안에 있다. 비밀번호를 주기적으로 교체해야 하는 규정 준수 요구사항이 있다면 Secrets Manager가 정답이다.

**AWS Systems Manager Parameter Store**는 설정값의 계층적 저장소다. 단순 문자열부터 암호화된 시크릿까지 다양한 타입을 지원하며, 비용이 훨씬 저렴하다.

### 비용 비교

Secrets Manager는 시크릿당 월 $0.40, API 호출 10,000건당 $0.05가 과금된다. 시크릿 100개면 매달 $40이 기본으로 나간다.

Parameter Store는 표준 파라미터를 무료로 제공한다. 처리량이 높거나 고급 파라미터(최대 8KB)가 필요한 경우에만 비용이 발생한다.

### 저장 크기

Secrets Manager는 시크릿 값 최대 65,536바이트를 지원한다. Parameter Store 표준 파라미터는 4KB, 고급 파라미터는 8KB까지 허용한다.

### 자동 로테이션

Secrets Manager는 자동 로테이션을 네이티브로 지원한다. RDS, Redshift, DocumentDB는 AWS 관리형 Lambda 함수가 이미 존재한다. Parameter Store는 로테이션 기능이 없으므로 직접 구현해야 한다.

### 선택 기준 요약

| 기준 | Secrets Manager | Parameter Store |
|------|----------------|-----------------|
| 자동 로테이션 필요 | 적합 | 미지원 |
| 비용 최소화 | 비쌈 | 무료(표준) |
| 단순 설정값 저장 | 과도함 | 적합 |
| 감사 로그 + 로테이션 | 적합 | 불가 |
| 계층적 설정 관리 | 불편 | 경로 기반 지원 |

단순한 설정값(환경명, 기능 플래그, 엔드포인트 URL)은 Parameter Store로 충분하다. 비밀번호나 API 키처럼 로테이션이 필요한 값은 Secrets Manager를 사용한다.

---

## Secrets Manager 실전 — RDS 비밀번호 자동 로테이션

### 시크릿 생성

RDS 데이터베이스 비밀번호를 Secrets Manager에 저장하는 가장 간단한 방법은 콘솔에서 "RDS 데이터베이스에 대한 자격 증명" 타입을 선택하는 것이다. AWS CLI로도 할 수 있다.

```bash
aws secretsmanager create-secret \
  --name "prod/myapp/rds" \
  --description "Production RDS master credentials" \
  --secret-string '{
    "username": "admin",
    "password": "초기비밀번호",
    "engine": "mysql",
    "host": "mydb.cluster-xxxx.ap-northeast-2.rds.amazonaws.com",
    "port": 3306,
    "dbname": "myapp"
  }'
```

시크릿 값을 JSON으로 구조화하면 애플리케이션에서 파싱하기 쉽다. RDS 통합 시크릿은 `host`, `port`, `dbname`, `username`, `password` 키를 표준으로 사용한다.

### 자동 로테이션 활성화

AWS 관리형 로테이션 Lambda를 사용하는 경우 콘솔에서 몇 번의 클릭으로 설정이 완료된다. CLI로는 다음과 같다.

```bash
aws secretsmanager rotate-secret \
  --secret-id "prod/myapp/rds" \
  --rotation-rules AutomaticallyAfterDays=30 \
  --rotate-immediately
```

로테이션이 활성화되면 Secrets Manager는 30일마다 Lambda 함수를 실행해 새 비밀번호를 생성하고, RDS에 적용하고, 시크릿을 업데이트한다. 애플리케이션은 항상 최신 비밀번호를 가져오기만 하면 된다.

### 로테이션 동작 방식

로테이션은 네 단계의 Lambda 호출로 이루어진다.

**1단계 createSecret**: 새 비밀번호를 생성하고 AWSPENDING 스테이징 레이블로 저장한다.

**2단계 setSecret**: 새 비밀번호를 데이터베이스에 실제로 적용한다.

**3단계 testSecret**: 새 비밀번호로 데이터베이스 연결을 검증한다.

**4단계 finishSecret**: AWSCURRENT 레이블을 새 버전으로 이동하고 이전 버전을 AWSPREVIOUS로 표시한다.

이 단계적 접근 덕분에 로테이션 도중 애플리케이션이 중단되지 않는다. 이전 비밀번호(AWSPREVIOUS)는 일정 기간 유지되어 캐시된 연결이 끊기지 않는다.

### Lambda 커스텀 로테이션

AWS가 지원하지 않는 서비스(예: 외부 API 키)의 경우 커스텀 Lambda를 작성해야 한다.

```python
import boto3
import json

def lambda_handler(event, context):
    arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']

    client = boto3.client('secretsmanager')

    if step == "createSecret":
        create_secret(client, arn, token)
    elif step == "setSecret":
        set_secret(client, arn, token)
    elif step == "testSecret":
        test_secret(client, arn, token)
    elif step == "finishSecret":
        finish_secret(client, arn, token)

def create_secret(client, arn, token):
    # 새 API 키 생성 로직
    new_key = generate_new_api_key()
    client.put_secret_value(
        SecretId=arn,
        ClientRequestToken=token,
        SecretString=json.dumps({"api_key": new_key}),
        VersionStages=["AWSPENDING"]
    )

def set_secret(client, arn, token):
    # 외부 서비스에 새 키 등록
    pending = client.get_secret_value(
        SecretId=arn,
        VersionStage="AWSPENDING"
    )
    new_value = json.loads(pending['SecretString'])
    register_key_with_external_service(new_value['api_key'])

def test_secret(client, arn, token):
    pending = client.get_secret_value(
        SecretId=arn,
        VersionStage="AWSPENDING"
    )
    new_value = json.loads(pending['SecretString'])
    # 새 키로 실제 API 호출 테스트
    verify_api_key(new_value['api_key'])

def finish_secret(client, arn, token):
    metadata = client.describe_secret(SecretId=arn)
    current_version = None
    for v, stages in metadata['VersionIdsToStages'].items():
        if "AWSCURRENT" in stages:
            current_version = v
            break

    client.update_secret_version_stage(
        SecretId=arn,
        VersionStage="AWSCURRENT",
        MoveToVersionId=token,
        RemoveFromVersionId=current_version
    )
```

Lambda에는 `secretsmanager:GetSecretValue`, `secretsmanager:PutSecretValue`, `secretsmanager:UpdateSecretVersionStage` 권한이 필요하다.

---

## KMS 키 정책 설계

### KMS의 역할

Secrets Manager와 Parameter Store는 저장 데이터를 암호화할 때 KMS를 사용한다. KMS는 암호화 키를 관리하는 전용 서비스이며, 키 자체가 절대 외부로 노출되지 않는다. 모든 암호화/복호화 연산은 KMS 내부에서 실행된다.

### 키 유형 선택

**AWS 관리형 키(aws/secretsmanager)**는 AWS가 자동으로 생성하고 관리한다. 비용이 없고 별도 설정이 필요 없다. 단, 키 정책을 커스터마이즈할 수 없다.

**고객 관리형 키(CMK)**는 사용자가 직접 생성하고 정책을 제어한다. 키 로테이션, 교차 계정 접근, 세밀한 권한 제어가 필요할 때 선택한다. 월 $1/키 비용이 발생한다.

### 키 정책 작성

키 정책은 KMS 키에 대한 접근을 제어하는 리소스 기반 정책이다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow Secrets Manager to use the key",
      "Effect": "Allow",
      "Principal": {
        "Service": "secretsmanager.amazonaws.com"
      },
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow application role to decrypt",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/myapp-ecs-task-role"
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Deny key deletion by non-admins",
      "Effect": "Deny",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "kms:ScheduleKeyDeletion",
        "kms:DeleteImportedKeyMaterial"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::123456789012:role/key-admin-role"
        }
      }
    }
  ]
}
```

첫 번째 Statement는 계정 루트에 전체 권한을 부여해 잠금 상태를 방지한다. 이것이 없으면 키 정책 실수로 키에 영구적으로 접근 불가능해질 수 있다.

### 봉투 암호화(Envelope Encryption)

KMS는 대용량 데이터를 직접 암호화하지 않는다. 대신 봉투 암호화 방식을 사용한다.

**원리는 다음과 같다:**

1. 애플리케이션이 KMS에 데이터 암호화 키(DEK) 생성을 요청한다.
2. KMS는 평문 DEK와 암호화된 DEK를 함께 반환한다.
3. 애플리케이션은 평문 DEK로 실제 데이터를 암호화한다.
4. 평문 DEK는 메모리에서 즉시 삭제하고, 암호화된 DEK만 데이터와 함께 저장한다.
5. 복호화 시에는 암호화된 DEK를 KMS에 보내 평문 DEK를 얻고, 그것으로 데이터를 복호화한다.

이 방식 덕분에 KMS 마스터 키는 절대 외부로 나오지 않는다. AWS SDK에서는 이를 투명하게 처리해준다.

```python
import boto3
import base64

kms = boto3.client('kms', region_name='ap-northeast-2')

# 데이터 암호화
def encrypt_data(plaintext: str, key_id: str) -> dict:
    # DEK 생성
    response = kms.generate_data_key(
        KeyId=key_id,
        KeySpec='AES_256'
    )
    plaintext_dek = response['Plaintext']      # 평문 DEK (메모리에만 유지)
    encrypted_dek = response['CiphertextBlob'] # 암호화된 DEK (저장)

    # DEK로 데이터 암호화 (실제로는 AES-GCM 사용 권장)
    encrypted_data = aes_encrypt(plaintext.encode(), plaintext_dek)

    # 평문 DEK 즉시 삭제
    del plaintext_dek

    return {
        "encrypted_dek": base64.b64encode(encrypted_dek).decode(),
        "encrypted_data": base64.b64encode(encrypted_data).decode()
    }

# 데이터 복호화
def decrypt_data(encrypted_payload: dict) -> str:
    encrypted_dek = base64.b64decode(encrypted_payload['encrypted_dek'])
    encrypted_data = base64.b64decode(encrypted_payload['encrypted_data'])

    # KMS에서 DEK 복호화
    response = kms.decrypt(CiphertextBlob=encrypted_dek)
    plaintext_dek = response['Plaintext']

    # DEK로 데이터 복호화
    plaintext = aes_decrypt(encrypted_data, plaintext_dek)
    del plaintext_dek

    return plaintext.decode()
```

Secrets Manager와 Parameter Store SecureString은 내부적으로 이 봉투 암호화를 자동으로 수행한다. 직접 구현이 필요한 경우(예: S3에 저장하는 민감 파일)에만 위 패턴을 사용하면 된다.

### 키 로테이션

고객 관리형 키는 연간 자동 로테이션을 활성화할 수 있다.

```bash
aws kms enable-key-rotation --key-id arn:aws:kms:ap-northeast-2:123456789012:key/mrk-xxx
```

KMS 키 로테이션은 기존에 암호화된 데이터를 재암호화하지 않는다. 이전 키 재료는 복호화를 위해 내부적으로 보존된다. 새로 암호화하는 데이터에만 새 키 재료가 사용된다. 따라서 애플리케이션 입장에서는 로테이션이 완전히 투명하다.

---

## 애플리케이션 통합 패턴

### AWS SDK로 직접 조회

가장 기본적인 방법이다. 애플리케이션 시작 시 Secrets Manager에서 값을 가져와 메모리에 캐시한다.

```python
import boto3
import json
from functools import lru_cache

@lru_cache(maxsize=None)
def get_secret(secret_name: str) -> dict:
    client = boto3.client('secretsmanager', region_name='ap-northeast-2')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

# 사용
db_secret = get_secret('prod/myapp/rds')
conn = create_db_connection(
    host=db_secret['host'],
    user=db_secret['username'],
    password=db_secret['password'],
    database=db_secret['dbname']
)
```

`lru_cache`를 사용하면 같은 프로세스 내에서 중복 호출을 방지한다. 단, 로테이션 이후 새 비밀번호가 반영되려면 프로세스를 재시작하거나 캐시를 주기적으로 무효화해야 한다.

로테이션을 고려한 캐시 전략은 다음과 같다.

```python
import time

_cache = {}
CACHE_TTL = 300  # 5분

def get_secret_with_ttl(secret_name: str) -> dict:
    now = time.time()
    if secret_name in _cache:
        value, timestamp = _cache[secret_name]
        if now - timestamp < CACHE_TTL:
            return value

    client = boto3.client('secretsmanager', region_name='ap-northeast-2')
    response = client.get_secret_value(SecretId=secret_name)
    value = json.loads(response['SecretString'])
    _cache[secret_name] = (value, now)
    return value
```

5분 TTL이면 로테이션 적용 후 최대 5분 내에 모든 인스턴스가 새 비밀번호를 사용한다.

### ECS 태스크 역할을 통한 권한 위임

ECS 태스크에서 Secrets Manager에 접근하려면 태스크 역할에 권한을 부여해야 한다. 자격 증명을 코드에 하드코딩하는 것이 아니라 IAM 역할로 위임하는 방식이다.

**태스크 실행 역할(Task Execution Role)**은 ECS 에이전트가 컨테이너를 실행하기 위해 사용하는 역할이다. ECR에서 이미지를 pull하거나 CloudWatch에 로그를 쓰는 데 사용된다.

**태스크 역할(Task Role)**은 컨테이너 내 애플리케이션이 AWS 서비스를 호출할 때 사용하는 역할이다. Secrets Manager 접근은 이 역할에 부여해야 한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:prod/myapp/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": [
        "arn:aws:kms:ap-northeast-2:123456789012:key/mrk-xxx"
      ]
    }
  ]
}
```

리소스 ARN에 와일드카드를 사용할 때는 주의가 필요하다. `prod/myapp/*`는 해당 애플리케이션의 모든 시크릿에 접근을 허용한다. 더 세밀하게 제어하려면 특정 시크릿 ARN을 명시한다.

ECS 태스크 정의에서 태스크 역할을 지정하면 컨테이너 내부에서 AWS SDK는 자동으로 IMDS(Instance Metadata Service)를 통해 임시 자격 증명을 획득한다.

```json
{
  "family": "myapp",
  "taskRoleArn": "arn:aws:iam::123456789012:role/myapp-ecs-task-role",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "myapp",
      "image": "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:latest",
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:prod/myapp/rds:password::"
        }
      ]
    }
  ]
}
```

`secrets` 필드를 사용하면 ECS가 컨테이너 시작 시 Secrets Manager에서 값을 가져와 환경변수로 주입한다. 이 경우 권한은 태스크 역할이 아닌 태스크 실행 역할에 있어야 한다.

### Parameter Store와 ECS 통합

Parameter Store도 같은 방식으로 ECS에 주입할 수 있다.

```json
{
  "secrets": [
    {
      "name": "APP_ENV",
      "valueFrom": "arn:aws:ssm:ap-northeast-2:123456789012:parameter/prod/myapp/app-env"
    },
    {
      "name": "LOG_LEVEL",
      "valueFrom": "arn:aws:ssm:ap-northeast-2:123456789012:parameter/prod/myapp/log-level"
    }
  ]
}
```

암호화된 파라미터(SecureString)를 주입할 때는 태스크 실행 역할에 `ssm:GetParameters`와 `kms:Decrypt` 권한이 모두 필요하다.

### Spring Cloud AWS 통합

Java Spring Boot 애플리케이션에서는 Spring Cloud AWS를 사용하면 `application.yml`에서 직접 Secrets Manager 값을 참조할 수 있다.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-secrets-manager</artifactId>
    <version>3.1.0</version>
</dependency>
```

```yaml
# application.yml
spring:
  config:
    import: "aws-secretsmanager:/prod/myapp/rds"
  datasource:
    url: jdbc:mysql://${host}:${port}/${dbname}
    username: ${username}
    password: ${password}
```

`aws-secretsmanager:` 접두사가 붙은 경로를 지정하면 Spring Boot 시작 시 해당 시크릿을 자동으로 로드한다. JSON 구조의 시크릿은 각 키가 Spring 프로퍼티로 flatten된다.

Parameter Store도 동일하게 사용할 수 있다.

```yaml
spring:
  config:
    import:
      - "aws-secretsmanager:/prod/myapp/rds"
      - "aws-parameterstore:/prod/myapp/"
```

`/prod/myapp/` 경로의 모든 파라미터가 자동으로 로드된다. 파라미터 이름이 `/prod/myapp/log-level`이면 `log-level` 프로퍼티로 접근 가능하다.

### Node.js 통합 패턴

Express 기반 애플리케이션에서는 시작 시 시크릿을 한 번 로드하는 패턴이 일반적이다.

```javascript
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

const client = new SecretsManagerClient({ region: 'ap-northeast-2' });

async function loadSecrets() {
  const command = new GetSecretValueCommand({
    SecretId: 'prod/myapp/rds',
  });

  const response = await client.send(command);
  return JSON.parse(response.SecretString);
}

async function startApp() {
  const secrets = await loadSecrets();

  const dbPool = createPool({
    host: secrets.host,
    user: secrets.username,
    password: secrets.password,
    database: secrets.dbname,
    port: secrets.port,
  });

  const app = express();
  // ... 라우트 설정

  app.listen(3000, () => {
    console.log('서버 시작');
  });
}

startApp().catch(console.error);
```

---

## 실전 운영 패턴

### 계층적 네이밍 전략

Parameter Store와 Secrets Manager 모두 네이밍 규칙을 일관되게 유지하면 IAM 정책 관리가 훨씬 쉬워진다.

```
/{환경}/{서비스}/{카테고리}/{키}

예시:
/prod/myapp/rds/password
/prod/myapp/redis/password
/prod/payment-service/stripe/api-key
/staging/myapp/rds/password
```

이 구조를 사용하면 IAM 정책에서 `Resource: "arn:aws:secretsmanager:*:*:secret:prod/myapp/*"` 한 줄로 특정 서비스의 모든 프로덕션 시크릿에 접근을 제어할 수 있다.

### 크로스 계정 접근

다른 AWS 계정의 서비스가 시크릿에 접근해야 할 때는 리소스 기반 정책을 사용한다.

```bash
aws secretsmanager put-resource-policy \
  --secret-id "prod/myapp/rds" \
  --resource-policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::999888777666:role/data-pipeline-role"
        },
        "Action": "secretsmanager:GetSecretValue",
        "Resource": "*"
      }
    ]
  }'
```

크로스 계정 접근 시 CMK를 사용한다면 KMS 키 정책에도 해당 계정의 접근을 명시적으로 허용해야 한다.

### 감사와 모니터링

CloudTrail을 통해 모든 시크릿 접근이 자동으로 기록된다. 의심스러운 접근을 감지하려면 CloudWatch Alarm을 설정한다.

```bash
# 특정 시크릿에 대한 접근 급증 감지
aws cloudwatch put-metric-alarm \
  --alarm-name "SecretAccessAnomaly-prod-myapp-rds" \
  --alarm-description "Unusual number of secret access attempts" \
  --namespace "AWS/SecretsManager" \
  --metric-name "GetSecretValue" \
  --dimensions Name=SecretId,Value="prod/myapp/rds" \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions "arn:aws:sns:ap-northeast-2:123456789012:security-alerts"
```

로테이션 실패도 모니터링 대상이다. Secrets Manager는 로테이션 실패 시 이벤트를 발생시키므로 EventBridge로 SNS 알림을 연결해두면 좋다.

```json
{
  "source": ["aws.secretsmanager"],
  "detail-type": ["AWS Service Event via CloudTrail"],
  "detail": {
    "eventName": ["RotationFailed"]
  }
}
```

### 로컬 개발 환경 처리

개발자 로컬 환경에서는 실제 Secrets Manager를 사용하거나, localstack을 통해 에뮬레이션할 수 있다. 단순한 경우에는 환경별 `.env` 파일을 사용하되 절대 Git에 커밋하지 않는다.

`.gitignore`에 반드시 포함해야 할 항목이다.

```gitignore
.env
.env.local
.env.production
*.secret
```

로컬에서도 실제 AWS 서비스를 사용하는 것이 베스트 프랙티스다. 개발자 IAM 사용자에게 개발 환경 시크릿(예: `dev/` 접두사)에만 접근 권한을 부여하면 프로덕션 시크릿 노출 위험 없이 실제 환경에 가까운 개발이 가능하다.

---

시크릿 관리는 "나중에 해결할 문제"가 아니다. 아키텍처 초기부터 Secrets Manager와 Parameter Store의 역할을 명확히 나누고, KMS CMK로 암호화 제어권을 확보하면 보안 감사 대응과 장애 대응 모두에서 여유가 생긴다. 특히 자동 로테이션은 인적 실수로 인한 비밀번호 유출 위험을 크게 낮춰주므로, 데이터베이스 자격증명부터 반드시 활성화해두는 것을 권장한다.

---

## 참고 자료

- [AWS Secrets Manager 공식 문서](https://docs.aws.amazon.com/secretsmanager/latest/userguide/)
- [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [AWS KMS 개발자 가이드](https://docs.aws.amazon.com/kms/latest/developerguide/)
- [Envelope Encryption 개념](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#enveloping)
- [Spring Cloud AWS — Secrets Manager](https://docs.awspring.io/spring-cloud-aws/docs/3.1.0/reference/html/index.html#secrets-manager)
- [ECS 태스크에 시크릿 주입](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/secrets-envvar-secrets-manager.html)
- [KMS 키 정책 모범 사례](https://docs.aws.amazon.com/kms/latest/developerguide/key-policy-overview.html)
