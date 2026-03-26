---
title: "[AWS / 클라우드] 5편 — 데이터 서비스: RDS, DynamoDB, S3"
date: 2026-03-20T14:04:00+09:00
draft: false
tags: ["RDS", "DynamoDB", "S3", "ElastiCache", "AWS", "서버"]
series: ["AWS / 클라우드"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "RDS Multi-AZ와 Read Replica·Aurora 아키텍처, DynamoDB 파티션 키 설계와 단일 테이블 패턴, S3 스토리지 클래스와 수명주기 정책, ElastiCache Redis 클러스터 모드와 장애 조치까지"
---

데이터베이스를 직접 설치하고 운영하는 시대는 지났다. AWS 관리형 서비스를 쓰면 패치, 백업, 복제를 자동화할 수 있다.

이 글은 RDS, DynamoDB, S3, ElastiCache의 아키텍처와 실전 설계 패턴을 다룬다.

---

## 목차

1. [RDS — 관계형 데이터베이스 완전 관리 서비스](#1-rds--관계형-데이터베이스-완전-관리-서비스)
2. [Aurora — AWS 독자 설계 RDBMS](#2-aurora--aws-독자-설계-rdbms)
3. [DynamoDB — 완전 관리형 NoSQL](#3-dynamodb--완전-관리형-nosql)
4. [S3 — 오브젝트 스토리지](#4-s3--오브젝트-스토리지)
5. [ElastiCache — 인메모리 캐시 클러스터](#5-elasticache--인메모리-캐시-클러스터)
6. [데이터 서비스 선택 기준 정리](#6-데이터-서비스-선택-기준-정리)

---

## 1. RDS — 관계형 데이터베이스 완전 관리 서비스

RDS(Relational Database Service)는 MySQL, PostgreSQL, MariaDB, Oracle, SQL Server 등을 AWS가 직접 운영해 주는 완전 관리형 서비스다.
OS 패치, 백업, 복구, 모니터링을 AWS가 처리하므로 DBA 없이도 안정적인 관계형 DB를 운영할 수 있다.

### 1-1. Multi-AZ 배포

Multi-AZ는 기본 인스턴스와 동일한 구성의 스탠바이 인스턴스를 다른 가용 영역에 동기적으로 복제한다.
기본 인스턴스에 장애가 발생하면 자동으로 스탠바이로 페일오버되며 DNS CNAME이 교체된다.

```
Primary (ap-northeast-2a)
     |  동기 복제 (Synchronous Replication)
Standby (ap-northeast-2b)
```

> **핵심 포인트**: Multi-AZ는 고가용성(HA) 목적이다. 스탠바이는 읽기 트래픽을 처리하지 않는다.

페일오버 시간은 일반적으로 60~120초다.
데이터 손실(RPO)은 동기 복제이므로 이론적으로 0이다.

### 1-2. Read Replica

읽기 전용 복제본은 비동기 복제로 최대 15개까지 생성할 수 있다.
읽기 요청을 분산시켜 기본 인스턴스의 부하를 줄인다.

```bash
# AWS CLI로 Read Replica 생성
aws rds create-db-instance-read-replica \
  --db-instance-identifier mydb-read-1 \
  --source-db-instance-identifier mydb-primary \
  --db-instance-class db.r6g.large \
  --availability-zone ap-northeast-2b
```

Read Replica는 다른 리전에도 만들 수 있어 재해 복구(DR) 전략으로 활용된다.
또한 Read Replica를 독립 인스턴스로 승격(promote)하면 쓰기도 가능한 별도 DB가 된다.

### 1-3. 자동 백업과 스냅샷

RDS는 백업 보존 기간(1~35일) 동안 지속적으로 트랜잭션 로그를 S3에 저장한다.
이를 통해 특정 시점으로 복구(PITR, Point-In-Time Recovery)가 가능하다.

```bash
# 수동 스냅샷 생성
aws rds create-db-snapshot \
  --db-instance-identifier mydb-primary \
  --db-snapshot-identifier mydb-snapshot-20260320

# 스냅샷에서 새 인스턴스 복구
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier mydb-restored \
  --db-snapshot-identifier mydb-snapshot-20260320
```

### 1-4. 파라미터 그룹과 옵션 그룹

파라미터 그룹은 DB 엔진 설정(innodb_buffer_pool_size, max_connections 등)을 관리한다.
변경 사항은 동적 파라미터는 즉시, 정적 파라미터는 재부팅 후 적용된다.

```bash
# 파라미터 그룹 생성 및 수정
aws rds create-db-parameter-group \
  --db-parameter-group-name mydb-params \
  --db-parameter-group-family mysql8.0 \
  --description "Custom MySQL 8.0 parameters"

aws rds modify-db-parameter-group \
  --db-parameter-group-name mydb-params \
  --parameters ParameterName=max_connections,ParameterValue=500,ApplyMethod=immediate
```

---

## 2. Aurora — AWS 독자 설계 RDBMS

Aurora는 MySQL/PostgreSQL 호환 엔진이지만 내부 아키텍처를 AWS가 완전히 재설계한 서비스다.
스토리지와 컴퓨트가 분리된 구조 덕분에 일반 RDS 대비 5배(MySQL), 3배(PostgreSQL) 빠른 성능을 제공한다.

### 2-1. Aurora 스토리지 아키텍처

Aurora는 데이터를 3개 가용 영역에 걸쳐 6개 복사본으로 자동 저장한다.
쓰기는 4개, 읽기는 3개의 복사본이 확인하면 성공으로 처리하는 쿼럼(Quorum) 방식이다.

```
Writer Endpoint ──── Primary Instance
                         |
Reader Endpoint ──── Read Replica 1 (AZ-a)
                ──── Read Replica 2 (AZ-b)
                ──── Read Replica 3 (AZ-c)

Shared Storage (6 copies across 3 AZs): 10GB ~ 128TB 자동 확장
```

스토리지는 10GB 단위로 자동 확장되므로 미리 용량을 프로비저닝하지 않아도 된다.

### 2-2. Aurora 클러스터 엔드포인트

Aurora는 세 가지 엔드포인트를 제공한다.

| 엔드포인트 | 역할 |
|---|---|
| Cluster Endpoint | 항상 현재 Writer로 라우팅 |
| Reader Endpoint | Read Replica들에 부하 분산 |
| Instance Endpoint | 특정 인스턴스에 직접 연결 |

페일오버 시 Reader 중 하나가 Writer로 승격되며, Cluster Endpoint가 자동으로 새 Writer를 가리킨다.
일반적으로 30초 이내에 페일오버가 완료된다.

### 2-3. Aurora Serverless v2

Aurora Serverless v2는 ACU(Aurora Capacity Unit) 단위로 0.5 ACU씩 세밀하게 스케일 인/아웃한다.
트래픽이 없으면 최솟값까지 줄고, 급증하면 수초 안에 스케일 아웃된다.

```bash
# Aurora Serverless v2 클러스터 생성
aws rds create-db-cluster \
  --db-cluster-identifier my-aurora-serverless \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.02.0 \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16 \
  --master-username admin \
  --master-user-password mypassword \
  --vpc-security-group-ids sg-12345678
```

개발/스테이징 환경이나 트래픽 예측이 어려운 서비스에 특히 적합하다.

### 2-4. Aurora Global Database

Aurora Global Database는 하나의 기본 리전과 최대 5개의 보조 리전으로 구성된다.
기본 리전에서 보조 리전으로의 복제 지연은 일반적으로 1초 미만이다.

재해 상황에서 보조 리전을 기본 리전으로 승격(promote)하면 RPO 1초, RTO 1분 미만의 DR이 가능하다.

---

## 3. DynamoDB — 완전 관리형 NoSQL

DynamoDB는 AWS의 완전 관리형 키-값 및 도큐먼트 데이터베이스다.
서버리스로 동작하며 초당 수백만 건의 요청을 밀리초 이하의 지연 시간으로 처리할 수 있다.

### 3-1. 핵심 개념: 파티션 키와 정렬 키

DynamoDB 테이블의 기본 키는 두 가지 형태다.

- **파티션 키(Partition Key)**: 단일 속성으로 구성된 단순 기본 키
- **복합 기본 키(Composite Key)**: 파티션 키 + 정렬 키(Sort Key) 조합

파티션 키는 내부적으로 해시 함수를 거쳐 데이터가 저장될 파티션을 결정한다.
카디널리티(cardinality)가 높은 속성을 파티션 키로 선택해야 트래픽이 고르게 분산된다.

```
테이블: Orders
┌─────────────────┬──────────────────┬──────────────────┐
│ PK: userId      │ SK: orderId      │ Attributes       │
├─────────────────┼──────────────────┼──────────────────┤
│ user#001        │ order#2026-001   │ status, amount   │
│ user#001        │ order#2026-002   │ status, amount   │
│ user#002        │ order#2026-003   │ status, amount   │
└─────────────────┴──────────────────┴──────────────────┘
```

### 3-2. GSI와 LSI

기본 키 이외의 조회 패턴을 지원하기 위해 보조 인덱스를 사용한다.

**LSI(Local Secondary Index)**
- 테이블 생성 시에만 정의 가능
- 파티션 키는 테이블과 동일, 정렬 키만 다른 인덱스
- 파티션 내에서 다른 기준으로 정렬/필터링할 때 사용

**GSI(Global Secondary Index)**
- 테이블 생성 후에도 추가 가능
- 완전히 다른 파티션 키와 정렬 키 조합으로 인덱스 생성
- 별도의 읽기/쓰기 용량(RCU/WCU) 또는 온디맨드 모드 설정

```bash
# GSI가 있는 테이블 생성
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
    AttributeName=userId,AttributeType=S \
    AttributeName=orderId,AttributeType=S \
    AttributeName=status,AttributeType=S \
    AttributeName=createdAt,AttributeType=S \
  --key-schema \
    AttributeName=userId,KeyType=HASH \
    AttributeName=orderId,KeyType=RANGE \
  --global-secondary-indexes \
    '[{
      "IndexName": "status-createdAt-index",
      "KeySchema": [
        {"AttributeName": "status", "KeyType": "HASH"},
        {"AttributeName": "createdAt", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
    }]' \
  --billing-mode PAY_PER_REQUEST
```

### 3-3. 단일 테이블 설계 (Single-Table Design)

DynamoDB에서는 여러 엔티티를 하나의 테이블에 저장하는 단일 테이블 패턴이 권장된다. 관계형 DB처럼 엔티티마다 테이블을 따로 만들면 DynamoDB에서는 JOIN이 없어 여러 번의 네트워크 왕복이 필요하다. 단일 테이블 패턴은 파일 서랍장 하나에 모든 서류를 분류해 넣는 방식처럼, 관련 데이터를 같은 파티션 키 아래 정렬 키(SK)로 구분해 단 한 번의 Query로 가져올 수 있다.
JOIN이 없는 NoSQL 특성상 관련 데이터를 하나의 파티션에 모아 단일 쿼리로 조회하는 것이 효율적이다.

```
테이블: App (Single Table)
┌──────────────────┬──────────────────┬─────────────────────────────┐
│ PK               │ SK               │ Attributes                  │
├──────────────────┼──────────────────┼─────────────────────────────┤
│ USER#user001     │ PROFILE          │ name, email, createdAt      │
│ USER#user001     │ ORDER#order001   │ amount, status, items       │
│ USER#user001     │ ORDER#order002   │ amount, status, items       │
│ PRODUCT#prod001  │ METADATA         │ name, price, category       │
│ PRODUCT#prod001  │ REVIEW#rev001    │ rating, comment, userId     │
└──────────────────┴──────────────────┴─────────────────────────────┘
```

이 패턴을 사용하면 `userId`로 사용자 프로필과 모든 주문을 한 번의 Query 호출로 가져올 수 있다.

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('App')

# 사용자의 모든 주문 조회 (단일 Query)
response = table.query(
    KeyConditionExpression='PK = :pk AND begins_with(SK, :sk)',
    ExpressionAttributeValues={
        ':pk': 'USER#user001',
        ':sk': 'ORDER#'
    }
)
```

### 3-4. 용량 모드: 프로비저닝 vs 온디맨드

**프로비저닝 모드(Provisioned)**
- RCU(Read Capacity Unit), WCU(Write Capacity Unit)를 미리 지정
- Auto Scaling 설정으로 자동 조정 가능
- 예측 가능한 트래픽에 비용 효율적

**온디맨드 모드(On-Demand)**
- 실제 사용한 요청 수 만큼만 과금
- 트래픽 예측이 어려운 서비스에 적합
- 프로비저닝 대비 요청당 단가가 약 7배 높음

```bash
# 온디맨드에서 프로비저닝으로 전환
aws dynamodb update-table \
  --table-name Orders \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=100,WriteCapacityUnits=50
```

### 3-5. DynamoDB Streams와 TTL

**DynamoDB Streams**는 테이블의 변경 사항(삽입/수정/삭제)을 24시간 동안 스트림으로 제공한다.
Lambda 트리거와 연결하면 변경 이벤트 기반의 서버리스 파이프라인을 구성할 수 있다.

**TTL(Time To Live)**은 아이템에 만료 시각을 Unix 타임스탬프로 지정하는 기능이다.
만료된 아이템은 48시간 이내에 자동 삭제되며 WCU를 소비하지 않는다.

```python
import time

# TTL 7일 후 만료되는 세션 아이템 저장
table.put_item(Item={
    'PK': 'SESSION#abc123',
    'SK': 'METADATA',
    'userId': 'user001',
    'ttl': int(time.time()) + 7 * 24 * 3600  # 7일 후 Unix timestamp
})
```

---

## 4. S3 — 오브젝트 스토리지

S3(Simple Storage Service)는 사실상 무제한 용량의 오브젝트 스토리지 서비스다.
정적 웹사이트 호스팅, 백업, 빅데이터 데이터 레이크, CDN 오리진 등 다양한 용도로 사용된다.

### 4-1. S3 스토리지 클래스

| 스토리지 클래스 | 가용성 | 최소 저장 기간 | 주요 용도 |
|---|---|---|---|
| S3 Standard | 99.99% | 없음 | 자주 접근하는 데이터 |
| S3 Standard-IA | 99.9% | 30일 | 월 1회 미만 접근, 빠른 조회 필요 |
| S3 One Zone-IA | 99.5% | 30일 | 재생성 가능한 데이터, 단일 AZ |
| S3 Glacier Instant | 99.9% | 90일 | 분기 1회 접근, 밀리초 조회 |
| S3 Glacier Flexible | 99.99% | 90일 | 연 1회 접근, 분~시간 조회 |
| S3 Glacier Deep Archive | 99.99% | 180일 | 장기 보관, 12시간 조회 |
| S3 Intelligent-Tiering | 99.9% | 없음 | 접근 패턴 불규칙, 자동 티어링 |

Standard에서 Glacier Deep Archive로 갈수록 저장 비용은 낮아지고 조회 비용은 높아진다.

### 4-2. 수명주기 정책 (Lifecycle Policy)

수명주기 정책으로 오브젝트를 자동으로 하위 스토리지 클래스로 전환하거나 삭제할 수 있다.

```json
{
  "Rules": [
    {
      "ID": "move-to-ia-then-glacier",
      "Status": "Enabled",
      "Filter": { "Prefix": "logs/" },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 365
      }
    }
  ]
}
```

```bash
# 수명주기 정책 적용
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-log-bucket \
  --lifecycle-configuration file://lifecycle.json
```

### 4-3. S3 버저닝과 MFA Delete

버저닝을 활성화하면 같은 키에 덮어쓰거나 삭제해도 이전 버전이 보존된다.
실수로 삭제한 파일을 복구하거나 이전 버전으로 롤백할 수 있다.

```bash
# 버저닝 활성화
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

# 특정 버전 복구
aws s3api get-object \
  --bucket my-bucket \
  --key myfile.txt \
  --version-id abc123xyz \
  restored-myfile.txt
```

MFA Delete를 활성화하면 버전 삭제 시 MFA 인증을 요구해 실수나 악의적인 삭제를 방지한다.

### 4-4. S3 보안 설정

**버킷 정책(Bucket Policy)**은 JSON 형식으로 리소스 기반 접근 제어를 설정한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonHTTPS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

**S3 Block Public Access**는 계정 또는 버킷 수준에서 퍼블릭 접근을 차단하는 설정이다.
신규 버킷은 기본으로 모든 퍼블릭 접근이 차단된다.

### 4-5. S3 Transfer Acceleration과 Multipart Upload

**Transfer Acceleration**은 CloudFront 엣지 로케이션을 경유해 장거리 업로드 속도를 높인다.
서울에서 미국 동부 버킷으로 업로드할 때 최대 60% 속도 향상 효과가 있다.

**Multipart Upload**는 100MB 이상 파일을 여러 파트로 나눠 병렬 업로드하는 방법이다.
네트워크 오류 시 실패한 파트만 재전송하므로 안정성이 높다.

```bash
# AWS CLI의 cp 명령은 자동으로 Multipart Upload 사용
aws s3 cp large-file.zip s3://my-bucket/ \
  --sse aws:kms \
  --multipart-threshold 100MB \
  --multipart-chunksize 50MB
```

### 4-6. S3 이벤트 알림

오브젝트 생성/삭제 시 SNS, SQS, Lambda로 이벤트를 전송할 수 있다.
이미지 업로드 후 Lambda로 자동 리사이징하는 파이프라인이 대표적인 패턴이다.

```bash
# S3 이벤트 -> Lambda 트리거 설정
aws s3api put-bucket-notification-configuration \
  --bucket my-image-bucket \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [
      {
        "LambdaFunctionArn": "arn:aws:lambda:ap-northeast-2:123456789:function:resize-image",
        "Events": ["s3:ObjectCreated:*"],
        "Filter": {
          "Key": {
            "FilterRules": [
              {"Name": "suffix", "Value": ".jpg"}
            ]
          }
        }
      }
    ]
  }'
```

---

## 5. ElastiCache — 인메모리 캐시 클러스터

ElastiCache는 Redis와 Memcached를 완전 관리형으로 제공하는 인메모리 캐시 서비스다.
DB 앞단에 캐시를 두어 읽기 지연 시간을 마이크로초 단위로 줄이고 DB 부하를 크게 낮춘다.

### 5-1. Redis vs Memcached 선택 기준

| 항목 | Redis | Memcached |
|---|---|---|
| 데이터 구조 | String, Hash, List, Set, Sorted Set, Stream 등 | String만 |
| 영속성 | RDB 스냅샷, AOF 로그 | 없음 |
| 복제 | 지원 (Primary/Replica) | 없음 |
| 클러스터 모드 | 지원 | 지원 |
| Pub/Sub | 지원 | 없음 |
| 트랜잭션 | 지원 | 없음 |

대부분의 신규 서비스는 Redis를 선택한다.
Memcached는 단순 캐시 목적으로 멀티스레드 처리가 중요할 때 고려한다.

### 5-2. Redis 클러스터 모드 비활성화 (Replication Group)

클러스터 모드 비활성화 구성은 하나의 Primary와 최대 5개의 Read Replica로 구성된다.
모든 데이터가 단일 샤드에 저장되므로 관리가 단순하다.

```
Primary (읽기/쓰기)
    |
    ├── Replica 1 (읽기 전용)
    └── Replica 2 (읽기 전용)
```

Primary 장애 시 Replica 중 하나가 자동으로 Primary로 승격된다.

```bash
# Redis Replication Group 생성
aws elasticache create-replication-group \
  --replication-group-id my-redis \
  --replication-group-description "My Redis cluster" \
  --engine redis \
  --cache-node-type cache.r6g.large \
  --num-cache-clusters 3 \
  --automatic-failover-enabled \
  --multi-az-enabled \
  --cache-subnet-group-name my-subnet-group
```

### 5-3. Redis 클러스터 모드 활성화 (Sharding)

클러스터 모드 활성화 구성은 데이터를 여러 샤드에 분산 저장한다.
각 샤드는 Primary와 Replica로 구성되며 최대 500개의 노드까지 확장 가능하다.

```
Shard 1: Primary + Replica (slot 0~5460)
Shard 2: Primary + Replica (slot 5461~10922)
Shard 3: Primary + Replica (slot 10923~16383)
```

16,384개의 해시 슬롯을 샤드에 균등 분배한다.
키는 `CRC16(key) % 16384`로 슬롯이 결정된다. 어떤 키가 어느 샤드에 저장되는지 일관된 규칙으로 계산하기 위해서다. 클라이언트는 이 규칙을 알고 있어 매 요청마다 중앙 라우터를 거치지 않고 해당 샤드로 직접 접근한다.

```bash
# 클러스터 모드 활성화 Redis 생성
aws elasticache create-replication-group \
  --replication-group-id my-redis-cluster \
  --replication-group-description "Redis cluster mode enabled" \
  --engine redis \
  --cache-node-type cache.r6g.large \
  --num-node-groups 3 \
  --replicas-per-node-group 2 \
  --automatic-failover-enabled \
  --cluster-mode enabled
```

### 5-4. 장애 조치와 자동 복구

`automatic-failover-enabled`를 설정하면 Primary 장애 시 자동으로 Replica가 승격된다.
일반적으로 장애 감지 후 60초 이내에 페일오버가 완료된다.

`multi-az-enabled`는 Primary와 Replica를 서로 다른 가용 영역에 배치한다.
AZ 단위 장애에도 서비스가 중단되지 않도록 보호한다.

```bash
# 수동 페일오버 테스트 (DR 연습)
aws elasticache test-failover \
  --replication-group-id my-redis \
  --node-group-id 0001
```

### 5-5. ElastiCache 캐싱 전략

**Cache-Aside (Lazy Loading)**
애플리케이션이 캐시를 먼저 확인하고 없으면 DB에서 읽어 캐시에 저장하는 패턴이다.
캐시 미스 시에만 DB 조회가 발생하므로 실제로 필요한 데이터만 캐시에 적재된다.

아래 코드는 Cache-Aside 패턴의 전형적인 구현이다. 캐시 히트 시 DB를 전혀 거치지 않고 바로 반환하며, 캐시 미스 시 DB에서 읽어 TTL 1시간으로 저장한다.

```python
import redis
import json

r = redis.Redis(host='my-redis.abc123.cache.amazonaws.com', port=6379)

def get_user(user_id):
    cache_key = f"user:{user_id}"

    # 캐시 확인
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # DB 조회
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)

    # 캐시 저장 (TTL 1시간)
    r.setex(cache_key, 3600, json.dumps(user))
    return user
```

**Write-Through**는 쓰기 시 캐시와 DB를 동시에 업데이트하는 패턴이다.
캐시가 항상 최신 상태이지만 모든 쓰기 연산에 지연이 추가된다.

### 5-6. Redis 데이터 구조 활용

Redis의 다양한 데이터 구조를 활용하면 복잡한 기능을 효율적으로 구현할 수 있다.

```python
# Sorted Set으로 실시간 랭킹 구현
r.zadd('leaderboard', {'user001': 9500, 'user002': 8700, 'user003': 9200})

# 상위 10명 조회 (내림차순)
top10 = r.zrevrange('leaderboard', 0, 9, withscores=True)

# Hash로 세션 데이터 저장
r.hset('session:abc123', mapping={
    'userId': 'user001',
    'role': 'admin',
    'loginAt': '2026-03-20T14:00:00'
})
r.expire('session:abc123', 3600)

# List로 작업 큐 구현
r.lpush('job-queue', json.dumps({'type': 'send-email', 'to': 'user@example.com'}))
job = r.brpop('job-queue', timeout=30)  # 블로킹 POP
```

---

## 6. 데이터 서비스 선택 기준 정리

각 AWS 데이터 서비스는 사용 사례에 따라 적합한 영역이 다르다.
서비스 선택 시 데이터 구조, 접근 패턴, 규모, 비용을 함께 고려해야 한다.

### 선택 플로우차트

```
데이터 저장이 필요하다
       |
       ├── 임시 데이터, 빠른 응답 필요 → ElastiCache (Redis)
       |
       ├── 정형 데이터, 복잡한 쿼리/JOIN 필요
       |       ├── 대용량/고성능 필요 → Aurora
       |       └── 일반적인 RDBMS → RDS
       |
       ├── 비정형/반정형 데이터, 수평 확장 필요
       |       └── 단순 키-값, 초저지연 → DynamoDB
       |
       └── 파일/오브젝트, 정적 콘텐츠
               └── S3
```

### 비용 최적화 포인트

| 서비스 | 비용 절감 방법 |
|---|---|
| RDS | Reserved Instance 1~3년 약정 (최대 60% 절감) |
| Aurora | Serverless v2로 비용 사용량 기반 전환 |
| DynamoDB | 예측 가능한 트래픽은 프로비저닝 + Auto Scaling |
| S3 | 수명주기 정책으로 Glacier 전환, Intelligent-Tiering 활용 |
| ElastiCache | Reserved Node 약정, 적정 노드 타입 선정 |

### 고가용성 구성 요약

- **RDS**: Multi-AZ + Read Replica 조합
- **Aurora**: 기본 6개 복사본 + Multi-AZ Read Replica
- **DynamoDB**: 글로벌 테이블로 다중 리전 액티브-액티브 구성 가능
- **S3**: 리전 내 최소 3개 AZ에 자동 복제, Cross-Region Replication으로 다중 리전 대응
- **ElastiCache**: Multi-AZ Replica + 자동 페일오버

---

## 참고 자료

- [Amazon RDS 공식 문서](https://docs.aws.amazon.com/rds/)
- [Amazon Aurora 사용 설명서](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/)
- [Amazon DynamoDB 개발자 안내서](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)
- [DynamoDB Best Practices — Single-Table Design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [Amazon S3 스토리지 클래스](https://aws.amazon.com/s3/storage-classes/)
- [Amazon ElastiCache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/)
- [AWS Well-Architected Framework — 데이터 계층](https://docs.aws.amazon.com/wellarchitected/latest/framework/)
